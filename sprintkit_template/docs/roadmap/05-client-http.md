# Étape 05 — Client HTTP robuste et secrets

> Durée indicative : 3 soirées. Prérequis : étape 04.

## Objectif

Écrire un client Jira Cloud de qualité production : authentification, pagination, timeouts, retries avec backoff, gestion du rate limiting et erreurs typées. Le tout est testé sans jamais toucher le vrai réseau.

## Notions (cours)

### httpx ou requests

Les deux sont des clients HTTP synchrones à l'API proche. httpx ajoute un mode `async`, le support HTTP/2, des timeouts explicites, et surtout un **transport injectable**, qui le rend très testable. On utilise un `httpx.Client` réutilisé (pool de connexions, keep-alive), jamais `httpx.get()` à chaque appel.

### Les timeouts

**Toujours** un timeout. Sans timeout, un serveur qui ne répond pas bloque ton process indéfiniment ; en CI, c'est un job qui pend pendant une heure. httpx distingue `connect`, `read`, `write` et `pool`.

### Les retries et le backoff

On réessaie uniquement ce qui est **transitoire** : erreurs de connexion, `502`, `503`, `504` et `429`. On ne réessaie pas un `400` (requête invalide) ni un `401` (token faux). Entre les essais, on attend un **backoff exponentiel avec jitter** (`base * 2**n + aléa`), pour ne pas marteler le serveur et pour désynchroniser les clients. Sur un `429`, l'en-tête `Retry-After` indique combien de temps attendre : on le respecte. On réessaie seulement les méthodes **idempotentes** (GET), ou celles qu'on sait sûres.

### La pagination

L'API Jira pagine ses résultats. Selon les endpoints, avec `startAt`/`maxResults`/`total`, ou avec un jeton `nextPageToken`. Le nouvel endpoint de recherche JQL de Jira Cloud (`/rest/api/3/search/jql`) utilise un jeton : vérifie dans la documentation Atlassian actuelle, car l'API a évolué. On expose la pagination sous forme de **générateur** (`Iterator[Issue]`), qui charge les pages à la demande. L'appelant ne voit pas la pagination, et la mémoire reste constante.

### L'authentification Jira Cloud

Basic Auth `email:api_token` sur HTTPS. Le token est un secret : il ne doit apparaître ni dans les logs, ni dans les messages d'erreur, ni dans `repr()` de tes objets. Utilise `pydantic.SecretStr` ou une classe dédiée.

### Tester un client HTTP

Trois techniques, de la plus isolée à la plus réaliste :

1. **Le transport mocké.** `respx` intercepte les appels httpx et renvoie des réponses définies dans le test. On vérifie aussi les **requêtes émises** : URL, paramètres, en-têtes, nombre d'appels. C'est pour les tests unitaires du client.
2. **Un faux serveur local.** C'est l'étape 08.
3. **La vraie API.** Quelques tests de recette manuels ou planifiés, jamais sur chaque PR.

### La gestion des secrets

- En local : variables d'environnement chargées depuis un `.env` ignoré par Git, ou le trousseau macOS via `keyring`.
- En CI : le gestionnaire de secrets du moteur (GitHub Secrets, credentials Jenkins avec `withCredentials`), avec masquage automatique dans les logs.
- **Prévention** : un scanner de secrets (gitleaks) bloque les commits qui contiennent un token.

## Consigne

Implémenter `sprintkit.jira.JiraClient`, qui implémente les ports de l'étape 04 :

- recherche JQL paginée ;
- worklogs d'une issue (paginés) ;
- sprints d'un board ;
- issues d'un sprint.

Le client gère les timeouts, les retries et le rate limiting, et produit des erreurs typées.

## Instructions

1. `uv add httpx`, `uv add --dev respx`.
2. `JiraClient.__init__(base_url, email, token, *, timeout, max_retries, transport=None)`. Le paramètre `transport` sert aux tests. Le client est utilisable en context manager (`with JiraClient(...) as c:`) pour fermer le pool proprement.
3. Une méthode privée `_request()` centralise :
   - les en-têtes (`Accept: application/json`, un `User-Agent` `sprintkit/<version>`) ;
   - le retry sélectif et le backoff avec jitter. L'attente passe par une fonction `sleep` injectable, pour que les tests soient instantanés ;
   - le mapping des statuts : `401`/`403` → `JiraAuthError`, `404` → `IssueNotFoundError` (ou équivalent), `429` épuisé → `JiraRateLimitError`, `5xx` épuisé → `JiraUnavailableError`.
