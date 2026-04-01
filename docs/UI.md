# SORK Web Interface

The SORK web console provides a Portainer-style Docker management UI with integrated orchestrator controls. It combines a FastAPI backend (Python) with a Vue 3 single-page application.

## Technology Stack

| Layer    | Technology                                           |
|----------|------------------------------------------------------|
| Backend  | Python 3, FastAPI, Uvicorn                           |
| Frontend | Vue 3 (Composition API), TypeScript, Vite            |
| Styling  | Tailwind CSS                                         |
| Icons    | Lucide                                               |
| Realtime | Server-Sent Events (SSE)                             |
| Auth     | Optional bearer token (SORK_UI_TOKEN env var)        |

## Authentication

Authentication is optional and controlled by the `SORK_UI_TOKEN` environment variable.

- If `SORK_UI_TOKEN` is not set or empty, the API is fully open (suitable for local/private use).
- If set, all `/api/*` endpoints (except `/api/ping`) require a valid token.

Token can be provided in three ways (checked in order):

1. `Authorization: Bearer <token>` header
2. `X-SORK-Token: <token>` header
3. `?token=<token>` query parameter (required for SSE/EventSource, which cannot set custom headers)

The `/metrics` endpoint has separate auth control via `SORK_METRICS_PROTECT=1` (requires `SORK_UI_TOKEN` to be set as well).

## Shared Components

### WizardModal

Unified multi-step wizard dialog. Manages step navigation, validation gates, and modal lifecycle. Used by `ContainerWizard` and other creation flows.

### ArrayField

Dynamic array input component. Users can add and remove string items from a list. Used for environment variables, volume mounts, port mappings, and similar repeating fields.

### ContainerWizard

Full container creation/editing wizard supporting three modes:

- **Create**: blank container from scratch.
- **Edit**: modify an existing container's configuration.
- **Template**: pre-fill fields from a service template.

Integrates `DeployProgress` to show real-time feedback during `docker build` and `docker run`.

### DeployProgress

Step-by-step deployment progress display with a visual progress bar. Each step (pull image, create container, start, health check) is shown with its status (pending, running, success, error).

### ConfirmModal

Simple confirmation dialog with customizable title, message, and action button text. Used before destructive operations (remove container, delete volume, etc.).

### DataTable

Sortable, filterable, paginated data table. Supports custom column renderers, row selection, and bulk actions.

### StatusBadge

Color-coded status pill component. Maps container/service states to appropriate colors (green for running, red for stopped, yellow for degraded, etc.).

### FeedbackToast

Non-blocking toast notification for user feedback. Supports success, error, warning, and info types with auto-dismiss.

### JsonViewer

JSON viewer with two modes:

- **Tree mode**: collapsible tree with syntax highlighting, using render functions for performance.
- **Raw mode**: formatted JSON text.

Used in container inspect views and API response displays.

## API Endpoints

### Docker Management

| Method | Endpoint                          | Description                        |
|--------|-----------------------------------|------------------------------------|
| GET    | `/api/docker/info`                | Docker daemon info                 |
| GET    | `/api/docker/version`             | Docker version                     |
| GET    | `/api/containers/`                | List all containers                |
| POST   | `/api/containers/`                | Create a container                 |
| GET    | `/api/containers/{id}`            | Container details                  |
| POST   | `/api/containers/{id}/start`      | Start a container                  |
| POST   | `/api/containers/{id}/stop`       | Stop a container                   |
| POST   | `/api/containers/{id}/restart`    | Restart a container                |
| DELETE | `/api/containers/{id}`            | Remove a container                 |
| GET    | `/api/containers/{id}/logs`       | Container logs                     |
| GET    | `/api/images/`                    | List images                        |
| POST   | `/api/images/pull`                | Pull an image                      |
| DELETE | `/api/images/{id}`                | Remove an image                    |
| POST   | `/api/images/build`               | Build an image                     |
| GET    | `/api/volumes/`                   | List volumes                       |
| POST   | `/api/volumes/`                   | Create a volume                    |
| DELETE | `/api/volumes/{name}`             | Remove a volume                    |
| GET    | `/api/networks/`                  | List networks                      |
| POST   | `/api/networks/`                  | Create a network                   |
| DELETE | `/api/networks/{id}`              | Remove a network                   |

### Stacks (Docker Compose)

| Method | Endpoint                          | Description                        |
|--------|-----------------------------------|------------------------------------|
| GET    | `/api/stacks/`                    | List stacks                        |
| POST   | `/api/stacks/`                    | Deploy a stack                     |
| GET    | `/api/stacks/{name}`              | Stack details                      |
| DELETE | `/api/stacks/{name}`              | Remove a stack                     |

### Orchestrator

| Method | Endpoint                          | Description                        |
|--------|-----------------------------------|------------------------------------|
| GET    | `/api/state`                      | Full orchestrator state            |
| GET    | `/api/manifest`                   | Read manifest                      |
| PUT    | `/api/manifest`                   | Write manifest                     |
| GET    | `/api/incidents`                  | Incident log                       |
| GET    | `/api/services`                   | SORK-managed services              |

### Autoscale

| Method | Endpoint                          | Description                        |
|--------|-----------------------------------|------------------------------------|
| GET    | `/api/autoscale/`                 | Autoscale status                   |

### Templates and App Store

| Method | Endpoint                          | Description                        |
|--------|-----------------------------------|------------------------------------|
| GET    | `/api/templates/`                 | List service templates             |
| POST   | `/api/templates/`                 | Create custom template             |

### Webhooks and Registries

