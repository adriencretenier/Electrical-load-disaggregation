# Projet NILM — Guide de démarrage

Désagrégation de consommation électrique (NILM) et flexibilité énergétique sur les datasets UK-DALE et REDD.

---

## Prérequis (à installer si pas déjà fait)

- **Git** → https://git-scm.com/downloads
- **VSCode** → https://code.visualstudio.com/
- **uv** (gestionnaire Python) :

  PowerShell sur Windows :
  ```powershell
  powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
  ```

  macOS / Linux :
  ```bash
  curl -LsSf https://astral.sh/uv/install.sh | sh
  ```

  **Ferme et rouvre ton terminal**, puis vérifie :
  ```bash
  uv --version
  ```

---

## 1. Cloner le repo

```powershell
cd <le-dossier-où-tu-veux-mettre-le-projet>
git clone https://github.com/Projet-scientifique-NILM/Projet-NILM.git projet-nilm
cd projet-nilm
```

---

## 2. Basculer sur ta branche personnelle

Liste les branches existantes :
```powershell
git fetch origin
git branch -a
```

Bascule sur ta branche (remplace `<ton-prenom>` par le nom de ta branche) :
```powershell
git checkout <ton-prenom>
```

Si ta branche n'existe pas encore, crée-la à partir de `main` :
```powershell
git checkout -b <ton-prenom> origin/main
git push -u origin <ton-prenom>
```

---

## 3. Installer l'environnement Python

```powershell
uv sync
```

Cette commande lit `pyproject.toml` + `uv.lock` et installe **exactement** les mêmes versions que les autres membres de l'équipe.

⏱️ Compte 5 à 15 minutes la première fois.

Active le venv :

**Windows** :
```powershell
.\.venv\Scripts\Activate.ps1
```

**macOS / Linux** :
```bash
source .venv/bin/activate
```

> ⚠️ Si PowerShell renvoie une erreur d'execution policy, lance une fois :
> ```powershell
> Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
> ```

---

## 4. Configurer ton identité git (uniquement la première fois sur cette machine)

```powershell
git config --global user.email "ton.email@gmail.com"
git config --global user.name "Ton Nom"
```

Utilise la même adresse email que ton compte GitHub.

---

## 5. Récupérer les datasets

Les fichiers HDF5 sont trop volumineux pour Git. Télécharge-les depuis les liens ci-dessous :

| Dataset | Taille | Lien de téléchargement |
|---|---|---|
| `ukdale.h5` | ~6 Go | https://data.ceda.ac.uk/edc/d1/7d78f943-f9fe-413b-af52-1816f9d968b0/data/version_0 ou  https://zenodo.org/records/13917372/files/ukdale.h5 |
| `redd.h5` | ~0.4 Go | https://zenodo.org/records/13917372/files/redd.h5 |
| `iawe.h5` | ~700 Mo |  https://zenodo.org/records/13917372/files/iawe.h5 |

Crée le dossier `data/` à la racine du projet et place les fichiers dedans :
```powershell
mkdir data
```

Arborescence attendue :
```
projet-nilm/
└── data/
    ├── ukdale.h5
    ├── redd.h5
    └── iawe.h5
```

---

## 6. Configurer VSCode

```powershell
code .
```

Une fois VSCode ouvert :

1. **Installer les extensions** (Marketplace, icône de 4 carrés à gauche) :
   - **Python** (Microsoft)
   - **Jupyter** (Microsoft)

2. **Sélectionner l'interpréteur Python** :
   - `Ctrl+Shift+P` → tape "Python: Select Interpreter"
   - Choisis celui qui pointe vers `.venv\Scripts\python.exe`
   - Vérifie en bas à droite : tu dois voir `3.13.x ('.venv': venv)`

> Si l'interpréteur n'apparaît pas, vérifie que tu as bien ouvert le dossier `projet-nilm` (et pas un dossier parent).

---

## 7-8. Vérifier que tout fonctionne et résoudre les blocages nilmtk et PyTorch / Windows (si applicable)

