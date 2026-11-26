# Étape 00 — Environnement, versions de Python et dépôt

> Durée indicative : une soirée. Prérequis : aucun.

## Objectif

Comprendre et maîtriser **quel Python** exécute ton code et **quelles dépendances** il voit. La moitié des « ça marche chez moi » en Python vient de là.

## Notions (cours)

### L'interpréteur

Plusieurs Python coexistent sur une machine : celui du système (macOS en fournit un, à ne jamais toucher), celui de Homebrew, ceux installés par un gestionnaire de versions. La commande `python3` pointe vers l'un d'eux selon le `PATH`. Le réflexe à prendre : `which python3`, `python3 -c "import sys; print(sys.executable, sys.version)"`.

### Les environnements virtuels

Un **venv** est un dossier (`.venv/`) qui contient un lien vers un interpréteur et son propre `site-packages`. Activer un venv revient à mettre son `bin/` en tête du `PATH`. Sans venv, `pip install` installe dans le Python global : les conflits de versions entre projets sont alors inévitables. Règle absolue : **un projet = un venv**, jamais versionné dans Git.

### uv

`uv` (Astral) est l'outil qui s'est imposé ces dernières années. Il remplace `pyenv` (installation de versions de Python), `venv` (création d'environnements), `pip` (installation) et `pip-tools`/Poetry (verrouillage), en étant beaucoup plus rapide. Commandes clés :

- `uv python install 3.13 3.14 3.15` : installer des interpréteurs ;
- `uv venv` / `uv sync` : créer l'environnement et installer exactement ce que décrit le lockfile ;
- `uv add httpx` / `uv add --dev pytest` : ajouter une dépendance (de prod ou de dev) ;
- `uv run pytest` : exécuter une commande dans l'environnement du projet, sans l'activer.

Les alternatives que tu croiseras en entreprise : pip + venv + pip-tools (le plus « standard bibliothèque »), Poetry, PDM, Hatch. Les concepts sont identiques, seules les commandes changent. Sache expliquer **lequel tu as choisi et pourquoi**.

### Dépendances déclarées et dépendances verrouillées

- Dans `pyproject.toml`, on déclare des **contraintes** larges : `httpx>=0.27`. C'est ce dont la bibliothèque a besoin.
- Dans `uv.lock`, on **fige** les versions exactes, y compris les dépendances transitives, avec leurs hashes. C'est ce qui rend l'environnement de dev et de CI reproductible.

Une **bibliothèque** publiée ne force pas les versions exactes chez ses utilisateurs, mais **ton CI** doit utiliser le lockfile.

## Consigne

Installer uv et deux versions de Python, créer le dépôt `sprintkit`, initialiser le projet et préparer l'agent Jenkins pour Python.

## Instructions

1. Installe uv (`brew install uv`, ou le script officiel), puis `uv python install 3.13 3.14 3.15`.
2. Crée le dépôt GitHub public `sprintkit`, clone-le, copie `docs/roadmap/`.
3. `uv init --lib --name sprintkit` : observe et comprends chaque fichier généré, puis supprime ce qui ne te sert pas.
4. Fichier `.python-version` à `3.14` : la version de dev par défaut.
5. `.gitignore` Python (`.venv/`, `__pycache__/`, `*.egg-info/`, `dist/`, `.coverage`, `htmlcov/`, `.mypy_cache/`, `.ruff_cache/`, `.pytest_cache/`, `.nox/`, `.env`), plus `LICENSE` et `README.md` provisoire.
6. `uv sync`, puis vérifie : `uv run python -c "import sys; print(sys.executable)"` doit pointer dans `.venv/`.
7. Crée un `.env.example` (versionné) avec `JIRA_BASE_URL=`, `JIRA_EMAIL=` et `JIRA_API_TOKEN=`. Le vrai `.env`, lui, est ignoré.
8. Côté Jira Cloud : crée l'instance gratuite, un projet Scrum avec une quinzaine d'issues, deux sprints, quelques worklogs, et ton token d'API. Teste avec `curl -u email:token https://<toi>.atlassian.net/rest/api/3/myself`.

## Contraintes et spécificités

- Pas de `pip install` hors d'un venv, jamais de `sudo pip`.
- Le token Jira n'apparaît dans aucun fichier versionné, ni dans l'historique de ton shell (utilise `read -s` ou un gestionnaire de mots de passe).

## Résultat attendu

- [x] `uv run python --version` donne 3.14.x, et `uv run --python 3.13 python --version` et `uv run --python 3.15 python --version` donnent les versions correspondantes.
- [x] `.venv/` existe et est ignoré par Git.
- [x] L'appel `curl` à `/myself` répond 200 avec ton compte.
- [x] Tu sais expliquer la différence entre `pyproject.toml` (contraintes) et `uv.lock` (versions exactes).

## Phase DevOps

**Attendu de l'intégrateur :** un dépôt gouverné, et un environnement CI qui sait exécuter uv.

1. Protection de `main` (PR obligatoire, historique linéaire), template de PR, Conventional Commits : même configuration que pour skylink.
2. **Jenkins :**
   - job Multibranch `sprintkit` ;
   - prérequis : la VM agent est prête (Docker installé, `jenkins` dans le groupe `docker`, 0 executor sur le controller). Voir « Infrastructure Jenkins : VM sur Proxmox » dans le README global ;
   - stratégie d'exécution : un conteneur Docker basé sur une image officielle (`ghcr.io/astral-sh/uv:python3.14-bookworm-slim`, ou `python:3.14-slim` avec uv installé), plutôt qu'un Python installé sur la VM. La VM ne porte que Docker, Java et Git : tout l'outillage Python vit dans l'image ;
   - cache uv : le dossier `/var/cache/ci/uv` de la VM, monté dans le conteneur (`args '-v /var/cache/ci/uv:/cache/uv'`), avec `UV_CACHE_DIR=/cache/uv`. Vérifie au second build que les paquets ne sont plus retéléchargés ;
   - snapshot Proxmox de la VM agent une fois qu'elle est fonctionnelle : c'est ton point de retour si tu casses quelque chose en avançant.
3. **GitHub Actions :** prends connaissance de `astral-sh/setup-uv` (cache intégré), que tu utiliseras à l'étape 01.
4. **Credentials Jenkins :** crée un credential « Username with password » `jira-cloud` (email + token). Il ne servira qu'à l'étape 08, mais pose dès maintenant la bonne pratique.

Critères :

- [x] Un push direct sur `main` est refusé.
- [ ] Un job Jenkins de test (même un `sh 'uv --version'` dans le conteneur) passe, et s'exécute sur la VM agent, pas sur le controller (vérifie le nœud indiqué dans la console).

## Pièges fréquents

- Activer le venv dans un terminal et oublier qu'on est dedans (ou l'inverse). `uv run` évite le problème.
- Versionner `.env` « juste pour tester ». S'il part sur GitHub, le token doit être **révoqué**, pas seulement supprimé de l'historique.
