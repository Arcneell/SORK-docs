# Reconciliation Engine

The reconciliation engine is the core of SORK. It compares the desired state (manifest) with the actual state (Docker containers) and takes corrective action to converge the two.

## Overview

The engine runs in a continuous loop (`bin/sork run`) or as a single pass (`bin/sork once`). Each pass executes `reconcile_all()`, which orchestrates the full reconciliation cycle.

## reconcile_all() Flow

Defined in `bin/sork`, this is the top-level function for each reconciliation pass.

```
reconcile_all()
  |
  +-- notify_load()                   # Load notify.ini for Discord alerts
  +-- manifest_load()                 # Parse manifest.ini into MNF[] array
  |     +-- (on failure) doctor fix   # Attempt auto-repair if manifest is unreadable
  |     +-- (on failure) notify       # Alert via Discord if manifest remains broken
  +-- apply_orchestrator_globals()    # Apply [orchestrator] settings to env vars
  +-- runtime_engine()                # Detect docker or podman
  +-- global_proxy_ensure()           # Start/verify global proxy if [proxy] is defined
  +-- for each app in MNF_APPS_ORDER:
  |     +-- skip if reserved section (orchestrator, proxy, notify, global)
  |     +-- reconcile_app(app)
  +-- remove_orphan_containers()      # Clean up sork-* containers not in manifest
```

**Manifest auto-repair**: if `manifest_load()` fails, the engine attempts `sork_manifest_try_repair()` using `manifest_doctor.py` with the example manifest as a reference. Discord notifications are sent on failure and recovery. Two debounce flags prevent repeated alerts: `manifest_load_warn.notified` and `manifest_repair_exhausted.notified`.

## reconcile_app() Flow

Defined in `lib/repair.sh`, this function handles a single service.

```
reconcile_app(app)
  |
  +-- Is autoscale enabled?
  |     +-- Yes: check manual pause, then delegate to autoscale_reconcile()
  |     +-- No: continue below
  |
  +-- Is reconciliation suspended? (create_fail_max_attempts exceeded)
  |     +-- Yes: log warning, return (requires `sork resume <app>`)
  |
  +-- Does the container exist?
  |     +-- No: container_create(app), reset counters, return
  |
  +-- detect_unexpected_restart()     # Track Docker restart counter changes
  +-- ensure_desired_revision()       # Check image + config_version match
  |
  +-- Is the container running?
  |     +-- Yes: deep_diagnose()
  |     |     +-- Healthy: reset fail count, record recovery if applicable
  |     |     +-- Unhealthy: increment fail count, repair or alert
  |     +-- No: check manual stop pause
  |           +-- manual_stop_pause enabled + exit code looks manual: enter pause mode
  |           +-- Otherwise: start/recreate the container
```

### Manual Pause

When `manual_stop_pause = 1` (default), SORK detects manual stops (exit codes 0, 137, 143) and enters pause mode instead of restarting the container. This prevents the orchestrator from fighting with an operator who intentionally stopped a service.

- Pause state is stored in `.sork/state/<app>.manual_pause`.
- A single Discord notification is sent when pause is activated.
- To resume: `sork resume <app>` or start the container manually (SORK detects it and clears the pause).

## sork reconcile-app <name>

For targeted single-service reconciliation without running a full pass:

```bash
bin/sork reconcile-app myapp
```

This loads the manifest, detects the runtime, and calls `reconcile_app()` for the specified service only. It does not run `remove_orphan_containers()` or ensure the global proxy.

## ensure_desired_revision()

Checks whether the running container matches the desired state. Two criteria must both match:

1. **Image**: the container's image must match the `image` key in the manifest (with normalization for registry prefixes and `:latest` tags).
2. **config_version**: the container's `sork.config_version` label must match the `config_version` key.

If either differs, the container is updated according to the `rollout_strategy`:

### recreate (default)

```
1. container_remove_force(app)
2. sleep 1
3. container_create(app)
4. reset_fail_count(app)
```

### blue_green