4. Méthodes publiques en générateurs :
   - `search_issues(jql, fields)` ;
   - `iter_worklogs(issue_key, since)` ;
   - `iter_sprints(board_id, state)` ;
   - `iter_sprint_issues(sprint_id)`.

   Chacune convertit les DTO en domaine.
5. Tests avec respx :
   - pagination sur 3 pages → tous les éléments, 3 requêtes, avec le bon jeton ou le bon `startAt` à chaque fois ;
   - `503` puis `200` → succès après 1 retry ;
   - `429` avec `Retry-After: 2` → attente de 2 s (vérifiée sur la fonction `sleep` injectée) ;
   - `401` → `JiraAuthError` immédiatement, **sans retry** ;
   - timeout de connexion répété → `JiraUnavailableError` après `max_retries` ;
   - les en-têtes d'auth sont présents, et le token n'apparaît **pas** dans `str(exception)` ni dans `repr(client)`.
6. Un script de recette `scripts/smoke_real_jira.py`, non lancé par pytest, qui interroge ta vraie instance avec les variables d'environnement.

## Contraintes et spécificités

- Aucun test unitaire ne fait d'appel réseau réel. respx doit lever une erreur sur toute route non déclarée (`assert_all_mocked`).
- Le jitter utilise un `random.Random` injectable : les tests sont déterministes.
- Logs : une ligne DEBUG par requête (méthode, chemin, statut, durée), sans en-têtes ni corps (le logging arrive à l'étape 06 ; prévois l'appel `logger.debug`).

## Résultat attendu

- [ ] Tous les scénarios ci-dessus sont testés et passent en moins d'1 s au total.
- [ ] Le script de recette liste réellement les issues de ton instance.
- [ ] Couverture de branches ≥ 90 % sur `jira/`.
- [ ] Tu sais expliquer pourquoi on ne réessaie pas un `400`, et ce qu'apporte le jitter.

## Phase DevOps

**Attendu de l'intégrateur :** les secrets sont gérés correctement à toutes les étapes de la chaîne, et les fuites sont bloquées automatiquement.

1. **gitleaks :**
   - hook pre-commit ;
   - job CI (`gitleaks/gitleaks-action` sur GitHub ; binaire dans un conteneur sur Jenkins) qui scanne tout l'historique de la PR ;
   - test volontaire : commite un faux token au format Atlassian dans une branche jetable et vérifie le blocage. Ne le pousse pas sur un dépôt public.
2. **Secrets en CI :**
   - GitHub : un environnement `jira-sandbox` avec les secrets `JIRA_*`, restreint à la branche `main` et aux déclenchements manuels ;
   - Jenkins : `withCredentials([usernamePassword(credentialsId: 'jira-cloud', usernameVariable: 'JIRA_EMAIL', passwordVariable: 'JIRA_API_TOKEN')])`. Vérifie le masquage dans la console ; aucune commande ne doit faire `echo` du secret.
3. **Job de recette manuel :** `workflow_dispatch` sur GitHub, et un job Jenkins paramétré, qui lance `smoke_real_jira.py` contre la vraie instance. Il n'est jamais déclenché par une PR : une PR venant d'un fork ne doit pas accéder aux secrets.
4. **Protection GitHub :** active « Push protection » de secret scanning dans les paramètres du dépôt.

Critères :

- [ ] Un commit contenant un token est bloqué localement **et** en CI.
- [ ] Le job de recette passe, et le token est masqué (`****`) dans les logs.
- [ ] Les jobs de PR n'ont accès à aucun secret Jira.

## Pièges fréquents

- Le token loggé via une exception qui contient l'URL complète avec les identifiants, ou via un `print(client.__dict__)`.
- Réessayer un POST non idempotent : création en double.
- `time.sleep` en dur dans le client : les tests de retry prennent 30 s.
