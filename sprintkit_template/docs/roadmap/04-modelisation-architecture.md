# Étape 04 — Modélisation et architecture

> Durée indicative : 2 soirées. Prérequis : étape 03.

## Objectif

Structurer le code en couches aux responsabilités claires, et modéliser les données de Jira avec des types solides. Le JSON brut de l'API ne doit pas se répandre dans tout le code.

## Notions (cours)

### Le problème du « dict partout »

Dans un script, on fait `r.json()["fields"]["customfield_10020"][0]["name"]` au milieu de la logique. Une faute de frappe ou un champ absent provoque une `KeyError` à l'exécution, loin de la cause. Et le moindre changement de l'API oblige à modifier tout le code.

### Architecture hexagonale (ports & adaptateurs), version légère

- **Domaine** (`domain/`) : les concepts métier (`Issue`, `Worklog`, `Sprint`, `WeekReport`) et les règles. Aucune I/O, aucun import de httpx ni de typer. C'est testable trivialement.
- **Ports** : des interfaces (`Protocol`) que le domaine ou les cas d'usage attendent, par exemple `class IssueRepository(Protocol): def search(self, jql: str) -> Iterator[Issue]: ...`.
- **Adaptateurs** (`jira/`, `cli/`, `reports/exporters`) : les implémentations concrètes. Le client Jira implémente le port ; la CLI appelle les cas d'usage.
- **Règle de dépendance** : les flèches pointent vers le domaine. `domain` n'importe jamais `jira` ni `cli`.

C'est le même principe que `ITransport` dans skylink : la logique ne connaît pas le réseau.

### dataclasses et pydantic

- `@dataclass(frozen=True, slots=True)` : léger, dans la bibliothèque standard, idéal pour le **domaine** (valeurs immuables, égalité structurelle).
- **pydantic** (v2) : validation et conversion à l'exécution depuis du JSON, avec des erreurs précises. Idéal à la **frontière** (désérialiser la réponse de l'API).

Pattern conseillé : des modèles pydantic « DTO » dans `jira/models.py`, qui reflètent l'API, et une fonction de conversion `to_domain()` vers les dataclasses du domaine. Le domaine reste indépendant de pydantic et de la forme de l'API.

### Hiérarchie d'exceptions

Une exception racine `SprintkitError`, avec des sous-classes métier (`InvalidWeekError`, `IssueNotFoundError`) et techniques (`JiraAuthError`, `JiraRateLimitError`, `JiraUnavailableError`). La CLI attrape `SprintkitError` pour afficher un message propre et un code de sortie, et laisse remonter le reste : un vrai bug doit produire une trace.

### Les types valeur

`IssueKey` (`"PROJ-123"`) validé à la construction plutôt qu'un `str` quelconque ; `IsoWeek(year, week)` plutôt qu'un tuple. Le typage et les tests attrapent ainsi plus d'erreurs.

## Consigne

Réorganiser le projet en couches, créer les modèles du domaine et les DTO pydantic de l'API Jira, et vérifier automatiquement que les dépendances entre couches respectent la règle.

## Instructions

1. `uv add pydantic`.
2. `domain/models.py` :
   - `IssueKey`, `IsoWeek` (déplace-y le parsing de l'étape 03), `Issue` (clé, résumé, statut, assigné, sprint courant éventuel), `Worklog`, `Sprint` (id, nom, état `Literal["future", "active", "closed"]`, dates) ;
   - tous en `frozen`.
3. `domain/errors.py` : la hiérarchie d'exceptions.
4. `domain/ports.py` : les `Protocol` `IssueRepository`, `WorklogRepository` et `SprintRepository`.
5. `jira/models.py` :
   - DTO pydantic pour les réponses JSON de recherche JQL, de worklog et de sprint ;
   - pars d'exemples réels : récupère 2 ou 3 réponses JSON de ton instance avec `curl`, anonymise-les et range-les dans `tests/fixtures/jira/*.json`.
6. Fonctions de conversion DTO → domaine, testées avec les fixtures JSON. Teste aussi les cas d'un champ facultatif absent, d'un assigné nul et d'un sprint absent.
7. Le module `reports/worklog.py` de l'étape 03 utilise désormais les types du domaine. Les tests existants sont adaptés, **pas** supprimés.
8. **Contrôle des dépendances :** `uv add --dev import-linter`, avec un contrat `layers` : `cli` → `reports` → `domain`, et `jira` → `domain`. `domain` n'importe rien du projet.

## Contraintes et spécificités

- `domain/` n'importe ni `httpx`, ni `pydantic`, ni `typer`.
- Le JSON brut (`dict[str, Any]`) ne sort jamais de `jira/`.
- Les fixtures JSON ne contiennent aucune donnée personnelle réelle ni URL d'instance.

## Résultat attendu

- [ ] `uv run lint-imports` passe, et échoue si tu ajoutes `import httpx` dans `domain/`.
- [ ] Les conversions DTO → domaine sont testées sur de vraies réponses anonymisées, y compris les champs manquants.
- [ ] Tous les tests de l'étape 03 sont toujours verts.
- [ ] `docs/adr/0001-architecture-en-couches.md` explique la décision en 15 lignes maximum.

## Phase DevOps

**Attendu de l'intégrateur :** les règles d'architecture deviennent des contrôles automatiques, pas des vœux pieux.

1. `lint-imports` ajouté au stage Quality (GitHub Actions et Jenkins).
2. **Revue assistée :** un fichier `CODEOWNERS` assigne automatiquement les PR touchant `domain/` (toi, ici, mais c'est la mécanique qui compte).
3. **Gouvernance des fixtures de test :** un hook pre-commit (script local) vérifie que `tests/fixtures/` ne contient ni `atlassian.net` ni adresses e-mail réelles.

Critères :

- [ ] Une PR qui viole l'architecture échoue au stage Quality, avec le contrat violé affiché.

## Pièges fréquents

- Réutiliser les modèles pydantic comme modèles du domaine « pour aller plus vite » : le domaine se met à épouser la forme de l'API.
- Une hiérarchie d'exceptions trop fine, avec une classe par message. Les sous-classes doivent correspondre à des **traitements différents** côté appelant.
