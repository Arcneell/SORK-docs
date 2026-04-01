# SORK Architecture

SORK (Shell Orchestrator) v1.2.0 -- a single-node Docker orchestrator combining a Bash reconciliation engine, a FastAPI backend, and a Vue 3 frontend.

## Directory Structure

```
shell-orchestrator/
  bin/
    sork                  # Entry point (bash). Commands: run, once, validate, doctor, show, status, resume, reconcile-app
  lib/
    common.sh             # Logging, paths, state helpers, heartbeat, version
    manifest.sh           # INI parser, MNF associative array, manifest_load / manifest_get
    runtime.sh            # Docker/Podman detection, container CRUD (create, remove, inspect, rename)
    health.sh             # HTTP/TCP probes, memory stats, deep_diagnose, disk usage, error rate tracking
    repair.sh             # Repair escalation, blue/green rollout, reconcile_app, manual pause, ensure_desired_revision
    autoscale.sh          # Horizontal scaling, replica management, proxy lifecycle, metric collection, scale decisions
    proxy.sh              # Standalone TCP reverse-proxy/load-balancer (bash + socat), round-robin, health-aware
    notify.sh             # Discord webhook notifications with French-language embeds
    incidents.sh          # Incident log (text + daily JSONL archive)
    audit.sh              # Bash-side audit hook (calls audit_log.py via python3)
    doctor.sh             # Manifest validation, environment checks, auto-repair
    audit_log.py          # Audit backend (JSONL or SQLite), invoked by bash and Python
    manifest_doctor.py    # Extended manifest validation and repair (Python)
  ui/
    backend/
      app/
        main.py           # FastAPI application, middleware, router registration
        config.py         # Path constants, env vars, rate limiting
        core/
          auth.py          # Bearer token / header / query param authentication
          docker.py        # Docker CLI wrappers, input sanitization
          manifest.py      # Manifest read/write helpers for Python
          state.py         # State file access (.sork/)
          logging.py       # Structured JSON logging, daily rotation, console formatter
          notifications.py # In-memory + file-persisted notification buffer
          sse.py           # Server-Sent Events helpers
        routers/
          containers.py    # Container lifecycle (list, create, start, stop, remove, exec, rename, logs)
          images.py        # Image list, pull, remove, build
          volumes.py       # Volume list, create, remove
          networks.py      # Network list, create, remove
          stacks.py        # Stack deploy (compose), list, remove
          orchestrator.py  # Orchestrator state, manifest CRUD, incidents, services, reconcile triggers
          autoscale.py     # Autoscale status and configuration
          templates.py     # Service templates (bundled + remote Portainer-compatible sources)
          webhooks.py      # Webhook management for deployment triggers
          registry.py      # Docker registry configuration
          host.py          # Host system info (CPU, memory, disk, uptime)
          docker_info.py   # Docker daemon info and version
          streaming.py     # SSE streams for events and state
          logs.py          # Log file access (/api/logs/*)
          backup.py        # Backup and restore of SORK data
          notifications.py # Notification list, mark-read, Discord integration
          metrics.py       # Prometheus-compatible /metrics endpoint
    frontend/
      src/
        views/
          DashboardView.vue          # Main dashboard with service overview
          docker/
            ContainersView.vue       # Container list and management
            ContainerDetailView.vue  # Single container detail (logs, stats, exec)
            CreateContainerView.vue  # Container creation form
            ImagesView.vue           # Docker images
            VolumesView.vue          # Docker volumes
            NetworksView.vue         # Docker networks
            StacksView.vue           # Compose stack list
            StackDetailView.vue      # Stack detail and services
            SystemView.vue           # System/host information
            EventsView.vue           # Docker events stream
          orchestrator/
            ServicesView.vue         # SORK-managed services overview
            EditorView.vue           # Manifest INI editor
            AutoscaleView.vue        # Autoscale dashboard
            IncidentsView.vue        # Incident journal
          AppStoreView.vue           # Template-based app deployment
          JournalView.vue            # Unified journal/log viewer
          LogsView.vue               # Structured log viewer (UI + daemon logs)
          SettingsView.vue           # Settings and authentication config
        components/
          shared/
            WizardModal.vue          # Multi-step wizard dialog pattern
            ArrayField.vue           # Dynamic array input (add/remove items)
            ContainerWizard.vue      # Container create/edit/template wizard with deploy
            DeployProgress.vue       # Step-by-step deploy progress with progress bar
            ConfirmModal.vue         # Confirmation dialog
            DataTable.vue            # Sortable, filterable data table
            StatusBadge.vue          # Color-coded status indicator
            FeedbackToast.vue        # Toast notification component
            JsonViewer.vue           # JSON tree/raw viewer with render functions
  etc/
    manifest.ini           # Active manifest (user configuration)
    manifest.ini.example   # Reference manifest with all documented keys
    notify.ini             # Active notification config
    notify.ini.example     # Reference notification config
  .sork/                   # Runtime data directory (created automatically)
    state/                 # Fail counters, heartbeat, pause flags, cooldown files
    incidents/             # incidents.log (text) + daily JSONL files
    archive/               # Daily incident archives
    audit/                 # container-audit.jsonl or container-audit.sqlite
    autoscale/             # Backends files, proxy PIDs, routes.conf, per-app state
    logs/                  # sork-daemon.log (bash), sork-ui.log (Python)
    stacks/                # Compose stack data
    notifications.json     # Persisted notification buffer
    webhooks.json          # Webhook definitions
    registries.json        # Registry credentials
    templates/             # Custom service templates
  scripts/
    deploy-ui.sh           # Build and deploy the UI container
    check-all.sh           # Run all checks (shellcheck, bash -n, python compile, validate, doctor)
    install-all.sh         # System installation script
  examples/
    apache-showcase/       # Example Apache service
    autoscale-web/         # Example autoscale-enabled web service
  docs/                    # This documentation directory
  VERSION                  # Version file (1.2.0)
  LICENSE
  README.md
  sork.global.service      # systemd unit file
```

