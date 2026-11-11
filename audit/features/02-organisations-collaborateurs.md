# Feature 02 : Organisations & Collaborateurs

## 1. Description Fonctionnelle
Permet de regrouper les activités de spectacle sous une entité juridique légale (l'Organisation). Les utilisateurs peuvent créer une organisation, renseigner ses coordonnées légales (SIREN, SIRET, adresse du siège, numéros SACD/SACEM), inviter des collaborateurs et naviguer entre plusieurs organisations.

## 2. Architecture Technique
- **API Tiers Publiques** :
  - **Annuaire des Entreprises** (`https://recherche-entreprises.api.gouv.fr/`) : Recherche et complétion automatique des données d'entreprise à partir du SIREN/SIRET.
  - **Base Adresse Nationale (BAN / Geopf)** (`https://data.geopf.fr/`) : Normalisation et validation des adresses postales françaises.
- **Autorisation tRPC** :
  - `isUserACollaboratorPartOfOrganization` : Middleware de contrôle d'accès garantissant l'isolation des données entre structures.

## 3. Modèles de Données Impliqués
- `Organization` :
  - `officialId` : Numéro SIREN (unique).
  - `officialHeadquartersId` : Numéro SIRET du siège (unique).
  - `headquartersAddressId` : Clé étrangère vers l'adresse normalisée (`Address`).
  - `sacemId` : Numéro de compte déclarant SACEM.
  - `sacdId` : Numéro de compte diffuseur/producteur SACD.
- `Collaborator` :
  - `userId`, `organizationId` (clé composite unique).
- `Address` :
  - `street`, `postalCode`, `city`, `countryCode`, `subdivision`.

## 4. Endpoints & Procédures tRPC
- Routeur `organizationRouter` (`src/server/routers/organization.ts`) :
  - `createOrganization` : Création de la structure, création de l'adresse liée, et rattachement immédiat de l'utilisateur créateur en tant que collaborateur.
  - `getOrganization` : Récupération des informations d'une organisation (vérifie l'appartenance de l'utilisateur).
  - `listOrganizations` : Liste des organisations auxquelles l'utilisateur connecté collabore.
  - `updateOrganization` : Mise à jour des identifiants légaux et numéros SACD/SACEM.
  - `addCollaborator` : Ajout d'un membre par son adresse email (envoi d'un email d'invitation/bienvenue).

## 5. Points d'Attention & Maintenance
- **Contrainte d'unicité SIREN/SIRET** : Si deux utilisateurs tentent de créer la même organisation (même SIREN), une `BusinessError` d'unicité est levée. Le support doit alors intervenir pour inviter l'utilisateur au lieu de dupliquer l'organisation.
- **Règles CSP** : Le middleware Next.js autorise les requêtes directes du frontend vers `https://recherche-entreprises.api.gouv.fr/` et `https://data.geopf.fr/`.
