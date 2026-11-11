# Plan d'Action Priorisé — Assistant Déclaration

Date : Novembre 2026
Objectif : Feuille de route opérationnelle classée par priorité d'urgence pour sécuriser et stabiliser la reprise du projet.

---

## Matrice de Priorisation Globale

| Priorité | Intitulé de l'Action | Catégorie | Risque initial | Effort estimé | Impact / Bénéfice |
| :---: | :--- | :---: | :---: | :---: | :--- |
| **P0** | **Chiffrer les clés d'API billetterie au repos** | Sécurité | Critique | 2 jours | Protection absolue des clés partenaires contre les fuites de BDD. |
| **P0** | **Mettre à jour les dépendances critiques (Storybook, shell-quote)** | Sécurité | Critique | 1 jour | Élimination des failles de RCE et d'injection de commande. |
| **P1** | **Sécuriser la validation d'accès / SSRF (Supersoniks & SoTicket)** | Sécurité | Élevé | 1 jour | Blocage des requêtes sortantes vers des réseaux ou métadonnées internes. |
| **P1** | **Ajouter un Rate Limiting sur l'authentification et les endpoints API** | Sécurité | Élevé | 1.5 jour | Protection contre les attaques par force brute et credential stuffing. |
| **P1** | **Écrire des tests d'intégration sur la transmission des déclarations** | Maintenance | Élevé | 2 jours | Sécurisation du cœur de valeur métier contre les régressions silencieuses. |
| **P2** | **Refactorer et segmenter le monolithe `DeclarationPage.tsx`** | Maintenance | Moyen | 3 jours | Amélioration drastique de la maintenabilité et réduction du risque de bug. |
| **P2** | **Mettre à jour les packages à vulnérabilité élevée (`sharp`, `zod`, `undici`)** | Sécurité | Moyen | 2 jours | Prévention des DoS mémoire et crashs de parsing d'images/flux. |
| **P2** | **Sécuriser la vérification de `MAINTENANCE_API_KEY` (timing safe)** | Sécurité | Moyen | 0.5 jour | Élimination du risque d'attaque par canal auxiliaire (timing attack). |
| **P3** | **Migrer le stockage des pièces jointes vers un Object Storage (S3)** | Architecture | Faible (court terme) | 4 jours | Déchargement de PostgreSQL, performance accrue des backups et de la RAM. |
| **P3** | **Harmoniser le routing vers Next.js App Router complet (Route Handlers)** | Architecture | Faible | 3 jours | Suppression du Pages Router legacy et simplification de la pile technique. |

---

## Détail des Actions par Phase

### Phase 1 : Urgences Critiques (P0 — Semaine 1)

#### Action 1.1 : Chiffrement des clés d'API des billetteries au repos
- **Contexte** : Dans `TicketingSystem`, `apiAccessKey` et `apiSecretKey` sont stockés en clair.
- **Plan de remédiation** :
  1. Définir une variable d'environnement `TICKETING_ENCRYPTION_KEY` (32 octets aléatoires).
  2. Créer un utilitaire cryptographique (`src/utils/crypto.ts`) utilisant l'algorithme AES-256-GCM avec IV aléatoire pour chaque enregistrement.
  3. Mettre à jour `src/server/routers/ticketing.ts` pour chiffrer les credentials avant `prisma.ticketingSystem.create` et `update`.
  4. Créer un script de migration (`src/scripts/migrations/encrypt_existing_ticketing_credentials.ts`) pour chiffrer les enregistrements existants en base.
  5. Déchiffrer les clés dans `src/core/ticketing/instance.ts` lors de l'instanciation des clients.
- **Critère d'acceptation** : Aucune clé secrète n'apparaît en texte clair lors d'un `SELECT * FROM "TicketingSystem";`.

#### Action 1.2 : Mise à jour des dépendances critiques
- **Contexte** : Vulnérabilités RCE dans Storybook (`GHSA-f2ph-cmg5-wxc8`) et injection de commande dans `shell-quote` (`GHSA-pqg4-j6r4-53mv`).
- **Plan de remédiation** :
  1. Mettre à jour `storybook` et ses addons vers les versions patchées (`>=10.6.2`).
  2. Mettre à jour `shell-quote` vers `>=1.10.1`.
  3. Relancer `npm audit` pour vérifier la disparition de ces avis critiques.
  4. Vérifier que la suite de tests Storybook s'exécute sans régression (`npm run build:storybook`).

