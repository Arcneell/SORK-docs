# CLI Reference

The `bin/sork` binary is the main entry point for SORK.

## Usage

```bash
bin/sork <command> [options]
```

Or with environment variables:

```bash
SORK_MANIFEST=etc/manifest.ini SORK_DATA=.sork bin/sork <command>
```

## Commands

### run

```bash
bin/sork run
```

Starts the infinite reconciliation loop (daemon mode).

- Executes `reconcile_all()` in a loop
- Interval configurable via `[orchestrator] interval` or `SORK_INTERVAL`
- Writes a heartbeat on each cycle
- Stops with `Ctrl+C` or `SIGTERM`

This is the primary command for production operation.

### once

```bash
bin/sork once
```

Executes a single reconciliation cycle then exits.

Use cases:
- First launch after installation
- Integration into a cron job
- Testing and debugging

### reconcile-app

```bash
bin/sork reconcile-app <service-name>
```

Reconciles only the specified service.

```bash
# Example
bin/sork reconcile-app web
```

### validate

```bash
bin/sork validate
```

Validates the manifest syntax and keys. Checks:

- Correct INI syntax
- Required keys present (`image`, `health_url` if http)
- Port format
- Valid values for strategies
- If Python is available, uses `manifest_doctor.py` for extended validation

### doctor

```bash
bin/sork doctor [--fix] [--strict-local]
```

Full system diagnostic.

**Options:**

| Option | Description |
|---|---|
| `--fix` | Attempt to automatically fix detected issues |
| `--strict-local` | Verify that health URLs target localhost |

**Checks performed:**

- Required keys present
- Valid port format
- Consistent rollout strategies (blue_green requires `candidate_publish`)
- Valid health, repair, autoscale metric types
- Write access to `SORK_DATA`
- Docker/Podman availability
- Presence of `notify.ini`

**Fix mode:**

```bash
bin/sork doctor --fix
```

- Backs up the original as `.bak`
- Fixes automatically detectable issues
- Rewrites the corrected manifest

### show

```bash
bin/sork show
```

Displays the loaded manifest, section by section. Useful to verify how SORK interprets the configuration.

### version

```bash
bin/sork version
```

Displays the version from the `VERSION` file.

### status

```bash
bin/sork status
```

Displays the daemon state:

- Last heartbeat
- Uptime since last start
- Detected runtime (Docker/Podman)
- Configured paths (`SORK_MANIFEST`, `SORK_DATA`)
- Number of services in the manifest

### resume

```bash
bin/sork resume <service-name>
```

Resumes reconciliation for a suspended or manually paused service.

Removes the files:
- `.sork/state/<app>.suspend_reconcile`
- `.sork/state/<app>.manual_pause`

```bash
# Example: resume a service suspended after creation failures
bin/sork resume api
```
