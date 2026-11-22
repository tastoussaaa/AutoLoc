# AutoLoc : Atelier 4
## Inversion de contrôle, injection de dépendances et couche Service

### A. Comparaison des modes d'injection

Dans cet atelier, nous avons comparé l'injection par constructeur et l'injection par attribut en utilisant `ContratServiceImpl`.

| Critère | Injection par constructeur | Injection par attribut |
|---|---|---|
| Champ `final` possible | Oui | Non |
| Dépendance explicite dans le constructeur | Oui | Non |
| Utilisation hors du conteneur Spring | Possible en fournissant le repository au constructeur | Le repository reste `null` sans injection |
| Dépendance obligatoire à la construction | Oui | Non |
| Signalement SonarQube observé | Aucun pour ce mode | `java:S6813` |
| Choix pour AutoLoc | Retenu | Non retenu |

Nous avons temporairement remplacé l'injection par constructeur par `@Autowired` sur un attribut. SonarQube for IDE a signalé la règle `java:S6813`, qui déconseille l'injection par attribut.

Nous avons ensuite restauré `@RequiredArgsConstructor` avec un champ `final`.

Cette approche permet de rendre les dépendances explicites et de construire les services avec les repositories nécessaires dès leur création.

### B. Les Services et leurs dépendances

Chaque Service est composé d'une interface `I...Service` et d'une classe `...ServiceImpl`.

Les implémentations utilisent `@Service` pour être détectées par Spring et `@RequiredArgsConstructor` pour bénéficier de l'injection par constructeur.

| Service | Dépendance injectée | Mode d'injection | Justification |
|---|---|---|---|
| ContratServiceImpl | IContratRepository | Constructeur | Gérer les contrats et appliquer les règles métier. |
| AgenceServiceImpl | IAgenceRepository | Constructeur | Gérer les agences et refuser un nom vide. |
| EmployeServiceImpl | IEmployeRepository | Constructeur | Gérer les employés et vérifier leur nom et leur rôle. |
| VehiculeServiceImpl | IVehiculeRepository | Constructeur | Gérer les véhicules et refuser un tarif journalier négatif. |
| EquipementServiceImpl | IEquipementRepository | Constructeur | Gérer les équipements et vérifier leur libellé. |
| ClientServiceImpl | IClientRepository | Constructeur | Gérer les clients et vérifier les informations obligatoires. |
| ReservationServiceImpl | IReservationRepository | Constructeur | Gérer les réservations et contrôler la cohérence des dates. |
| PaiementServiceImpl | IPaiementRepository | Constructeur | Consulter les paiements uniquement. |
| MaintenanceServiceImpl | IMaintenanceRepository | Constructeur | Gérer les maintenances et contrôler la cohérence des dates. |

Tous les services dépendent d'une interface Repository plutôt que d'une implémentation concrète.

Nous n'avons pas besoin de créer manuellement ces dépendances avec `new`. Spring les fournit automatiquement.

Les services restent également sans état propre à une requête, car ils sont gérés par défaut comme des beans singleton.

### C. Diagnostic des erreurs du conteneur Spring

#### Message 1 : Bean introuvable

**Exception :** `NoSuchBeanDefinitionException`, éventuellement accompagnée de `UnsatisfiedDependencyException`.

**Bean concerné :** `IContratService`, demandé par `ContratController`.

**Causes probables :**

- L'annotation `@Service` est absente de `ContratServiceImpl`.
- La classe est placée en dehors du package analysé par Spring.

**Corrections :**

Ajouter `@Service` à l'implémentation et vérifier qu'elle appartient au package `tn.esprit.autoloc.service`.

#### Message 2 : Plusieurs beans disponibles

**Exception :** `NoUniqueBeanDefinitionException`.

**Bean concerné :** La dépendance `INotificateur` demandée par `NotificationController`.

**Cause :**

Spring trouve deux implémentations : `emailNotificateur` et `smsNotificateur`, alors qu'une seule est attendue.

**Corrections possibles :**

- Ajouter `@Primary` sur la classe de l'implémentation choisie par défaut.
- Ajouter `@Qualifier("smsNotificateur")` sur le paramètre du constructeur pour sélectionner explicitement cette implémentation.

`@Primary` établit une préférence générale, tandis que `@Qualifier` effectue un choix précis au point d'injection.

#### Message 3 : Dépendance circulaire

**Erreur :** Cycle de dépendances entre `ClientServiceImpl` et `ReservationServiceImpl`.

**Cause :**