---

### Phase 2 : Sécurisation & Robustesse (P1 — Semaine 2)

#### Action 2.1 : Blocage SSRF sur les clients Supersoniks et SoTicket
- **Contexte** : L'URL de base est construite via `https://${accessKey}/api/v2/`.
- **Plan de remédiation** :
  1. Améliorer la validation Zod dans `src/models/actions/ticketing.ts` pour interdire les domaines réservés (`localhost`, `.local`, `.internal`, `.arpa`).
  2. Résoudre le nom DNS au niveau applicatif et vérifier que l'adresse IP résultante n'appartient pas aux plages privées RFC 1918 (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16), loopback (127.0.0.0/8) ou cloud metadata (169.254.169.254).

#### Action 2.2 : Protection contre la force brute (Rate Limiting)
- **Contexte** : `/api/auth/[...nextauth]` et `/api/maintenance/` n'ont pas de limitation de fréquence des requêtes.
- **Plan de remédiation** :
  1. Intégrer un limiteur de requêtes mémoire ou basé sur PostgreSQL (ou Redis si disponible) via `bottleneck` ou `rate-limiter-flexible`.
  2. Restreindre les tentatives d'authentification à 5 échecs par minute par adresse IP.
  3. Restreindre les appels aux endpoints de maintenance.

#### Action 2.3 : Couverture de tests sur le flux de transmission des déclarations
- **Contexte** : `transmitDeclaration` dans `src/server/routers/declaration.ts` n'a pas de test unitaire ou d'intégration automatisé.
- **Plan de remédiation** :
  1. Créer un test d'intégration `src/server/routers/declaration.spec.ts` en utilisant la base de test / Testcontainers.
  2. Simuler une transmission SACD complète avec `MockSacdClient`.
  3. Simuler une transmission SACEM avec interception du flux email et validation du PDF généré.

---

### Phase 3 : Maintenabilité & Dette Technique (P2 — Semaines 3 & 4)

#### Action 3.1 : Découpage de `DeclarationPage.tsx`
- **Contexte** : 1 671 lignes de code dans un seul composant React.
- **Plan de remédiation** :
  1. Extraire la logique d'état et de synchronisation dans un hook `useDeclarationWorkflow.ts`.
  2. Isoler les composants visuels :
     - `DeclarationGeneralInfoForm.tsx` (nom de série, producteur, type de spectacle).
     - `DeclarationEventsTable.tsx` (liste des dates, jauges, recettes billetterie, exemptions).
     - `DeclarationExpensesForm.tsx` (dépenses du spectacle, frais d'approche).
     - `DeclarationAttachmentsSection.tsx` (téléversement et sélection des contrats et programmes).
     - `DeclarationSubmissionSummary.tsx` (récapitulatif avant transmission).

#### Action 3.2 : Mise à niveau des dépendances à sévérité élevée
- **Plan de remédiation** :
  1. Monter `sharp` vers `>=0.35.5` pour corriger la CVE librsvg.
  2. Mettre à niveau `undici` et `zod` vers leurs versions stables récentes.
  3. Vérifier la compatibilité avec Next.js et tRPC.

#### Action 3.3 : Sécurisation de l'authentification de maintenance
- **Plan de remédiation** :
  1. Remplacer `maintenanceApiKey === apiKeyHeader` dans `src/utils/maintenance.ts` par :
     ```typescript
     crypto.timingSafeEqual(
       Buffer.from(maintenanceApiKey, 'utf-8'),
       Buffer.from(apiKeyHeader, 'utf-8')
     )
     ```
  2. Prévenir toute erreur si les longueurs de chaînes diffèrent en normalisant avec un hash SHA-256 intermédiaire à temps constant.

---

### Phase 4 : Évolution Architecturale (P3 — Moyen Terme)

#### Action 4.1 : Déport des pièces jointes vers Scalingo Object Storage / S3
- Remplacer le champ `value Bytes` dans PostgreSQL par un identifiant d'objet S3 (`s3Key`).
- Adapter le serveur `tus` pour utiliser `@tus/s3-store`.
- Bénéfice : Base de données allégée de 80%, sauvegardes instantanées et consommation mémoire Node.js maîtrisée.

#### Action 4.2 : Migration vers les Route Handlers Next.js App Router
- Migrer les fichiers de `src/pages/api/` vers `src/app/api/.../route.ts`.
- Uniformiser toute l'application sous l'App Router Next.js.
