# Projet NILM — Désagrégation de consommation électrique & flexibilité énergétique

Désagrégation non intrusive de charge (*Non-Intrusive Load Monitoring*) : reconstituer la
consommation de chaque appareil à partir du seul signal agrégé du compteur, puis estimer le
potentiel de flexibilité énergétique disponible.

> **Premier passage sur le projet ?** Lire d'abord `README_start.md` (installation de
> l'environnement Python, téléchargement des datasets, configuration du dépôt). Revenir
> ensuite à ce README pour le pipeline complet.

## État d'avancement
La répartition des rôles de chaque notebook est davantage présenté dans la ligne-guide du projet (cf document ligne_guide_PROJET.md).

| Notebook | Rôle | Plateforme | Statut |
|----------|------|------------|--------|
| `00_test_setup.ipynb`         | Vérification environnement + accès NILMTK aux `.h5`              | Local | ✅ |
| `01_exploration_donnees.ipynb`| Inventaire appareils, qualité signal, choix cibles & split       | Local | ✅ |
| `02_preprocessing.ipynb`      | Extraction, alignement, segmentation, normalisation              | Local | ✅ |
| `03_baselines_nilmtk.ipynb`   | Baselines CO et FHMMExact (NILMTK)                               | Local | ✅ |
| `04_seq2point.ipynb`          | Seq2Point (perte MSE vs combinée), entraînement                  | **Colab GPU** | ✅ |
| `04b_inference_seq2point.ipynb`| Inférence des modèles Seq2Point, sauvegarde des prédictions     | Local | ✅ |
| `05_modeles_complementaires`  | LSTM / GRU / Seq2Seq                                             | — | ❌ Non livré (hors périmètre) |
| `06_evaluation_comparaison`   | Agrégation et comparaison des modèles (tableaux + figures)       | Local | ✅ |
| `07_ablation_resolution`      | Effet de la résolution temporelle (question Linky)               | **Colab GPU** | ✅ |
| `08_flexibilite.ipynb`        | Indicateur de potentiel de report de charge                      | Local | ✅ |
| `09_synthese.ipynb`           | Synthèse autonome du projet (réponses aux questions du sujet)    | Local | ✅ |

## Installation

Suivre d'abord `README_start.md` pour l'environnement Python, le clone du dépôt et le
téléchargement des datasets. En résumé :

```bash
git clone <url-du-depot>
cd projet-nilm

# Environnement (au choix)
uv sync                       # si uv installé
# ou
pip install -r requirements.txt
```

Python 3.13. NILMTK est installé depuis GitHub (cf. `pyproject.toml`) ; il n'est nécessaire
que pour les notebooks 00, 01, 02 et 03 (lecture des `.h5` et baselines). Les notebooks de
modélisation (04 et au-delà) ne dépendent que de PyTorch, NumPy, pandas, hmmlearn et
scikit-learn.

## Données (NON versionnées — à télécharger)

Les datasets `.h5` sont volumineux et exclus du dépôt (`.gitignore`). À télécharger et
placer dans `data/` :

- **UK-DALE** (dataset principal) : https://data.ceda.ac.uk/edc/d1/7d78f943-f9fe-413b-af52-1816f9d968b0/data/version_0
- **REDD** (validation inter-dataset) : https://zenodo.org/records/13917372/files/redd.h5
- **iAWE** (test de robustesse, non utilisé in fine) : https://zenodo.org/records/13917372/files/iawe.h5

Arborescence locale attendue après téléchargement :

```
data/
  ukdale.h5
  redd.h5
  iawe.h5
```

## Exécution du pipeline

Lancer les notebooks **dans l'ordre numérique**. Certains tournent en local, d'autres
nécessitent un GPU (Colab recommandé). Les artefacts produits par chaque étape alimentent
les suivantes.

### 1. `00_test_setup` — **local**
Vérification que NILMTK lit les `.h5` et que les datasets sont bien en place.

**Produit :** aucun artefact persisté.

### 2. `01_exploration_donnees` — **local**
Inventaire des appareils par maison, qualité du signal, détermination du split inter-foyers,
choix des seuils ON/OFF.

