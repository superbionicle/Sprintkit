# Étape 12 — Release et publication

> Durée indicative : 3 soirées. Prérequis : étape 11, et l'étape 11 du parcours C++ si tu veux la shared library commune.

## Objectif

Publier sprintkit 1.0.0 : sur TestPyPI (puis PyPI si tu le souhaites), sur GHCR et sur GitHub Releases. Tout est déclenché par un tag Git, sans aucune action manuelle. Factoriser enfin les deux pipelines (C++ et Python) dans une shared library Jenkins commune.

## Notions (cours)

### SemVer pour une bibliothèque Python

Ton API publique comprend les fonctions et classes exportées, les commandes et options CLI, le schéma de sortie JSON et les codes de sortie. Toute rupture impose une version majeure. Avant 1.0, tout peut casser (0.x) ; la 1.0 est un **engagement de stabilité**. On déclare ce qui est public (`__all__`, documentation) et ce qui est interne (préfixe `_`).

### La version depuis Git

`hatch-vcs` (basé sur `setuptools-scm`) dérive la version du tag : `v1.0.0` donne `1.0.0`, et un commit après le tag donne `1.0.1.dev3+gabc1234`. La version n'est plus écrite **nulle part** dans les fichiers, ce qui rend toute divergence impossible.

### PyPI et TestPyPI

- TestPyPI est l'instance de bac à sable : on peut y publier sans conséquence.
- Un nom publié sur PyPI est **définitif** : une version publiée ne peut jamais être republiée, même supprimée.

Vérifie que le nom `sprintkit` est libre, sinon choisis-en un autre.

### Trusted Publishing (OIDC)

Au lieu d'un token PyPI stocké en secret, PyPI fait confiance à un workflow GitHub précis (dépôt, fichier de workflow, environnement) grâce à un jeton OIDC éphémère. Aucun secret longue durée n'existe. C'est la méthode recommandée par PyPI. Côté Jenkins, il n'y a pas d'OIDC natif vers PyPI : on utilise un token à portée limitée au projet.

### Attestations et provenance

