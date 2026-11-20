# Étape 03 — Tests unitaires avec pytest et TDD

> Durée indicative : 3 soirées. Prérequis : étape 02.

## Objectif

Maîtriser pytest en profondeur (organisation, fixtures, paramétrage, couverture, tests de propriétés) en développant en TDD la logique la plus « métier » du projet : l'agrégation des worklogs par semaine ISO.

## Notions (cours)

### Pourquoi pytest

Avec `unittest` (dans la bibliothèque standard), on écrit des classes et des `self.assertEqual`. Avec pytest, on écrit des fonctions `test_*` avec un simple `assert` : pytest réécrit l'assertion pour afficher un diff détaillé en cas d'échec. pytest exécute aussi les tests `unittest` existants.

### Découverte et organisation

pytest cherche les fichiers `test_*.py` et les fonctions `test_*`. Organisation conseillée :

- `tests/unit/` miroir de `src/sprintkit/` ;
- `tests/integration/` séparé ;
- `conftest.py` pour les fixtures partagées (pytest le charge automatiquement, il ne s'importe pas).

### Structure d'un test

Arrange / Act / Assert. Un test vérifie **un** comportement, et son nom le décrit : `test_entries_spanning_midnight_are_counted_on_start_day`.

### Les fixtures

Les fixtures (`@pytest.fixture`) sont l'injection de dépendances de pytest. Un test demande une ressource en la nommant en paramètre :

- elles peuvent dépendre d'autres fixtures ;
- elles ont une **portée** (`function` par défaut, `module`, `session`) ;
- elles gèrent le nettoyage avec `yield`.

Fixtures intégrées utiles : `tmp_path` (dossier temporaire), `monkeypatch` (modifier l'environnement ou un attribut le temps d'un test), `capsys` (capturer stdout/stderr), `caplog` (capturer les logs).

### Le paramétrage

`@pytest.mark.parametrize("raw, expected", [...], ids=[...])` exécute le même test sur une table de cas. Chaque cas est un test distinct dans le rapport.

### Les marqueurs

`@pytest.mark.integration`, `@pytest.mark.slow` : déclarés dans la configuration (`markers = [...]`), avec `--strict-markers` pour refuser les fautes de frappe. On sélectionne avec `-m "not integration"`.

### Les exceptions

`with pytest.raises(InvalidWeek, match="W54"):` vérifie qu'une exception est levée et que son message correspond.

### Les tests de propriétés (hypothesis)

Au lieu d'écrire des exemples, on énonce une **propriété** vraie pour toute entrée, par exemple « la somme des totaux par jour égale le total de la semaine ». hypothesis génère des centaines d'entrées, et **réduit** automatiquement un contre-exemple au cas minimal. C'est l'équivalent Python du fuzzing de skylink.

### La couverture

`pytest-cov` mesure les lignes et les **branches** exécutées (`--cov-branch`). Elle sert à trouver le code non testé, pas à prouver la qualité. Un seuil (`--cov-fail-under=90`) empêche la dérive.

### Le temps dans les tests

Une fonction qui appelle `datetime.now()` est non déterministe. On passe la date en paramètre, ou on injecte une horloge. Même principe que la `FakeClock` en C++.

### TDD

Red → Green → Refactor, voir le parcours C++, étape 02. En Python, la boucle est encore plus courte : `uv run pytest -x --lf` (s'arrête au premier échec, relance les derniers échoués). Tu peux aussi utiliser `pytest-watch` ou ton IDE.

## Consigne

En TDD, implémenter `sprintkit.reports.worklog` : à partir d'une liste de worklogs (auteur, issue, début, durée), produire un rapport par semaine ISO et par auteur, avec le total par jour et par issue.

## Instructions

1. `uv add --dev pytest pytest-cov hypothesis`. Configuration dans `[tool.pytest.ini_options]` :
   - `testpaths = ["tests"]` ;
   - `addopts = "-ra --strict-markers --strict-config"` ;
   - `markers` ;
   - `xfail_strict = true`.
2. Modèle temporaire minimal pour l'étape : une `@dataclass(frozen=True)` `Worklog`. Il sera retravaillé à l'étape 04.
3. Écris **d'abord** les tests (Red), commités séparément :
   - parsing d'une semaine `"2026-W47"` → (lundi, dimanche) ; cas invalides (`"2026-W54"`, `"2026-47"`, `""`) → exception explicite ;
   - semaine 1 d'une année dont le 1er janvier tombe un jeudi, un vendredi ou un dimanche (les cas limites de l'ISO 8601) ;
   - semaine 53 (2026 en a une : vérifie-le) ;
   - un worklog à cheval sur minuit est compté sur son jour de début (règle métier à documenter) ;
   - filtrage par auteur ; totaux par jour, par issue et pour la semaine ;
   - fuseaux horaires : un worklog à 23h30 UTC le dimanche appartient à la semaine suivante en heure de Paris. Le rapport prend un `tz` en paramètre ;
   - **propriété hypothesis** : pour toute liste de worklogs, la somme des totaux journaliers égale la somme des totaux par issue, qui égale le total de la semaine.
4. Implémente (Green) avec `datetime.date.fromisocalendar` / `isocalendar()` et `zoneinfo`.
5. Refactor : les fixtures partagées (une liste de worklogs réaliste) vont dans `tests/unit/conftest.py`.
6. Couverture : `uv run pytest --cov=sprintkit --cov-branch --cov-report=term-missing`. Lis les lignes manquantes et complète.

## Contraintes et spécificités

- Aucun appel à `datetime.now()` / `date.today()` dans `reports/`.
- Les durées sont des `timedelta`, jamais des secondes en `int` qui circulent.
- Aucun test ne dépend de l'ordre d'exécution. Vérifie avec `pytest -p randomly` (plugin `pytest-randomly`).

## Résultat attendu

- [ ] Au moins 20 tests (paramétrages compris), tous verts, y compris en ordre aléatoire.
- [ ] Couverture de branches ≥ 95 % sur `reports/`.
- [ ] Le test hypothesis tourne et passe. En cassant volontairement un total, hypothesis trouve et réduit un contre-exemple.
- [ ] L'historique Git montre les commits de tests avant ceux d'implémentation.
- [ ] Tu sais expliquer ce qu'est une fixture, sa portée, et la différence entre un exemple paramétré et une propriété hypothesis.

## Phase DevOps

**Attendu de l'intégrateur :** des résultats de tests et une couverture lisibles et bloquants, sans ouvrir les logs.

1. Commande de test unique, réutilisée partout :
   ```
   uv run pytest -m "not integration" --junitxml=reports/junit-unit.xml --cov=sprintkit --cov-branch --cov-report=xml:reports/coverage.xml --cov-report=term --cov-fail-under=90
   ```
   À l'étape 08, elle sera encapsulée dans nox.
2. **GitHub Actions :**
   - job `tests` ;
   - rapport JUnit affiché dans la PR ;
   - résumé de couverture dans `$GITHUB_STEP_SUMMARY` ;
   - `reports/` en artefact.
3. **Jenkins :**
   - stage `Unit tests` ;
   - `junit 'reports/junit-*.xml'` ;
   - `recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'reports/coverage.xml']], qualityGates: [[threshold: 90.0, metric: 'BRANCH', baseline: 'PROJECT']])` ;
   - les deux dans `post { always { } }`.
4. **Profil CI de hypothesis :** un profil `ci` avec plus d'exemples (`max_examples=500`), sélectionné par une variable d'environnement. La base d'exemples (`.hypothesis/`) est conservée en cache, pour que les contre-exemples trouvés soient rejoués.

Critères :

- [ ] Une PR qui fait passer la couverture sous 90 % échoue, avec la raison affichée.
- [ ] Le nom exact d'un test en échec est visible dans l'onglet Tests de Jenkins et dans la PR GitHub.

## Pièges fréquents

- Des tests qui passent seuls et échouent ensemble : un état global modifié (variable de module, variable d'environnement). Utilise `monkeypatch`, qui restaure automatiquement.
- Une fixture `session` qui renvoie un objet mutable modifié par un test : les autres tests en héritent.
- Tester le format de sortie de `print` au lieu de la valeur retournée. Sépare calcul et affichage.