```
1. Create candidate container with candidate_publish ports
2. Wait grace period (candidate_grace, default 2s)
3. Run preflight_cmd in candidate (if configured)
4. Health check the candidate (using health_url_candidate if set)
5. If candidate is healthy:
   a. Remove the live container
   b. Rename candidate to the live name (or recreate if ports differ)
6. If candidate is unhealthy:
   a. Remove the candidate
   b. Leave the live container unchanged
```

The candidate container name includes a timestamp: `sork-<app>-candidate-<epoch>`.

## repair_execute() Escalation

When `deep_diagnose()` detects a problem, SORK escalates through repair phases based on the consecutive failure count:

| Failure Count | Phase      | Action                                               |
|---------------|------------|------------------------------------------------------|
| 0--1          | `restart`  | `docker restart` the container                       |
| 2--3          | `recreate` | Remove and recreate the container                    |
| 4+            | `purge`    | Remove container, optionally purge volumes, recreate |

The `repair_strategy` manifest key can override this escalation:

| Strategy        | Behavior                                     |
|-----------------|----------------------------------------------|
| `auto`          | Default escalation (restart -> recreate -> purge) |
| `restart-only`  | Always restart, never recreate               |
| `recreate-only` | Always recreate, never restart or purge      |
| `purge-only`    | Always purge and recreate                    |

Volume purge only occurs when `purge_on_escalation = 1` in the manifest. Named volumes listed in the `volumes` key are deleted individually.

When `max_repair` consecutive failures are reached without recovery, a Discord alert is sent.

## remove_orphan_containers()

When `remove_orphans = 1` in `[orchestrator]`, SORK scans for Docker containers whose names start with `sork-` but do not correspond to any service in the manifest. These containers are forcefully removed.

Exclusions:
- Containers matching `sork-<app>-candidate-*` (blue/green candidates in progress).
- Containers matching `sork-<app>-r*` (autoscale replicas, managed by `autoscale.sh`).
- Containers matching `sork-<app>-lb` (autoscale proxy containers).

An incident is recorded for each orphan removed.

## Proxy State Cleanup

When a service with autoscale is removed from the manifest:
- Its replicas become orphans and are cleaned up by `remove_orphan_containers()`.
- The proxy process (PID file in `.sork/autoscale/`) is stopped by `autoscale_lb_stop()`.
- Backend files and route entries are cleaned up on the next global proxy route update.

## State Files

The reconciliation engine maintains state in `.sork/state/`:

| File                              | Purpose                                        |
|-----------------------------------|-------------------------------------------------|
| `sork-daemon-heartbeat`           | Timestamp of last heartbeat (daemon alive check)|
| `<app>.restart_count`             | Docker restart counter tracking                 |
| `<app>.fail_count`                | Consecutive health failure counter              |
| `<app>.manual_pause`              | Manual pause flag file                          |
| `<app>.manual_pause_notified`     | Debounce for manual pause notification          |
| `<app>.suspend_reconcile`         | Reconciliation suspended (creation loop)        |
| `<app>.suspend_reconcile_notified`| Debounce for suspension notification            |
| `<app>.autoscale_cooldown`        | Autoscale decision streak counter               |
| `<app>.autoscale_max.notified`    | Debounce for autoscale max-reached alert        |
| `manifest_load_warn.notified`     | Debounce for manifest load failure alert        |
| `manifest_repair_exhausted.notified` | Debounce for manifest repair failure alert   |

## CLI Commands

| Command                    | Description                                   |
|----------------------------|-----------------------------------------------|
| `sork run`                 | Infinite reconciliation loop                  |
| `sork once`                | Single reconciliation pass                    |
| `sork reconcile-app <app>` | Reconcile a single service                    |
| `sork validate`            | Validate the manifest (bash + python checks)  |
| `sork doctor [--fix] [--strict-local]` | Diagnose and optionally repair    |
| `sork show`                | Display loaded manifest summary               |
| `sork version`             | Display SORK version                          |
| `sork status`              | Show version, paths, heartbeat, runtime       |
| `sork resume <app>`        | Clear suspension and manual pause for a service|
