# Étape 09 — Documentation

> Durée indicative : 2 soirées. Prérequis : étape 08.

## Objectif

Produire une documentation qui sert trois publics :

- l'utilisateur de la CLI ;
- le développeur qui utilise la bibliothèque ;
- le contributeur, c'est-à-dire toi dans six mois.

Elle est générée, vérifiée et publiée automatiquement.

## Notions (cours)

### Les quatre types de documentation (Diátaxis)

| Type | Orienté vers | Exemple |
|---|---|---|
| Tutoriels | l'apprentissage | « Ton premier rapport de worklog en 5 minutes » |
| Guides pratiques | une tâche | « Exporter les worklogs d'un sprint en Excel » |
| Référence | l'exhaustivité | Toutes les commandes, toutes les options, l'API Python, le schéma JSON |
| Explications | la compréhension | Architecture, choix de conception, gestion du rate limiting |

Mélanger les quatre dans un seul README donne une documentation que personne ne lit.

### Docstrings

On choisit un style unique (Google, ou NumPy) et on s'y tient. Un docstring d'API publique contient le rôle, les paramètres, ce qui est retourné, les exceptions levées et un exemple. ruff peut vérifier leur présence et leur forme (règles `D`, pydocstyle).

### Doctests

Les exemples dans les docstrings peuvent être **exécutés** comme des tests (`pytest --doctest-modules`). Une doc qui ment devient un test rouge.

### MkDocs

- **MkDocs + Material** : un site statique depuis du Markdown, avec recherche, thème soigné et navigation.
- **mkdocstrings** : génère la référence API depuis les docstrings et les annotations de type.

Sphinx est l'alternative historique (reStructuredText, très utilisé dans l'écosystème scientifique et par la doc de CPython).

### CHANGELOG

Au format « Keep a Changelog » : sections Added, Changed, Deprecated, Removed, Fixed et Security, par version. Il s'adresse aux **utilisateurs**, pas aux développeurs : ce n'est pas un `git log`.

### Le README

C'est ta vitrine. Il contient le pitch, une démo (GIF ou capture), l'installation en une ligne, trois exemples, les liens vers la doc et les badges.

## Consigne

Écrire les docstrings de toute l'API publique, mettre en place le site MkDocs (les 4 types de documentation), le CHANGELOG et le README final.

## Instructions

1. Active les règles ruff `D` avec la convention `google` sur `src/`. Rédige tous les docstrings publics.
2. Doctests sur au moins 3 fonctions du domaine (`IsoWeek.parse`, les conversions de durée…), lancés par la session `tests`.
3. `uv add --dev mkdocs-material "mkdocstrings[python]"`. Fichier `mkdocs.yml` :
   - nav : Accueil, Tutoriel, Guides, Référence (CLI, API Python, schéma JSON de sortie, codes de sortie, configuration), Explications (architecture, ADR), Changelog ;
   - option `strict: true`, pour qu'un lien cassé fasse échouer le build.
4. La référence CLI est **générée**, pas écrite à la main : Typer peut produire de la doc Markdown (`typer <module> utils docs`), ou tu écris un petit script. Elle est ainsi toujours synchronisée avec le code.
5. Schéma JSON des sorties `--format json`, généré depuis les modèles (pydantic `model_json_schema()`), et inclus dans la doc.
6. `CHANGELOG.md` de 0.1.0 jusqu'à l'état actuel.
7. README final : pitch, GIF de démo (asciinema + agg, ou vhs), `pipx install sprintkit` (ou `uv tool install sprintkit`), 3 exemples, badges (CI, couverture, version de Python, licence), lien vers la doc.
8. Session nox `docs` : `mkdocs build --strict`.

## Contraintes et spécificités

- Aucun élément public sans docstring : ruff le vérifie.
- La référence CLI et le schéma JSON ne sont jamais édités à la main.
- La doc se construit sans accès à Jira ni secret.

## Résultat attendu

- [ ] `nox -s docs` construit le site sans warning, en mode strict.
- [ ] Les doctests passent.
- [ ] Le site contient les 4 types de documentation et une référence API générée.
- [ ] Quelqu'un qui découvre le dépôt installe et lance un premier rapport à partir du seul tutoriel.

## Phase DevOps

**Attendu de l'intégrateur :** la documentation est un livrable vérifié et publié comme le code.

1. **Build de la doc sur chaque PR** (`nox -s docs`), avec le site construit en artefact pour pouvoir le relire.
2. **Publication sur GitHub Pages** depuis `main` (`actions/upload-pages-artifact` + `actions/deploy-pages`). Lien dans le README et dans les métadonnées `[project.urls]`.
3. **Doc versionnée (optionnel) :** `mike` publie une version de la doc par release (`latest`, `1.0`, `dev`). Pertinent dès que tu as plusieurs versions maintenues.
4. **Jenkins :** stage `Docs`, avec `publishHTML` (plugin HTML Publisher) pour consulter le site construit directement depuis le build.
5. **Contrôle du CHANGELOG :** un job vérifie qu'une PR modifiant `src/` modifie aussi `CHANGELOG.md` (section Unreleased), sauf si elle porte le label `skip-changelog`. Un petit script, ou une action existante, suffit.

Critères :

- [ ] Le site est en ligne et se met à jour à chaque merge.
- [ ] Une PR fonctionnelle sans entrée de CHANGELOG est signalée ou bloquée.
- [ ] Un lien cassé dans la doc fait échouer la PR.

## Pièges fréquents

- Une référence écrite à la main, qui diverge du code dès la release suivante.
- Un CHANGELOG écrit le jour de la release, de mémoire. Alimente la section Unreleased dans chaque PR.
