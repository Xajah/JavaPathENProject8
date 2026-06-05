# TourGuide

![CI](https://github.com/Xajah/JavaPathENProject8/actions/workflows/build.yml/badge.svg)

Application Spring Boot de planification de voyages personnalisés, reprise et améliorée dans le cadre du **Projet 8 du parcours Java d'OpenClassrooms** (correction de bugs, optimisation des performances, mise en place d'un pipeline d'intégration continue).

---

## Contexte fonctionnel

TourGuide propose à ses utilisateurs des attractions touristiques à proximité, ainsi que des forfaits de voyage associés. Chaque action de l'utilisateur lui permet d'accumuler des points de fidélité auprès des partenaires.

Le projet original présentait trois problèmes que j'ai eu à traiter :
- des tests instables liés à des accès concurrents non protégés,
- un bug fonctionnel dans la recommandation d'attractions,
- des performances insuffisantes sur les opérations de masse (100 000 utilisateurs).

Ce dépôt contient la version corrigée.

---

## Architecture technique

- Java 17, Spring Boot 3.1.1, Maven
- Bibliothèques métier locales (non publiées sur Maven Central) : `gpsUtil`, `RewardCentral`, `TripPricer`, fournies dans `TourGuide/libs/`
- Tests : JUnit 5
- CI : GitHub Actions

Côté composants :

- `TourGuideController` expose les endpoints REST.
- `TourGuideService` orchestre la logique principale et délègue les appels lents à un pool de threads dédié pour la récupération des localisations (`gpsUtil`).
- `RewardsService` calcule les points de fidélité et utilise son propre pool de threads pour les appels à `RewardCentral`.
- Les collections internes de l'utilisateur (`visitedLocations`, `userRewards`) sont des `CopyOnWriteArrayList` pour rester sûres en accès concurrent.

---

## Prérequis

- JDK 17 ou supérieur
- Maven 3.9+

J'ai personnellement testé le build avec Eclipse Temurin 17 (utilisé par la CI) et Oracle JDK 23 (en local).

---

## Build et exécution

Toutes les commandes Maven se lancent depuis le dossier `TourGuide/`.

### Build complet (avec tests)

```bash
cd TourGuide
mvn clean package
```

Le JAR exécutable est généré dans `TourGuide/target/tourguide-0.0.1-SNAPSHOT.jar`.

### Lancement de l'application

```bash
java -jar TourGuide/target/tourguide-0.0.1-SNAPSHOT.jar
```

L'application est ensuite accessible sur `http://localhost:8080`.

### Endpoints REST principaux

| Endpoint | Description |
|---|---|
| `GET /getLocation?userName=<user>` | Dernière localisation connue d'un utilisateur |
| `GET /getNearbyAttractions?userName=<user>` | 5 attractions les plus proches, avec distance et points de récompense |
| `GET /getRewards?userName=<user>` | Récompenses accumulées par un utilisateur |
| `GET /getTripDeals?userName=<user>` | Forfaits voyages personnalisés |

Pour tester rapidement, des utilisateurs internes sont générés au démarrage : `internalUser0`, `internalUser1`, etc.

---

## Tests

```bash
cd TourGuide
mvn test
```

Suites présentes :

- `TestTourGuideService` — services principaux (3 tests)
- `TestRewardsService` — calcul des récompenses, dont `nearAllAttractions` (2 tests)
- `TestPerformance` — tests de charge à 100 000 utilisateurs (2 tests)
- `TourguideApplicationTests` — vérification du contexte Spring (1 test)

---

## Performances

Mesures obtenues sur ma machine de test (AMD Ryzen 9 7900X, 32 Go de RAM, JDK 23) :

| Test | Objectif OC | Mesure |
|---|---|---|
| `highVolumeTrackLocation` (100 000 users) | ≤ 15 min | 3 min 23 s |
| `highVolumeGetRewards` (100 000 users) | ≤ 20 min | 1 min 45 s |

La démarche complète (profiling VisualVM, choix de la parallélisation par `CompletableFuture`, tuning empirique des pools, sécurisation des collections concurrentes) est détaillée dans la documentation technique livrée avec le projet.

---

## Pipeline d'intégration continue

Workflow GitHub Actions (`.github/workflows/build.yml`) déclenché sur :
- push vers `dev` ou `master`,
- pull request vers `master`,
- déclenchement manuel via `workflow_dispatch`.

Étapes : checkout → setup Java 17 → installation des JARs locaux dans le dépôt Maven du runner → `mvn test` → `mvn package -DskipTests` → upload du JAR comme artefact téléchargeable (rétention 30 jours).

---

## Organisation du dépôt

Le projet Maven se trouve dans le sous-dossier `TourGuide/` (sources, tests et `pom.xml`). Les trois JARs locaux non publiés sur Maven Central sont conservés dans `TourGuide/libs/`. La configuration du pipeline est dans `.github/workflows/build.yml`.

---

## Auteur

Antoine Filho — Parcours Développeur d'application Java, OpenClassrooms.