Sur certaines machines, l'import de PyTorch est bloqué (par **Smart App Control**, **Microsoft Defender** ou équivalent). 

Les commandes suivantes peuvent prendre du temps à être exécutée la première fois (5mn).
Les lancer les unes après les autres.
Pour la première commande : python -c "import nilmtk; print('NILMTK:', nilmtk.__version__)" , il se peut que Microsoft Defender bloque le téléchargement des fichiers source de nilmtk. Dans ce cas, aller dans Paramètres > Confidentialité et Sécurité > Sécurité Windows > Protection contre les virus et menaces > Aller dans l'onglet de l'application en charge de la protection sur l'ordinateur (Microsoft Defender, McAffee ou autre). Puis aller dans Exclusions / Gérer les exclusions d'analyse. Sélectionner le dossier projet-nilm nouvellement créé.
Il sera alors considérer comme un dossier libre pour le téléchargement de fichiers non vérifiés.

```powershell
python -c "import nilmtk; print('NILMTK:', nilmtk.__version__)"
python -c "import torch; print('PyTorch:', torch.__version__)"
python -c "import pandas, numpy; print('pandas:', pandas.__version__, '| numpy:', numpy.__version__)"
```

Les trois doivent afficher une version sans planter. Les warnings (`UnclosedFileWarning`, etc.) sont normaux.

Lance ensuite le notebook de test :
```powershell
jupyter lab notebooks/00_test_setup.ipynb
```

Ou ouvre-le directement dans VSCode et exécute les cellules. Si la dernière cellule affiche un graphique de consommation, **tout fonctionne**.

---

## ✅ Checklist finale

- [ ] `uv --version` répond
- [ ] Le repo est cloné dans `projet-nilm/`
- [ ] Je suis sur ma branche personnelle (`git branch` affiche `*` devant mon nom)
- [ ] `uv sync` s'est terminé sans erreur
- [ ] Le venv est activé (`(projet-nilm)` apparaît dans le prompt)
- [ ] `data/ukdale.h5`, `data/redd.h5`, `data/iawe.h5` sont en place
- [ ] Les 3 imports Python passent sans planter
- [ ] Le notebook `00_test_setup.ipynb` s'exécute et affiche un graphique
- [ ] VSCode utilise le bon interpréteur (`.venv\Scripts\python.exe`)

---

## Workflow git du quotidien

```powershell
# Récupérer les modifs des collègues sur main
git checkout main
git pull origin main

# Revenir sur ta branche et rapatrier les modifs de main
git checkout <ton-prenom>
git merge main

# Faire des modifs, puis commit + push
git add .
git commit -m "Description claire du changement"
git push origin <ton-prenom>
```

---

## Commandes utiles

| Action | Commande |
|---|---|
| Activer le venv | `.\.venv\Scripts\Activate.ps1` |
| Ajouter une dépendance | `uv add <package>` |
| Supprimer une dépendance | `uv remove <package>` |
| Re-synchroniser après un `git pull` | `uv sync` |
| Lancer Jupyter Lab | `jupyter lab` |

---

## En cas de problème

1. Vérifie que tu es bien dans le dossier `projet-nilm` (`pwd` ou `Get-Location`)
2. Vérifie que le venv est activé (`(projet-nilm)` dans le prompt)
3. Vérifie que tu es sur ta branche (`git branch`)
4. Copie le **message d'erreur complet** avant de demander de l'aide

---

## Datasets utilisés

- **UK-DALE** (Kelly & Knottenbelt 2015) — DOI: [10.1038/sdata.2015.7](https://doi.org/10.1038/sdata.2015.7)
- **REDD** (Kolter & Johnson 2011) — [redd.csail.mit.edu](http://redd.csail.mit.edu)
- **iAWE** — [iawe.github.io](https://iawe.github.io/)

## Stack technique

- Python 3.13, gérée par `uv`
- NILMTK 0.4.x (depuis git)
- PyTorch 2.x (pour les modèles deep learning)
- Jupyter + pandas + matplotlib + seaborn
