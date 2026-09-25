## Plan proposé

### `01_exploration_donnees.ipynb` — Comprendre ce qu'il y a vraiment dans les datasets

L'identification des maisons et du nombre d'appareils constitue une première étape. L'exploration doit être approfondie selon les axes suivants :

- Inventaire des appareils par maison : identification des appareils présents dans plusieurs maisons. C'est un point crucial pour le protocole inter-foyers — l'entraînement et le test ne pouvant s'effectuer que sur des appareils communs.
- Fréquences d'échantillonnage réelles par maison et par sous-compteur (l'annonce "1/6 Hz" pour UK-DALE cachant des variations).
- Qualité du signal : détection des trous, des NaN, et évaluation des durées des sections continues exploitables. La fonction `elec.mains().get_timeframe()` ne signalant pas la présence de trous internes.
- Statistiques par appareil : puissance moyenne en état ON, durée typique des cycles, fréquence d'usage. Données indispensables pour le seuillage ON/OFF ultérieur.
- Vérification du « bilan » : pour quelques fenêtres, comparaison du signal agrégé à la somme des sous-compteurs. La différence (= appareils non monitorés) mesure à quel point la décomposition est *résiduellement bruitée*.
- Choix des appareils cibles : définition dès à présent d'un ensemble de 4-5 appareils sur lesquels travailler de manière systématique. Les classiques de la littérature : **fridge, washing machine, dishwasher, microwave, kettle**. Ils possèdent des signatures bien distinctes et sont communs entre UK-DALE et REDD.

Sortie : un tableau récapitulatif des appareils retenus × maisons où ils sont monitorés, et la décision finale sur le split inter-foyers.

### `02_preprocessing.ipynb` — Documenter chaque décision

Cette partie nécessite de trancher et de justifier explicitement les choix méthodologiques suivants :

- **Resampling** : la cible standard dans la littérature est de 6 s pour UK-DALE, et de 1-3 s pour REDD. Règle à appliquer : aligner mains et submeters à la même fréquence, et choisir la plus grossière des deux par paire.
- **Fenêtres glissantes** : la valeur de référence pour Seq2Point est une fenêtre de 599 points centrée. Option à retenir et à documenter.
- **Normalisation** : méthode standard dans la littérature = soustraire la moyenne et diviser par l'écart-type **calculés sur le train uniquement** (afin d'éviter toute fuite de données). Conservation des paramètres pour le test.
- **Gestion des trous** : suppression (*drop*) des fenêtres qui en contiennent, plutôt qu'une imputation.
- **Sauvegarde** : sérialisation des tenseurs prétraités en `.npy` ou `.pt` dans un dossier `data/processed/`. Recharger depuis NILMTK à chaque run représentant un coût de plusieurs minutes inutiles. Ajout de ce dossier dans le `.gitignore`.

### `03_baselines_nilmtk.ipynb` — CO et FHMM

NILMTK fournit ces deux modèles de manière native (`CombinatorialOptimisation`, `FHMM` dans `nilmtk.disaggregate`). Entraînement à réaliser sur le split inter-foyers, suivi de la sauvegarde des prédictions. Ces baselines servent de référence à battre, et leur structure (états discrets) est conceptuellement éclairante.

⚠️ Attention compatibilité : NILMTK upstream présente parfois des frictions avec les versions récentes de pandas 2.x / numpy 2.x. En cas de bugs, l'utilisation de `nilmtk-contrib` peut aider ; dans tous les cas, il convient de documenter clairement quels modèles fonctionnent et lesquels posent problème.

### `04_seq2point.ipynb` — Le modèle de référence

Débuter par Seq2Point seul. Un modèle par appareil cible. Architecture : 5 couches Conv1D + dense, fenêtre de 599 points, prédiction du point central. C'est la configuration largement reproduite dans la littérature.

Conseils pratiques :
- Entraîner d'abord sur **un seul appareil, une seule maison** pour vérifier la décroissance de la loss. Ne pas lancer de parallélisation au début.
- GPU recommandé (sur CPU, compter 1-2 h par appareil sur UK-DALE house 1).
- Enregistrement des courbes train/val et sauvegarde des poids (`models/seq2point_fridge_h1.pt`).
- Maintien d'un script `train.py` réutilisable à `import`er depuis le notebook plutôt que d'insérer tout le code dans des cellules.

