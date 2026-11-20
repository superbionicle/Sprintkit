# Étape 08 — Tests d'intégration et multi-versions

> Durée indicative : 3 soirées. Prérequis : étape 07.

## Objectif

Tester le client contre un **vrai serveur HTTP** (un faux Jira contrôlé) plutôt que contre des mocks, sur plusieurs versions de Python, et automatiser toutes les tâches du projet derrière une seule interface : nox.

## Notions (cours)

### Les limites des mocks

respx vérifie que ton code envoie ce que **tu crois** que Jira attend. Si ta compréhension de l'API est fausse, le mock est faux aussi, et le test passe. Les tests d'intégration contre un serveur réel, même simulé, détectent d'autres classes de bugs : sérialisation réelle, en-têtes, encodage, comportement de la pile HTTP, timeouts réels.

### Le faux serveur ou le test contre le service réel

On ne lance pas les tests de PR contre Jira Cloud : c'est lent, non déterministe, soumis au rate limit, et ça exige des secrets. On construit un **fake** : un petit serveur FastAPI qui implémente le sous-ensemble d'endpoints utilisés, avec des données initiales fixes et des **scénarios de panne pilotables** (forcer un 503, un 429, de la latence). C'est une pratique courante : les *contract tests* sur le service réel, en fréquence réduite, vérifient que le fake ne diverge pas.

### Les fixtures de portée session et les services

Une fixture `session` démarre le fake une fois pour tous les tests d'intégration, soit en sous-processus uvicorn sur un port libre, soit en conteneur. Variantes outillées : `pytest-docker`, testcontainers-python.

### nox

nox est un automatiseur de tâches **en Python** (`noxfile.py`) : chaque session crée son environnement isolé et exécute des commandes, par exemple :

- `nox -s lint` ;
- `nox -s tests-3.13 tests-3.14 tests-3.15` ;
- `nox -s integration` ;
- `nox -s docs` ;
- `nox -s build`.

Il remplace les commandes à rallonge et unifie local et CI : la pipeline n'appelle plus que `nox -s <session>`. tox est l'équivalent historique, configuré en INI ou TOML. nox s'intègre avec uv (`nox.options.default_venv_backend = "uv"`).

### Supporter plusieurs versions de Python

Une bibliothèque déclare `requires-python = ">=3.13"`. Elle doit alors être **testée** sur chaque version supportée. Les classifiers PyPI reflètent la matrice testée.

### La version toute neuve

Quelques semaines après la sortie d'une version (3.15 ici), une dépendance qui contient une extension compilée (pydantic-core, par exemple) peut ne pas encore publier de wheel pour elle. L'installation tente alors de compiler depuis les sources, ou échoue. Deux réponses propres :

- marquer temporairement le job de la nouvelle version comme **autorisé à échouer** (informatif), avec une issue de suivi ;
- ou exclure la version de `requires-python` jusqu'à ce que l'écosystème suive.

Ce qu'on ne fait pas : retirer silencieusement la version de la matrice.

## Consigne

Créer le faux Jira (FastAPI), les tests d'intégration du client et de la CLI contre ce fake, un `noxfile.py` couvrant toutes les tâches du projet, et la matrice de versions 3.13, 3.14 et 3.15.

## Instructions

