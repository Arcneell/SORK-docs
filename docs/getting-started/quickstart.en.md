# Getting Started

This guide walks you through launching SORK with your first service.

## 1. Create the Manifest

```bash
cd shell-orchestrator
cp etc/manifest.ini.example etc/manifest.ini
```

Edit `etc/manifest.ini` to define your first service:

```ini
[orchestrator]
interval = 10
max_repair = 5
remove_orphans = 1
log_level = info

[web]
image = nginx:latest
publish = 8080:80
health_type = http
health_url = http://127.0.0.1:8080/
health_expect_codes = 200
health_timeout = 5
repair_strategy = auto
```

## 2. Validate the Configuration

```bash
bin/sork validate
```

Expected output: no errors. For a more thorough diagnostic:

```bash
bin/sork doctor
```

## 3. Run a First Pass

```bash
bin/sork once
```

SORK will:

1. Load the manifest
2. Detect the runtime (Docker or Podman)
3. Determine that the `sork-web` container does not exist
4. Create it with the `nginx:latest` image and port `8080:80`
5. Run a health check on `http://127.0.0.1:8080/`
6. Display the result

## 4. Verify the Result

```bash
# Check SORK status
bin/sork status

# View the created container
docker ps | grep sork-web
```

Your Nginx service is now accessible at `http://localhost:8080`.

## 5. Start the Daemon

To have SORK continuously monitor your services:

```bash
bin/sork run
```

The reconciliation loop will run every 10 seconds (configurable via `interval`).

!!! tip "Stop the daemon"
    `Ctrl+C` to stop. In systemd mode: `sudo systemctl stop sork`.

## 6. Test Automatic Repair

Simulate a failure by manually stopping the container:

```bash
docker stop sork-web
```

On the next reconciliation cycle, SORK will detect that the container is stopped and automatically restart it.

!!! note "Manual pause"
    If `manual_stop_pause = 1` (default), SORK will not restart a manually stopped container. Use `bin/sork resume web` to resume reconciliation.

## Next Steps

- [Manifest configuration](../configuration/manifest.md) to add more services
- [Health checks](../modules/health.md) to configure monitoring
- [Discord notifications](../configuration/notifications.md) to receive alerts