Chaque service dépend de l'autre. Avec l'injection par constructeur, chacun doit recevoir sa dépendance avant de pouvoir être instancié.

Spring ne peut donc pas terminer normalement la construction des deux beans.

**Correction :**

Revoir la répartition des responsabilités pour supprimer la dépendance mutuelle. Une solution consiste à extraire la logique commune dans un troisième service indépendant.

### D. Analyse SonarQube for IDE

Nous avons réalisé des expériences d'analyse statique dans `ContratServiceImpl`.

#### Anomalie 1 : Injection par attribut

**Fichier :** `ContratServiceImpl.java`

**Règle détectée :** `java:S6813`

**Problème :** Utilisation de `@Autowired` directement sur un attribut.

**Explication :**

Cette méthode masque les dépendances obligatoires et ne permet pas de déclarer le champ injecté comme `final`.

**Correction :**

Nous avons restauré l'injection par constructeur avec `@RequiredArgsConstructor` et un attribut `final` de type `IContratRepository`.

#### Anomalie 2 : Import inutilisé

Règle SonarQube : java:S1128

Problème : Un import java.time.LocalDate était présent
sans être utilisé dans ContratServiceImpl.

Correction : Suppression de l'import inutile.

#### Anomalie 3 : Utilisation de System.out.println()

Règle SonarQube : java:S106

Problème : Une instruction System.out.println()
était utilisée pour afficher un message dans le service.

Correction : Suppression de cette instruction,
qui n'était pas nécessaire au fonctionnement du service.

### E. Questions de compréhension

#### 1. Où est le `new` de `ContratServiceImpl` et qui l'exécute ?

Nous n'écrivons pas explicitement `new ContratServiceImpl(...)` dans notre projet.

Spring crée automatiquement le bean grâce à son conteneur IoC. Il utilise le constructeur généré par Lombok et fournit le repository nécessaire.

C'est le principe de l'inversion de contrôle : la gestion des objets est confiée au framework.

#### 2. Pourquoi un contrôleur dépendra-t-il de `IContratService` et non de `ContratServiceImpl` ?

Le contrôleur doit dépendre d'une abstraction plutôt que d'une classe concrète.

Cela réduit le couplage entre les couches et permet de remplacer l'implémentation du service sans modifier le contrôleur, par exemple pour effectuer des tests.

#### 3. Pourquoi un service singleton doit-il rester sans état ?

Un bean singleton est partagé entre plusieurs appels et peut être utilisé simultanément par différents threads.

S'il conserve des informations propres à un utilisateur dans un attribut modifiable, une requête risque d'écraser les données d'une autre.

Par exemple, enregistrer un `contratCourant` dans un attribut du service pourrait mélanger les contrats de deux utilisateurs.

Les données propres à une opération doivent donc rester dans les paramètres et les variables locales.

#### 4. Pourquoi `update()` charge-t-il l'entité existante avant `save()` ?

L'objet reçu pour une modification peut contenir seulement une partie des informations.

Si nous l'enregistrons directement, nous risquons de remplacer des valeurs existantes ou de modifier involontairement des associations.

Nous récupérons donc d'abord l'entité enregistrée, puis nous recopions uniquement les attributs autorisés.

Cette précaution est particulièrement importante pour les contrats, car l'association avec les paiements utilise `orphanRemoval`.

#### 5. Différences entre les annotations et particularité du Service Paiement

**`@Component` et `@Bean` :**

`@Component` permet à Spring de détecter automatiquement une classe lors du scan.

`@Bean` est placé sur une méthode de configuration pour déclarer explicitement un objet géré par Spring. Il est notamment utile lorsqu'on ne peut pas annoter directement la classe.

**`@Primary` et `@Qualifier` :**

`@Primary` définit le bean à privilégier lorsqu'il existe plusieurs candidats du même type.

`@Qualifier` permet de sélectionner explicitement le bean souhaité lors de l'injection.

**Pourquoi `IPaiementService` est-il en lecture seule ?**

Les paiements appartiennent à la gestion des contrats.

Le professeur demande donc que leur création, leur modification et leur suppression passent par le contrat afin de respecter les règles métier et les associations JPA.

`IPaiementService` expose uniquement `findById()` et `findAll()`.

### Conclusion

Cet atelier nous a permis de compléter l'architecture multicouche d'AutoLoc en ajoutant la couche Service.

Nous avons mis en pratique l'inversion de contrôle, l'injection de dépendances par constructeur et les règles de gestion propres aux entités.

La séparation entre Service et Repository facilite l'organisation du code et prépare l'intégration des contrôleurs REST lors de l'Atelier 5.