1. `fakejira/` : un projet à part, avec son propre `pyproject` minimal ou dans le groupe de dev (FastAPI + uvicorn) :
   - endpoints utilisés par sprintkit, en suivant la forme réelle des réponses (réutilise les fixtures JSON de l'étape 04) ;
   - pagination réelle ;
   - authentification Basic vérifiée (identifiants de test fixes) ;
   - endpoint d'administration `POST /_fake/scenario` pour injecter : les N prochaines requêtes en 503, un 429 avec `Retry-After`, de la latence ;
   - `Dockerfile` pour `fakejira`.
2. `tests/integration/conftest.py` : une fixture `session` qui démarre `fakejira`, en sous-processus uvicorn sur un port libre en local, ou à une URL fournie par la variable `FAKEJIRA_URL` (cas de la CI avec le conteneur). Attente active de disponibilité sur un endpoint de santé, avec un timeout.
3. Tests d'intégration (marqueur `integration`) :
   - recherche paginée sur 250 issues ;
   - retry réel sur un 503 injecté ;
   - respect d'un `Retry-After` réel (petite valeur) ;
   - identifiants faux → code 3 de la CLI ;
   - `worklog report` de bout en bout en JSON, comparé à un résultat attendu.
4. `noxfile.py`, avec les sessions :
   - `lint` (ruff, format, import-linter) ;
   - `typecheck` ;
   - `tests` (paramétrée en 3.13, 3.14 et 3.15, unitaires + couverture) ;
   - `integration` ;
   - `build` ;
   - `smoke` (l'installation de wheel de l'étape 07) ;
   - `docs` (pour l'étape 09) ;
   - `audit` (pour l'étape 10).

   Sessions par défaut : `lint`, `typecheck` et `tests`.
5. Classifiers `Programming Language :: Python :: 3.13`, `3.14` et `3.15`, alignés sur la matrice.
6. Un test de **contrat** (marqueur `contract`) : le même jeu de tests d'intégration, lancé contre la vraie instance Jira avec des identifiants réels. Il vérifie que le fake reste fidèle.

## Contraintes et spécificités

- Pas de port fixe pour le fake en local.
- Les tests d'intégration sont idempotents : chaque test réinitialise les scénarios via `/_fake/reset`.
- Pas de `sleep()` pour attendre le démarrage : utilise une attente active bornée.

## Résultat attendu

- [ ] `nox` (sessions par défaut) passe en local.
- [ ] `nox -s integration` démarre le fake, passe, puis l'arrête proprement (aucun processus résiduel).
- [ ] `nox -s tests-3.13 tests-3.14 tests-3.15` passe sur les trois versions.
- [ ] Le test de contrat passe contre ta vraie instance.
- [ ] Tu sais expliquer ce que les tests d'intégration attrapent et que respx ne peut pas attraper.

## Phase DevOps

**Attendu de l'intégrateur :** une pipeline complète, fondée **uniquement** sur des sessions nox, avec une matrice de versions et des services Docker.

1. **GitHub Actions :**
   - job `tests` avec `strategy.matrix.python: ["3.13", "3.14", "3.15"]` → `nox -s tests-${{ matrix.python }}` ;
   - si une dépendance bloque sur 3.15 : `continue-on-error` sur cette seule case de la matrice (via `matrix.include` avec une clé `experimental: true`), et issue de suivi liée dans un commentaire ;
   - job `integration` avec un bloc `services:` qui démarre l'image `fakejira` (construite dans un job précédent, ou depuis GHCR), et `FAKEJIRA_URL` en variable ;
   - job `contract` planifié chaque semaine et déclenchable à la main, dans l'environnement `jira-sandbox` (secrets).
2. **Jenkins :**
   - stages `parallel` pour 3.13, 3.14 et 3.15 (bloc `matrix` déclaratif, axe `PYTHON`) ;
   - même logique « autorisé à échouer » pour 3.15 si besoin : `catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE')` sur cette case uniquement ;
   - stage `Integration` : `docker.image('fakejira:...').withRun('-p 0:8000') { c -> ... }`, ou `docker compose up -d` avec `--wait`, puis `nox -s integration`, puis arrêt garanti dans `post { always { } }` ;
   - job hebdomadaire `contract`, avec `withCredentials` et `cron('H 6 * * 1')`.
3. **Jenkinsfile mince :** chaque stage n'appelle plus qu'une session nox. Si tu dois écrire de la logique dans le Jenkinsfile, elle a sa place dans le `noxfile`.
4. **Pipeline complète à ce stade :**
   ```
   Quality → Tests (3.13 ∥ 3.14 ∥ 3.15) → Build → Smoke → Integration (fakejira) → [hebdo] Contract
   ```

Critères :

- [ ] La matrice 3.13/3.14/3.15 tourne en parallèle sur les deux moteurs.
- [ ] Le conteneur `fakejira` est toujours arrêté, même si les tests échouent.
- [ ] Le job de contrat hebdomadaire est en place et vert.

## Pièges fréquents

- Un fake qui diverge silencieusement de la vraie API : c'est le rôle des tests de contrat.
- `docker compose up -d` suivi immédiatement des tests, sans attendre la disponibilité : échecs aléatoires.
- Des sessions nox qui réinstallent tout à chaque run. Utilise le backend uv et `--reuse-existing-virtualenvs` (`-R`) en local.
