# Étape 02 — Qualité : lint, formatage, typage

> Durée indicative : 2 soirées. Prérequis : étape 01.

## Objectif

Installer les garde-fous automatiques qui attrapent une grande partie des erreurs avant même d'exécuter le code, et comprendre le typage statique en Python : c'est la plus grosse montée en gamme de ce parcours.

## Notions (cours)

### Formatage et lint

- Le **formateur** réécrit le code dans un style unique (espaces, guillemets, retours à la ligne). Il n'y a plus de débat en revue.
- Le **linter** détecte du code suspect : import inutilisé, variable non définie, `except:` nu, argument par défaut mutable, comparaison à `None` avec `==`, etc.

**ruff** fait les deux, et remplace flake8, isort, pyupgrade, une partie de pylint et bandit, ainsi que black. On choisit des familles de règles (`select`) :

| Code | Famille |
|---|---|
| `E`, `W` | pycodestyle |
| `F` | pyflakes, bugs évidents |
| `I` | tri des imports |
| `B` | bugbear, pièges courants |
| `UP` | modernisation de la syntaxe |
| `S` | sécurité |
| `SIM` | simplifications |
| `PT` | style pytest |
| `RUF` | règles propres à ruff |

### Le typage statique

Python reste dynamique à l'exécution : les annotations ne sont **pas vérifiées au runtime**. Un vérificateur statique (mypy, pyright) les lit et signale les incohérences sans exécuter le code. Bases à maîtriser :

- annotations de fonctions : `def week_total(entries: list[Worklog]) -> timedelta: ...` ;
- types optionnels : `str | None` (depuis 3.10, plus besoin de `Optional[str]`), le *narrowing* par `if x is None: return` ;
- génériques intégrés : `list[int]`, `dict[str, list[Issue]]`, `tuple[int, ...]` ;
- `typing` : `Protocol` (typage structurel, équivalent d'une interface implicite), `TypeAlias`, `Literal["csv", "json"]`, `Final`, `TypedDict` (pour les dicts JSON), `Self`, `@overload`, `TypeVar` / la syntaxe générique PEP 695 (`def first[T](xs: list[T]) -> T`) ;
- `Any` : l'échappatoire qui désactive la vérification. Elle est à proscrire hors des frontières (JSON brut entrant).

**Mode strict** : `mypy --strict` exige des annotations partout et interdit les `Any` implicites. Sur un projet neuf, on démarre strict, car il est très coûteux d'y venir plus tard.

### Pourquoi c'est rentable

Le typage documente le code, permet à l'IDE d'autocompléter correctement et trouve des bugs que seuls des tests très ciblés attraperaient : un `None` non géré, une clé mal orthographiée dans un `TypedDict`, une fonction appelée avec les arguments dans le mauvais ordre.

### pre-commit

Il exécute ruff (et d'autres hooks) automatiquement avant chaque `git commit`, sur les fichiers modifiés. La CI le relance sur tout le dépôt.

## Consigne

Configurer ruff (lint + format), mypy en mode strict et pre-commit, puis rendre le code existant conforme.

## Instructions

1. `uv add --dev ruff mypy pre-commit`.
2. Section `[tool.ruff]` dans `pyproject.toml` :
   - `line-length = 100`, `target-version = "py313"` (la plus ancienne version supportée) ;
   - `select = ["E", "W", "F", "I", "B", "UP", "S", "SIM", "PT", "RUF", "N", "C4", "PTH", "RET"]` ;
   - exceptions par chemin : autorise `S101` (`assert`) dans `tests/`.
3. Section `[tool.mypy]` : `strict = true`, `python_version = "3.13"`, `files = ["src", "tests"]`. Pour toute dépendance non typée, une surcharge `[[tool.mypy.overrides]]` ciblée, **justifiée** par un commentaire.
4. `.pre-commit-config.yaml` :
   - `ruff-pre-commit` (hooks `ruff` avec `--fix`, et `ruff-format`) ;
   - `pre-commit-hooks` (`check-toml`, `check-yaml`, `end-of-file-fixer`, `trailing-whitespace`, `check-added-large-files`) ;
   - mypy en hook `local`, via `uv run mypy`, pour qu'il voie les dépendances du projet.
5. `uv run pre-commit install`, puis `uv run pre-commit run --all-files`.
6. Exercice volontaire : écris une fonction `def parse_duration(s: str) -> int | None`, appelle-la sans gérer `None`, et observe l'erreur mypy. Corrige-la. Garde le résultat comme premier vrai code du domaine.

## Contraintes et spécificités

- Aucun `# type: ignore` sans code d'erreur (`# type: ignore[attr-defined]`) **et** sans commentaire.
- Aucun `# noqa` sans code de règle.
- La version de ruff du hook pre-commit est la même que celle du lockfile.

## Résultat attendu

- [ ] `uv run ruff check .`, `uv run ruff format --check .` et `uv run mypy` passent à 0 erreur.
- [ ] Un commit avec un import inutilisé est corrigé automatiquement par le hook.
- [ ] Tu sais expliquer pourquoi les annotations ne sont pas vérifiées au runtime, et ce que fait un `Protocol`.

## Phase DevOps

**Attendu de l'intégrateur :** une quality gate rapide, qui échoue avant les étapes coûteuses.

1. **GitHub Actions :**
   - job `quality` (ruff check avec `--output-format=github` pour annoter la PR, ruff format `--check`, mypy) ;
   - le job `build` le précède ou en dépend, selon ta logique : justifie ton choix dans `ci/README.md`.
2. **Jenkins :**
   - stage `Quality` avec des stages `parallel` `Lint` et `Typecheck` ;
   - plugin Warnings NG : `recordIssues(tools: [pyLint(pattern: 'ruff.txt'), myPy(pattern: 'mypy.txt')])`. Fais produire à ruff un format que le plugin comprend (`--output-format=pylint`) ;
   - quality gate à 0 issue, sinon échec.
3. **Cache :** les caches de mypy et de ruff (`.mypy_cache`, `.ruff_cache`) sont conservés entre builds, avec une clé basée sur le hash du lockfile.
4. **pre-commit en CI :** `uv run pre-commit run --all-files --show-diff-on-failure`. Le diff affiché en cas d'échec indique exactement quoi corriger.

Critères :

- [ ] Une PR avec une erreur de type est bloquée, et l'erreur apparaît ligne à ligne dans l'interface de la PR.
- [ ] Le stage Quality dure moins de 1 min.

## Pièges fréquents

- Activer `strict` puis ajouter des `Any` partout pour faire taire mypy. Le typage ne sert plus à rien.
- Faire tourner mypy dans pre-commit avec l'environnement isolé du hook, qui n'a pas les dépendances du projet : faux positifs « module not found ». Utilise le hook `local` avec `uv run`.
