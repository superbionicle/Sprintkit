# Étape 06 — Configuration et logging

> Durée indicative : 2 soirées. Prérequis : étape 05.

## Objectif

Paramétrer l'outil sans toucher au code (fichier, environnement, arguments), avec une priorité claire. Rendre son comportement observable grâce à des logs utiles, aussi bien pour un humain que pour une machine.

## Notions (cours)

### Les sources de configuration et leur priorité

Ordre classique, du plus faible au plus fort :

1. valeurs par défaut dans le code ;
2. fichier de configuration utilisateur (`~/.config/sprintkit/config.toml`, en respectant XDG ; sous macOS, `~/Library/Application Support` est aussi possible, choisis et documente) ;
3. fichier projet (`./sprintkit.toml`) ;
4. variables d'environnement (`SPRINTKIT_*`, ou `JIRA_*` pour les identifiants) ;
5. arguments de ligne de commande.

C'est le principe des **12-factor apps** : la configuration qui varie entre environnements passe par l'environnement.

### TOML

`tomllib` est dans la bibliothèque standard depuis 3.11, en lecture seule. TOML est typé (entiers, booléens, dates, tables), contrairement à INI, et plus sûr que YAML pour de la configuration.

### Valider la configuration

`pydantic-settings` charge et **valide** la configuration depuis l'environnement et les fichiers, avec des messages d'erreur clairs (« `jira.timeout` doit être un nombre positif »). On valide **au démarrage** : mieux vaut échouer en 10 ms avec un message précis qu'au milieu d'un traitement.

### Le module `logging`

- **Loggers** hiérarchiques nommés par module : `logging.getLogger(__name__)` donne par exemple `sprintkit.jira.client`.
- **Niveaux** : DEBUG, INFO, WARNING, ERROR, CRITICAL.
- **Handlers** : où écrire (stderr, fichier). **Formatters** : sous quelle forme.

Règle d'or : **une bibliothèque ne configure jamais le logging**. Elle émet des messages, et ajoute au plus un `NullHandler`. C'est **l'application** (ici la CLI) qui configure, via `logging.config.dictConfig`, une seule fois au démarrage.

### stdout ou stderr

stdout porte le **résultat** (la table, le JSON, le CSV), qu'on peut rediriger ou piper. stderr porte les logs et les messages. Sans cette discipline, `sprintkit issues --format json | jq` casse dès qu'un log s'affiche.

### Les logs structurés

En plus d'un format texte lisible, un format JSON (une ligne = un objet) est exploitable par une machine (Loki, ELK, `jq`). On ajoute du contexte (`extra={"issue": key}`), et éventuellement un identifiant de corrélation par exécution.

### Ce qu'on ne logge pas

Les secrets, les données personnelles inutiles, et les corps de réponse entiers en INFO. Un **filtre** de logging peut masquer les motifs de token en dernier rempart.

## Consigne

Implémenter le chargement de configuration multi-sources avec validation, et une configuration de logging pilotée par la CLI (`-v`/`-vv`/`-q`, `--log-format text|json`).

## Instructions

1. `uv add pydantic-settings`. Classe `Settings` :
   - sous-sections `jira` (`base_url`, `email`, `token` en `SecretStr`, `timeout`, `max_retries`), `report` (`timezone`, `default_board`) et `output` (`format`) ;
   - sources : défauts < fichier utilisateur < fichier projet < environnement. Les arguments CLI s'appliquent à l'étape 07.
2. Une commande de diagnostic, préparée pour la CLI : `config show` affiche la configuration effective **et la source de chaque valeur**, avec le token masqué.
3. `sprintkit/logging.py` :
   - `configure_logging(verbosity: int, fmt: Literal["text", "json"])` ;
   - un formatter JSON maison (une vingtaine de lignes), ou une lib si tu préfères, à justifier ;
   - un filtre de masquage des tokens.
4. Le `NullHandler` sur le logger `sprintkit` dans `__init__.py`.
5. Tests :
   - priorité des sources (avec `monkeypatch.setenv` et `tmp_path` pour les fichiers) ;
   - configuration invalide → erreur claire, qui contient le nom du champ ;
   - `caplog` : le client de l'étape 05 émet bien une ligne DEBUG par requête, et le token n'apparaît dans aucun log, même en DEBUG ;
   - le format JSON produit un JSON valide par ligne.

## Contraintes et spécificités

- Aucun `print()` dans `src/` hors de la couche CLI de rendu.
- Aucun appel à `logging.basicConfig()` dans la bibliothèque.
- Le token n'est jamais affiché en clair, quel que soit le niveau de log ou la commande.

## Résultat attendu

- [ ] `config show` affiche la valeur effective et la source de chaque paramètre.
- [ ] `SPRINTKIT_JIRA__TIMEOUT=abc` → message d'erreur clair, code de sortie non nul, pas de stack trace.
- [ ] Les logs vont sur stderr, et la sortie JSON sur stdout reste valide avec `-vv`.
- [ ] Tu sais expliquer pourquoi une bibliothèque ne doit pas configurer le logging.

## Phase DevOps

**Attendu de l'intégrateur :** la configuration par environnement fonctionne en pipeline, et les logs produits en CI sont exploitables.

1. **Configuration par stage :** les variables non secrètes (`SPRINTKIT_REPORT__TIMEZONE`, `SPRINTKIT_LOG_FORMAT=json`) sont déclarées dans le bloc `environment { }` du Jenkinsfile et dans `env:` du workflow. Les secrets restent dans `withCredentials`.
2. **Logs JSON en artefact :** le job de recette (étape 05) écrit ses logs JSON dans un fichier archivé. Un petit script `jq` dans la pipeline en tire un résumé (nombre de requêtes, nombre de retries, durée max) affiché dans le résumé du job.
3. **Échec rapide :** un stage `Config check` (`sprintkit config show --validate`) avant le job de recette. Une mauvaise configuration échoue en quelques secondes, avec un message lisible.

Critères :

- [ ] Le job de recette produit un résumé des requêtes extrait des logs JSON.
- [ ] Une variable de configuration invalide fait échouer le stage `Config check` avec un message explicite.

## Pièges fréquents

- Créer des loggers avec un nom fixe (`getLogger("sprintkit")`) partout : on perd la granularité par module.
- Le formatter JSON qui plante sur un objet non sérialisable passé en `extra` : on perd le log au moment où on en a le plus besoin. Prévois un `default=str`.
