# Feature 03 : Billetteries & Synchronisation des Recettes

## 1. Description Fonctionnelle
Permet de connecter l'organisation aux logiciels de billetterie du marché afin d'importer automatiquement le catalogue de spectacles, les dates de représentations et les volumes de billetterie (billets payants, exonérés, montants HT et TTC).

## 2. Billetteries Supportées
1. **BILLETWEB** : Connexion via API access key & secret key. Rate limiting strict géré par `Bottleneck` (90 requêtes/minute avec fenêtre glissante étendue).
2. **SHOTGUN** : Connexion par clé d'API. Pagination des événements et des jauges.
3. **HELLOASSO** : Connexion OAuth2 Client Credentials (génération de token JWT temporaire).
4. **MAPADO** : Connexion via Token API et SDK généré OpenAPI `@hey-api/client-fetch`.
5. **SUPERSONIKS** : Connexion via Secret Key et domaine client dynamique.
6. **SOTICKET** : Connexion sur le même principe que Supersoniks.
7. **MANUAL** : Mode sans connecteur externe permettant la saisie et gestion manuelle intégrale des événements dans l'application.

## 3. Architecture Technique de Synchronisation
- **Streaming HTTP anti-timeout** (`src/pages/api/synchronize-data-from-ticketing-systems.ts`) :
  - Scalingo coupe les requêtes HTTP après **60 secondes d'inactivité** (H12 request timeout).
  - Pour éviter ce blocage sur les catalogues volumineux, l'endpoint renvoie des paquets de ping réguliers toutes les 10 secondes :
    ```
    ping:true\n
    ...
    data:done\n
    ```
  - Le frontend consomme ce flux textuel et affiche une barre de progression interactive.
- **Orchestration métier** (`src/core/ticketing/synchronize.ts`) :
  - Pour chaque connecteur actif, instanciation du client via `getTicketingSystemClient`.
  - Comparaison différentielle des séries et des événements existants vs importés.
  - Sauvegarde transactionnelle dans PostgreSQL.
  - Préservation des modifications manuelles : si l'utilisateur a édité manuellement un événement (`lastManualTicketingDataUpdateAt`), la synchronisation n'écrase pas ses corrections sauf demande explicite.

## 4. Modèles de Données Impliqués
- `TicketingSystem` :
  - `name` : Enum (`BILLETWEB`, `SHOTGUN`, `MAPADO`, etc.).
  - `apiAccessKey`, `apiSecretKey` : Identifiants d'accès (actuellement stockés en clair).
  - `lastSynchronizationAt` : Horodatage du dernier succès.
  - `lastProcessingError` : Message d'erreur en cas d'échec de synchronisation.

## 5. Endpoints & Procédures tRPC
- `src/server/routers/ticketing.ts` :
  - `connectTicketingSystem` : Vérification des droits, test immédiat de connexion via `ticketingSystemClient.testConnection()`, création en base.
  - `listTicketingSystems` : Liste des billetteries connectées (masque les clés secrètes pour le frontend).
  - `updateTicketingSystem` : Mise à jour des identifiants et re-test de connexion.
  - `disconnectTicketingSystem` : Soft-delete (`deletedAt`) et purge immédiate des clés d'API.
- Endpoint de streaming : `POST /api/synchronize-data-from-ticketing-systems`.

## 6. Points d'Attention & Maintenance
- **Clés stockées en clair** : Priorité P0 de remédiation cryptographique.
- **SSRF sur Supersoniks / SoTicket** : Priorité P1 pour valider que le domaine fourni ne résout pas vers des IPs privées ou Cloud Metadata.
- **Environnements non-production** : Par défaut, si `APP_MODE !== 'prod'`, le système utilise `MockTicketingSystemClient` pour ne pas appeler les vraies APIs partenaires, sauf si l'UUID de l'utilisateur est listé dans `DISABLE_TICKETING_SYSTEM_MOCK_FOR_USER_IDS`.
