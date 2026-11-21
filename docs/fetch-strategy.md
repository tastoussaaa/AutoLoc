# AutoLoc : Stratégies Fetch et Cascade

## 1. Introduction

Dans le premier atelier, nous avons créé les neuf entités du projet AutoLoc et configuré leur persistance avec JPA et Hibernate.

Dans ce deuxième atelier, nous avons relié ces entités en respectant les associations définies dans notre modèle. L'objectif n'était pas seulement de créer les clés étrangères dans MySQL, mais aussi de définir comment les objets liés sont chargés et ce qui se passe lorsqu'une entité est enregistrée, modifiée ou supprimée.

Pour cela, nous avons utilisé les stratégies **FetchType** et **CascadeType**, en tenant compte du rôle de chaque association dans l'application.

## 2. Choix des stratégies pour chaque association

| Association | Côté propriétaire | Fetch | Cascade | Justification |
|---|---|---|---|---|
| Contrat / Paiement | Paiement | LAZY | ALL + orphanRemoval | Un paiement est lié à un contrat précis. Si le contrat est supprimé, ses paiements doivent également l'être. Un paiement retiré de la collection est considéré comme orphelin. |
| Agence / Vehicule | Vehicule | LAZY | Aucune | Une voiture peut être transférée vers une autre agence. Elle ne doit donc pas être supprimée automatiquement avec son agence. |
| Agence / Employe | Employe | LAZY | Aucune | Un employé peut changer d'agence sans être supprimé du système. Les deux entités ont des cycles de vie distincts. |
| Vehicule / Equipement | Vehicule | LAZY | Aucune | Un équipement, comme un GPS, peut être associé à plusieurs véhicules. Il ne doit pas disparaître lorsqu'un véhicule est supprimé. |
| Client / Reservation | Reservation | LAZY | PERSIST | Lors de la création d'un client, ses nouvelles réservations associées peuvent être enregistrées avec lui. En revanche, leur suppression ne doit pas être automatique. |
| Reservation / Vehicule | Reservation | LAZY | Aucune | Un véhicule peut faire l'objet de plusieurs réservations successives. Supprimer une réservation ne doit pas supprimer le véhicule. |
| Reservation / Contrat | Contrat | LAZY | ALL côté Reservation | Le contrat est lié à une réservation. Les opérations effectuées sur la réservation sont propagées à son contrat. |
| Vehicule / Maintenance | Maintenance | LAZY | PERSIST | Les nouvelles interventions peuvent être enregistrées avec leur véhicule. Toutefois, l'historique de maintenance ne doit pas être supprimé automatiquement. |

Ces choix ne sont pas identiques pour toutes les associations, car les entités ne dépendent pas toutes les unes des autres de la même manière.

## 3. Stratégies de chargement : LAZY et EAGER

### FetchType.LAZY

Nous avons principalement retenu **FetchType.LAZY** pour les associations d'AutoLoc.

Prenons le cas d'une agence qui possède plusieurs véhicules. Lorsqu'on récupère les informations de cette agence, il n'est pas toujours nécessaire de charger immédiatement toute sa liste de véhicules.

Avec LAZY, Hibernate peut différer le chargement de cette collection jusqu'au moment où le programme en a réellement besoin.

Cette stratégie permet d'éviter des chargements inutiles, notamment lorsque les collections deviennent importantes.

Il faut cependant faire attention au contexte de persistance : si une association LAZY n'a pas encore été initialisée et qu'on essaie d'y accéder après la fermeture de la session Hibernate, une `LazyInitializationException` peut être déclenchée.

### FetchType.EAGER

Nous avons également étudié **FetchType.EAGER** pour comprendre la différence.

Avec EAGER, les données associées sont récupérées lors du chargement de l'entité principale, même si elles ne sont pas directement utilisées. Cela peut être pratique dans certains cas, mais risque aussi d'augmenter le volume de données récupérées et le nombre de requêtes SQL.

Par exemple, charger systématiquement tous les paiements à chaque consultation d'un contrat serait inutile si nous voulons simplement afficher sa date de signature ou son montant total.

C'est pourquoi nous avons conservé **LAZY** pour l'association `Contrat / Paiement`, comme recommandé dans l'atelier.

## 4. Stratégies de cascade

### CascadeType.ALL

Nous avons utilisé `CascadeType.ALL` pour les associations où les entités sont fortement dépendantes.

C'est notamment le cas de **Contrat et Paiement**.

Un contrat peut contenir plusieurs paiements. Grâce à la cascade, les opérations de persistance, de mise à jour, de suppression, de rafraîchissement et de détachement peuvent être propagées du contrat vers ses paiements.