## Component Overview

### Bash Engine (bin/sork + lib/*.sh)

The core reconciliation engine runs as a bash process. `bin/sork run` enters an infinite loop that:

1. Loads `etc/manifest.ini` into the `MNF` associative array via `manifest_load()`.
2. Applies `[orchestrator]` globals (interval, max_repair, remove_orphans, log_level).
3. Detects the container runtime (`docker` or `podman`).
4. Ensures the global proxy is running if `[proxy]` is configured.
5. Iterates over all non-reserved sections, calling `reconcile_app()` for each.
6. Removes orphan containers (`sork-*` names not in the manifest).
7. Writes a heartbeat timestamp, then sleeps `SORK_INTERVAL` seconds.

Each library module has a focused responsibility:

| Module         | Role                                                                 |
|----------------|----------------------------------------------------------------------|
| `common.sh`    | Logging (`sork_log`), paths, state directories, heartbeat, version   |
| `manifest.sh`  | INI parser, `MNF[]` table, `manifest_load`, `manifest_get`          |
| `runtime.sh`   | Docker/Podman detection, container CRUD, image comparison            |
| `health.sh`    | HTTP/TCP probes, memory stats, `deep_diagnose`, latency, error rate  |
| `repair.sh`    | Repair escalation, blue/green rollout, `reconcile_app`, manual pause |
| `autoscale.sh` | Replica management, proxy lifecycle, metric collection, decisions    |
| `proxy.sh`     | Standalone socat-based TCP reverse-proxy with health monitoring      |
| `notify.sh`    | Discord notifications with French-language formatting                |
| `incidents.sh` | Incident log (text + JSONL), archive rotation                        |
| `audit.sh`     | Audit event dispatch (bash to `audit_log.py`)                        |
| `doctor.sh`    | Manifest validation, environment diagnostics, auto-repair            |

### Python Backend (ui/backend/app/)

A FastAPI application serving the REST API and static frontend assets. It runs inside the `sork-sork-ui` container, with the project root mounted at `/workspace`.

**Core modules** (`core/`):

