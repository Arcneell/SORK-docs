# Frontend

Le frontend SORK est une Single Page Application (SPA) construite avec Vue 3 et TypeScript.

## Stack

| Outil | Rôle |
|---|---|
| **Vue 3** | Framework réactif |
| **TypeScript** | Typage statique |
| **Vite** | Build tool & dev server |
| **Tailwind CSS** | Styles utilitaires |
| **Lucide** | Icônes |
| **Vue Router** | Navigation SPA |

## Structure des vues

### Dashboard

La page d'accueil affiche :

- **Statut du daemon** : indicateur basé sur le heartbeat (actif, inactif, inconnu)
- **Services** : carte résumée de chaque service avec état, actions rapides
- **Système** : CPU, mémoire, disque du serveur hôte
- **Alertes récentes** : dernières notifications

### Docker

Gestion des ressources Docker organisée en sous-vues :

- **Containers** : tableau avec filtres, actions en ligne, logs en modal
- **Images** : galerie avec pull, build, suppression
- **Volumes** : liste avec taille et attachements
- **Networks** : topologie des réseaux
- **Stacks** : gestion Docker Compose
- **System Info** : version Docker, storage driver, nombre de conteneurs
- **Events** : flux en direct des événements Docker

### Orchestrator

Interface spécifique SORK :

- **Services** : état détaillé de chaque service orchestré (santé, replicas, dernière action)
- **Manifest Editor** : édition syntaxique du fichier INI avec validation en direct
- **Autoscale Dashboard** : graphiques de métriques, nombre de replicas, seuils actifs
- **Incidents** : tableau filtrable par date, service, sévérité
- **Audit Journal** : timeline des opérations conteneur avec filtrage

### AppStore

Déploiement simplifié :

- Catalogue de templates (services préconfigurés)
- Sources de templates distantes
- **WizardModal** : assistant multi-étapes pour le déploiement

### Logs

Visionneuse centralisée :

- Logs daemon SORK (JSON formatté)
- Logs conteneurs (avec suivi temps réel)
- Logs backend UI

### Settings

- Gestion des utilisateurs (admin uniquement)
- Gestion des notifications lues/non lues
- Préférences d'affichage

## Composants réutilisables

| Composant | Description |
|---|---|
| `DataTable` | Tableau avec tri, filtrage, pagination |
| `StatusBadge` | Badge coloré selon l'état (running, stopped, unhealthy) |
| `JsonViewer` | Affichage formatté de JSON |
| `ConfirmModal` | Dialogue de confirmation pour les actions destructives |
| `FeedbackToast` | Notification temporaire (succès, erreur) |
| `DeployProgress` | Barre de progression pour les déploiements |
| `WizardModal` | Assistant multi-étapes |
| `ArrayField` | Champ de formulaire pour les listes (ports, volumes, env) |
| `ContainerWizard` | Formulaire complet de création de conteneur |

## Build de production

```bash
cd ui/frontend
npm install
npm run build
```

Les assets sont générés dans `dist/` et servis en tant que fichiers statiques par le backend FastAPI.

## Développement

```bash
cd ui/frontend
npm install
npm run dev
```

Le serveur de développement Vite tourne sur `http://localhost:5173` avec hot-reload. Il proxifie les requêtes `/api` vers le backend FastAPI.