Ainsi, lorsqu'un contrat est supprimé par JPA, les paiements qui lui sont associés sont également supprimés.

Nous avons également appliqué cette stratégie du côté `Reservation` pour la relation avec `Contrat`.

### CascadeType.PERSIST

Pour les associations `Client / Reservation` et `Vehicule / Maintenance`, nous avons retenu `CascadeType.PERSIST`.

Cette stratégie permet de propager uniquement l'opération de persistance.

Par exemple, lorsqu'un nouveau client est enregistré avec de nouvelles réservations correctement associées, celles-ci peuvent être persistées au cours de la même opération.

En revanche, cette configuration ne propage pas automatiquement la suppression du client à ses réservations.

### Associations sans cascade

Pour les autres associations, nous n'avons pas ajouté de cascade.

Prenons l'exemple de `Agence / Vehicule`. Une voiture peut être affectée à une autre agence et continuer à exister indépendamment de son affectation actuelle.

Appliquer `CascadeType.REMOVE` dans ce cas serait risqué, car la suppression d'une agence pourrait entraîner celle de véhicules que nous souhaitons conserver.

La même logique s'applique aux équipements, qui peuvent être partagés entre plusieurs véhicules.

L'absence de cascade ne signifie pas que toutes les suppressions seront automatiquement autorisées par MySQL. Les contraintes de clés étrangères doivent également être respectées.

## 5. Utilisation de orphanRemoval

Dans l'association `Contrat / Paiement`, nous avons ajouté :

```java
@OneToMany(
    mappedBy = "contrat",
    cascade = CascadeType.ALL,
    orphanRemoval = true,
    fetch = FetchType.LAZY
)
private List<Paiement> paiements = new ArrayList<>();
```

Il est important de distinguer `orphanRemoval` de `CascadeType.REMOVE`.

Avec `CascadeType.REMOVE`, supprimer un contrat entraîne la suppression de ses paiements.

Avec `orphanRemoval = true`, nous pouvons également supprimer un paiement simplement en le retirant de la collection de son contrat, dans un contexte JPA approprié.

Le contrat reste donc enregistré, mais le paiement qui ne lui est plus associé est supprimé lors de la synchronisation avec la base de données.

Cette configuration correspond au modèle retenu dans l'atelier, où un paiement est considéré comme dépendant de son contrat.

## 6. Gestion des associations bidirectionnelles

Pour les associations bidirectionnelles, nous avons distingué le **côté propriétaire** du **côté inverse**.

Dans la relation `Contrat / Paiement`, le côté propriétaire est `Paiement`, car c'est sa table qui contient la clé étrangère vers `Contrat`.

Dans `Paiement.java`, nous avons :

```java
@ManyToOne(fetch = FetchType.LAZY)
private Contrat contrat;
```

Et dans `Contrat.java`, nous utilisons :

```java
@OneToMany(mappedBy = "contrat")
```

L'attribut `mappedBy` indique à Hibernate que l'association est déjà gérée par le champ `contrat` de l'entité `Paiement`.

Il évite ainsi de créer une deuxième association indépendante et une table de jointure inutile.

Lorsqu'on ajoute un paiement à un contrat en Java, il faut aussi veiller à mettre à jour les deux côtés de la relation pour conserver des objets cohérents.

Pour la relation `Vehicule / Equipement`, nous avons utilisé `@ManyToMany` et une table de jointure `vehicule_equipement`. Nous avons choisi une collection `Set` afin d'éviter les associations en double selon les règles d'égalité des objets.

Enfin, nous avons conservé les annotations Lombok ciblées (`@Getter`, `@Setter`, `@NoArgsConstructor`, `@AllArgsConstructor`) plutôt que `@Data`, afin d'éviter notamment les problèmes de récursion dans les méthodes générées pour les associations bidirectionnelles.

## 7. Conclusion

Cet atelier nous a permis de passer de simples entités JPA indépendantes à un modèle dans lequel les objets sont réellement associés.

Nous avons constaté que le choix d'une stratégie ne dépend pas uniquement du type d'association. Il dépend aussi du fonctionnement de l'application.

**FetchType** détermine quand récupérer les données associées, tandis que **CascadeType** détermine quelles opérations doivent être propagées entre les entités.

Dans AutoLoc, nous avons donc privilégié le chargement LAZY et limité les cascades aux relations où elles sont justifiées par le cycle de vie des données.

Cette organisation nous permet de conserver un modèle cohérent tout en évitant les chargements inutiles et les suppressions automatiques non souhaitées.