**Produit dans `outputs/exploration/`** :
- `exploration_config.json` — paramètres validés (seuils, split, appareils cibles).
- Tableaux CSV de qualité signal et statistiques par appareil.

### 3. `02_preprocessing` — **local**
Étape lourde de préparation des données. Pipeline en deux temps : alignement à 6 s par
maison, puis extraction des segments continus utilisables.

**Produit dans `data/processed/`** :
- `aligned/UK-DALE_house{1,2,4,5}.pkl` et `aligned/REDD_house*.pkl` — DataFrames alignés
  par maison (cache pour les étapes suivantes).
- `seq2point/{appliance}__UK-DALE__{train,test}.npz` et `seq2point/{appliance}__REDD__*.npz`
  — séquences 1D mains + cible + offsets de segments.
- `norm_params.json` — paramètres de normalisation μ/σ par dataset et appareil
  (calculés sur le train uniquement, pas de fuite).
- `manifest.json` — paramètres de prétraitement (résolution, fenêtre, seuils).

### 4. `03_baselines_nilmtk` — **local**
Entraînement et évaluation des baselines CO et FHMMExact sur le même protocole que
Seq2Point (train UK-DALE 1+2+4, test maison 5).

**Produit dans `outputs/baselines/`** :
- `metrics_co.json` — métriques CO par appareil (F1, MAE, erreur énergie).
- `metrics_fhmm.json` — métriques FHMMExact par appareil.

### 5. `04_seq2point` — **Colab GPU recommandé** (~40 min sur T4)
Entraînement de Seq2Point sur les 5 appareils, en deux variantes de perte (MSE et
combinée). Lance le notebook sur Colab après avoir uploadé `data/processed/` sur ton
Google Drive. La cellule `§0bis` du notebook gère le montage du Drive.

**Produit sur Google Drive (puis à rapatrier en local)** :
- `models/mse/seq2point_{appliance}_UK-DALE.pt` — 5 modèles entraînés perte MSE.
- `models/combined/seq2point_{appliance}_UK-DALE.pt` — 5 modèles entraînés perte combinée.
- `outputs/seq2point/metrics_mse.json` et `metrics_combined.json` — métriques par appareil.

**⚠️ Étape manuelle à faire après l'exécution Colab :**

Télécharger depuis Google Drive vers son arborescence locale :

| Sur le Drive | Vers le local |
|---|---|
| `MyDrive/Projet_scientifique/projet-nilm/models/combined/*.pt` | `models/combined/` |
| `MyDrive/Projet_scientifique/projet-nilm/models/mse/*.pt`      | `models/mse/`      |
| `MyDrive/Projet_scientifique/projet-nilm/outputs/seq2point/metrics_mse.json`      | `outputs/seq2point/` |
| `MyDrive/Projet_scientifique/projet-nilm/outputs/seq2point/metrics_combined.json` | `outputs/seq2point/` |

Sans ce rapatriement, les notebooks 04b, 06 et 08 ne pourront pas lire les artefacts.

### 6. `04b_inference_seq2point` — **local CPU** (~5-10 min)
Inférence des modèles Seq2Point entraînés (perte combinée) sur le test UK-DALE house 5.
Sauvegarde les prédictions pour réutilisation par les notebooks 06 et 08. Inclut un
contrôle de cohérence (MAE recalculée vs métriques de référence).

