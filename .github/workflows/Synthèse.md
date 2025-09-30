# Synthèse de la Pipeline d'Intégration et de Déploiement Continus (CI/CD)

Ce document formalise les étapes de la pipeline CI/CD configuré pour le projet BobApp.
La pipeline est définie dans le fichier `.github/workflows/ci.yml` et s'exécute automatiquement à chaque `push` ou `pull request` sur la branche `main`.

## 1. Workflow CI/CD Général

L'objectif de la pipeline est d'automatiser les tests, l'analyse de qualité du code avec Sonar et la publication des applications front-end et back-end sur docker hub. Elle est composée de plusieurs jobs qui s'exécutent selon un ordre logique pour garantir la qualité avant déploiement.

**Ordre d'exécution :**

1. **`backend-ci`** et **`frontend-ci`** : S'exécutent en parallèle pour compiler, tester et vérifier la couverture de code.
2. **`sonarcloud-analysis`** : Lance l'analyse de qualité centralisée pour les deux parties du projet (attend la fin des deux jobs précédents).
3. **`build-and-push-backend-on-dockerhub`** et **`build-and-push-frontend-on-dockerhub`** : Construisent et publient les images Docker uniquement si l'analyse qualité réussit.

---

## 2. Workflow des Tests

L'objectif de cette partie est de s'assurer que le code existant n'a pas de régression et que les nouvelles fonctionnalités sont fiables.

### Back-end (`backend-ci`)

1.  **`Checkout code`**: Récupère la dernière version du code source depuis le dépôt Git.
2.  **`Set up JDK 11`**: Configure l'environnement d'exécution avec Java version 11.
3.  **`Build, test and analyze with Maven`**:
    - **Objectif**: Compile le code, exécute tous les tests unitaires et lance l'analyse de qualité.
    - **Commande**: `mvn -B verify ...` exécute le cycle de vie Maven jusqu'à la phase `verify`. Cela inclut automatiquement l'exécution des tests (phase `test`) et la vérification de la couverture de code configurée dans le `pom.xml` (via le plugin JaCoCo).

### Front-end (`frontend-ci`)

1.  **`Checkout code`**: Récupère le code source.
2.  **`Set up Node.js`**: Configure l'environnement avec Node.js version 16.
3.  **`Install dependencies`**: Installe les dépendances du projet de manière reproductible avec `npm ci`.
4.  **`Run tests with coverage`**:
    - **Objectif**: Exécute les tests unitaires du front-end et génère un rapport de couverture de code.
    - **Commande**: `npm run test -- --code-coverage` lance les tests dans un mode non-interactif adapté à la CI.

---

## 3. Workflow de Qualité

L'objectif est de maintenir un haut niveau de qualité, de sécurité et de maintenabilité du code de manière automatisée.

### Back-end (`backend-ci`)

1.  **Vérification de la couverture de code (JaCoCo)**:
    - **Objectif**: Forcer un seuil minimum de couverture de code pour l'ensemble du projet back-end.
    - **Étape**: Intégrée à la commande Maven, cette vérification est configurée dans le `pom.xml` pour faire échouer le build si la couverture de lignes est inférieure à 30%.

### Front-end (`frontend-ci`)

1.  **`Enforce minimum coverage`**:
    - **Objectif**: Forcer un seuil minimum de couverture de code pour l'ensemble du projet front-end.
    - **Étape**: Ce script lit le fichier `coverage-summary.json` généré par les tests, en extrait le pourcentage de couverture de lignes et fait échouer le job si ce dernier est inférieur à 80%.

### Analyse Centralisée (`sonarcloud-analysis`)

**Ce job s'exécute uniquement si `backend-ci` ET `frontend-ci` ont réussi.**

1.  **Préparation de l'environnement**:

    - **Objectif**: Configurer Java et Node.js pour analyser les deux parties du projet.
    - **Étapes**: Installation de JDK 11 et Node.js 16.

2.  **Re-exécution des tests front-end**:

    - **Objectif**: Régénérer les rapports de couverture nécessaires à SonarCloud (les artefacts des jobs précédents ne sont pas accessibles).
    - **Étape**: `npm run test -- --code-coverage` dans le dossier front.

3.  **Compilation back-end**:

    - **Objectif**: Préparer les fichiers compilés nécessaires à l'analyse SonarCloud.
    - **Étape**: `mvn -B compile test-compile` dans le dossier back.

4.  **`SonarCloud Scan`**:
    - **Objectif**: Envoyer une analyse complète et centralisée du projet (front-end + back-end) à SonarCloud.
    - **Étape**: Utilise l'action `SonarSource/sonarqube-scan-action` avec un fichier de configuration global qui détecte automatiquement les deux parties du projet.

---

## 4. Workflow de Publication sur Docker Hub

L'objectif est de "packer" les applications dans des images Docker et de les rendre disponibles dans un registre central (Docker Hub) pour un déploiement futur.

