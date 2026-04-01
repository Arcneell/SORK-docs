# Authentification

SORK intègre un système d'authentification multi-utilisateurs basé sur **SQLite** et **JWT**. Aucune base de données externe n'est requise : tout est stocké dans un fichier unique au sein du container.

## Vue d'ensemble

| Composant | Technologie |
|-----------|-------------|
| Stockage utilisateurs | SQLite (`/workspace/.sork/auth.db`) |
| Hachage mots de passe | bcrypt (passlib) |
| Tokens d'accès | JWT HS256 (python-jose) |
| Clé de signature | Auto-générée et persistée dans `/workspace/.sork/jwt_secret.key` |

## Rôles

SORK définit deux rôles :

### Administrateur (`admin`)

Accès complet à toutes les fonctionnalités :

- Création, suppression, modification de conteneurs, images, volumes, réseaux, stacks
- Exécution de commandes dans les conteneurs (`exec`)
- Gestion du manifeste (édition, validation, doctor)
- Backup / restauration de la configuration
- Prune des ressources Docker
- Gestion des utilisateurs (CRUD)
- Gestion des webhooks, registries, templates, proxy, notifications

### Technicien (`technicien`)

Accès en lecture et actions opérationnelles limitées :

- Consultation de tous les tableaux de bord, logs, stats, inspections
- Start / stop / restart des services et conteneurs
- Acquittement des alertes et incidents
- Réconciliation d'une application individuelle
- Lecture des variables d'environnement des stacks
- Navigation dans les volumes (lecture seule)

Les actions destructives (suppression, kill, prune, exec, deploy, backup/restore, édition manifeste) sont **interdites** aux techniciens — le backend retourne `403 Forbidden`.

## Compte par défaut

Au premier démarrage, si aucun utilisateur n'existe dans la base, SORK crée automatiquement :

- **Nom d'utilisateur** : `admin`
- **Mot de passe** : `admin` (ou la valeur de `SORK_ADMIN_PASSWORD`)
- **Rôle** : `admin`

!!! warning "Changement obligatoire"
    Après la première connexion avec `admin/admin`, l'interface force le changement de mot de passe avant de pouvoir accéder au tableau de bord.

## Variables d'environnement

| Variable | Description | Défaut |
|----------|-------------|--------|
| `SORK_ADMIN_PASSWORD` | Mot de passe initial du compte admin | `admin` |
| `SORK_JWT_SECRET` | Clé secrète pour signer les tokens JWT. Si non défini, une clé est auto-générée et persistée | Auto-généré |
| `SORK_JWT_EXPIRE_MINUTES` | Durée de validité des tokens JWT en minutes | `480` (8 heures) |
| `SORK_UI_TOKEN` | Token legacy (rétro-compatibilité). **Déprécié** — migrer vers les comptes utilisateurs | - |

## Sécurité

### Protection contre les attaques par force brute

Le endpoint de connexion intègre un rate limiter :

- **5 tentatives échouées** dans une fenêtre de **5 minutes** déclenchent un blocage
- Le blocage dure **10 minutes** pour l'adresse IP concernée
- Les tentatives échouées sont journalisées dans l'audit

### Invalidation des tokens

Les tokens JWT incluent un horodatage (`pw_ts`) correspondant au `updated_at` de l'utilisateur. Après un changement de mot de passe :

- Tous les anciens tokens sont automatiquement invalidés
- Un nouveau token est émis au moment du changement
- Les sessions actives sur d'autres appareils sont déconnectées

### Protection contre les attaques par timing

Le endpoint de connexion exécute toujours une vérification bcrypt, même si l'utilisateur n'existe pas. Cela empêche l'énumération des noms d'utilisateurs par mesure du temps de réponse.

### Audit

Tous les événements d'authentification sont tracés dans l'audit :

- `login` — connexion réussie (avec IP)
- `login_failed` — tentative échouée (avec IP et nom d'utilisateur)
- `login_disabled` — tentative sur un compte désactivé
- `password_change` — changement de mot de passe
- `user_create` / `user_update` / `user_delete` — gestion des comptes

## API REST

### Endpoints publics

| Méthode | Chemin | Description |
|---------|--------|-------------|
| `POST` | `/api/auth/login` | Connexion (retourne un JWT) |

### Endpoints authentifiés

| Méthode | Chemin | Rôle requis | Description |
|---------|--------|-------------|-------------|
| `GET` | `/api/auth/me` | Tout rôle | Informations de l'utilisateur courant |
| `POST` | `/api/auth/change-password` | Tout rôle | Changer son mot de passe |

### Endpoints admin

| Méthode | Chemin | Description |
|---------|--------|-------------|
| `GET` | `/api/auth/users` | Lister les utilisateurs |
| `POST` | `/api/auth/users` | Créer un utilisateur |
| `PUT` | `/api/auth/users/{id}` | Modifier un utilisateur (rôle, actif, mot de passe) |
| `DELETE` | `/api/auth/users/{id}` | Supprimer un utilisateur |

### Exemple de connexion

```bash
# Connexion
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "votre_mot_de_passe"}'

# Réponse
# {"ok": true, "token": "eyJ...", "user": {"username": "admin", "role": "admin"}}

# Utilisation du token
curl http://localhost:8080/api/state \
  -H "Authorization: Bearer eyJ..."
```

## Gestion des utilisateurs (interface)

Les administrateurs peuvent gérer les utilisateurs depuis **Paramètres > Utilisateurs** :

- Créer de nouveaux comptes (admin ou technicien)
- Modifier le rôle ou l'état actif/inactif
- Réinitialiser le mot de passe d'un utilisateur
- Supprimer un utilisateur (sauf soi-même)

## Rétro-compatibilité

L'ancien mécanisme `SORK_UI_TOKEN` (token unique partagé) reste fonctionnel en mode fallback. Si un token ne peut pas être décodé en JWT valide, SORK vérifie s'il correspond à `SORK_UI_TOKEN` et accorde un accès admin temporaire.

!!! warning "Dépréciation"
    Ce mécanisme est **déprécié**. Un avertissement est journalisé à chaque utilisation. Il sera supprimé dans une version future. Migrez vers les comptes utilisateurs.
