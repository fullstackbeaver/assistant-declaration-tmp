# Audit de Sécurité 360° — Assistant Déclaration

Date de l'audit : Novembre 2026
Statut : Révision initiale avant reprise de maintenance
Périmètre : Code source applicatif, configuration, dépendances et intégrations tierces.

---

## 1. Synthèse Exécutive de Sécurité

L'application **Assistant Déclaration** présente une architecture défensive globale bien pensée (utilisation rigoureuse de Zod, requêtes typées via Prisma ORM et tRPC, absence d'injections SQL brutes, chiffrement des mots de passe utilisateurs via bcrypt, secrets NextAuth et tokens d'accès fichiers signés via JWT).

Cependant, plusieurs **vulnérabilités critiques à élevées** et des **angles morts architecturaux** nécessitent une remédiation urgente avant toute montée en charge ou modification du code :

1. **Stockage en clair des clés d'API partenaires dans PostgreSQL** (`TicketingSystem.apiAccessKey` et `apiSecretKey`). En cas de dump ou d'accès en lecture à la base, l'ensemble des clés des billetteries des entrepreneurs de spectacle est exposé.
2. **Vulnérabilités de dépendances critiques & élevées (117 signalées par `npm audit`)**, dont Storybook (RCE/code injection via dev server & missing origin validation GHSA-f2ph-cmg5-wxc8, GHSA-px85-f4q5-jh77), `shell-quote` (command injection GHSA-pqg4-j6r4-53mv), `nodemailer` / `next-auth` (DoS & injection d'en-têtes), et `sharp` (libvips/librsvg CVE-2026-96889).
3. **Endpoint de maintenance CLI (`/api/maintenance/cli.ts`)** : Bien que protégé par une clé d'API statique (`X-API-KEY`), la liste blanche `commandsPrefixesWhitelist` est vide (`[]`), désactivant de fait l'exécution mais créant un risque si une commande générique y est ajoutée sans validation stricte des arguments passés à Commander.js (`parseAsync` direct).
4. **Risque de SSRF partiel sur Supersoniks / SoTicket** : L'accès API construit dynamiquement une URL `https://${accessKey}/api/v2/`. Même si une regex valide le nom de domaine, la résolution DNS interne n'est pas bloquée (possibilité de cibler des hôtes internes ou endpoints Cloud metadata via DNS rebinding ou noms pointant vers 127.0.0.1).
5. **Gestion des fichiers en base (`bytea`) et DoS mémoire** : Le streaming de fichiers volumineux ou la synchronisation de données non paginées (chargement de 100 000 000 items d'un coup dans Supersoniks) peut saturer la mémoire du conteneur Node.js (limité à 512 Mo sur Scalingo).
6. **CSP assouplie** : `style-src-elem 'unsafe-inline'` et `'unsafe-eval'` (en dev), injection de domaines tiers (Crisp, Matomo, Geopf, Metabase).

---

## 2. Analyse des Dépendances et CVEs

Le scan automatisé via `npm audit` remonte **117 vulnérabilités** :
- **4 Critiques**
- **19 Élevées**
- **82 Modérées**
- **12 Faibles**

### 2.1. Vulnérabilités Critiques (Impact Immédiat)

| Paquet | Versions | CVE / Advisory | Gravité | Risque & Vecteur d'Attaque |
| :--- | :--- | :--- | :--- | :--- |
| **storybook** | `10.0.0 - 10.6.1` | GHSA-f2ph-cmg5-wxc8 / GHSA-px85-f4q5-jh77 | **CRITIQUE** | Injection de code arbitraire à distance via le serveur de dev Storybook et absence de validation d'origine dans la prévisualisation. Si un développeur ou une instance publique fait tourner Storybook, un site tiers malveillant peut exécuter du code. |
| **shell-quote** | `1.8.4 - 1.10.0` | GHSA-pqg4-j6r4-53mv | **CRITIQUE** | Injection de commande shell via des séparateurs de ligne après un bloc `{ comment }`. |

### 2.2. Vulnérabilités Élevées (Haute Priorité)

| Paquet | Versions | CVE / Advisory | Gravité | Risque & Vecteur d'Attaque |
| :--- | :--- | :--- | :--- | :--- |
| **sharp** | `<0.35.5` | GHSA-wq5f-xc86-pv6w (CVE-2026-96889) | **ÉLEVÉE** | Faille mémoire dans `librsvg` intégrée pouvant mener à un crash serveur ou exécution de code lors du traitement de formats SVG malformés. |
| **undici** | `4.0.0 - 6.29.0` | GHSA-4qv8-2p9j-2685, GHSA-4w7w-5xrc-9h26 | **ÉLEVÉE** | Déni de service par consommation mémoire non bornée (EventSource parser) et fuite de données inter-utilisateurs via redirections cross-origin. |
| **zod** | `<4.6.6` (actuel `^4.1.12`) | GHSA-mrhx-c7xc-m3w5 | **ÉLEVÉE** | Épuisement mémoire / DoS sur la validation de collections volumineuses avec contraintes. |
| **webpack-dev-middleware** | `<7.4.5` | GHSA-g84c-rxfj-3j2c | **ÉLEVÉE** | Path traversal via `publicPath` non terminé par un slash. |
| **vite** | `<=6.4.3` | GHSA-9jrq-w75r-8gcw, GHSA-r825-w8mm-5pxw | **ÉLEVÉE** | Exécution de commandes et cross-origin script execution via middleware personnalisé. |
| **source-map-js** | `1.0.0 - 1.2.1` | GHSA-68fv-2mgg-jv7q | **ÉLEVÉE** | DoS de l'event loop via décalages de section source-map indexés. |

### 2.3. Failles Next-Auth & Nodemailer (Risque Opérationnel)

- **nodemailer** (`^7.0.13`) dépendance transitive de `next-auth@4.24.15` : 7 vulnérabilités GHSA (GHSA-39m8-27wv-hr27, GHSA-8c8m-p3m8-cjg8, GHSA-m943-m63m-w62w, GHSA-85x2-q946-cp5h, etc.) liées au parsing d'adresses email (injection de destinataires non souhaités via des caractères `;` ou `>` dans les commentaires/noms d'affichage).
- Risque dans le projet : L'envoi d'emails transactionnels (notifications SACEM, réinitialisation de mot de passe) transmet des emails construits dynamiquement. Un attaquant qui s'enregistre avec un nom ou une adresse artisanale pourrait potentiellement altérer les destinataires SMTP.

---

## 3. Analyse Approfondie du Code Applicatif

### 3.1. Gestion des Secrets & Identifiants Partenaires (CRITIQUE)

- **Fichier impacté** : `src/prisma/schema.prisma` (modèle `TicketingSystem`), `src/server/routers/ticketing.ts`.
- **Problème** : Les champs `apiAccessKey` et `apiSecretKey` sont stockés sous forme de texte brut (`String?`) dans PostgreSQL.
- **Impact** : En cas de compromission de la base (ou d'erreur d'un utilisateur du connecteur Metabase ayant accès à la table `TicketingSystem`), l'ensemble des clés d'accès aux billetteries externes (Billetweb, Shotgun, HelloAsso, Mapado, Soticket, Supersoniks) est compromis.
- **Recommandation** : Implémenter un chiffrement applicatif au repos (chiffrement symétrique AES-256-GCM via une clé serveur `ENCRYPTION_SECRET_KEY`) avant insertion/mise à jour, et déchiffrer uniquement à l'instanciation des clients dans `src/core/ticketing/instance.ts`.

### 3.2. Vulnérabilité Potentielle de SSRF (Server-Side Request Forgery)

- **Fichiers impactés** : `src/core/ticketing/supersoniks.ts`, `src/models/actions/ticketing.ts`.
- **Problème** : Pour Supersoniks et SoTicket, l'utilisateur fournit son domaine en tant qu'access key :
  ```typescript
  const baseUrl = new URL(`https://${accessKey}/api/v2/`);
  ```
  La validation Zod utilise :
  ```typescript
  const domainNameRegexp = /^(?!:\/\/)([a-zA-Z0-9-_]{1,63}\.)+[a-zA-Z]{2,63}$/;
  ```
- **Faiblesse** : Cette regex autorise n'importe quel nom de domaine public ou privé (par ex. `sub.internal.domain.fr` ou un domaine public résolvant sur `127.0.0.1` ou `169.254.169.254` via DNS Rebinding). Le client effectue ensuite des appels `fetch` vers cette cible.
- **Recommandation** : Valider et interdire la résolution d'adresses IP privées/locales (RFC 1918 / loopback / link-local) avant d'émettre des requêtes sortantes, ou passer par un proxy de sortie filtrant.

### 3.3. Endpoints d'Upload et Contrôle des Fichiers

- **Fichier impacté** : `src/pages/api/upload/[[...file]].ts`.
- **Analyse des défenses en place** :
  - Utilise `tus` avec `FileStore` local temporaire puis insertion dans PostgreSQL (`Attachment.value`).
  - Validation du type MIME via `magic-bytes.js` (`filetypeinfo(attachmentContent)`).
  - Contrôle d'authentification pour les pièces jointes privées via session `next-auth`.
  - Nettoyage des fichiers temporaires via `fs.unlink`.
- **Faiblesses identifiées** :
  1. **Stockage temporaire local** : Si le serveur redémarre ou subit un crash pendant l'upload, des fichiers peuvent persister dans `.files/` jusqu'au prochain déploiement.
  2. **Taille mémoire et DoS** : `fs.readFileSync(attachmentPath)` charge l'intégralité du fichier en mémoire RAM avant de l'insérer dans Prisma. Si plusieurs utilisateurs envoient simultanément des pièces jointes à la limite autorisée, le conteneur Node (512 Mo RAM) risque un Out-Of-Memory (OOM crash).
  3. **Stockage PostgreSQL ByteA** : Le TAD et le README confirment que les fichiers sont stockés dans la base Postgres. Au-delà d'un usage modéré, cela dégrade les performances des sauvegardes, gonfle le cache mémoire PostgreSQL et sature le pool de connexions Scalingo.

### 3.4. Téléchargement Sécurisé de Fichiers (`/api/file/[fileId]`)

- **Fichier impacté** : `src/pages/api/file/[fileId].ts`.
- **Mécanisme** : Vérification d'un JWT signé (`HS256`) avec le secret `FILE_AUTH_SECRET`.
- **Sécurité** :
  - Les liens ont une durée de vie limitée (15 minutes).
  - Le claim `urn:claim:file_id` est vérifié contre le `fileId` demandé pour empêcher qu'un token valide pour le fichier A serve à télécharger le fichier B.
  - La vérification d'expiration est doublement contrôlée (`jwtVerify` + vérification manuelle de `exp`).
- **Point d'attention** : `res.setHeader('Cache-Control', 'private, max-age=900')` autorise la mise en cache navigateur pendant 15 minutes même si l'autorisation change. Pour des documents hautement confidentiels (contrats de cession, déclarations financières), privilégier `private, no-cache, no-store`.

### 3.5. Endpoints de Maintenance et Contrôle d'Accès

- **Fichiers impactés** : `src/pages/api/maintenance/cli.ts`, `src/pages/api/maintenance/jobs/replay.ts`.
- **Authentification** : Protégé par l'en-tête `X-API-KEY` comparé à `process.env.MAINTENANCE_API_KEY`.
- **Faiblesses** :
  1. **Timing attack sur la clé** : Comparaison par égalité stricte `maintenanceApiKey === apiKeyHeader` dans `src/utils/maintenance.ts` au lieu d'une comparaison à temps constant `crypto.timingSafeEqual`.
  2. **Commander.js dynamique** : Dans `cli.ts`, la commande reçue est découpée (`body.command.split(' ')`) et exécutée via `cloneProgram.parseAsync`. Bien que `commandsPrefixesWhitelist` soit actuellement vide (rejetant tout appel), toute ouverture future sans assainissement strict des arguments peut entraîner des comportements imprévus ou l'écrasement de données critiques.

### 3.6. Authentification NextAuth & Sessions

- **Fichier impacté** : `src/pages/api/auth/[...nextauth].ts`.
- **Stratégie** : JWT stateless (`strategy: 'jwt'`).
- **Points forts** :
  - `bcrypt.compare` utilisé pour la validation des mots de passe.
  - Vérification du statut de l'utilisateur (`CONFIRMED`).
  - Redirections vérifiées pour contrer les Open Redirects (`new URL(url).origin === baseUrl`).
- **Points faibles** :
  - Absence de politique de lockout ou de limitation de débit (Rate Limiting) sur les tentatives de connexion échouées (`authorize`). Une attaque par force brute ou credential stuffing est possible sur `/api/auth/callback/credentials`.
  - La mise à jour de `lastActivityAt` est exécutée de façon asynchrone sans await à chaque revalidation de token : en cas de forte charge, cela génère une requête `prisma.user.update` supplémentaire par requête API.

---

## 4. Audit des Tiers et Partenaires

### 4.1. Organismes de Déclaration (SACD & SACEM)

| Tiers | Mode d'Intégration | Données Échangées | Risques Identifiés & Sécurité |
| :--- | :--- | :--- | :--- |
| **SACD** | API SOAP/HTTP REST (`auto-access.json`, `ExploitFileTransmitETicketingDeclarationWS`) | Données billetterie, recettes, dépenses de cession, SIREN diffuseur/producteur | Authentification via hash MD5 du mot de passe (`this.md5Password`) transmis en POST. Les communications doivent obligatoirement se faire en HTTPS strict. Risque d'erreur métier silencieuse si l'API SACD renvoie du JSON dans un statut HTTP 200 au lieu de XML (contournement géré dans le code mais fragile). |
| **SACEM** | Génération de PDF et transmission par email via Nodemailer (`DeclarationToSacemAgency`) | Synthèse financière, nombre de billets, répartition des jauges, adresses, contacts | Le routage vers l'agence dépend de la table `SacemAgency` issue d'un fichier CSV (`./data/sacem-agencies.csv`). Si la base n'est pas synchronisée ou si le code postal est erroné, la déclaration peut être envoyée à la mauvaise agence ou échouer silencieusement. |

### 4.2. Connecteurs de Billetterie

| Billetterie | Protocole & Authentification | Flux de Données | Vulnérabilités & Risques |
| :--- | :--- | :--- | :--- |
| **Billetweb** | Clé publique + Clé secrète HTTP query params / headers | Événements, tarifs, catégories, participants | Rate limiting strict (90 req/min). Géré via `Bottleneck` mais peut causer des timeouts Scalingo (60s). |
| **HelloAsso** | OAuth2 Client Credentials (Token éphémère) | Formulaires, collectes, paiements | Gestion correcte du cycle de vie du token OAuth2. Nécessite une gestion rigoureuse des scopes. |
| **Mapado** | API Token Bearer HTTP | Séries, événements, tickets | Utilise le SDK `@hey-api/client-fetch`. |
| **Shotgun** | API Access Key + Secret | Événements, ventes | Requêtes directes paginées. |
| **Supersoniks** | Clé secrète + domaine client dynamique | Bordereaux de recettes, événements | Risque de SSRF sur le domaine client (`accessKey`), requête chargeant 100 000 000 items d'un coup (risque de crash mémoire). |
| **SoTicket** | Clé secrète + domaine client dynamique | Clôtures et recettes | Même architecture que Supersoniks. |

### 4.3. Services d'Infrastructure et d'Observabilité

| Service | Rôle | Flux & Authentification | Points d'Attention Sécurité |
| :--- | :--- | :--- | :--- |
| **Scaleway & Brevo** | Envoi d'emails transactionnels (primaire et fallback) | SMTP TLS (ports 465 / 587) | Les credentials SMTP doivent être impérativement distincts entre dev et prod. Risque d'usurpation de réputation de domaine si les enregistrements SPF/DKIM/DMARC ne sont pas rigoureusement surveillés. |
| **Crisp** | Chat de support utilisateur | Script client + signature cryptographique HMAC backend | La signature HMAC (`CRISP_SIGNING_SECRET_KEY`) empêche l'usurpation d'identité sur le chat en production. Néanmoins, le CSP requiert l'autorisation de multiples domaines Crisp (`*.crisp.chat`, `wss://client.relay.crisp.chat/`). |
| **Sentry** | Suivi d'erreurs | DSN client et serveur | Risque de fuite de données nominatives ou de tokens dans les traces. Un filtre `rrweb` et une exclusion des `BusinessError` sont en place, mais un audit régulier des données capturées est indispensable. |
| **Metabase** | Dashboarding / Business Intelligence | Connexion directe PostgreSQL (schéma isolé `metabase` + utilisateur dédié restreint) | Le document TAD mentionne l'absence de 2FA natif sur Metabase. L'accès direct à la base de données via un utilisateur restreint limite l'impact mais requiert une isolation stricte des colonnes contenant des mots de passe ou des clés d'API. |

---

## 5. Synthèse des Vulnérabilités et Niveau de Risque

```
+---------------------------------------------------------------------------------------+
| SÉVÉRITÉ   | VULNÉRABILITÉ / ANOMALIE                              | RISQUE PRINCIPAL |
+---------------------------------------------------------------------------------------+
| CRITIQUE   | Clés d'API billetteries stockées en clair en base    | Fuite de données |
| CRITIQUE   | Failles Storybook & shell-quote (CVE RCE/Injection)  | Exécution code   |
| ÉLEVÉE     | SSRF potentiel sur connecteurs Supersoniks/SoTicket   | Accès interne    |
| ÉLEVÉE     | Absence de Rate Limiting sur `/auth/sign-in`         | Force brute      |
| ÉLEVÉE     | Déni de service mémoire (Chargement gros fichiers)   | OOM crash        |
| MOYENNE    | Comparaison non sécurisée de MAINTENANCE_API_KEY      | Timing attack    |
| MOYENNE    | CSP avec 'unsafe-inline' et 'unsafe-eval'            | Contournement CSP|
| MOYENNE    | Dépendances vulnérables (undici, zod, sharp)         | DoS / Stabilité  |
+---------------------------------------------------------------------------------------+
```
