# Installation

## Quick Install (recommended)

Install SORK on any Linux server in two steps:

**1. Authenticate to the SORK registry** (credentials provided with your license):

```bash
echo "YOUR_TOKEN" | docker login ghcr.io -u Arcneell --password-stdin
```

**2. Install:**

```bash
docker run --rm ghcr.io/arcneell/sork:latest cat /opt/sork/install.sh | bash -s -- --with-systemd
```

### What the Script Does

1. Checks for and installs Docker if missing
2. Extracts the orchestration engine to `/opt/sork/`
3. Creates default configuration files
4. Runs an initial reconciliation (`sork once`)
5. Installs the systemd service (if `--with-systemd`)

### Prerequisites

| Dependency | Minimum Version | Installed Automatically |
|---|---|---|
| **Docker** | 20.10+ | Yes |
| **Bash** | 4.0+ | No (present on most systems) |
| **curl** | 7.0+ | Yes |
| **socat** | 1.7+ | No (optional, for autoscale) |

!!! note "Python not required"
    Unlike the source installation, the image-based installation does **not** require Python or Node.js on the host. The Python backend is compiled and embedded in the Docker image.

### Install Options

```bash
install.sh [options]

  --root PATH            Installation directory (default: /opt/sork)
  --image IMAGE          Docker image (default: ghcr.io/arcneell/sork:latest)
  --with-systemd         Install and start the systemd service
  --no-install-docker    Don't install Docker (fail if missing)
  --skip-engine          Don't extract the engine (UI image only)
  --skip-pull            Don't pull the image (use local image)
  --dry-run              Print actions without executing
```

### Environment Variables

```bash
SORK_IMAGE="ghcr.io/arcneell/sork:latest"   # Image to use
SORK_ROOT="/opt/sork"                         # Installation directory
```

## Directory Structure After Install

```
/opt/sork/
├── bin/sork                  # Orchestrator CLI
├── lib/                      # Engine modules
│   ├── common.sh
│   ├── manifest.sh
│   ├── health.sh
│   ├── repair.sh
│   ├── ...
│   ├── audit_log.py
│   └── manifest_doctor.py
├── etc/
│   ├── manifest.ini          # Service configuration
│   └── notify.ini            # Notification configuration
├── .sork/                    # Runtime data
│   ├── state/
│   ├── incidents/
│   ├── audit/
│   └── logs/
└── VERSION
```

## Post-Install Configuration

### Edit the Manifest

Add your services to `/opt/sork/etc/manifest.ini`:

```ini
[orchestrator]
interval = 10
max_repair = 5
remove_orphans = 1

[my-service]
image = nginx:latest
publish = 8080:80
health_type = http
health_url = http://127.0.0.1:8080/
repair_strategy = auto
```

### Validate the Configuration

```bash
sork validate
sork doctor
```

### Configure Notifications

Edit `/opt/sork/etc/notify.ini` to enable Discord, Slack, Teams, Telegram, or SMTP alerts.

## systemd Service

If you didn't use `--with-systemd` during installation:

```bash
# Re-run the installer with the option
install.sh --with-systemd --skip-pull --skip-engine

# Or install manually
sudo tee /etc/systemd/system/sork.service <<EOF
[Unit]
Description=SORK orchestrator (run mode)
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=simple
User=$(whoami)
WorkingDirectory=/opt/sork
Environment=SORK_MANIFEST=/opt/sork/etc/manifest.ini
Environment=SORK_NOTIFY_CONF=/opt/sork/etc/notify.ini
Environment=SORK_HOME=/opt/sork
ExecStart=/opt/sork/bin/sork run
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now sork
```

## Updating

To update SORK to the latest version:

```bash
# Pull the new image and re-run the installer
docker run --rm ghcr.io/arcneell/sork:latest cat /opt/sork/install.sh | bash -s -- --with-systemd
```

The systemd service will restart automatically. The UI will be recreated on the next reconciliation cycle.

## Web Console

After installation, the web console is available at **http://127.0.0.1:18100**.

- Default login: `admin` / `admin` (password change forced on first login)
- To expose on the LAN, change `publish` in the manifest:

```ini
[sork-ui]
publish = 0.0.0.0:18100:8080
```

Then increment `config_version` and wait for the next reconciliation cycle (or run `sork once`).

---

## Source Installation (developers)

For contributors and developers:

### Additional Prerequisites

| Dependency | Minimum Version | Usage |
|---|---|---|
| **Python 3** | 3.11+ | UI backend, manifest validation, audit |
| **Node.js** | 20+ | Vue 3 frontend build |
| **ShellCheck** | — | Bash script linting |

### Procedure

```bash
git clone https://github.com/Arcneell/SORK.git
cd shell-orchestrator
./scripts/install-all.sh
```

See [WORKFLOW-MODIFICATIONS.md](../WORKFLOW-MODIFICATIONS.en.md) for the full development workflow.
