# Documentation Fonctionnelle Globale — Assistant Déclaration

Date : Novembre 2026
Objectif : Fournir une vision complète des concepts métier, des acteurs, du modèle de données et des processus de l'application.

---

## 1. Vision et Objectifs du Produit

**Assistant Déclaration** est une plateforme nationale française (incubée au sein de beta.gouv.fr) destinée aux **entrepreneurs du spectacle vivant**.

### Le Problème Métier Résolu
Pour chaque spectacle ou série de représentations, les producteurs et diffuseurs doivent déclarer leurs recettes et dépenses auprès des organismes de perception des droits d'auteur (principalement la **SACEM** pour la musique et la **SACD** pour le théâtre, l'humour, la danse, etc.).
Cette démarche administrative est historiquement :
- **Fastidieuse** : Saisie manuelle redondante de chiffres entre différents formulaires.
- **Source d'erreurs** : Calculs complexes de TVA, déductions de commissions, proratas de recettes accessoires (consommations, programmes).
- **Éparpillée** : Chaque organisme exige son propre format (télétransmission XML pour la SACD, document récapitulatif pour la SACEM).

### La Solution Apportée par Assistant Déclaration
1. **Connecter les logiciels de billetterie** (Billetweb, Shotgun, Mapado, HelloAsso, Supersoniks, SoTicket) pour importer automatiquement les événements, le nombre de billets (payants / exonérés) et les montants hors taxes et toutes taxes comprises.
2. **Offrir un formulaire unique et unifié** : L'entrepreneur renseigne ses informations une seule fois (nom du spectacle, lieu, jauge, producteur, dépenses de plateau, frais d'approche).
3. **Automatiser la transmission** :
   - Pour la **SACD** : Télétransmission automatisée via leur API web services (génération d'un flux XML conforme et envoi en tâche de fond).
   - Pour la **SACEM** : Génération automatique du document PDF officiel certifié et expédition par email à l'agence régionale compétente (déterminée selon le code postal du lieu du spectacle).

---

## 2. Acteurs et Gestion des Permissions

### 2.1. Les Utilisateurs (`User`)
- Un utilisateur représente une personne physique (prénom, nom, email, mot de passe hashé).
- Statuts possibles :
  - `REGISTERED` : Compte créé, en attente de confirmation par lien email.
  - `CONFIRMED` : Compte actif, capable de se connecter et d'agir.
  - `DISABLED` : Compte suspendu.

### 2.2. Les Organisations (`Organization`)
- L'organisation est la personne morale (compagnie, producteur, théâtre, association, SAS) responsable légale des déclarations.
- Données légales obligatoires :
  - Nom commercial
  - Numéro SIREN (`officialId`)
  - Numéro SIRET du siège (`officialHeadquartersId`)
  - Adresse du siège (`headquartersAddress`)
  - Numéro d'adhérent SACEM (`sacemId`) et SACD (`sacdId`) optionnels mais indispensables pour les télétransmissions respectives.

### 2.3. Les Collaborateurs (`Collaborator`)
- Table de liaison Many-to-Many entre `User` et `Organization`.
- Un utilisateur peut être membre de plusieurs organisations (ex. un administrateur de production gérant plusieurs compagnies).
- Les vérifications de sécurité dans tRPC (`isUserACollaboratorPartOfOrganization`) garantissent qu'un utilisateur n'accède qu'aux organisations auxquelles il est rattaché.

### 2.4. Les Administrateurs Système (`Admin`)
- Enregistrement dans la table `Admin` lié à un `User`.
- Possède le drapeau `canEverything: true`.
- Donne accès aux fonctionnalités d'administration transverse (non exposées aux déclarants standards).

---

## 3. Modèle Métier & Concepts Clés

```
[ Organization ]
       | 1
       +----< [ TicketingSystem ] (Billetweb, Shotgun, Mapado...)
       | 1
       +----< [ Collaborator ] >----+ [ User ]
       |
[ EventSerie ] (Série de représentations : ex. "Tournée Automne")
       |
       +--- 1:N ---> [ Event ] (Représentation individuelle : Date, Recettes)
       |
       +--- 1:N ---> [ EventSerieDeclaration ] (SACD / SACEM, statut PENDING/PROCESSED)
       |
       +--- N:M ---> [ AttachmentsOnEventSeries ] ---> [ Attachment ] (Contrats, Programmes)
```

### 3.1. Système de Billetterie (`TicketingSystem`)
- Représente un compte ou une clé d'accès vers un logiciel de billetterie partenaire.
- Peut également être de type `MANUAL` pour les entrepreneurs n'utilisant pas de billetterie informatisée connectée (saisie manuelle totale).
- Stocke l'état de la dernière synchronisation (`lastSynchronizationAt`) et les erreurs éventuelles (`lastProcessingError`).

### 3.2. Série d'Événements (`EventSerie`)
- Entité pivot représentant une pièce de théâtre, un concert ou une tournée.
- Porte les métadonnées communes à toutes les dates :
  - Nom de la série (`name`)
  - Producteur (`producerName`, `producerOfficialId`)
  - Type de spectacle (`performanceType` : Théâtre, Danse, Musique, Humour, etc.)
  - Organismes cibles (`expectedDeclarationTypes`: `[SACEM, SACD]`)
  - Lieu principal (`Place`) et jauge (`placeCapacity`)
  - Taux de TVA billetterie par défaut (`ticketingRevenueTaxRate`)
  - Dépenses globales du plateau : montant HT/TTC des dépenses de cession, frais d'approche et dépenses spécifiques cirque.

### 3.3. Événement / Représentation (`Event`)
- Représente une date précise de spectacle (`startAt`, `endAt`).
- Détail des recettes :
  - Recettes billetterie HT et TTC (`ticketingRevenueIncludingTaxes`, `ticketingRevenueExcludingTaxes`).
  - Nombre de billets payants (`paidTickets`) et exonérés / gratuits (`freeTickets`).
  - Recettes annexes (consommations bar/restauration, vente de programmes, merchandising).
- Dérogations possibles par date : override du lieu (`placeOverride`), de la jauge ou du taux de TVA si une date particulière a des conditions différentes de la série.

### 3.4. Déclaration (`EventSerieDeclaration`)
- Enregistre l'état de transmission pour chaque organisme (`SACD` ou `SACEM`) :
  - Statuts : `PENDING` (en cours de saisie), `PROCESSED` (transmis avec succès), `CANCELED` (annulé).
  - Horodatage de transmission (`transmittedAt`).
  - Erreur éventuelle renvoyée par le tiers (`lastTransmissionError`).

### 3.5. Pièces Jointes (`Attachment` et `AttachmentsOnEventSeries`)
- Types de documents associés à une série :
  - `ARTISTIC_CONTRACT` : Contrat de cession ou de coréalisation (souvent requis par la SACD).
  - `PERFORMED_WORK_PROGRAM` : Programme des œuvres jouées / feuille de déclaration SACEM.
  - `REVENUE_STATEMENT` : État des recettes certifié.
  - `OTHER` : Tout document complémentaire.

---

## 4. Parcours Utilisateur & Cycle de Vie d'une Déclaration

### Étape 1 : Connexion & Onboarding
1. L'entrepreneur crée son compte et confirme son adresse email.
2. Il crée son organisation en renseignant son numéro SIREN (auto-complété via l'API publique de l'INSEE / Annuaire des Entreprises).
3. Il renseigne les numéros d'adhérent SACD et SACEM de la structure.

### Étape 2 : Connexion de la Billetterie & Synchronisation
1. L'utilisateur sélectionne son logiciel de billetterie (ex. Billetweb ou Shotgun) et fournit les identifiants d'API.
2. Le système teste immédiatement la connexion.
3. Une synchronisation est déclenchée (via un stream HTTP long-polling pour éviter le timeout de 60 secondes de Scalingo).
4. Les événements et séries sont créés ou mis à jour en base de données.

### Étape 3 : Complétion du Formulaire Unique
1. L'utilisateur sélectionne une série d'événements à déclarer.
2. Il coche les organismes destinataires : **SACEM**, **SACD**, ou les deux.
3. Il vérifie et ajuste les montants (les recettes billetterie sont pré-remplies par la synchronisation).
4. Il renseigne les dépenses de plateau (frais de cession, frais d'approche).
5. Il téléverse les pièces jointes requises (contrats, programme des œuvres).

### Étape 4 : Validation & Transmission
1. Le système effectue une validation stricte Zod des champs requis selon les organismes cochés.
2. **Pour la SACD** :
   - Le système génère le document XML conforme à la spécification SACD.
   - Connexion à l'API SACD (`auto-access.json`), envoi du XML (`ExploitFileTransmitETicketingDeclarationWS`).
   - Si des pièces jointes sont jointes, envoi d'un email récapitulatif à l'agence SACD de rattachement.
3. **Pour la SACEM** :
   - Le système génère le PDF récapitulatif officiel via `@react-pdf/renderer`.
   - Il détermine l'agence SACEM compétente grâce au code postal du lieu du spectacle (recherche dans `SacemAgency`).
   - Il envoie un email certifié avec le PDF et les pièces jointes à l'agence SACEM ainsi qu'en copie au déclarant.
4. Le statut de la déclaration passe à `PROCESSED`. La déclaration est verrouillée en modification pour garantir l'intégrité comptable.
