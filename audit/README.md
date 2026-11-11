# Dossier d'Audit & Documentation Technique — Assistant Déclaration

Bienvenue dans le dossier d'audit et de transmission technique pour le projet **Assistant Déclaration** (beta.gouv.fr).
Ce dossier a été conçu pour vous fournir une vue à 360°, claire, exhaustive et actionnable afin de reprendre la maintenance et le développement du projet en toute sérénité.

---

## 🗂️ Sommaire & Structure du Dossier

Le dossier est découpé de manière modulaire :

```
audit/
├── README.md                           # Ce document d'accueil et de synthèse
├── 01-audit-securite.md                # Audit de sécurité 360° (code, failles, CVEs, tiers)
├── 02-bonnes-pratiques-maintenance.md  # Audit qualité, dette technique et architecture
├── 03-plan-action-priorise.md          # Feuille de route classée par urgence (P0 à P3)
├── 04-documentation-fonctionnelle.md   # Documentation métier globale & modèle de données
└── features/                           # Documentation modulaire par feature
    ├── 01-authentification-utilisateurs.md
    ├── 02-organisations-collaborateurs.md
    ├── 03-billetteries-synchronisation.md
    ├── 04-gestion-evenements-series.md
    ├── 05-declarations-sacd-sacem.md
    ├── 06-fichiers-pieces-jointes.md
    └── 07-taches-de-fond-maintenance.md
```

---

## 🚀 Synthèse Exécutive pour le Nouveau Mainteneur

### 1. Qu'est-ce que l'application ?
Une plateforme nationale aidant les entrepreneurs du spectacle vivant (théâtre, concerts, danse, cirque, humour) à pré-remplir et télétransmettre leurs déclarations administratives obligatoires de droits d'auteur vers la **SACEM** et la **SACD**, en important automatiquement les données de leur logiciel de billetterie (Billetweb, Shotgun, HelloAsso, Mapado, Supersoniks, SoTicket).

### 2. Les 3 Urgences Immédiates (P0)
1. **Sécuriser les clés d'API des billetteries** : Actuellement stockées en clair dans la table PostgreSQL `TicketingSystem`. Il faut implémenter un chiffrement applicatif au repos (AES-256-GCM).
2. **Corriger les vulnérabilités de dépendances critiques** : En particulier Storybook (RCE / code injection dev server) et `shell-quote` (injection shell).
3. **Sécuriser les connecteurs avec domaine dynamique (Supersoniks / SoTicket)** : Valider que le domaine client ne résout pas vers des adresses IP privées ou Cloud Metadata (SSRF).

### 3. Les 3 Chantiers de Maintenabilité (P1/P2)
1. **Écrire des tests d'intégration sur la transmission** : Le routeur `declaration.ts` (génération XML SACD, PDF SACEM et emails) n'a pas de test d'intégration automatisé.
2. **Découper le composant monolithe `DeclarationPage.tsx`** : 1 671 lignes de React concentrant toute la complexité du formulaire unique.
3. **Planifier la migration des pièces jointes vers du stockage S3** : Actuellement stockées en binaire `ByteA` dans PostgreSQL, ce qui surchargera la base à moyen terme.

---

## 🛠️ Démarrage Rapide pour Développer

```bash
# 1. Lancer l'infrastructure locale (PostgreSQL + Mailcatcher)
docker-compose up -d

# 2. Initialiser la base de données
npm run db:migration:deploy
npm run db:schema:compile
npm run db:seed

# 3. Lancer Storybook (recommandé pour travailler sur l'UI avec mocks complets)
npm run dev:storybook

# 4. Lancer l'application web complète
npm run dev:app
```

Pour toute question détaillée sur une feature ou une règle métier, consultez le fichier correspondant dans le sous-dossier `audit/features/`.
