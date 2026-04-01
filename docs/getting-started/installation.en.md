# Installation

## Prerequisites

| Dependency | Minimum Version | Usage |
|---|---|---|
| **Bash** | 5.0+ | Reconciliation engine |
| **Docker** or **Podman** | 20.10+ / 4.0+ | Container runtime |
| **curl** | 7.0+ | HTTP health checks |
| **python3** | 3.11+ | Manifest validation, audit, UI backend |
| **socat** | 1.7+ | TCP reverse proxy (optional, for autoscale) |
| **Node.js** | 20+ | Frontend build (optional, for web console) |

## Automatic Installation

The `install-all.sh` script handles everything:

```bash
git clone https://github.com/Arcneell/SORK.git
cd shell-orchestrator
./scripts/install-all.sh
```

### Available Options

```bash
./scripts/install-all.sh [OPTIONS]

Options:
  --dry-run         Display actions without executing them
  --with-systemd    Install the systemd service
  --skip-build      Skip Docker image rebuilds
  --skip-sork-once  Skip the initial reconciliation run
```

### What the Script Does

1. Checks for and installs Docker if missing
2. Installs base packages (`curl`, `python3`)
3. Builds the orchestrator and UI images
4. Creates configuration files from examples
5. Runs an initial reconciliation (`sork once`)
6. Installs the systemd service (if `--with-systemd`)

## Manual Installation

### 1. Clone the Project

```bash
git clone https://github.com/Arcneell/SORK.git
cd shell-orchestrator
```

### 2. Create the Configuration

```bash
cp etc/manifest.ini.example etc/manifest.ini
cp etc/notify.ini.example etc/notify.ini
```

### 3. Edit the Manifest

Add your services to `etc/manifest.ini`:

```ini
[orchestrator]
interval = 10
max_repair = 5
remove_orphans = 1

[mon-service]
image = nginx:latest
publish = 8080:80
health_type = http
health_url = http://127.0.0.1:8080/
```

### 4. Validate the Configuration

```bash
bin/sork validate
bin/sork doctor
```

### 5. Start the Orchestrator

```bash
# Single pass
bin/sork once

# Daemon mode (infinite loop)
bin/sork run
```

## systemd Installation

To run as a service:

```bash
sudo cp sork.global.service /etc/systemd/system/sork.service
sudo systemctl daemon-reload
sudo systemctl enable sork
sudo systemctl start sork
```

!!! warning "Service User"
    The systemd service runs by default with the `deploy` user. Adjust `User=` and `Group=` in the unit file as needed.

The unit file configures:

- **WorkingDirectory**: `/opt/shell-orchestrator`
- **Variables**: `SORK_MANIFEST` and `SORK_NOTIFY_CONF` pointing to `/opt/shell-orchestrator/etc/`
- **Automatic restart**: `Restart=always` with a 3-second delay
- **Dependencies**: Starts after Docker and the network

## Web Console Installation

```bash
# Build and deploy the UI
./scripts/deploy-ui.sh
```

The UI Docker image is a multi-stage build:

1. **Stage 1**: Node 20 Alpine — Vue 3 frontend build
2. **Stage 2**: Python 3.12 Alpine — FastAPI runtime + static assets

The UI is accessible on port `8080` by default.
