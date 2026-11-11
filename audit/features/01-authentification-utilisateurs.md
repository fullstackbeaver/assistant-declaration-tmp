# Feature 01 : Authentification & Gestion des Utilisateurs

## 1. Description Fonctionnelle
Permet aux entrepreneurs et administrateurs de créer un compte, de valider leur adresse email par un token sécurisé, de s'authentifier par couple email / mot de passe, de réinitialiser un mot de passe oublié et de gérer leur session de travail.

## 2. Architecture Technique
- **Framework d'Authentification** : NextAuth.js v4 (`src/pages/api/auth/[...nextauth].ts`).
- **Stratégie de Session** : JWT stateless (`session: { strategy: 'jwt' }`).
- **Adaptateur de Base de Données** : `@next-auth/prisma-adapter`.
- **Hachage des Mots de Passe** : `bcrypt` (10 rounds par défaut) avec table dédiée `UserSecrets` isolée de la table principale `User`.
- **Protection CSRF** : Gérée nativement par NextAuth via tokens de formulaire et cookies sécurisés SameSite.

## 3. Modèles de Données Impliqués
- `User` : Identité publique (`id`, `email`, `firstname`, `lastname`, `status`, `lastActivityAt`).
- `UserSecrets` : `passwordHash` (relation 1-to-1 avec `User`).
- `VerificationToken` : Tokens temporaires pour la confirmation d'inscription (`SIGN_UP`) et réinitialisation de mot de passe (`RESET_PASSWORD`).
- `Admin` : Rôle d'administration transverse.

## 4. Endpoints & Procédures tRPC
- `src/pages/api/auth/[...nextauth].ts` :
  - `CredentialsProvider` : Vérification des identifiants et validation du statut `CONFIRMED`.
  - Callback `jwt` : Injection des informations utilisateur (`id`, `email`, nom, prénom) dans le token JWT.
  - Callback `session` : Exposition du profil utilisateur et horodatage asynchrone de `lastActivityAt`.
- Routeur tRPC `authRouter` (`src/server/routers/auth.ts`) :
  - `register` : Inscription d'un nouvel utilisateur, hachage du mot de passe, création du token et envoi de l'email de confirmation via Nodemailer.
  - `confirmEmail` : Validation du token `VerificationToken` et passage du statut à `CONFIRMED`.
  - `requestPasswordReset` : Génération d'un token à usage unique et envoi de l'email avec lien de réinitialisation.
  - `resetPassword` : Mise à jour du hash du mot de passe dans `UserSecrets` et invalidation du token.

## 5. Points d'Attention & Maintenance
- **Absence de Rate Limiting** : Les tentatives de connexion répétées sur `/api/auth/callback/credentials` ne sont pas limitées (risque de brute force).
- **Mise à jour `lastActivityAt`** : Déclenchée sans `await` à chaque vérification de session. En cas de panne de base de données, l'erreur est logguée (`console.warn`) sans interrompre la session.
