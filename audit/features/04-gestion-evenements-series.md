# Feature 04 : Gestion des Séries & Événements

## 1. Description Fonctionnelle
Permet de visualiser, enrichir et corriger les séries de représentations (titre du spectacle, producteur, lieu, discipline artistique, dépenses de plateau) et le détail de chaque événement (date/heure, jauge de la salle, billets payants/exonérés, recettes billetterie et recettes annexes).

## 2. Règles Métier & Fiscalité des Droits d'Auteur
- **Série vs Événement** :
  - Une `EventSerie` regroupe un ensemble de représentations (`Event`) ayant le même titre et le même modèle de production.
  - Les dépenses de plateau (contrat de cession, frais d'approche) sont attachées au niveau de la série.
  - Les recettes de billetterie et le nombre de spectateurs sont attachés au niveau de chaque événement.
- **TVA et Recettes** :
  - Les montants sont stockés en Decimal(10, 2) : HT (`ticketingRevenueExcludingTaxes`) et TTC (`ticketingRevenueIncludingTaxes`).
  - Taux de TVA usuels dans le spectacle vivant : 5,5 % ou 2,1 % (pour les 140 premières représentations de certaines disciplines) ou 20 % pour les recettes annexes (bars/merchandising).
  - L'application calcule automatiquement le montant de la TVA : `Montant TTC - Montant HT`.
- **Overrides par Événement** :
  - Un événement hérite du lieu (`Place`), de la jauge (`placeCapacity`) et du taux de TVA de sa série.
  - L'utilisateur peut surcharger ponctuellement ces valeurs pour une date donnée (ex. festival délocalisé ou jauge réduite) via les champs `placeOverrideId`, `placeCapacityOverride`, `ticketingRevenueTaxRateOverride`.

## 3. Modèles de Données Impliqués
- `EventSerie` :
  - Métadonnées spectacle : `name`, `producerName`, `producerOfficialId`, `performanceType` (Enum de 14 disciplines).
  - Discipline artistique : Théâtre, Danse, Musique populaire, Humour, etc.
  - Dépenses : `expensesExcludingTaxes`, `expensesIncludingTaxes`, `introductionFeesExpensesExcludingTaxes`, `circusSpecificExpensesExcludingTaxes`.
- `Event` :
  - `startAt`, `endAt`.
  - Billetterie : `paidTickets`, `freeTickets`, `ticketingRevenueIncludingTaxes`, `ticketingRevenueExcludingTaxes`.
  - Recettes annexes : `consumptionsRevenue...`, `cateringRevenue...`, `programSalesRevenue...`, `otherRevenue...`.
- `Place` :
  - `name`, `addressId` (relié à `Address`).

## 4. Endpoints & Procédures tRPC
- Routeur `eventRouter` (`src/server/routers/event.ts`) :
  - `listEventSeries` : Liste des séries d'une organisation avec statut de déclaration et totaux calculés.
  - `getEventSerie` : Détail complet d'une série et de l'ensemble de ses représentations.
  - `updateEventSerie` : Mise à jour manuelle des métadonnées de la série.
  - `updateEvent` : Mise à jour manuelle des chiffres d'une date (marquage automatique de `lastManualTicketingDataUpdateAt`).
  - `createManualEventSerie` / `createManualEvent` : Création unitaire pour les billetteries de type `MANUAL`.

## 5. Points d'Attention & Maintenance
- **Précision des calculs flottants** : Utilisation stricte de `Prisma.Decimal` en base pour éviter les erreurs d'arrondi sur les centimes d'euros.
- **Impact sur les déclarations transmises** : Si une série a déjà fait l'objet d'une déclaration au statut `PROCESSED`, toute modification ultérieure des événements ou de la série est bloquée par l'erreur `transmittedDeclarationCannotBeUpdatedError`.
