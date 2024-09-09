# Catalogue de pipelines Gitlab

Ce dépôt à pour but de proposer des pipelines compatibles avec la plateforme Cloud π Native pour faciliter l'utilisation des services dépendants des CI/CD Gitlab.

## Contributions

Cf. [Conventions - MIOM Fabrique Numérique](https://docs.fabrique-numerique.fr/conventions/nommage.html).

Les commits doivent suivre la spécification des [Commits Conventionnels](https://www.conventionalcommits.org/en/v1.0.0/), il est possible d'ajouter l'[extension VSCode](https://github.com/vivaxy/vscode-conventional-commits) pour faciliter la création des commits.

Une PR doit être faite avec une branche à jour avec la branche `main` en rebase (et sans merge) avant demande de fusion, et la fusion doit être demandée dans `main`.

## Read Secret

L'etape ```read secret``` permet de récupérer les secrets interne lié au projet depuis le Vault. Cette étape est réalisée par l'import du fichier [vault-ci](./vault-ci.yml) et permet d'ajouter de générer les fichiers de configuration aux différents outils pour son build.

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

L'etape ```build app``` permet de construire l'artefact applicatif (hors image Docker) et est à adapter en fonciton de la technologie applicative. Cette étape peut egalement se faire lors de l'etape ```build-docker``` dans le cas d'un multi stage build docker.
L'équipe CPiN fourni quelques exemple de technologie notamment java par le fichier [java-mvn](./java-mvn.yml)

Ce fichier comporte deux jobs :
 - .java:build permettant de constuire un artefact par la commande ```mvn package```
 - .java:sonar permettant de lancer les tests unitaire et l'analyse qualimétrique

 ### Build java

Le build d'un artefact Java/Maven

L'étape ```build-java``` permet de construire un artefact java (jar) via le gestionnaire de cycle de vie Maven et la commande ```mvn clean package```
Il prend les paramètres suivants :
  - MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository" # a positionner à minima a cette valeur
  - MAVEN_CLI_OPTS: "" # goal maven supplémentaire ajoutés à ```clean package```
  - MVN_CONFIG_FILE: $MVN_CONFIG # Fichier de configuration Maven, ici on utilise la variable $MVN_CONFIG qui contient la configuration pour le projet.
  - BUILD_IMAGE_NAME: maven:3.8-openjdk-17 #Image utilisée pour builder l'artefact
  - WORKING_DIR: . # Répertoire de travail, la racine contenant le fihcier pom.xml du projet
  - ARTEFACT_DIR: ./target/*.jar # Répertoire contenant les artefacts générés par le build

 ### Sonar Java et Sonar Java jacoco

 2 jobs permettent une intégration sonarqube pour les projets Java :
  - .java:sonar-jacoco: Intégration maven / SonarQube avec l'executor Jacoco
  - .java:sonar: Intégration maven / SonarQube sans l'executor Jacoco

> A noter que les jobs sonar acceptent les erreurs et possède l'attribut ```allow_failure: true```

## Build docker

L'etape ```kaniko-build``` permet de construire une image Docker à partir d'un dockerfile et de l'envoyer sur Harbor. La construction d'image docker étant réalisé dans Docker sur Openshit, il est nécessaire d'utiliser Kaniko.

Ce fichier comporte deux jobs :
 - .kaniko:simple-build-push permettant de constuire une image Docker et de la pousser sur Harbor avec des paramètres standards
 - .kaniko:build-push permettant de constuire une image Docker et de la pousser sur Harbor avec plus de paramètres possible

### Simple-build-push

Ce job prend pour paramètres :
 - DOCKERFILE: Nom du dockerfile
 - WORKDIR : Répertoire de travail
 - IMAGE_NAME: Au format image:tag à pousser
 - EXTRA_BUILD_ARGS: paramètre supplémentaire à passer à la construction de l'image

Le dockerfile doit se trouver à l'emplacement ```$WORKDIR/$DOCKERFILE```

> IMAGE_NAME ne peut prendre qu'une seule valeur de couple image:tag

### Build-push

Ce job premet de gérer les cas où le dockerfile n'est pas à la racine du workdir, ainsi il prend les paramètres
 - DOCKERFILE: Chemin et nom du dockerfile, par exemple backend/Dockerfile
 - WORKDIR : Répertoire de travail
 - IMAGE_NAMES: Au format image:tag séparé par des espace, par défaut prend les valeurs ```$CI_PROJECT_NAME:$CI_COMMIT_BRANCH $CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA $CI_PROJECT_NAME:latest```
 - EXTRA_BUILD_ARGS: paramètre supplémentaire à passer à la construction de l'image

> IMAGE_NAME peut prendre plueiurs valeurs de couple image:tag séparées par des espaces

## Sonar

Ce job permet de lancer une analyse sonarqube du projet afin de faire une analyse statique du code pour les projets hors stack applicative Java.

paramètre :
 - SONAR_WAIT_QUALITY_GATE : permettant d'attendre le retour Sonar avant de valider l'étape de build.

## Mirror

Ce job est utilisé pour la CI interne du CPiN pour le projet de copie des repos externes / internes. Ce job n'est à priori pas à utiliser dans le fichier .gitlab-ci-dso.yml d'un projet applicatif.