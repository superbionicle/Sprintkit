# Étape 11 — Conteneurisation

> Durée indicative : 2 soirées. Prérequis : étape 10.

## Objectif

Distribuer sprintkit sous forme d'image Docker petite, sûre et reproductible. On peut ainsi l'utiliser dans une pipeline (par exemple un job Jenkins ou un workflow n8n) sans installer Python.

## Notions (cours)

### Multi-stage build

- Un stage `builder` (avec uv et les outils de build) construit la wheel ou installe dans un venv.
- Un stage `runtime` minimal ne reçoit **que** le venv ou les fichiers nécessaires.

Résultat : pas de compilateur ni de cache pip dans l'image finale. Elle est plus petite, avec moins de surface d'attaque.

### Le choix de l'image de base

- `python:3.14-slim` (Debian) : le bon compromis.
- `alpine` : petite, mais musl libc, avec des incompatibilités de wheels et des builds plus longs. À éviter en Python sauf raison précise.
- **distroless** (`gcr.io/distroless/python3`) : pas de shell, surface minimale, mais un débogage plus difficile.

On épingle la base par **digest** (`@sha256:...`) pour la reproductibilité, et Renovate la met à jour.

### Le cache de couches

Docker met en cache chaque instruction. On copie d'abord les fichiers de dépendances (`pyproject.toml`, `uv.lock`), on installe les dépendances, **puis** on copie le code. Changer le code ne réinstalle pas les dépendances. Le `--mount=type=cache` de BuildKit garde le cache d'uv entre builds.

### Les bonnes pratiques de sécurité

- Utilisateur non root (`USER`).
- Pas de secret dans l'image, ni dans `ENV`, ni dans une couche : `docker history` les révèle. Les secrets sont fournis au runtime (`-e`, `--env-file`, secrets de l'orchestrateur).
- `.dockerignore` (sans lui, `.git`, `.venv` et `.env` partent dans le contexte de build).
- Variables utiles : `PYTHONDONTWRITEBYTECODE=1` et `PYTHONUNBUFFERED=1` (des logs immédiats, importants en conteneur).

### ENTRYPOINT et CMD

`ENTRYPOINT ["sprintkit"]` + `CMD ["--help"]` : `docker run sprintkit worklog report ...` se comporte comme la CLI. En forme exec (tableau JSON), les signaux sont bien transmis au processus.

### Les labels OCI

`org.opencontainers.image.source`, `version`, `revision` et `licenses` relient l'image au dépôt et au commit. GHCR les utilise pour afficher la page du paquet.

## Consigne

Écrire un Dockerfile multi-stage optimisé pour sprintkit, le linter, scanner l'image et la publier en multi-arch.

## Instructions

1. `.dockerignore` strict : exclusion par défaut, puis réinclusion du nécessaire.
2. `Dockerfile` :
   - stage `builder` : image uv, `uv sync --frozen --no-dev --no-install-project` (dépendances seules), puis copie de `src/` et `uv sync --frozen --no-dev` (le projet) ;
   - stage `runtime` : `python:3.14-slim@sha256:...`, copie du venv depuis le builder, `PATH` vers le venv, utilisateur `sprintkit` non root, labels OCI (version et révision passées en `ARG`), `ENTRYPOINT`/`CMD`.

   L'extra `excel` est inclus dans l'image.
3. Lint avec hadolint, et corrige tout.
4. Mesure : taille de l'image (`docker image ls`) et nombre de couches. Objectif : moins de 150 Mo. Compare avec une version naïve mono-stage, et note les chiffres dans `docs/explications/docker.md`.
5. Test : `docker run --rm --env-file .env sprintkit:dev worklog report --week <semaine> --format json`.
6. Session nox `docker`, qui construit l'image et lance un smoke test (`--version`, `--help`).

## Contraintes et spécificités

- L'image ne contient ni uv, ni les tests, ni `.git`, ni de secret. Vérifie avec `docker run --rm --entrypoint sh` (si tu n'es pas en distroless) et `docker history`.
- L'image tourne en non-root : `docker run --rm --entrypoint id sprintkit:dev` ne doit pas afficher `uid=0`.
- Le build doit être reproductible : même commit et même base donnent les mêmes dépendances.

## Résultat attendu

- [ ] L'image fait moins de 150 Mo, tourne en non-root, et hadolint ne signale rien.
- [ ] Modifier uniquement le code ne reconstruit pas la couche des dépendances : vérifie dans la sortie du build.
- [ ] `docker run` avec `--env-file` produit un vrai rapport.
- [ ] Le comparatif naïf/optimisé est documenté.

## Phase DevOps

**Attendu de l'intégrateur :** une chaîne de build d'image complète (lint, build, scan, test, push), multi-arch et traçable.

1. **PR :** hadolint dans Quality ; build de l'image sans push (buildx, cache `type=gha`) ; smoke test ; scan Trivy (`--exit-code 1 --severity HIGH,CRITICAL --ignore-unfixed`).
2. **Merge sur `main` :** push multi-arch (`linux/amd64,linux/arm64`) sur GHCR, avec les tags `sha-<court>` et `main`.
3. **Jenkins :**
   - stage `Image` avec `docker buildx` (builder créé une fois sur l'agent) ;
   - push sur GHCR avec un credential token, via `withCredentials` + `docker login --password-stdin` ;
   - `docker logout` dans `post { always }`.
4. **Signature (optionnel, valorisant) :** `cosign sign` en mode keyless (OIDC GitHub) sur l'image publiée, et vérification documentée (`cosign verify`). C'est la provenance de l'artefact, un sujet supply chain.
5. **SBOM de l'image :** `syft` sur l'image, ou attestation `--sbom=true` avec buildx. À rapprocher du SBOM Python de l'étape 10.

Critères :

- [ ] Une vulnérabilité HIGH corrigeable dans l'image (par exemple une base ancienne) fait échouer la PR.
- [ ] L'image multi-arch est publiée sur GHCR, avec ses labels OCI visibles sur la page du paquet.
- [ ] Les identifiants du registre n'apparaissent jamais dans les logs.

## Pièges fréquents

- `COPY . .` en début de Dockerfile : tout le cache saute au moindre changement.
- Un `ENV JIRA_API_TOKEN=...` « pour tester » : le token est gravé dans une couche et publié avec l'image.
- La forme shell de l'ENTRYPOINT (`ENTRYPOINT sprintkit`) : le processus n'est pas PID 1, et les signaux d'arrêt ne lui parviennent pas.
