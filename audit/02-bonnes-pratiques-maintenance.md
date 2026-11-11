# Audit Qualité & Bonnes Pratiques de Maintenance — Assistant Déclaration

Date de l'audit : Novembre 2026
Statut : Synthèse technique pour la reprise en main du projet
Périmètre : Architecture logicielle, organisation du code, dette technique, tests, CI/CD et outillage.

---

## 1. Vue d'Ensemble de l'Architecture Technique

Le projet repose sur une pile moderne articulée autour de l'écosystème TypeScript :
- **Framework Full-Stack** : Next.js (App Router pour les pages sous `src/app/`, Pages Router sous `src/pages/api/` pour les endpoints tRPC, upload tus, et maintenance).
- **Couche Données & ORM** : Prisma ORM v6 sur base PostgreSQL v15+ (hébergée sur Scalingo osc-secnum-fr1 en production).
- **Communication Client-Serveur** : tRPC v11 avec sérialisation `superjson` et typage bout-en-bout sans code-generation.
- **UI & Système de Design** : DSFR officiel (`@gouvfr/dsfr`, `@codegouvfr/react-dsfr`), couplé à Material UI (`@mui/material`) et SASS Modules.
- **Tâches Asynchrones & Files d'Attente** : `pg-boss` (queue persistée directement dans PostgreSQL sous le schéma `pgboss`).
- **Composants & Visual Testing** : Storybook v10 et Chromatic.

---

## 2. Dette Technique et Risques Majeurs pour la Maintenance

### 2.1. Fichiers « Monstres » et Complexité Cyclomatique Élevée

Plusieurs composants clés centralisent une quantité anormale de logique métier, d'effets de bord et de rendu UI :

1. **`DeclarationPage.tsx`** : **1 671 lignes de code**
   - Emplacement : `src/app/(private)/dashboard/organization/[organizationId]/serie/[eventSerieId]/declaration/DeclarationPage.tsx`
   - Contenu : Formulaire complet de déclaration unifiée (SACD + SACEM), calcul des taux de TVA, gestion des jauges, synchronisation avec le serveur, sélection des pièces jointes, overrides par événement.
   - Problème : Toute modification (même mineure) comporte un risque élevé de régression. Le composant gère à la fois l'état local du formulaire, la synchronisation temps réel, les dialogues d'erreur et les transitions visuelles.
   - Recommandation : Découper en sous-composants métier (`DeclarationSummary`, `TaxBreakdownSection`, `EventListEditor`, `AttachmentUploaderSection`) et extraire la logique d'état dans des hooks personnalisés (`useDeclarationForm`).

2. **`declaration.ts` (Routeur tRPC)** : **1 000+ lignes de code**
   - Emplacement : `src/server/routers/declaration.ts`
   - Contenu : Mutation géante `transmitDeclaration` et `fillDeclaration`, gestion des transactions Prisma, génération des PDFs SACEM via `@react-pdf/renderer`, télétransmission SACD via SOAP, et envoi d'emails via Nodemailer.
   - Problème : Mélange des responsabilités entre la validation d'entrée tRPC, la logique de persistance, la composition d'emails et la communication avec des APIs tierces.
   - Recommandation : Introduire un service applicatif dédié (`DeclarationService`) et découper les adapters (`SacdTransmissionAdapter`, `SacemTransmissionAdapter`).

3. **`OrganizationPage.tsx`** : **~450 lignes** combinant gestion de la billetterie, listing des séries, et paramétrage d'organisation.

### 2.2. Cohabitation App Router / Pages Router

- Next.js utilise l'**App Router** (`src/app/`) pour l'ensemble des écrans et layouts (sections `(public)`, `(private)`, `(visitor-only)`).
- Mais l'ensemble des routes API réside dans le **Pages Router** (`src/pages/api/`) : `/api/trpc/[trpc]`, `/api/upload/[[...file]]`, `/api/file/[fileId]`, `/api/maintenance/`, `/api/auth/[...nextauth]`.
- Risque pour la maintenance : Confusion pour un nouvel arrivant sur le cycle de vie des requêtes, incompatibilité de certains middlewares et nécessité de configurations hybrides dans `next.config.ts`.

### 2.3. Multiples « Workarounds » et Patchs Temporaires