**Requiert en local :** `models/combined/*.pt` (téléchargés depuis le Drive à l'étape 5),
`data/processed/seq2point/*__UK-DALE__test.npz`, `data/processed/norm_params.json`.

**Produit dans `outputs/seq2point/`** :
- `predictions_combined_UK-DALE_test.npz` — prédictions et vérité terrain pour les 5
  appareils, aux mêmes positions stride que la métrique d'évaluation.

### 7. `07_ablation_resolution` — **Colab GPU recommandé** (~20-25 min sur T4)
Étude d'ablation sur la résolution temporelle : 6 s → 1 min → 5 min → 15 min → 30 min,
sur fridge et kettle. Réentraîne un Seq2Point dédié à chaque résolution.

**⚠️ Étape manuelle à faire après l'exécution Colab :**

Télécharger depuis Google Drive vers son arborescence locale:

| Sur le Drive | Vers le local |
|---|---|
| `MyDrive/Projet_scientifique/projet-nilm/outputs/ablation/metrics_ablation.csv` | `outputs/ablation/` |
| `MyDrive/Projet_scientifique/projet-nilm/outputs/ablation/ablation_f1_score.png` | `outputs/ablation/` |

**Produit dans `outputs/ablation/`** :
- `metrics_ablation.csv` — métriques par (appareil × résolution).
- `metrics_ablation.json` — mêmes données au format JSON.
- Modèles intermédiaires `.pt` (peuvent rester sur le Drive, non requis pour 06/08).

### 8. `06_evaluation_comparaison` — **local CPU** (~30 s)
Agrégation et comparaison de tous les modèles entraînés. Produit le tableau de synthèse
et les figures principales du rapport.

**Requiert en local** : `outputs/seq2point/metrics_*.json` (notebook 04 rapatrié),
`outputs/baselines/metrics_*.json` (notebook 03), `outputs/ablation/metrics_ablation.csv`
(notebook 07 rapatrié).

**Produit dans `outputs/evaluation/`** :
- `metrics_all.csv` — tableau unifié de toutes les métriques.
- `fig_comparaison_globale.png`, `fig_heatmap_f1.png`, `fig_ablation_resolution.png` —
  figures pour le rapport.

### 9. `08_flexibilite` — **local CPU** (<1 min)
Indicateur de potentiel de report de charge : classification des appareils,
détection des cycles flexibles, estimation de l'énergie décalable, extrapolation
nationale (borne supérieure).

**Requiert en local :** `outputs/seq2point/predictions_combined_UK-DALE_test.npz`
(produit par 04b).

**Produit dans `outputs/flexibilite/`** :
- `cycles_summary.json` — statistiques par appareil et estimation nationale.
- Figures sur la distribution des cycles et énergies.

### 10. `09_synthese` — **local CPU** (~30 s)
Synthèse autonome du projet : lit dynamiquement tous les artefacts produits, propose une
narration cohérente et répond aux questions du sujet. C'est le document de référence pour
l'évaluateur.

**Requiert** : la majorité des artefacts produits ci-dessus.

**Produit dans `outputs/synthese/`** : rien (lecture seule). À exporter en HTML ou PDF
pour le rendu final.

## Décisions de conception (résumé)

- **Cibles** : fridge (charge froide regroupée : fridge + freezer + fridge freezer),
  washing machine, dish washer, microwave, kettle (UK-DALE uniquement — absent de REDD).
- **Résolution commune 6 s** (= 1/6 Hz) sur UK-DALE et REDD. Extraction native + re-bin
  pandas + interpolation bornée des micro-trous (le resampling NILMTK direct introduit des
  NaN fantômes sur les sections non contiguës).
- **Split inter-foyers** : UK-DALE train [1, 2, 4] / test [5] ; REDD train [1, 2, 3] /
  test [4, 5]. Train et test sur maisons disjointes (généralisation inter-foyers = enjeu
  industriel réel).
- **Fenêtre** : 599 points (Seq2Point, Zhang 2018), ~1 h de contexte à 6 s.
- **Normalisation** : μ/σ calculés sur le train uniquement, sauvegardés dans `norm_params.json`.
- **Seuils ON/OFF** : constants par appareil — fridge 50 W, kettle 2000 W, microwave 200 W,
  dish washer 10 W, washing machine 20 W. Ceux par défaut de NILMTK étaient incohérents
  entre maisons.

## Résultats clés (test inter-foyers UK-DALE maison 5)

**Seq2Point Combined** est le meilleur modèle sur les 5 appareils :

| Appareil | F1 | MAE (W) | Erreur énergie |
|---|---|---|---|
| fridge          | 0,83 | 24,5 | 0,8 %  |
| kettle          | 0,85 | 9,2  | 27,5 % |
| dish washer     | 0,37 | 20,4 | 46,4 % |
| washing machine | 0,35 | 31,4 | 45,4 % |
| microwave       | 0,03 | 51,6 | 68,4 % (non évaluable, ~14 cycles dans h5) |

**Comparé aux meilleures baselines classiques (CO ou FHMMExact)** : Seq2Point Combined
divise l'erreur (MAE) par 4 à 6 selon l'appareil, avec un F1 doublé ou plus.

**Question Linky (résolution 30 min)** : F1 nul sur fridge ET kettle. La désagrégation
fine n'est pas viable à cette résolution. Seule l'estimation énergétique cumulée
sur les cycles longs reste partiellement possible (fridge).

**Flexibilité résidentielle** : borne supérieure théorique ~8-9 TWh/an pour
lave-linge + lave-vaisselle en France (~2 % de la consommation nationale annuelle),
sous l'hypothèse forte que 100 % des cycles sont décalables.

Détails complets dans le notebook `09_synthese.ipynb`.

## Limites connues

- **REDD** : enregistrements fragmentés (mains 50-90 % de pas non couverts en temps
  calendaire) → données utiles réduites, test mince. UK-DALE reste le protocole principal.
- **Dishwasher / microwave** : appareils rares dans h5 → peu d'exemples, métriques fragiles.
- **FHMMExact NILMTK** contraint à 2 états par appareil → baseline FHMM sous-spécifiée pour
  les appareils multi-paliers.
- **Modèles complémentaires** (LSTM / GRU / Seq2Seq, notebook 05) non livrés.

## Structure du dépôt et arborescence finale

### Dépôt GitHub (versionné)

```
projet-nilm/
  notebooks/
    00_test_setup.ipynb
    01_exploration_donnees.ipynb
    02_preprocessing.ipynb
    03_baselines_nilmtk.ipynb
    04_seq2point.ipynb
    04b_inference_seq2point.ipynb
    06_evaluation_comparaison.ipynb
    07_ablation_resolution.ipynb
    08_flexibilite.ipynb
    09_synthese.ipynb
  pyproject.toml
  requirements.txt
  uv.lock
  .python-version
  .gitignore
  README.md          (ce fichier)
  README_start.md    (installation initiale)
```

### Arborescence locale complète après exécution du pipeline

Ce qui n'est pas versionné mais est nécessaire à l'exécution complète :

```
projet-nilm/
  data/
    ukdale.h5
    redd.h5
    iawe.h5
    processed/
      norm_params.json
      manifest.json
      aligned/
        UK-DALE_house{1,2,4,5}.pkl
        REDD_house{1,2,3,4,5}.pkl
      seq2point/
        {appliance}__UK-DALE__{train,test}.npz       (10 fichiers)
        {appliance}__REDD__{train,test}.npz          (10 fichiers, sauf kettle)
  models/
    mse/
      seq2point_{appliance}_UK-DALE.pt               (5 fichiers, ~150 Mo)
    combined/
      seq2point_{appliance}_UK-DALE.pt               (5 fichiers, ~150 Mo)
  outputs/
    exploration/
      exploration_config.json
      *.csv
    baselines/
      metrics_co.json
      metrics_fhmm.json
    seq2point/
      metrics_mse.json
      metrics_combined.json
      predictions_combined_UK-DALE_test.npz
    ablation/
      metrics_ablation.csv
      metrics_ablation.json
    evaluation/
      metrics_all.csv
      fig_*.png
    flexibilite/
      cycles_summary.json
      *.png
    synthese/
      (pas de sortie, lecture seule)
```

### Arborescence Drive (pour utilisateurs Colab)

```
MyDrive/Projet_scientifique/projet-nilm/
  data/
    processed/                                (uploadé depuis local étape 3)
      aligned/, seq2point/
      norm_params.json, manifest.json
  models/
    mse/, combined/                           (généré par notebook 04 sur Colab)
  outputs/
    seq2point/metrics_{mse,combined}.json     (généré par notebook 04 sur Colab)
    ablation/metrics_ablation.{csv,json}      (généré par notebook 07 sur Colab)
```

Les artefacts produits sur le Drive doivent ensuite être **rapatriés en local** (cf.
étapes 5 et 7 ci-dessus) pour que les notebooks de post-traitement (04b, 06, 08, 09)
puissent les lire.