| Module             | Role                                                              |
|--------------------|-------------------------------------------------------------------|
| `auth.py`          | Token extraction (Bearer header, X-SORK-Token, query param)      |
| `docker.py`        | Docker CLI execution, input sanitization, name validation         |
| `manifest.py`      | Manifest INI read/write using Python `configparser`               |
| `state.py`         | Read/write `.sork/` state files                                   |
| `logging.py`       | Structured JSON logging with daily rotation (30-day retention)    |
| `notifications.py` | Thread-safe notification buffer, file persistence, 200 entry cap  |
| `sse.py`           | Server-Sent Events helpers                                        |

**Routers** (mounted at `/api/`):

| Router            | Prefix                | Description                                  |
|-------------------|-----------------------|----------------------------------------------|
| `docker_info`     | `/api/docker`         | Docker daemon info and version               |
| `containers`      | `/api/containers`     | Container lifecycle management               |
| `images`          | `/api/images`         | Image list, pull, remove, build              |
| `volumes`         | `/api/volumes`        | Volume CRUD                                  |
| `networks`        | `/api/networks`       | Network CRUD                                 |
| `stacks`          | `/api/stacks`         | Docker Compose stack management              |
| `streaming`       | `/api`                | SSE streams (events, state)                  |
| `orchestrator`    | `/api`                | Orchestrator state, manifest, incidents      |
| `autoscale`       | `/api/autoscale`      | Autoscale status and configuration           |
| `templates`       | `/api/templates`      | Service templates                            |
| `webhooks`        | `/api/webhooks`       | Webhook management                           |
| `registry`        | `/api/registries`     | Registry configuration                       |
| `host`            | `/api/host`           | Host system metrics                          |
| `backup`          | `/api`                | Backup and restore                           |
| `notifications`   | `/api/notifications`  | Notification buffer                          |
| `metrics`         | `/metrics`            | Prometheus-compatible metrics                |
| `logs`            | `/api/logs`           | Structured log access                        |

A `/api/ping` endpoint is registered directly on the app (no auth required).

### Vue 3 Frontend (ui/frontend/)

Single-page application built with Vue 3, Vite, TypeScript, and Tailwind CSS. Lucide provides the icon set.

**Views** are organized by domain:

- **Dashboard**: service health overview, daemon heartbeat, quick actions.
- **Docker views**: Containers, ContainerDetail, CreateContainer, Images, Volumes, Networks, Stacks, StackDetail, System, Events.
- **Orchestrator views**: Services, Editor (manifest INI), Autoscale, Incidents.
- **Cross-cutting**: AppStore (template-based deploy), Journal, Logs, Settings.

**Shared components** provide reusable UI patterns:

| Component          | Purpose                                                        |
|--------------------|----------------------------------------------------------------|
| `WizardModal`      | Multi-step wizard dialog with navigation                       |
| `ArrayField`       | Dynamic add/remove array input                                 |
| `ContainerWizard`  | Create/edit/template container with integrated deploy progress  |
| `DeployProgress`   | Step-by-step progress bar for container deployment              |
| `ConfirmModal`     | Confirmation dialog                                            |
| `DataTable`        | Sortable, filterable, paginated table                          |
| `StatusBadge`      | Color-coded status pill                                        |
| `FeedbackToast`    | Non-blocking notification toast                                |
| `JsonViewer`       | JSON tree/raw viewer with render functions                     |

## Data Flow

```
manifest.ini
    |
    v
bin/sork (bash engine)
    |
    +-- manifest_load() --> MNF[] associative array
    +-- reconcile_all()
    |     +-- reconcile_app() per service
    |     |     +-- container_create() / ensure_desired_revision()
    |     |     +-- deep_diagnose() --> repair_execute()
    |     |     +-- autoscale_reconcile() (if autoscale=1)
    |     +-- remove_orphan_containers()
    +-- Docker daemon (docker/podman CLI)
    +-- .sork/ state files (incidents, audit, heartbeat)
    +-- Discord webhooks (notify.sh)

Browser (Vue 3 SPA)
    |
    v
FastAPI backend (:8080)
    |
    +-- Docker CLI (subprocess)
    +-- .sork/ state files (read/write)
    +-- manifest.ini (read/write)
    +-- bin/sork (reconcile triggers)
```

The bash engine and FastAPI backend share state through the filesystem (`.sork/` directory, `etc/manifest.ini`). Both can invoke Docker CLI commands. The UI provides a graphical interface to the same operations the bash engine performs automatically.
