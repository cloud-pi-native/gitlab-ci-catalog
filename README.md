# Catalogue de pipelines GitLab

Ce dépôt à pour but de proposer des pipelines compatibles avec la plateforme Cloud π Native (CPiN) pour faciliter l'utilisation des services dépendants des CI/CD GitLab.

## Contributions

Cf. [Conventions - MIOM Fabrique Numérique](https://docs.fabrique-numerique.fr/conventions/nommage.html).

Les commits doivent suivre la spécification des [Commits Conventionnels](https://www.conventionalcommits.org/en/v1.0.0/), il est possible d'ajouter l'[extension VSCode](https://github.com/vivaxy/vscode-conventional-commits) pour faciliter la création des commits.

Important : Toute demande de modification de ce dépôt doit se faire à l'aide de demande de fusion à partir d'une branche à jour avec la branche `main`. Les PR/MR sont intégrées uniquement en mode rebasage de `main` (pas de fusion/merge ni d'écrasement/squash merge).

## Read Secret

L'étape ```read secret``` permet de récupérer les secrets internes liés au projet depuis le Vault. Cette étape est réalisée par l'import du fichier [vault-ci](./vault-ci.yml) et permet d'ajouter de générer les fichiers de configuration aux différents outils pour son build.

Ce build est appelé depuis le fichier .gitlab-ci-dso.yml comme suit. Aucun paramètre ou variable supplémentaire n'est nécessaire à ce job :

```
# Import des catalog des jobs CPiN
include:
  - project: $CATALOG_PATH
    file:
      - vault-ci.yml
      [...]
    ref: main

[...]
stages:
  - read-secret

[...]

# Lecture des secrets CI du build
read_secret:
  stage: read-secret
  extends:
    - .vault:read_secret
```

## Build applicatif (optionnel)

L'étape `build app` permet de construire l'artefact applicatif (hors image Docker) et est à adapter en fonction de la technologie applicative. Cette étape est incluse dans l'étape `build-docker` dans le cas d'un multi-stage build docker.

L'équipe CPiN fourni quelques exemples de technologie, notamment java par le fichier [java-mvn](./java-mvn.yml).
Ce fichier comporte deux jobs :
 - .java:build permettant de construire un artefact par la commande `mvn package`
 - .java:sonar permettant de lancer les tests unitaire et l'analyse de la qualimétrie

### Build java

Le build d'un artefact Java/Maven

L'étape `build-java` permet de construire un artefact java (jar) via le gestionnaire de cycle de vie Maven et la commande `mvn clean package`.

Il prend les paramètres suivants :
  - `MAVEN\_OPTS` : `-Dmaven.repo.local=$CI\_PROJECT\_DIR/.m2/repository`. À positionner a minima à cette valeur
  - `MAVEN\_CLI\_OPTS` : `""`. Options Maven supplémentaires ajoutées à `clean package`
  - `MVN\_CONFIG\_FILE` : `$MVN\_CONFIG`.Fichier de configuration Maven, ici on utilise la variable `$MVN\_CONFIG` qui contient la configuration pour le projet.
  - `BUILD\_IMAGE\_NAME` : `maven:3.8-openjdk-17`. Image utilisée pour construire l'artefact
  - `WORKING\_DIR` : `.`. Répertoire de travail, la racine contenant le fichier pom.xml du projet
  - `ARTEFACT\_DIR` : `./target/*.jar`. Répertoire contenant les artefacts générés par le build

 ### Sonar Java et Sonar Java JaCoCo

 2 jobs permettent une intégration SonarQube pour les projets Java :
  - `.java:sonar-jacoco` : Intégration Maven / SonarQube avec l'executor JaCoCo
  - `.java:sonar` : Intégration Maven / SonarQube sans l'executor JaCoCo

> À noter que les jobs SonarQube acceptent les erreurs et possède l'attribut `allow_failure: true`

### Build Node.js

Le fichier `node-npm.yml` contient le job suivant : `.npm:build-publish`.
Cette étape va construire l'application/paquet Node.js avec NPM. Les paquets sont récupérés depuis internet, et depuis le Nexus DSO pour vos paquets privés. Si le projet est un paquet, il est automatiquement poussé vers Nexus.

> NOTE: Il n'y a pas, actuellement, de proxy Nexus-NPM automagique

Vous pouvez avoir par exemple un premier dépôt qui construit `@mon-groupe/mon-paquet` et le publie sur Nexus.
Ensuite dans un deuxième dépôt, vous voulez utiliser ce paquet en dépendance.

Il faut dire à NPM de récupérer ce paquet non pas depuis internet, mais depuis le Nexus de votre projet. Pour cela, configurez la variable `PRIVATE_GROUPS` avec vos groupes séparés par des espaces (en incluant l'arobase `@`).
Normalement, la CI devrait détecter automatiquement les dépendances privées si vous ne travaillez pas uniquement sur le GitLab DSO. En revanche elle n'arrive pas à détecter les dépendances transitives.

Paramètres :
  - `WORKING_DIR: .` : Répertoire de travail, contenant le fichier `package.json` du projet
  - `AUTODETECT_PRIVATE_GROUP: 1` : Mettre à `0` pour désactiver la détection automatique des paquets privés
  - `ARTEFACT_DIR: out` : Le dossier contenant les fichiers générés par votre build NPM
  - `BUILD_IMAGE_NAME: node:lts-bullseye` : Image utilisée pour construire l'artefact

## Build docker

L'étape `kaniko-build` permet de construire une image Docker à partir d'un Dockerfile et de l'envoyer sur Harbor. La construction d'image docker étant réalisé dans Docker sur OpenShift, il est nécessaire d'utiliser Kaniko.

Ce fichier comporte deux jobs :
 - `.kaniko:simple-build-push` : Permet de construire une image Docker et de la pousser sur Harbor avec des paramètres standards
 - `.kaniko:build-push` : Permet de construire une image Docker et de la pousser sur Harbor avec plus de paramètres possible

### Simple-build-push

Ce job prend pour paramètres :
 - `DOCKERFILE` : Nom du Dockerfile
 - `WORKDIR` : Répertoire de travail
 - `IMAGE_NAME` : Au format `image:tag` à pousser
 - `EXTRA_BUILD_ARGS` : Paramètre supplémentaire à passer à la construction de l'image

Le Dockerfile doit se trouver à l'emplacement `$WORKDIR/$DOCKERFILE`.

> `IMAGE_NAME` ne peut prendre qu'une seule valeur de couple `image:tag`.

### Build-push

Ce job permet de gérer les cas où le Dockerfile n'est pas à la racine du workdir, ainsi il prend les paramètres
 - `DOCKERFILE` : Chemin et nom du Dockerfile, par exemple `backend/Dockerfile`
 - `WORKDIR` : Répertoire de travail
 - `IMAGE_NAMES` : Au format `image:tag` séparé par des espaces, par défaut prend les valeurs `$CI_PROJECT_NAME:$CI_COMMIT_BRANCH $CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA $CI_PROJECT_NAME:latest`
 - `EXTRA_BUILD_ARGS` : Paramètre supplémentaire à passer à la construction de l'image

> `IMAGE_NAME` peut prendre plusieurs valeurs de couple `image:tag` séparées par des espaces.

## Helm Build

Le fichier `helm-ci.yml` contient le job `.helm:build-push` permettant de packager et pousser des charts Helm vers Harbor en tant qu'artefacts OCI.

### Fonctionnement

Ce job détecte automatiquement les charts modifiés en comparant les commits Git :

- Si aucun commit précédent n'est trouvé (`CI_COMMIT_BEFORE_SHA` absent ou nul), tous les charts sont empaquetés
- Sinon, seuls les charts modifiés (dans le répertoire `charts/`) sont empaquetés

### Paramètres

| Variable           | Description                                    | Valeur par défaut         |
| ------------------ | ---------------------------------------------- | ------------------------- |
| `CHARTS_DIR`       | Répertoire contenant les charts Helm           | `./charts`                |
| `IMAGE_REPOSITORY` | URL du registre OCI (Harbor)                   | Fourni par `vault-ci.yml` |
| `DOCKER_AUTH`      | Configuration d'authentification Docker (JSON) | Fourni par `vault-ci.yml` |

> Les variables `IMAGE_REPOSITORY` et `DOCKER_AUTH` sont automatiquement récupérées depuis le Vault par le job `read_secret` de `vault-ci.yml`.

### Prérequis

- Les charts doivent être placés dans le répertoire défini par `CHARTS_DIR` (par défaut `./charts`)
- Chaque chart doit contenir un fichier `Chart.yaml` valide avec les champs `name` et `version`
- La variable `DOCKER_AUTH` doit contenir les credentials pour Harbor

### Exemple d'utilisation

```yaml
include:
  - project: $CATALOG_PATH
    file:
      - vault-ci.yml
      - helm-build.yml
    ref: main

stages:
  - read-secret
  - build

helm_build:
  stage: build
  extends:
    - .helm:build-push
  variables:
    CHARTS_DIR: ./charts
    IMAGE_REPOSITORY: harbor.example.com/my-project/helm
  needs:
    - read_secret
```

> **Note** : Le job gère automatiquement les dépendances des charts si un fichier `Chart.lock` existe ou si des dépendances sont définies dans `Chart.yaml`.

## Sonar

Ce job permet de lancer une analyse SonarQube du projet afin de faire une analyse statique du code pour les projets hors stack applicative Java.

Paramètres :
 - `SONAR_WAIT_QUALITY_GATE` : permettant d'attendre le retour Sonar avant de valider l'étape de build.

## Mirror

Ce job est utilisé pour la CI interne du CPiN pour le projet de copie des repos externes / internes.
Ce job n'est a priori pas à utiliser dans le fichier `.gitlab-ci-dso.yml` d'un projet applicatif.