Les attestations de build (PEP 740, générées par l'action de publication PyPA) prouvent qu'un artefact a été construit par tel workflow, à partir de tel commit.

### Le processus de release

1. Section Unreleased du CHANGELOG → `## [1.0.0] - <date>`.
2. PR de release, revue, merge.
3. Tag annoté `v1.0.0` sur le commit de merge.
4. La pipeline fait le reste.

On ne construit **jamais** une release sur son poste.

## Consigne

Brancher la version sur Git et automatiser la release complète déclenchée par tag : PyPI (Test puis réel), image Docker, GitHub Release avec CHANGELOG, SBOM et checksums. Factoriser les pipelines Jenkins C++ et Python dans une shared library commune et testée.

## Instructions

1. `hatch-vcs` : `dynamic = ["version"]`, `[tool.hatch.version] source = "vcs"`. `sprintkit --version` affiche la version dérivée. Vérifie le cas d'un build depuis la sdist, sans `.git` : hatch-vcs écrit la version dans les métadonnées de la sdist.
2. Configure le Trusted Publisher sur TestPyPI (et PyPI) pour ton dépôt, avec le workflow `release.yml` et l'environnement `pypi`.
3. Prépare le CHANGELOG 1.0.0, revois l'API publique (`__all__`), et passe le classifier en `Development Status :: 5 - Production/Stable`.
4. Release candidate d'abord : tag `v1.0.0rc1`, publication sur TestPyPI seulement, puis installation depuis TestPyPI dans un venv vierge (`--index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/`) et test d'un vrai rapport.
5. Tag `v1.0.0` → publication finale.
6. Annonce : la release GitHub contient l'extrait du CHANGELOG, la wheel, la sdist, `SHA256SUMS`, le SBOM, et la commande `docker pull`.

## Contraintes et spécificités

- La publication sur PyPI ne se fait qu'après la réussite de **toute** la pipeline sur le commit taggé (quality, tests multi-versions, intégration, sécurité, smoke).
- Les tags de pré-release (`rc`, `a`, `b`) ne publient que sur TestPyPI, et ne déplacent pas le tag Docker `latest`.
- Les artefacts publiés sont exactement ceux qui ont été testés : build **une** fois, test, puis promotion du même fichier. On ne reconstruit pas pour publier.

## Résultat attendu

- [ ] `pip install sprintkit==1.0.0` (ou depuis TestPyPI si tu ne publies pas sur PyPI) fonctionne dans un venv vierge.
- [ ] `docker pull ghcr.io/<toi>/sprintkit:1.0.0` fonctionne en amd64 et en arm64.
- [ ] La release GitHub contient tous les artefacts listés.
- [ ] Aucun token PyPI n'existe dans les secrets GitHub (Trusted Publishing).
- [ ] Tu sais expliquer pourquoi on ne reconstruit pas l'artefact entre le test et la publication.

## Phase DevOps

**Attendu de l'intégrateur :** une pipeline de livraison complète et sûre, et une CI Jenkins mutualisée entre les langages. C'est la synthèse des deux parcours.

1. **`release.yml`** sur `push: tags: ['v*']` :
   - job `verify` : réutilise le workflow CI via `workflow_call` (un **reusable workflow** : la CI est définie une fois et appelée par la PR et par la release) ;
   - job `build` : sdist + wheel une seule fois, en artefact ;
   - job `publish-testpypi` / `publish-pypi` : environnement protégé, `pypa/gh-action-pypi-publish` avec OIDC et attestations, condition sur le format du tag ;
   - job `image` : tags `1.0.0`, `1.0`, `1` et `latest` (sauf pré-release) ;
   - job `github-release` : notes extraites du CHANGELOG, artefacts, checksums, SBOM.
2. **Approbation manuelle :** l'environnement `pypi` exige une validation (« required reviewers ») avant publication. C'est une porte de promotion explicite.
3. **Shared library Jenkins commune** (le dépôt `jenkins-shared-lib` créé à l'étape 11 de skylink) :
   - `vars/pythonPipeline.groovy`, sur le modèle de `cppPipeline` : configuration par map (`pythonVersions`, `integrationService`, `coverageThreshold`, `publish`) ;
   - code commun extrait dans `src/` : `Stages.quality()`, `Reports.publishJUnit()`, `Docker.buildAndPush()`, `Release.isReleaseTag()`, `Notify.telegram()` (bot Telegram via ton n8n, optionnel). Les deux pipelines l'utilisent ;
   - tests JenkinsPipelineUnit pour les deux pipelines, et sa propre CI ;
   - release de la lib en `v2.0.0`, et les deux projets épinglés dessus.

   Le Jenkinsfile de sprintkit devient :
   ```groovy
   @Library('homelab-lib@v2.0.0') _
   pythonPipeline(pythonVersions: ['3.13', '3.14', '3.15'], integrationService: 'fakejira', coverageThreshold: 90)
   ```
4. **Stage de release Jenkins :** sur un build de tag (`buildingTag()`), `input` de validation, puis publication sur TestPyPI avec `withCredentials` (token à portée limitée au projet).
5. **`ci/README.md` final :** un schéma Mermaid de la pipeline complète, les déclencheurs, les environnements, les secrets (leur nom, pas leur valeur), et la procédure de release en 5 lignes.

Critères :

- [ ] `git push origin v1.0.0` produit, sans autre intervention qu'une approbation, la publication sur PyPI, GHCR et GitHub Releases.
- [ ] La définition de la CI GitHub n'existe qu'une fois (reusable workflow), appelée par la PR et par la release.
- [ ] Les Jenkinsfile de skylink et de sprintkit font chacun moins de 10 lignes, et reposent sur la même shared library testée et versionnée.

## Ce que tu peux mettre sur ton CV à l'issue du parcours

> **sprintkit**, bibliothèque et CLI Python (3.13+) pour l'API Jira Cloud, publiée sur PyPI : typage strict (mypy), architecture en couches contrôlée (import-linter), client HTTP résilient (retry/backoff, rate limiting), tests unitaires, de propriétés (hypothesis), d'intégration contre un faux serveur conteneurisé et de contrat. Pipeline GitHub Actions (reusable workflows, Trusted Publishing OIDC, SBOM, scan Trivy/pip-audit/CodeQL) et Jenkins (shared library Groovy testée, commune avec un projet C++).

Ne garde que ce que tu as réellement fait, et sois prêt à détailler chaque mot-clé en entretien.
