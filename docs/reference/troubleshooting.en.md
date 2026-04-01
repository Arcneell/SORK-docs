# Troubleshooting

Guide for resolving common issues.

## Quick Diagnostic

```bash
# Check overall state
bin/sork status

# Validate configuration
bin/sork validate

# Full diagnostic
bin/sork doctor

# View daemon logs
tail -50 .sork/logs/sork-daemon.log | python3 -m json.tool

# Latest incidents
tail -20 .sork/incidents/incidents.log
```

## Common Issues

### The daemon does not start

**Symptom**: `bin/sork run` fails immediately.

**Checks**:

1. Is Docker accessible?
   ```bash
   docker info
   ```

2. Is the manifest valid?
   ```bash
   bin/sork validate
   ```

3. Is the SORK_DATA directory writable?
   ```bash
   ls -la .sork/
   ```

### A service does not start

**Symptom**: the container is never created.

**Checks**:

1. Does the image exist?
   ```bash
   docker pull <image>
   ```

2. Is the service suspended?
   ```bash
   ls .sork/state/<app>.suspend_reconcile
   # If the file exists:
   bin/sork resume <app>
   ```

3. Is the service manually paused?
   ```bash
   ls .sork/state/<app>.manual_pause
   # If the file exists:
   bin/sork resume <app>
   ```

### Health checks keep failing

**Symptom**: the `.fail` counter keeps increasing.

**Checks**:

1. Is the health URL accessible?
   ```bash
   curl -v <health_url>
   ```

2. Is the port correctly exposed?
   ```bash
   docker port sork-<app>
   ```

3. Does the service take time to start? Increase the grace period:
   ```ini
   post_repair_grace = 10
   ```

4. In strict mode, does the URL target localhost?
   ```bash
   SORK_STRICT_LOCAL=1 bin/sork doctor
   ```

### The service keeps restarting

**Symptom**: constant restarts, notification flood.

**Possible causes**:

- **OOM**: the container is killed due to out of memory
  ```bash
  docker inspect sork-<app> | grep OOMKilled
  ```
  Solution: increase `memory_limit_mb`

- **Crash at startup**: the application crashes immediately
  ```bash
  docker logs sork-<app>
  ```

- **Port in use**: the port is already used by another process
  ```bash
  ss -tlnp | grep <port>
  ```

### Discord notifications are not working

**Checks**:

1. Is the webhook configured?
   ```bash
   cat etc/notify.ini
   ```

2. Is the webhook valid?
   ```bash
   curl -X POST <webhook_url> \
     -H "Content-Type: application/json" \
     -d '{"content": "Test SORK"}'
   ```

3. Is `enabled` set to `1`?

### Autoscale is not working

**Checks**:

1. Is `autoscale = 1` defined?
2. Is `socat` installed?
   ```bash
   which socat
   ```

3. Is the port range free?
   ```bash
   ss -tlnp | grep -E '185[0-9]{2}'
   ```

4. Check the backends:
   ```bash
   cat .sork/autoscale/<app>.backends
   ```

### The web console does not connect

**Checks**:

1. Is the UI container running?
   ```bash
   docker ps | grep sork-ui
   ```

2. Is the Docker socket mounted?
   ```bash
   docker inspect sork-ui | grep docker.sock
   ```

3. Is the auth token correct (if enabled)?

## Logs

### Daemon Logs (JSON lines)

```bash
# Latest logs
tail -20 .sork/logs/sork-daemon.log

# Format as readable JSON
tail -5 .sork/logs/sork-daemon.log | python3 -m json.tool

# Filter by level
grep '"level":"error"' .sork/logs/sork-daemon.log
```

### Container Logs

```bash
docker logs sork-<app> --tail 50
docker logs sork-<app> -f  # real-time follow
```

### Incidents

```bash
# Text
cat .sork/incidents/incidents.log

# JSONL (with jq)
cat .sork/incidents/$(date +%Y-%m-%d).jsonl | jq .
```

## Debug Mode

For more detail in logs:

```bash
SORK_LOG_LEVEL=debug bin/sork once
```

Or in the manifest:

```ini
[orchestrator]
log_level = debug
```

## Full Reset

!!! danger "Warning"
    This operation deletes all SORK state. Containers are not affected.

```bash
# Delete all state
rm -rf .sork/

# Relaunch
bin/sork once
```

SORK will recreate the `.sork/` directory and reconverge toward the desired state.
