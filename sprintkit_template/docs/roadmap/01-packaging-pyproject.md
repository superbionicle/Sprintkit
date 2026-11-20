# Étape 01 — Packaging : pyproject.toml et structure src

> Durée indicative : 2 soirées. Prérequis : étape 00.

## Objectif

Comprendre comment du code Python devient un **paquet installable**, et faire de sprintkit un paquet qui expose une commande `sprintkit` dès l'installation.

## Notions (cours)

### Module, paquet, distribution

- Un **module** est un fichier `.py`.
- Un **paquet** (import package) est un dossier avec un `__init__.py`.
- Une **distribution** (le « paquet » au sens de PyPI) est une archive qu'on installe avec pip.

Le nom de distribution (`sprintkit` sur PyPI) et le nom d'import (`import sprintkit`) coïncident souvent, mais pas toujours (`pip install pyyaml` donne `import yaml`).

### sdist et wheel

- Une **sdist** (`.tar.gz`) contient les sources et les métadonnées. Elle doit être « construite » à l'installation.
- Une **wheel** (`.whl`) est un zip prêt à être décompressé dans `site-packages`. Elle n'exécute aucun code à l'installation.

Son nom encode la compatibilité, par exemple `sprintkit-0.1.0-py3-none-any.whl` : Python 3, pas d'ABI spécifique, toute plateforme, car c'est du pur Python. Un paquet avec du code C (numpy) a une wheel par plateforme.

### Le build : frontend et backend (PEP 517/518)

`pyproject.toml` déclare un **build backend** (`hatchling`, `setuptools`, `flit-core`, `pdm-backend`…) dans `[build-system]`. Un **frontend** (`uv build`, `python -m build`, pip) crée un environnement isolé, installe le backend, puis lui demande de produire sdist et wheel. `setup.py` est l'ancien modèle. Tu en verras encore, mais tu n'en écris plus.

### Les métadonnées (PEP 621)

La table `[project]` contient `name`, `version` (ou `dynamic = ["version"]`), `description`, `readme`, `requires-python`, `license`, `authors`, `classifiers`, `dependencies`, `optional-dependencies` et `scripts`.

### Les points d'entrée

`[project.scripts] sprintkit = "sprintkit.cli:app"` fait générer par pip un exécutable `sprintkit` dans le `bin/` du venv, qui appelle cette fonction. C'est ainsi que naissent toutes les commandes CLI Python (`black`, `pytest`…).

### Le layout `src/`

Le code vit dans `src/sprintkit/` plutôt qu'à la racine. Pourquoi ? Sans `src/`, `import sprintkit` dans les tests importe le dossier local, et non le paquet **installé**. Tu ne testes donc pas ce que tu livres : un fichier oublié dans le paquet passe inaperçu. Avec `src/`, les tests sont obligés de passer par le paquet installé.

### L'installation éditable

`uv sync`, ou `pip install -e .`, installe le paquet en mode « lien » : les modifications du code sont visibles sans réinstaller. C'est le mode de dev. La CI, elle, doit aussi tester **la wheel construite** (étape 07).

### Les groupes de dépendances

- Dépendances de **prod** : `[project.dependencies]`.
- Extras optionnels pour l'utilisateur : `[project.optional-dependencies]`, par exemple `excel = ["openpyxl"]`.
- Dépendances de **dev** : `[dependency-groups]` (PEP 735), qui ne sont jamais publiées.

### La source de la version

Pour l'instant, la version est statique dans `pyproject.toml`. Le code la lit avec `importlib.metadata.version("sprintkit")`, plutôt qu'avec une constante dupliquée. À l'étape 12, elle viendra du tag Git.

## Consigne

Configurer `pyproject.toml` complet, avec le layout `src/`, une commande `sprintkit --version` fonctionnelle, et la construction de la wheel et de la sdist.

## Instructions

1. `pyproject.toml` :
   - `[build-system]` hatchling ;
   - `[project]` complet (description, `readme`, `requires-python = ">=3.13"`, licence SPDX, classifiers, URLs du dépôt) ;
   - dépendances de prod : `httpx` et `typer` (rien d'autre pour l'instant) ;
   - extra `excel = ["openpyxl"]` ;
   - groupe de dev `dev = ["pytest"]` (on complétera).
2. `src/sprintkit/__init__.py` expose `__version__`, lu via `importlib.metadata`.
3. `src/sprintkit/cli/__init__.py` : une app Typer minimale avec `--version`.
4. `src/sprintkit/__main__.py`, pour que `python -m sprintkit` fonctionne aussi.
5. `uv sync`, puis `uv run sprintkit --version` affiche `sprintkit 0.1.0`.
6. `uv build` : inspecte le contenu de la wheel (`unzip -l dist/*.whl`) et de la sdist (`tar tzf`). Vérifie qu'il n'y a ni tests, ni `.env`, ni `docs/roadmap` dans la wheel. Ajuste `[tool.hatch.build]` si besoin.
7. Installe la wheel dans un venv **vierge**, hors du projet (`uv venv /tmp/t && uv pip install --python /tmp/t dist/*.whl`), et lance `/tmp/t/bin/sprintkit --version`.

## Contraintes et spécificités

- La version n'est écrite qu'à **un seul** endroit.
- Le fichier `py.typed` (vide) dans `src/sprintkit/` signale que le paquet fournit des annotations de type (PEP 561).
- Aucune dépendance de dev dans `[project.dependencies]`.

## Résultat attendu

- [ ] `uv run sprintkit --version` et `uv run python -m sprintkit --version` fonctionnent.
- [ ] `uv build` produit `sprintkit-0.1.0-py3-none-any.whl` et `sprintkit-0.1.0.tar.gz`.
- [ ] La wheel installée dans un venv vierge fournit la commande `sprintkit`.
- [ ] Tu sais expliquer ce qu'est un build backend, la différence wheel/sdist, et à quoi sert le layout `src/`.

## Phase DevOps

**Attendu de l'intégrateur :** chaque PR prouve que le projet **se construit et s'installe**.

1. **GitHub Actions `ci.yml`**, job `build` :
   - `astral-sh/setup-uv` avec cache ;
   - `uv build` ;
   - `uvx twine check dist/*` (validation des métadonnées et du rendu du README pour PyPI) ;
   - upload de `dist/` en artefact.
2. **Jenkinsfile déclaratif :**
   - agent Docker uv ;
   - stages `Setup` (`uv sync --locked`) et `Build` (`uv build` + `twine check`) ;
   - `archiveArtifacts 'dist/*'` ;
   - options : timestamps, ansiColor, timeout, buildDiscarder.
3. **`--locked` en CI :** `uv sync --locked` échoue si `uv.lock` ne correspond pas à `pyproject.toml`. Ça détecte l'oubli de mettre à jour le lockfile.
4. **Status check** `build` requis sur `main`, et badge dans le README.

Critères :

- [ ] Une PR qui modifie `pyproject.toml` sans mettre à jour `uv.lock` échoue.
- [ ] La wheel est téléchargeable depuis les artefacts GitHub **et** Jenkins.

## Pièges fréquents

- `__version__ = "0.1.0"` en dur dans `__init__.py` **et** dans `pyproject.toml` : les deux divergent au bout de deux releases.
- Tester uniquement en mode éditable, puis découvrir après publication qu'un fichier de données n'est pas inclus dans la wheel.
