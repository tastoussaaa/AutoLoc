# AutoLoc : Couche Repository et analyse du code

## 1. Introduction

Après avoir créé les entités JPA et leurs associations dans les deux premiers ateliers, nous avons mis en place la couche Repository d'AutoLoc.

Cette couche permet d'accéder aux données stockées dans MySQL sans devoir écrire manuellement les requêtes SQL correspondant aux opérations CRUD les plus courantes.

Nous avons également utilisé SonarQube for IDE pour analyser le code et corriger les problèmes signalés.

## 2. Choix des interfaces Repository

Nous avons commencé avec `IContratRepository`, en utilisant `CrudRepository<Contrat, Long>` pour comprendre les opérations de base : enregistrement, recherche, consultation et suppression.

Nous sommes ensuite passés à `JpaRepository`, qui reprend ces fonctionnalités et fournit également le tri, la pagination et des opérations spécifiques à JPA.

Ce choix est adapté à AutoLoc, car nous aurons besoin de récupérer des listes, de rechercher des enregistrements et de gérer les données des différentes entités.

### Repositories créés

| Interface | Interface étendue | Justification |
|---|---|---|
| IAgenceRepository | JpaRepository<Agence, Long> | Gérer les agences et consulter leurs informations. |
| IEmployeRepository | JpaRepository<Employe, Long> | Enregistrer, rechercher et consulter les employés. |
| IVehiculeRepository | JpaRepository<Vehicule, Long> | Gérer le parc automobile, avec possibilité de tri et de pagination. |
| IEquipementRepository | JpaRepository<Equipement, Long> | Consulter et gérer les équipements disponibles. |
| IClientRepository | JpaRepository<Client, Long> | Enregistrer les clients et retrouver leurs informations. |
| IReservationRepository | JpaRepository<Reservation, Long> | Accéder aux réservations et à leurs données. |
| IContratRepository | JpaRepository<Contrat, Long> | Gérer les contrats avec les opérations CRUD et les méthodes JPA. |
| IPaiementRepository | JpaRepository<Paiement, Long> | Consulter les paiements, tout en privilégiant le contrat pour leur création et leur suppression. |
| IMaintenanceRepository | JpaRepository<Maintenance, Long> | Gérer les interventions de maintenance des véhicules. |

Toutes les interfaces utilisent `Long` comme type d'identifiant, conformément aux clés primaires de nos entités.

Spring Data JPA génère leurs implémentations au démarrage. Nous n'avons donc pas besoin d'écrire une classe d'implémentation pour chaque repository.

## 3. Compréhension des opérations CRUD

Les principales méthodes utilisées sont :

- `save()` : enregistrer une nouvelle entité ou mettre à jour une entité existante.
- `findById()` : rechercher une entité par son identifiant et retourner un `Optional`.
- `findAll()` : récupérer les enregistrements d'une entité.
- `deleteById()` : supprimer un enregistrement à partir de son identifiant.
- `count()` : connaître le nombre d'enregistrements.

La différence entre `CrudRepository` et `JpaRepository` apparaît notamment avec `findAll()` : la première interface renvoie un `Iterable`, tandis que la seconde permet de récupérer directement une `List`.

Nous devons également faire attention aux suppressions en lot proposées par `JpaRepository`, car elles peuvent contourner le mécanisme habituel des cascades JPA. C'est particulièrement important pour l'association `Contrat / Paiement`, où nous avons configuré `CascadeType.ALL` et `orphanRemoval = true`.

## 4. Analyse avec SonarQube for IDE

Nous avons installé SonarQube for IDE dans IntelliJ et lancé une analyse du projet.

Le premier rapport a signalé trois problèmes dans deux fichiers.

| Fichier | Problème détecté | Règle Sonar | Correction apportée |
|---|---|---|---|
| Agence.java, ligne 5 | Import dupliqué | java:S1128 | Suppression de l'import répété. |
| Agence.java, ligne 6 | Deuxième import dupliqué | java:S1128 | Suppression du second import répété. |
| pom.xml, ligne 2 | Caractères placés avant la déclaration XML | xml:S1778 | Correction du début du fichier XML. |

### Explication des corrections

**Imports dupliqués dans Agence.java**

Deux imports étaient déclarés plusieurs fois. Ces répétitions ne sont pas nécessaires au fonctionnement de la classe et rendent le code moins propre.

Nous avons conservé uniquement les imports utiles, sans modifier les attributs ni les associations JPA de l'entité.

**Déclaration XML dans pom.xml**

SonarQube a également signalé des caractères situés avant la déclaration XML du fichier Maven.

Nous avons corrigé le début du fichier afin de respecter la structure attendue d'un document XML.

### Résultat de la nouvelle analyse

Après les corrections, une nouvelle analyse a été lancée le 22 novembre 2026.

Le rapport affichait :

- No issues to display.
- No Security Hotspots to display.
- No Taint Vulnerabilities to display.

Les trois signalements précédents avaient donc disparu du rapport.

Nous avons constaté que l'analyse statique permet de détecter des défauts qui n'empêchent pas nécessairement le projet de compiler, mais qui peuvent nuire à la qualité du code.

## 5. Conclusion

Ce troisième atelier nous a permis de mettre en place la couche Repository du projet AutoLoc en utilisant Spring Data JPA.

Nous avons commencé par `CrudRepository` pour comprendre les méthodes CRUD, avant de retenir `JpaRepository` pour les neuf entités du projet.

L'analyse avec SonarQube for IDE nous a également permis d'identifier et de corriger des problèmes concrets dans notre code.

La couche Repository est maintenant structurée pour être utilisée par la couche Service, qui sera développée dans la suite du projet.