Le code contient plusieurs annotations `[WORKAROUND]` ou `TODO` critiques témoignant de verrous techniques :
- **RsPack non activé** : `next.config.ts` contient des traces d'essais d'intégration RsPack abandonnés (`withRspack` commenté en raison de bugs d'incompatibilité avec Sentry et Next.js).
- **TypeScript `ignoreBuildErrors: true`** : Dans `next.config.ts`, les erreurs TypeScript sont volontairement ignorées au build Next.js (`ignoreBuildErrors: true`) sous prétexte qu'elles sont vérifiées dans une étape de CI (`npm run lint:ts`). Si un développeur lance un build local sans lint préalable, des erreurs de typage passent inaperçues.
- **`patch-package` actif** : Présence d'un dossier `patches/` pour corriger des bibliothèques externes lors du `postinstall`. Ces patchs constituent un point de rupture lors des montées de version.
- **Règles eslint assouplies** : Configuration hybride ESLint 9 / legacy `.eslintrc.js` avec de nombreuses règles désactivées ou contournées.

---

## 3. Qualité du Code, Typage et Architecture

### 3.1. Points Forts Observés

- **Validation Zod omniprésente** : Chaque mutation et query tRPC valide rigoureusement ses entrées (`input(MySchema)`).
- **Gestion structurée des erreurs** : Présence d'une classe `BusinessError` dans `src/models/entities/errors/` permettant de distinguer les erreurs attendues du parcours utilisateur des vraies anomalies techniques envoyées à Sentry.
- **Centralisation des mappings Prisma vers Modèles** : Dossier `src/server/routers/mappers/` qui isole la transformation des entités de base de données en objets transférés au client, évitant la fuite de colonnes sensibles.
- **Fixtures et Mocks de qualité** : Les clients externes (billetteries, SACD) disposent de mocks (`MockSacdClient`, `MockTicketingSystemClient`) permettant de tester l'UI sans accès aux environnements partenaires.

### 3.2. Points d'Amélioration Prioritaires

- **Couplage direct à Prisma dans les routeurs** : Les routeurs tRPC appellent directement `prisma.eventSerie.findUnique(...)` ou `prisma.eventSerie.update(...)` sans couche d'abstraction ou de repository.
- **Stockage de fichiers volumineux en base de données** : Le modèle `Attachment` stocke les fichiers binaires dans une colonne PostgreSQL `value Bytes @db.ByteA`. C'est une anti-pratique à moyen terme qui va surcharger la base de données de production et compliquer les backups/restaurations.
- **Absence de centralisation des constantes métier** : Les taux de TVA (0.055, 0.021, 0.20), les délais de rétention et les limites de taille de fichier sont parfois dispersés entre Zod schemas et fonctions utilitaires.

---

## 4. État des Tests et Automatisation CI/CD

### 4.1. Couverture des Tests Unitaires & Intégration

- **Outil** : Jest 29 configuré avec `ts-jest` et `--experimental-vm-modules`.
- **Observation** : Les tests unitaires couvrent principalement les connecteurs de billetterie (`billetweb.spec.ts`, `helloasso.spec.ts`, etc.), les fonctions de formatage et quelques modèles (`event.spec.ts`, `catcher.spec.ts`).
- **Angle mort majeur** : Absence totale de tests d'intégration automatisés pour la mutation la plus critique de l'application : `transmitDeclaration` dans `declaration.ts`. La conformité de la génération XML SACD et du PDF SACEM n'est couverte que par des tests de rendu isolés.
- **Tests UI** : Forte dépendance vis-à-vis des stories Storybook et de Playwright/test-runner pour valider le rendu visuel.

### 4.2. Pipeline CI/CD (GitHub Actions)

- Fichier principal : `.github/workflows/ci.yaml` (et configurations associées).
- Déploiement : Scalingo PaaS avec buildpacks Herokuish / standalone bundle.
- Points d'attention :
  - La compilation dépend de secrets Scalingo et de variables d'environnement (`SCALINGO_POSTGRESQL_URL`, `FILE_AUTH_SECRET`, etc.).
  - Le script `prepare-standalone.sh` et `start-and-wait-to-init.sh` assure les migrations automatiques au démarrage du conteneur en production.

---

## 5. Guide Pratique pour la Reprise de Maintenance

Pour un développeur reprenant le projet, voici la check-list indispensable avant toute intervention :

1. **Environnement Local** :
   - Cloner le dépôt et lancer `docker-compose up` (Postgres local + Mailcatcher).
   - Initialiser la base : `npm run db:migration:deploy && npm run db:schema:compile`.
   - Seeder la base : `npm run db:seed`.
2. **Cycle de Développement** :
   - UI uniquement : Lancer Storybook via `npm run dev:storybook` (permet de manipuler tous les états avec fausses données sans credentials tiers).
   - Application complète : `npm run dev:app`.
3. **Précautions avant déploiement** :
   - Ne jamais modifier le schéma Prisma sans créer une migration formelle : `npm run db:migration:prepare:manually`.
   - Toujours exécuter `npm run lint` et `npm run test:unit` manuellement avant de pousser sur `dev` ou `main`.