### `05_modeles_complementaires.ipynb` — Seq2Seq et un modèle récurrent

Une fois Seq2Point validé, l'implémentation de Seq2Seq constitue une variante directe (modification de la tête du modèle). Ajout d'un LSTM ou GRU comme point de comparaison entre approches récurrentes et convolutives. BERT4NILM est une option intéressante mais coûteuse en temps d'implémentation et de calcul — à réserver en cas de marge en fin de projet.

### `06_evaluation_comparaison.ipynb` — Le cœur scientifique

Calcul de trois métriques minimum, de manière cohérente sur tous les modèles :

- **MAE** sur la puissance (W) — sensibilité aux erreurs absolues.
- **F1-score** sur l'état ON/OFF (avec un seuil par appareil, justifié) — qualité de la détection d'usage.
- **Erreur relative sur l'énergie totale** sur la fenêtre de test — pertinence opérationnelle pour la facturation et la flexibilité.

Produit attendu : un tableau modèles × appareils × métriques. Discussion des compromis : un modèle pouvant gagner en MAE et perdre en F1 si ses prédictions sont lissées.

### `07_ablation_resolution.ipynb` — La question Linky

Réalisation d'une étude d'ablation par ré-échantillonnage progressif du signal d'entrée : 6 s (natif) → 1 min → 5 min → 15 min → 30 min (Linky). À chaque résolution, les questions suivantes sont à analyser :
- Quels appareils restent détectables (F1 > seuil) ?
- Sur quels appareils l'erreur augmente-t-elle de façon critique ?
- Quelle architecture résiste le mieux à la dégradation ?

Cette partie constitue l'apport original du rendu et répond directement à la question opérationnelle de la faisabilité du NILM en sortie de compteur Linky standard.

### `08_flexibilite.ipynb` — Du désagrégé à l'opérationnel

Transition de la pure désagrégation vers l'application pratique :
- Classer les appareils par flexibilité intrinsèque : *fortement flexibles* (lave-linge, lave-vaisselle, sèche-linge, ballon ECS), *peu flexibles* (frigo, congélateur — cycles imposés par la thermique), *non flexibles* (éclairage, cuisson).
- Pour chaque appareil flexible identifié dans une trace désagrégée, calculer une "fenêtre de décalage admissible" — par exemple : un cycle lave-linge détecté entre 18h et 19h pourrait être décalé vers la fenêtre 23h-6h, soit X kWh × Y heures de flexibilité.
- Agréger : sur l'ensemble des maisons de test, évaluation du potentiel total de décalage vers les heures creuses (expression en kWh/jour/foyer puis extrapolation).

Une analyse descriptive rigoureuse est suffisante pour ce premier jet, sans recours nécessaire à un modèle d'optimisation sophistiqué.

### `09_synthese.ipynb` — Conclusions

Tableaux finaux, figures-clés, limites de l'étude et pistes d'extension. En cas de rendu sous forme de notebook unique, ce module central doit importer les résultats sauvegardés des autres notebooks.

---

## Quelques conseils transverses

- **Mise en cache systématique** : tout résultat dont le calcul prend > 30 s doit être sauvegardé (chargements NILMTK, modèles entraînés, prédictions) afin de fluidifier les itérations.
- **Séparation du code et des notebooks** : création d'un dossier `src/` contenant les classes de dataset PyTorch, le modèle Seq2Point et les fonctions de métriques. Les notebooks se limitent à l'importation de ces modules pour garantir la lisibilité du rendu.
- **Reproductibilité** : configuration des graines aléatoires (`torch.manual_seed`, `np.random.seed`) en haut de chaque notebook impliquant un entraînement. Documentation des versions GPU/CPU.
- **Périmètre** : focalisation sur 4-5 appareils, 2 datasets (UK-DALE + REDD) et 3-4 modèles bien évalués, plutôt que sur un catalogue de 10 modèles à moitié comparés. La valeur réside dans la rigueur du protocole.

Le preprocessing (notebook n° 2) et le protocole d'évaluation inter-foyers constituent les deux étapes concentrant le plus de biais potentiels.