| Method | Endpoint                          | Description                        |
|--------|-----------------------------------|------------------------------------|
| GET    | `/api/webhooks/`                  | List webhooks                      |
| POST   | `/api/webhooks/`                  | Create webhook                     |
| DELETE | `/api/webhooks/{id}`              | Remove webhook                     |
| GET    | `/api/registries/`                | List registries                    |
| POST   | `/api/registries/`                | Add registry                       |
| DELETE | `/api/registries/{id}`            | Remove registry                    |

### Host and System

| Method | Endpoint                          | Description                        |
|--------|-----------------------------------|------------------------------------|
| GET    | `/api/host/info`                  | Host system metrics                |
| GET    | `/api/ping`                       | Health check (no auth)             |
| GET    | `/metrics`                        | Prometheus metrics                 |

### Streaming (SSE)

| Method | Endpoint                          | Description                        |
|--------|-----------------------------------|------------------------------------|
| GET    | `/api/events/stream`              | Docker events SSE stream           |
| GET    | `/api/state/stream`               | Orchestrator state SSE stream      |

For SSE endpoints, pass the token as a query parameter: `?token=<token>`.

### Notifications

| Method | Endpoint                          | Description                        |
|--------|-----------------------------------|------------------------------------|
| GET    | `/api/notifications/`             | List notifications                 |
| POST   | `/api/notifications/read`         | Mark all as read                   |
| GET    | `/api/notifications/stream`       | Notification SSE stream            |

### Logs

| Method | Endpoint                          | Description                        |
|--------|-----------------------------------|------------------------------------|
| GET    | `/api/logs/`                      | List available log files           |
| GET    | `/api/logs/{name}`                | Read a log file                    |

### Backup

| Method | Endpoint                          | Description                        |
|--------|-----------------------------------|------------------------------------|
| GET    | `/api/backup`                     | Download SORK data backup          |
| POST   | `/api/restore`                    | Restore from backup                |

## Logging System

The backend uses a structured JSON logging system (`core/logging.py`):

- **JSON log file**: `$SORK_DATA/logs/sork-ui.log` with daily rotation and 30-day retention.
- **Console output**: colored, human-readable format for `docker logs`.
- **Request middleware**: logs all `/api/*` requests (excluding `/api/ping` and SSE endpoints) with method, path, status code, duration, and client IP.
- **Action logging**: `log_action(category, action, detail, level)` for audit-trail events.

Log entries follow this JSON schema:

```json
{
  "ts": "2025-01-15T10:30:00.123Z",
  "level": "INFO",
  "logger": "sork.routers.containers",
  "message": "Container started: sork-webapp",
  "category": "container",
  "action": "start",
  "detail": "sork-webapp"
}
```

HTTP request logs include additional fields: `method`, `path`, `status`, `duration_ms`, `client_ip`.

## Notification System

Notifications are managed by `core/notifications.py`:

- **In-memory buffer**: thread-safe list with a 200-entry maximum.
- **File persistence**: saved to `.sork/notifications.json` on every write, loaded on first access. Survives container restarts.
- **Types**: `info`, `warning`, `error`, `success`.
- **Discord integration**: notifications can also be sent to Discord via the backend's Discord helper (separate from the bash engine's `notify.sh`).
- **API**: list, mark-all-read, and SSE stream via `/api/notifications/`.

## Deployment

### Using deploy-ui.sh

The recommended deployment method:

```bash
./scripts/deploy-ui.sh
```

This script:

1. Builds the Docker image `sork-ui:2` from `ui/`.
2. Removes any existing `sork-sork-ui` container.
3. Runs a new container with:
   - Port mapping: `127.0.0.1:18100:8080` (host:container).
   - Volume mount: project root at `/workspace` (gives access to `etc/`, `lib/`, `.sork/`).
   - Volume mount: Docker socket at `/var/run/docker.sock`.
   - Volume mount: `ui/backend/app` at `/app/app` for backend hot-reload.

### Bind Address

By default, the UI listens on `127.0.0.1` only (local access). To expose on the network:

```bash
SORK_UI_PUBLISH_BIND=0.0.0.0 ./scripts/deploy-ui.sh
```

If `sork.service` (systemd) is active, update `etc/manifest.ini` to match:

```ini
[sork-ui]
publish = 0.0.0.0:18100:8080
config_version = 2     # increment to trigger reconciliation
```

### Environment Variables

| Variable               | Default     | Description                                    |
|------------------------|-------------|------------------------------------------------|
| `PORT`                 | `8080`      | Listen port inside the container               |
| `SORK_UI_BIND`         | `0.0.0.0`  | Bind address inside the container              |
| `SORK_UI_TOKEN`        | (empty)     | API authentication token                       |
| `SORK_RUNTIME`         | (auto)      | Force `docker` or `podman`                     |
| `SORK_UI_TLS_CERT`     | (empty)     | TLS certificate path for HTTPS                 |
| `SORK_UI_TLS_KEY`      | (empty)     | TLS private key path for HTTPS                 |
| `SORK_METRICS_PROTECT` | `0`         | Require auth for `/metrics`                    |
| `SORK_LOG_LEVEL`       | `INFO`      | Logging level (DEBUG, INFO, WARNING, ERROR)    |

Pass environment variables via the manifest `env` key (semicolon-separated):

```ini
env = SORK_UI_TOKEN=my_secret;SORK_METRICS_PROTECT=1
```

### Backend Hot-Reload

The deploy script mounts `ui/backend/app` as a volume at `/app/app`. To apply backend code changes, restart the container:

```bash
docker restart sork-sork-ui
```

### Frontend Development

For frontend hot-reload during development:

```bash
cd ui/frontend
npm install
npm run dev
```

For production, rebuild the Docker image:

```bash
docker build -t sork-ui:2 ./ui
```
