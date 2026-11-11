# Feature 05 : Déclarations SACD & SACEM

## 1. Description Fonctionnelle
Cœur de valeur de l'application : permet à l'entrepreneur de valider un formulaire unifié et de télétransmettre ses déclarations administratives directement aux sociétés de gestion collective des droits d'auteur (SACD et SACEM).

---

## 2. Le Flux SACD (Société des Auteurs et Compositeurs Dramatiques)

### 2.1. Principes et Spécifications
- Concerne le spectacle dramatique, la danse, l'humour, le cirque, la marionnette, etc.
- Nécessite le numéro SACD du déclarant (`organization.sacdId`).
- La transmission s'opère par **télétransmission directe via API SOAP/XML**.

### 2.2. Cinématique Technique (`src/core/declaration/sacd.ts`)
1. **Authentification** : Appel POST `/ticketing/auto-access.json` avec identifiants SACD (`SACD_API_PROVIDER_NAME`, `SACD_API_PROVIDER_REFFILE`, `SACD_API_CONSUMER_KEY`, mot de passe hashé MD5) pour obtenir un `AuthToken`.
2. **Génération du XML** : Construction de l'arbre XML via la librairie `xmlbuilder2` (`prepareDeclarationParameter`) :
   - En-tête : Référence déclarant, date, nombre de représentations.
   - Détail par représentation (`Representation`) : Titre, date début/fin, horaire, montant HT, TVA calculée, nombre de billets payants et exonérés, SIREN diffuseur et producteur, nom et jauge de la salle.
   - Dépenses de plateau : Les dépenses de cession et frais d'approche (portés au niveau de la série) sont rattachés au premier événement du flux XML.
3. **Transmission du flux** : Appel POST `/ticketing/broker/ExploitFileTransmitETicketingDeclarationWS` avec le paramètre `parameters[Declaration]`.
4. **Analyse de la réponse** :
   - Vérification de la présence d'erreurs au format JSON ou XML.
   - Détection des codes d'erreur connus (ex. `ERR_WS_DEC_REF` : numéro déclarant erroné).
   - Analyse des statuts par représentation (`statut: 'KO'` ou `'WARNING'`).
5. **Déconnexion** : Appel POST `/ticketing/logout`.
6. **Pièces jointes SACD** : L'API SACD n'acceptant pas les binaires, les pièces jointes (contrat de cession) sont transmises par email à l'agence SACD de rattachement (`SacdAgency`), déterminée par le préfixe du code postal du lieu (`matchingFrenchPostalCodesPrefixes`).

---

## 3. Le Flux SACEM (Société des Auteurs, Compositeurs et Éditeurs de Musique)

### 3.1. Principes et Spécifications
- Concerne les concerts, festivals, spectacles musicaux, musique de scène.
- Nécessite le numéro de compte client SACEM (`organization.sacemId`).
- La transmission s'opère par **génération d'un PDF officiel et transmission par email certifié**.

### 3.2. Cinématique Technique (`src/server/routers/declaration.ts`)
1. **Validation des Données** : Vérification des champs requis SACEM via Zod (`SacemDeclarationSchema`).
2. **Génération du Document PDF** :
   - Généré à la volée côté serveur via `@react-pdf/renderer` (`renderToBuffer`).
   - Modèle : `SacemDeclarationDocument` (`src/components/documents/templates/SacemDeclaration.tsx`).
   - Respecte fidèlement la charte graphique et la disposition des formulaires papier SACEM : en-tête avec coordonnées du déclarant, tableau ventilé des recettes par date, dépenses du spectacle, déclaration sur l'honneur.
3. **Routage Postal Automatisé** :
   - Recherche dans la table `SacemAgency` via le code postal de la salle (`Place.address.postalCode`).
   - La table `SacemAgency` contient la liste complète des adresses emails des délégations régionales SACEM.
4. **Envoi de l'Email Certifié** :
   - Modèle MJML : `DeclarationToSacemAgency` (`src/components/emails/templates/DeclarationToSacemAgency.tsx`).
   - Pièce jointe principale : Le PDF SACEM fraîchement généré.
   - Pièces jointes annexes : Le programme des œuvres jouées et les états de billetterie associés à la série.
   - Destinataire principal : L'email de la délégation SACEM compétente.
   - Copie (CC) : L'adresse email de l'utilisateur déclarant.

---

## 4. Modèles de Données Impliqués
- `EventSerieDeclaration` :
  - `eventSerieId`, `type` (`SACD` ou `SACEM`), `status` (`PENDING`, `PROCESSED`, `CANCELED`).
  - `transmittedAt` : Date et heure de transmission.
  - `lastTransmissionError`, `lastTransmissionErrorAt`.
- `SacdAgency` : Mapping préfixe département postal -> email agence SACD.
- `SacemAgency` : Mapping code postal complet -> email agence régionale SACEM.

## 5. Endpoints & Procédures tRPC
- `declarationRouter` (`src/server/routers/declaration.ts`) :
  - `getDeclaration` : Récupère l'état courant de la déclaration pour une série, pré-rempli avec les données calculées.
  - `fillDeclaration` : Sauvegarde intermédiaire des modifications utilisateur sans transmission finale.
  - `transmitDeclaration` : Mutation principale déclenchant les flux SACD et/ou SACEM de façon atomique et transactionnelle.

## 6. Points d'Attention & Maintenance
- **Absence de tests d'intégration automatisés** : Priorité P1 pour créer des scénarios de test end-to-end de transmission.
- **Mise à jour des listings agences** : En cas de changement des découpages territoriaux SACD ou SACEM, les commandes CLI `npm run cli sacd agency sync` et `npm run cli sacem agency sync` doivent être exécutées avec les nouveaux fichiers CSV sous `./data/`.