### `build-and-push-backend-on-dockerhub`

**Ce job s'exécute uniquement si `sonarcloud-analysis` a réussi.**

1.  **`Log in to Docker Hub`**: Se connecte de manière sécurisée à Docker Hub en utilisant un nom d'utilisateur et un token stockés dans les secrets du dépôt GitHub.
2.  **`Build and push Docker image`**:
    - **Objectif**: Construire l'image Docker et la pousser sur Docker Hub.
    - **Étape**: Utilise l'action `docker/build-push-action` qui lit le `Dockerfile` situé dans le dossier `back/`, construit l'image, et la publie sur Docker Hub sous le nom `[votre_username]/bobapp-backend:latest`.

### `build-and-push-frontend-on-dockerhub`

**Ce job s'exécute uniquement si `sonarcloud-analysis` a réussi.**

1.  **`Log in to Docker Hub`**: Étape identique à celle du back-end.
2.  **`Build and push Docker image`**:
    - **Objectif**: Construire l'image Docker du front-end et la pousser sur Docker Hub.
    - **Étape**: Lit le `Dockerfile` situé dans le dossier `front/` et publie l'image sous le nom `[votre_username]/bobapp-frontend:latest`.

---

## 5. Indicateurs Clés de Performance (KPI) et Contrôles de Qualité

L'objectif de cette section est de définir les métriques de qualité utilisées pour garantir un niveau élevé de fiabilité et de maintenabilité du code.

### 5.1 KPI de Couverture de Code

#### Front-end

- **Métrique** : Couverture de lignes globale du projet
- **Seuil minimum** : 80%
- **Implémentation** : Script personnalisé dans l'étape `Enforce minimum coverage` du job `frontend-ci`
- **Mécanisme** :
  - Analyse du fichier `coverage-summary.json` généré par les tests
  - Extraction de la valeur `total.lines.pct` via l'outil `jq`
  - Échec du job si la couverture est inférieure au seuil défini

#### Back-end

- **Métrique** : Couverture de lignes globale du projet
- **Seuil minimum** : 30%
- **Implémentation** : Plugin JaCoCo configuré dans le fichier `pom.xml`
- **Mécanisme** :
  - Vérification automatique durant l'étape `Build, test and analyze with Maven`
  - Configuration dans la section `<execution id="jacoco-check">` du plugin
  - Échec du build Maven si la couverture est inférieure au seuil défini

### 5.2 Quality Gates SonarCloud

SonarCloud utilise des "Quality Gates" (portes de qualité) pour évaluer automatiquement la qualité du code à chaque analyse.

#### Principe de Fonctionnement

- **Évaluation** : Ensemble de conditions appliquées lors de l'analyse du code
- **Portée** : Conditions définies soit sur le nouveau code, soit sur le code global
- **Résultat** : Le code passe la Quality Gate avec succès ou échoue
- **Visibilité** : L'état apparaît dans la page "Aperçu du projet" et sur les Pull Requests GitHub

#### Quality Gate "Sonar Way" (Configuration par Défaut)

Cette Quality Gate est recommandée par Sonar et se concentre sur le maintien de standards élevés pour le **nouveau code** plutôt que sur la correction de l'ancien code.

**Les quatre conditions de la Quality Gate "Sonar Way" :**

1. **Aucun nouveau problème critique n'est introduit**

   - Objectif : Empêcher l'introduction de bugs ou vulnérabilités de haute sévérité
   - Portée : Nouveau code uniquement

2. **Tous les nouveaux Security Hotspots sont examinés**

   - Objectif : Garantir que tous les points sensibles de sécurité sont analysés
   - Portée : Nouveau code uniquement

3. **La couverture de tests du nouveau code ≥ 80%**

   - Objectif : Assurer un niveau de test suffisant pour les nouvelles fonctionnalités
   - Portée : Nouveau code uniquement

4. **La duplication dans le nouveau code ≤ 3%**
   - Objectif : Limiter la duplication de code pour maintenir la maintenabilité
   - Portée : Nouveau code uniquement

#### Intégration dans la Pipeline

- **Analyse centralisée** : Un job unique `sonarcloud-analysis` traite l'ensemble du projet (front-end + back-end)
- **Ordre d'exécution** : L'analyse SonarCloud s'exécute après la réussite des tests individuels
- **Déploiement conditionnel** : Les images Docker ne sont publiées que si la Quality Gate SonarCloud passe avec succès
- **Échec de la pipeline** : Si la Quality Gate échoue, le statut "Failed" est renvoyé à GitHub, empêchant la fusion du code et bloquant le déploiement

### 5.3 Stratégie Qualité Globale

La combinaison de ces KPI offre une approche à deux niveaux :

1. **Niveau projet global** : Maintien de la qualité globale via les seuils de couverture personnalisés
2. **Niveau incrémental** : Garantie de qualité pour chaque contribution via les Quality Gates SonarCloud

