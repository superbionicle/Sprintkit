# sprintkit — cours et feuille de route

**sprintkit** est une bibliothèque Python et un outil en ligne de commande pour l'API REST de **Jira Cloud** : recherche d'issues en JQL, rapports de worklogs par semaine ISO, état des sprints, export CSV/JSON/Excel. Le sujet t'est familier. L'objectif n'est pas la fonctionnalité, mais **tout ce qu'il y a autour** : la manière dont un projet Python professionnel est structuré, testé, packagé, documenté, sécurisé et livré.

Ce parcours est aussi un **cours**. Chaque étape commence par une section « Notions » plus développée que dans le parcours C++ : lis-la entièrement avant de coder.

## Versions de Python ciblées

Aucune contrainte de version : le projet suit la pratique courante d'une bibliothèque moderne, qui consiste à supporter les **trois dernières versions stables**.

- **Supportées et testées :** 3.13, 3.14 et 3.15 (sortie le 9 octobre 2026). `requires-python = ">=3.13"`.
- **Version de dev par défaut :** 3.14, la plus récente avec un écosystème entièrement mûr.
- **3.15 :** testée en CI dès le départ. Une dépendance qui ne publie pas encore de wheel pour une version toute neuve est une situation réelle, et l'étape 08 apprend à la gérer.

Quand une nouvelle version sort, on l'ajoute à la matrice et on retire la plus ancienne. C'est une release MINEURE, documentée dans le CHANGELOG.

## Prérequis

- Un compte **Jira Cloud gratuit** (offre Free, jusqu'à 10 utilisateurs) avec un projet de démo (« Scrum »), quelques issues, sprints et worklogs, plus un **token d'API** (id.atlassian.com → Security → API tokens).
- Python 3.13, 3.14 et 3.15 installés via `uv` (étape 00).
- Docker, et ton Jenkins homelab.

## Architecture cible

```
sprintkit/
├── pyproject.toml          # métadonnées, dépendances, config de tous les outils
├── uv.lock                 # versions exactes résolues (lockfile)
├── noxfile.py              # sessions automatisées : tests, lint, typage, docs, build
├── src/sprintkit/
│   ├── __init__.py
│   ├── __main__.py         # python -m sprintkit
│   ├── domain/             # modèles et logique pure (aucune I/O)
│   ├── jira/               # client HTTP (adaptateur)
│   ├── reports/            # agrégations et exports
│   ├── config.py
│   ├── logging.py
│   └── cli/                # interface Typer
├── tests/
│   ├── unit/
│   ├── integration/        # contre un faux serveur Jira en Docker
│   └── conftest.py
├── fakejira/               # faux serveur Jira (FastAPI) pour les tests d'intégration
├── docs/                   # MkDocs (+ ce parcours)
├── Dockerfile
├── Jenkinsfile
└── .github/workflows/
```

## Les étapes

| # | Étape | Phase DevOps |
|---|---|---|
| 00 | [Environnement, versions de Python et dépôt](00-environnement.md) | Dépôt, protections, agent Jenkins Python |
| 01 | [Packaging : pyproject.toml et structure src](01-packaging-pyproject.md) | Build sdist/wheel en CI, validation des métadonnées |
| 02 | [Qualité : lint, formatage, typage](02-qualite-lint-typage.md) | Quality gate rapide, pre-commit en CI |
| 03 | [Tests unitaires avec pytest et TDD](03-tests-pytest.md) | Rapports JUnit et couverture, seuil bloquant |
| 04 | [Modélisation et architecture](04-modelisation-architecture.md) | Contrôle des dépendances entre couches |
| 05 | [Client HTTP robuste et secrets](05-client-http.md) | Gestion des secrets, scan de secrets |
| 06 | [Configuration et logging](06-configuration-logging.md) | Variables d'environnement par stage, logs exploitables en CI |
| 07 | [Interface en ligne de commande](07-cli.md) | Smoke test du paquet installé |
| 08 | [Tests d'intégration et multi-versions](08-integration-nox.md) | Matrice de versions, services Docker en pipeline |
| 09 | [Documentation](09-documentation.md) | Build et publication du site de doc |
| 10 | [Sécurité de la chaîne d'approvisionnement](10-supply-chain.md) | SCA, Renovate, SBOM |
| 11 | [Conteneurisation](11-conteneurisation.md) | Image durcie, scan, push |
| 12 | [Release et publication](12-release.md) | Release sur tag, TestPyPI, shared library Jenkins commune |

## Carte mentale de « tout ce qui englobe un projet »

| Préoccupation | Question à laquelle elle répond | Outil dans ce parcours |
|---|---|---|
| Environnement | Avec quel Python et quelles dépendances exactes ? | uv, `.python-version`, `uv.lock` |
| Packaging | Comment le code devient-il installable ? | `pyproject.toml`, hatchling, wheel |
| Style et lint | Le code est-il homogène et sans erreur évidente ? | ruff |
| Typage | Les types sont-ils cohérents ? | mypy (strict) |
| Tests | Le comportement est-il correct ? | pytest, pytest-cov, respx, hypothesis |
| Automatisation locale | Comment lancer tout ça en une commande ? | nox, pre-commit |
| Configuration | Comment paramétrer sans modifier le code ? | TOML + variables d'environnement |
| Observabilité | Que s'est-il passé quand ça a planté ? | logging |
| Documentation | Comment un autre peut-il s'en servir ? | MkDocs, mkdocstrings, CHANGELOG |
| Sécurité | Mes dépendances et mes secrets sont-ils sains ? | pip-audit, gitleaks, Renovate, SBOM |
| Livraison | Comment le distribuer ? | wheel sur (Test)PyPI, image Docker |
| CI/CD | Comment tout est-il vérifié et livré sans intervention humaine ? | GitHub Actions + Jenkins |
