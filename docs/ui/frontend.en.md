# Frontend

The SORK frontend is a Single Page Application (SPA) built with Vue 3 and TypeScript.

## Stack

| Tool | Role |
|---|---|
| **Vue 3** | Reactive framework |
| **TypeScript** | Static typing |
| **Vite** | Build tool & dev server |
| **Tailwind CSS** | Utility-first styles |
| **Lucide** | Icons |
| **Vue Router** | SPA navigation |

## View Structure

### Dashboard

The home page displays:

- **Daemon status**: indicator based on heartbeat (active, inactive, unknown)
- **Services**: summary card for each service with state, quick actions
- **System**: host server CPU, memory, disk
- **Recent alerts**: latest notifications

### Docker

Docker resource management organized in sub-views:

- **Containers**: table with filters, inline actions, logs in modal
- **Images**: gallery with pull, build, removal
- **Volumes**: list with size and attachments
- **Networks**: network topology
- **Stacks**: Docker Compose management
- **System Info**: Docker version, storage driver, container count
- **Events**: live Docker events stream

### Orchestrator

SORK-specific interface:

- **Services**: detailed state of each orchestrated service (health, replicas, last action)
- **Manifest Editor**: syntax-aware INI file editing with live validation
- **Autoscale Dashboard**: metric graphs, replica count, active thresholds
- **Incidents**: table filterable by date, service, severity
- **Audit Journal**: container operation timeline with filtering

### AppStore

Simplified deployment:

- Template catalog (preconfigured services)
- Remote template sources
- **WizardModal**: multi-step deployment assistant

### Logs

Centralized log viewer:

- SORK daemon logs (formatted JSON)
- Container logs (with real-time streaming)
- UI backend logs

### Settings

- Authentication token configuration
- Read/unread notification management
- Display preferences

## Reusable Components

| Component | Description |
|---|---|
| `DataTable` | Table with sorting, filtering, pagination |
| `StatusBadge` | Colored badge based on state (running, stopped, unhealthy) |
| `JsonViewer` | Formatted JSON display |
| `ConfirmModal` | Confirmation dialog for destructive actions |
| `FeedbackToast` | Temporary notification (success, error) |
| `DeployProgress` | Deployment progress bar |
| `WizardModal` | Multi-step assistant |
| `ArrayField` | Form field for lists (ports, volumes, env) |
| `ContainerWizard` | Complete container creation form |

## Production Build

```bash
cd ui/frontend
npm install
npm run build
```

Assets are generated in `dist/` and served as static files by the FastAPI backend.

## Development

```bash
cd ui/frontend
npm install
npm run dev
```

The Vite development server runs on `http://localhost:5173` with hot-reload. It proxies `/api` requests to the FastAPI backend.
