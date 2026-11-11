# Feature 07 : Tâches de Fond & Maintenance

## 1. Description Fonctionnelle
Fournit les mécanismes asynchrones d'arrière-plan pour le traitement des opérations différées (nettoyage des fichiers orphelins, purge des sessions expirées), ainsi que les outils d'administration et de maintenance (rejeu des jobs échoués, commandes CLI sans accès shell direct).

---

## 2. File d'Attente et Planification avec `pg-boss`

Plutôt que d'introduire une dépendance externe comme Redis ou RabbitMQ, le projet utilise **`pg-boss`** (`src/server/queueing/client.ts`), qui transforme PostgreSQL en gestionnaire de files d'attente transactionnelles :

### 2.1. Avantages de ce Choix Architectural
- **Atomisme et Persistance** : Les jobs résident dans le même moteur de données que l'application (schéma `pgboss`), simplifiant les sauvegardes globales.
- **Gestion de la Concurrence** : Verrous SQL natifs (`SELECT ... FOR UPDATE SKIP LOCKED`) garantissant qu'une même tâche n'est pas exécutée deux fois simultanément lors d'un déploiement multi-instances sur Scalingo.
- **Monitoring Intégré** : Suivi des états de chaque job (`created`, `active`, `completed`, `failed`, `cancelled`, `archived`) directement consultable via DBeaver.

### 2.2. Workers et Planification (`src/server/queueing/schedule.ts`)
- Initialisé au démarrage du conteneur Node.js via le script d'entrée `start-and-wait-to-init.sh` et `instrumentation.ts`.
- Worker `clean-pending-uploads` (`src/server/queueing/workers/clean-pending-uploads.ts`) :
  - Fréquence : Planifié quotidiennement via cron expression.
  - Action : Supprime de la table `Attachment` les fichiers restant en statut `PENDING_UPLOAD` depuis plus de 48 heures.

---

## 3. Endpoints HTTP de Maintenance

Pour faciliter l'administration sur Scalingo sans ouvrir de shell interactif, deux endpoints dédiés sont exposés :

### 3.1. Rejeu de Tâches : `POST /api/maintenance/jobs/replay`
- **Utilité** : Lorsqu'une tâche de fond échoue (ex. coupure réseau temporaire avec un tiers), elle est marquée `failed`. Cet endpoint permet de la relancer proprement avec les mêmes paramètres tout en traçant l'historique du rejeu via des métadonnées `__maintenance__`.
- **Paramètres attendus** (validés par Zod) :
  ```json
  {
    "topicName": "clean-pending-uploads",
    "jobId": "e5c6b412-1f48-43d9-95a7-47b7a149f159"
  }
  ```
- **Sécurité** : Requiert l'en-tête HTTP `X-API-KEY` égal à `MAINTENANCE_API_KEY`.

### 3.2. Exécution de Commandes CLI : `POST /api/maintenance/cli`
- **Utilité** : Permet de déclencher les sous-commandes de maintenance CLI (synchronisation SACD/SACEM, migrations de données) via une simple requête HTTP sans accès SSH au serveur.
- **Fonctionnement** :
  - Reçoit `{ "command": "sacem agency sync" }`.
  - Valide la commande par rapport à une liste blanche (`commandsPrefixesWhitelist`).
  - Exécute le parseur asynchrone de `Commander.js` (`program.parseAsync`).
- **Sécurité** : Protégé par l'en-tête `X-API-KEY`. (Note : `commandsPrefixesWhitelist` est actuellement vide par défaut pour interdire tout appel non explicitement activé).

---

## 4. Outils CLI & Scripts de Maintenance Locale

Le projet fournit une suite de commandes exécutables via `npm run cli` (`src/cli/index.ts`) :
- `sacd agency sync` : Synchronise la table `SacdAgency` à partir du fichier `./data/sacd-agencies.csv`.
- `sacem agency sync` : Synchronise la table `SacemAgency` à partir du fichier `./data/sacem-agencies.csv`.
- Scripts de base de données sous `src/scripts/` :
  - `init_metabase.sql` : Création du schéma isolé et des rôles pour le serveur Metabase.
  - `init_metabase_connector.sql` : Création de l'utilisateur PostgreSQL restreint pour les dashboards de métriques.

---

## 5. Points d'Attention & Maintenance
- **Vérification cryptographique de la clé de maintenance** : Utiliser `crypto.timingSafeEqual` (action P2 du plan) pour éliminer le risque d'attaque par timing.
- **Rétention des logs pg-boss** : `pg-boss` archive ses jobs terminés. Configurer une politique de purge automatique pour éviter que la table `pgboss.archive` ne croisse indéfiniment.
