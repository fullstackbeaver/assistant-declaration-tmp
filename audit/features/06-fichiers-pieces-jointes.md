# Feature 06 : Fichiers & Pièces Jointes

## 1. Description Fonctionnelle
Permet aux déclarants de téléverser des documents justificatifs requis par les organismes (contrats artistiques, programmes d'œuvres jouées, états de billetterie certifiés), de prévisualiser ces documents dans l'interface et de les associer aux séries de représentations.

---

## 2. Architecture Technique du Téléversement

L'application implémente le protocole standard ouvert **Tus** (Resumable File Upload) combiné à un stockage en base PostgreSQL :

```
[ Navigateur / Uppy Client ]
       |
       | (Protocole Tus en morceaux - Chunked Upload)
       v
[ /api/upload - @tus/server + FileStore local ]
       |
       | (Validation MIME 'magic-bytes.js' + contrôle session NextAuth)
       v
[ Sauvegarde finale dans PostgreSQL : Attachment (ByteA) ]
       |
       | (Suppression du fichier temporaire local via fs.unlink)
```

### 2.1. Client Frontend
- Librairie `@uppy/core` avec plugin `@uppy/tus` et `@uppy/drag-drop` (`src/components/uploader/`).
- Prise en charge du téléversement par morceaux, reprise automatique sur perte de connexion et barre de progression.

### 2.2. Serveur Tus (`src/pages/api/upload/[[...file]].ts`)
- Définition `bodyParser: false` pour laisser le serveur Tus gérer le streaming des octets.
- Répertoire tampon local : `./.files/`.
- Hook `onUploadCreate` :
  - Vérification de la présence des métadonnées (`kind`, `type`, `size`).
  - Contrôle de la taille maximale autorisée selon le type de document (`attachmentKind.maxSize`).
  - Filtrage des extensions et types MIME autorisés (`attachmentKind.allowedFileTypes`).
  - Contrôle d'authentification utilisateur via cookies NextAuth si `requiresAuthToUpload: true`.
- Hook `onUploadFinish` :
  - Lecture du fichier local et analyse des signatures binaires réelles (Magic Bytes) via `magic-bytes.js`.
  - Insertion du document dans la table `Attachment` avec le statut `PENDING_UPLOAD`.
  - Suppression immédiate du fichier local via `fs.unlink`.

---

## 3. Sécurité du Téléchargement (`/api/file/[fileId]`)

Pour consulter ou télécharger un fichier privé :
1. Le client ne dispose pas d'une URL publique statique.
2. Le backend génère une URL signée temporaire via `generateSignedAttachmentLink` (`src/server/routers/common/attachment.ts`).
3. Le lien contient un token JWT signé (`HS256`) avec le secret `FILE_AUTH_SECRET` et les claims :
   - `urn:claim:file_id` : L'UUID strict du fichier autorisé.
   - `iat` / `exp` : Date d'émission arrondie au modulo de 10 minutes et expiration à 15 minutes (permettant la mise en cache navigateur contrôlée).
4. Lors de la requête GET `/api/file/[fileId]?token=...` :
   - L'endpoint vérifie la signature cryptographique et la non-expiration du token.
   - Si valide, il envoie les en-têtes HTTP sécurisés (`Content-Type`, `Content-Disposition: inline`) et délivre le contenu en flux Node.js (`Readable.from(attachment.value).pipe(res)`).

---

## 4. Modèles de Données Impliqués
- `Attachment` :
  - `kind` : `EVENT_SERIE_DOCUMENT`.
  - `contentType` : Type MIME validé (ex. `application/pdf`).
  - `value` : Contenu binaire stocké dans PostgreSQL (`Bytes @db.ByteA`).
  - `size` : Taille en octets.
  - `status` : `PENDING_UPLOAD` (téléversé mais non encore associé), `VALID` (associé à une série), `EXPIRED` (à purger).
- `AttachmentsOnEventSeries` :
  - Table de liaison entre `EventSerie` et `Attachment`.
  - `type` : Enum `EventSerieAttachmentType` (`ARTISTIC_CONTRACT`, `PERFORMED_WORK_PROGRAM`, `REVENUE_STATEMENT`, `OTHER`).

---

## 5. Cycle de Vie et Nettoyage des Orphelins
1. Lorsqu'un utilisateur téléverse un fichier, il est créé en statut `PENDING_UPLOAD`.
2. Lors de la soumission du formulaire de série, la fonction `formatSafeAttachmentsToProcess` passe le statut à `VALID`.
3. Si un utilisateur téléverse un fichier mais quitte la page sans enregistrer, le fichier reste orphelin en base.
4. Une tâche de fond planifiée (`src/server/queueing/workers/clean-pending-uploads.ts`) s'exécute régulièrement pour supprimer de PostgreSQL les fichiers en `PENDING_UPLOAD` vieux de plus de quelques jours.

---

## 6. Points d'Attention & Maintenance
- **Anti-Pattern ByteA en base de données** : Le stockage de fichiers binaires directement dans PostgreSQL sature la RAM et ralentit les sauvegardes Scalingo. Une migration vers du stockage d'objets (S3 / Scalingo Object Storage) est planifiée en priorité P3.
- **Empreinte mémoire temporaire** : `fs.readFileSync` dans `onUploadFinish` charge l'intégralité du fichier en RAM. Surveiller la consommation mémoire en cas de téléversement simultané de plusieurs pièces jointes volumineuses.
