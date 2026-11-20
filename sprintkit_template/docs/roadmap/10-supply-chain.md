# Étape 10 — Sécurité de la chaîne d'approvisionnement

> Durée indicative : 2 soirées. Prérequis : étape 09.

## Objectif

Maîtriser ce qui entre dans ton logiciel : les dépendances et leurs vulnérabilités, leurs mises à jour et leurs licences. Savoir le prouver (SBOM). C'est un sujet de plus en plus incontournable, notamment en défense, et un vrai terrain DevSecOps.

## Notions (cours)

### La supply chain logicielle

Ton paquet dépend d'une poignée de bibliothèques directes, qui en tirent des dizaines d'autres (dépendances transitives). Chacune est un risque :

- vulnérabilité connue (CVE) ;
- paquet compromis : *typosquatting* (`reqeusts`), compte de mainteneur piraté ;
- licence incompatible avec ton usage ;
- projet abandonné.

### SCA (Software Composition Analysis)

C'est la comparaison de tes dépendances résolues avec les bases de vulnérabilités (PyPI Advisory Database, OSV). En Python, l'outil est **pip-audit**, maintenu par la PyPA. Il lit un environnement ou un fichier d'exigences exporté depuis le lockfile.

### Lockfile et hashes

`uv.lock` fige les versions **et** les hashes des artefacts. Une installation qui vérifie les hashes refuse un fichier modifié sur le dépôt de paquets : c'est l'intégrité.

### Mises à jour automatisées

Renovate ou Dependabot ouvrent des PR de mise à jour. Ta CI complète (tests, intégration) les valide, et tu fusionnes. Stratégie : regrouper les mises à jour mineures, traiter les majeures une par une, auto-merge éventuel pour les correctifs des dépendances de dev quand la CI est verte. C'est là que l'investissement des étapes précédentes paie : sans tests, tu ne peux pas mettre à jour sereinement.

### SAST

C'est l'analyse statique de sécurité de **ton** code. Les règles ruff `S` (issues de bandit) sont déjà actives depuis l'étape 02. On peut ajouter CodeQL (GitHub) ou Semgrep pour aller plus loin.

### SBOM (Software Bill of Materials)

L'inventaire formel des composants d'un livrable (nom, version, licence, hash), aux formats CycloneDX ou SPDX. Il est exigé par certaines réglementations et de plus en plus par les clients industriels. On le génère à chaque release et on le publie avec.

### Licences

Elles peuvent être permissives (MIT, BSD, Apache-2.0) ou copyleft (GPL, LGPL). Un outil comme `pip-licenses` les liste. Une politique « liste autorisée » se vérifie en CI.

## Consigne

Mettre en place l'audit de vulnérabilités, la mise à jour automatisée des dépendances, l'analyse de sécurité du code, le contrôle des licences et la génération d'un SBOM.

## Instructions

1. Session nox `audit` :
   - `uv export --frozen --no-hashes --format requirements-txt` (vérifie les options de ta version d'uv), puis `pip-audit -r <fichier> --strict` ;
   - en cas de vulnérabilité, l'échec affiche l'identifiant (CVE ou GHSA) et la version corrigée.
2. Exercice : installe temporairement une ancienne version vulnérable d'une dépendance (par exemple une vieille version de `jinja2`, tirée par mkdocs, ou une dépendance que tu choisis) dans une branche jetable. Observe le rapport, puis reviens en arrière.
3. `renovate.json` (application Renovate installée sur ton dépôt GitHub) :
   - gestion de `pyproject.toml`, de `uv.lock`, des actions GitHub (`github-actions`), du Dockerfile et des hooks pre-commit ;
   - regroupement des mises à jour mineures des dépendances de dev ;
   - `schedule` hebdomadaire.
4. CodeQL (workflow GitHub fourni par défaut pour Python) : activation et lecture des premiers résultats.
5. Licences : `pip-licenses` dans une session nox `licenses`, avec une liste autorisée (`MIT`, `BSD`, `Apache-2.0`, `ISC`, `PSF`, `MPL-2.0`) et un échec sur toute autre licence.
6. SBOM : `cyclonedx-py` (paquet `cyclonedx-bom`) sur l'environnement de prod, ce qui produit `sbom.cdx.json`. Vérifie qu'il ne contient pas les dépendances de dev.
7. `SECURITY.md` : comment signaler une vulnérabilité, et quelles versions sont supportées.

## Contraintes et spécificités

- Toute exception de vulnérabilité (`--ignore-vuln`) est accompagnée d'un commentaire : identifiant, raison (non exploitable dans ton usage) et date de revue.
- Les actions GitHub sont épinglées par **SHA de commit**, pas par tag (`actions/checkout@<sha> # v4.x`). Renovate les maintient à jour.

## Résultat attendu

- [ ] `nox -s audit` et `nox -s licenses` passent.
- [ ] Renovate a ouvert au moins une PR, que ta CI a validée.
- [ ] CodeQL est actif, et ses éventuelles alertes sont traitées ou justifiées.
- [ ] `sbom.cdx.json` est généré et contient uniquement les dépendances de prod.
- [ ] Tu sais expliquer ce qu'est une dépendance transitive, et pourquoi on épingle les actions par SHA.

## Phase DevOps

**Attendu de l'intégrateur :** la sécurité est un stage de la pipeline, avec des règles d'échec claires. C'est la partie DevSecOps.

1. **Stage `Security`** (en parallèle des tests) : `nox -s audit licenses` + gitleaks (étape 05).
2. **Audit nocturne sur `main`** : une CVE publiée **après** le dernier merge doit être détectée même sans nouveau commit. Jenkins : `cron('H 3 * * *')` ; GitHub : `schedule`. En cas d'échec, notification (e-mail, ou un bot Telegram via ton n8n).
3. **SARIF :** les résultats pip-audit ou CodeQL sont envoyés à l'onglet Security de GitHub (`upload-sarif`).
4. **Jenkins :** `recordIssues` avec un parser adapté, ou un rapport HTML publié ; badge ou résumé du nombre de vulnérabilités.
5. **Durcissement de la CI elle-même :**
   - `permissions:` minimales en tête de chaque workflow (`contents: read` par défaut) ;
   - pas de `pull_request_target` sur du code non fiable ;
   - credentials Jenkins limités au dossier ou au job qui les utilise.

Critères :

- [ ] Une dépendance vulnérable introduite volontairement fait échouer la PR au stage Security.
- [ ] L'audit nocturne tourne, et une notification arrive en cas d'échec.
- [ ] Tous les workflows GitHub déclarent leurs `permissions`.

## Pièges fréquents

- Ignorer une CVE « temporairement » sans date de revue : elle y est encore un an plus tard.
- Auditer l'environnement de dev (qui contient pytest, mkdocs…) et confondre les vulnérabilités de l'outillage avec celles du produit livré. Audite les deux, mais rapporte-les séparément.
