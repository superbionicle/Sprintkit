# Étape 07 — Interface en ligne de commande

> Durée indicative : 3 soirées. Prérequis : étape 06.

## Objectif

Construire une CLI agréable et **scriptable** (sous-commandes, formats de sortie, codes de retour), testée automatiquement. Vérifier aussi que le paquet **installé** fonctionne, pas seulement le code source.

## Notions (cours)

### argparse, click, typer

- argparse est dans la bibliothèque standard, verbeux mais sans dépendance.
- click est basé sur des décorateurs, mature.
- typer est construit sur click et utilise les **annotations de type** pour définir les options. C'est cohérent avec le typage strict du projet.

Le concept est le même partout : commandes, sous-commandes, arguments positionnels, options.

### Une CLI scriptable

- **Codes de sortie** : `0` succès, `1` erreur métier, `2` mauvaise utilisation (convention click/argparse), et éventuellement des codes dédiés (par exemple `3` authentification, `4` réseau), documentés. Les scripts et les pipelines s'appuient sur ces codes.
- **Formats** : `table` pour l'humain, `json`/`csv` pour les machines. Le JSON est stable : il fait partie de ton API publique.
- **Pas d'interaction** si stdin ou stdout n'est pas un terminal (`sys.stdout.isatty()`).
- Respecter `NO_COLOR`.

### Une couche CLI fine

La commande parse les arguments, construit les objets (configuration, client), appelle un **cas d'usage** et rend le résultat. Aucune logique métier dans la fonction de commande. C'est ce qui rend les cas d'usage testables sans la CLI, et la CLI testable avec des faux.

### Tester une CLI

`typer.testing.CliRunner` invoke l'app en mémoire, et donne la sortie et le code de retour. On injecte un **faux repository** (une implémentation en mémoire du `Protocol` de l'étape 04) à la place du vrai client. On teste aussi le binaire installé en sous-processus : c'est le smoke test.

### Le rendu

`rich` affiche de belles tables dans le terminal. On l'isole dans une couche `render`, en fonctions pures `render_table(report) -> rich.Table`, ou des fonctions qui produisent une chaîne, testables par comparaison.

## Consigne

Implémenter les commandes :

```
sprintkit config show [--validate]
sprintkit issues search "<JQL>" [--fields ...] [--limit N] [--format table|json|csv]
sprintkit worklog report --week 2026-W47 [--author ...] [--format table|json|csv|xlsx] [--output FILE]
sprintkit sprint list --board ID [--state active|closed|future]
sprintkit sprint status --board ID      # sprint actif : avancement par statut, temps passé
```

## Instructions

1. `uv add rich`. L'export xlsx utilise l'extra `excel` (openpyxl). Si l'extra n'est pas installé, un message clair indique `pip install sprintkit[excel]`.
2. `cli/` : une app racine et des sous-apps Typer (`issues`, `worklog`, `sprint`, `config`). Options globales : `-v`, `-q`, `--log-format` et `--config FILE`.
3. Une **fabrique** `build_context(settings) -> AppContext` crée le client et les repositories. Les tests la remplacent par des faux (via un paramètre, un `typer.Context.obj` ou un `monkeypatch` ciblé).
4. Les cas d'usage dans `sprintkit/usecases.py` (ou un paquet) : `search_issues`, `weekly_worklog_report` et `sprint_status`. Ils prennent des ports et retournent des objets du domaine.
5. Gestion d'erreurs centralisée : `SprintkitError` → message sur stderr + code dédié ; `KeyboardInterrupt` → code 130, sans trace.
6. Tests avec `CliRunner` et des faux repositories :
   - chaque commande dans chaque format ;
   - le JSON produit est valide et suit un schéma stable (vérifié par un snapshot, `tests/snapshots/*.json`, ou par `syrupy`) ;
   - `--week` invalide → code 2 et message utile ;
   - erreur d'authentification simulée → code 3, sans stack trace ;
   - le CSV est lisible par `csv.DictReader`, et le xlsx par openpyxl (dans `tmp_path`).
7. `--help` : chaque commande a une description et des exemples (`epilog`).

## Contraintes et spécificités

- Les fonctions de commande font moins de 15 lignes chacune : tout le reste vit dans les cas d'usage et le rendu.
- Le format JSON est documenté (étape 09). Le changer sera un changement **cassant** au sens de SemVer.
- Aucune couleur ni caractère de contrôle dans les sorties `json`/`csv`.

## Résultat attendu

- [ ] Toutes les commandes fonctionnent contre ta vraie instance Jira.
- [ ] `sprintkit worklog report --week <semaine courante> --format xlsx --output rapport.xlsx` produit un fichier ouvrable dans Excel.
- [ ] `sprintkit issues search "project = DEMO" --format json | jq length` fonctionne, même avec `-vv`.
- [ ] Tous les tests CLI passent sans réseau.
- [ ] Les codes de sortie sont documentés dans le README.

## Phase DevOps

**Attendu de l'intégrateur :** tester **l'artefact livré**, et pas seulement les sources.

1. **Smoke test de la wheel**, dans un stage dédié après `Build` :
   - création d'un venv vierge ;
   - installation de `dist/*.whl` (avec et sans l'extra `excel`) ;
   - `sprintkit --version`, `sprintkit --help`, et `sprintkit worklog report --help`, dont le code de retour vaut 0 ;
   - import du paquet et vérification de la présence de `py.typed`.

   Ça attrape les fichiers oubliés dans la wheel, les points d'entrée cassés et les dépendances manquantes dans `[project.dependencies]`.
2. **Ordre de la pipeline à ce stade :**
   ```
   Quality (lint, types, archi, secrets) → Unit tests → Build → Smoke test wheel
   ```
3. **Jenkins :** le stage Smoke se lance dans un conteneur `python:3.14-slim` **différent** de celui du build. Il n'a ni uv ni les sources : seulement la wheel, via `unstash`.

Critères :

- [ ] Retirer `rich` de `[project.dependencies]` (en le laissant en dev) fait échouer le smoke test, mais pas les tests unitaires. C'est exactement le cas que ce stage doit attraper.

## Pièges fréquents

- Tester la CLI uniquement avec `CliRunner` : le point d'entrée déclaré dans `pyproject.toml` n'est jamais vérifié.
- Mélanger logs et résultat sur stdout.
- Des imports lourds (openpyxl, rich) au niveau module : `sprintkit --help` met 1 s à répondre. Importe-les dans les fonctions qui en ont besoin, et mesure avec `python -X importtime`.
