# Health, Notifications, and Audit

## Health Diagnostics

### deep_diagnose() / deep_diagnose_name()

Defined in `lib/health.sh`, `deep_diagnose_name(app, container_name)` performs a comprehensive health assessment. It checks multiple dimensions in order, returning on the first failure. The global variable `SORK_HEALTH_REASON` is set to describe the cause.

#### Check Order

| Priority | Check              | Condition                                  | SORK_HEALTH_REASON                      |
|----------|--------------------|--------------------------------------------|-----------------------------------------|
| 1        | Container exists   | Container ID resolvable                    | `conteneur_introuvable`                 |
| 2        | OOM killed         | Docker OOMKilled flag is `true`            | `oom_killed`                            |
| 3        | Memory (hard)      | Usage >= `memory_hard_mb`                  | `memoire_critique:<used>>=<limit>MiB`   |
| 4        | Memory (soft)      | Usage >= `memory_soft_mb`                  | `memoire_suspecte:<used>>=<limit>MiB`   |
| 5        | Health (TCP)       | TCP connect fails on `health_tcp_port`     | `tcp_refuse`                            |
| 5        | Health (HTTP)      | HTTP probe fails or unexpected status code | `http_code_invalide:<code>` or `curl_echec_ou_timeout` or `http_erreur_serveur:<code>` |
| 6        | Log anomaly        | Regex match in recent container logs       | `logs_anomaly:<matched_line>`           |
| 7        | HTTP latency       | Response time >= `monitoring_http_latency_max_ms` | `http_latency_high:<ms>ms>=<max>ms` |
| 8        | HTTP error rate    | Error rate >= threshold in sliding window  | `http_error_rate_high:<rate>%>=<threshold>%` |
| 9        | Disk usage         | Usage >= `monitoring_disk_usage_max_pct`   | `disk_usage_high:<path>=<pct>%>=<max>%` |

Each check is gated by `monitoring_enabled(app, type)`. If `monitoring_types = none`, all checks are skipped. If `monitoring_types = all` (default), all checks run.

### Monitoring Types

The `monitoring_types` manifest key accepts a comma-separated list of types to enable:

| Type              | What it checks                                                |
|-------------------|---------------------------------------------------------------|
| `health`          | HTTP or TCP health probe (based on `health_type`)             |
| `memory`          | Container memory usage against soft/hard thresholds           |
| `oom`             | Docker OOMKilled flag                                         |
| `restart`         | Docker restart counter changes (unexpected restarts)          |
| `logs`            | Recent container log lines against error regex pattern        |
| `http_latency`    | HTTP response time against `monitoring_http_latency_max_ms`   |
| `http_error_rate` | 5xx/timeout rate in a sliding window                          |
| `disk`            | Disk usage on SORK_DATA and bound volume host paths           |

Special values: `all` (enable everything), `none` (disable all monitoring).

### HTTP Health Probes

HTTP probes (`health_http()`) perform a full request cycle:

1. Send HTTP GET with `User-Agent: Shell-Orchestrator/1.0`.
2. Compare response code against `health_expect_codes` (comma-separated, default `200,204`).
3. Reject any 5xx response regardless of whitelist.
4. Optionally check response body size against `health_max_bytes`.
5. Respect `SORK_STRICT_LOCAL=1`: refuse URLs not targeting `127.0.0.1`, `localhost`, or `::1`.

### Log Anomaly Detection

When `logs` monitoring is enabled:

1. Fetch the last N lines of container logs (`monitoring_log_tail`, default 120).
2. Search for the first line matching `monitoring_log_error_regex` (case-insensitive).
3. If matched, capture a resource snapshot (CPU%, memory) and include it in the reason.
4. The matched line is truncated to 180 characters.

### HTTP Error Rate Tracking

A sliding window tracks HTTP error rate per service:

1. On each check, probe the health URL and record whether the response code is 5xx or 000 (timeout).
2. Maintain `fail` and `total` counters in `.sork/state/<app>.http_errrate`.
3. When the window exceeds `monitoring_http_error_rate_window` (default 20), proportionally reduce both counters.
4. Minimum 5 samples required before the threshold is evaluated.

### Disk Usage Check

Checks disk usage percentage on:

1. The `SORK_DATA` directory (`.sork/`).
2. Each host path from the service's `volumes_bind` entries.

Triggers when usage meets or exceeds `monitoring_disk_usage_max_pct` (default 90%).

## Notifications

### Discord Notifications (Bash Engine)

The bash engine sends Discord notifications via `notify_all()` in `lib/notify.sh`.

**Configuration**: `etc/notify.ini` with a `[discord]` section:

```ini
[discord]
enabled = 1
webhook_url = https://discord.com/api/webhooks/XXX/YYY
```

**Message format**: Discord embeds with French-language formatting. Each notification includes:

- A translated event title (e.g., "Service signale comme non sain").
- A detailed explanation of the cause, translated from internal reason codes.
- Key-value data extracted from the incident detail string.
- Color-coded severity (info=blue, warn=yellow, critical=red, ok=green).

**Events that trigger notifications**:

- Health failures (`unhealthy`)
- Repair actions (`repair_restart`, `repair_recreate`, `repair_purge`)
- Repair failures (`repair_failed`, `escalade_max`)
- Container state changes (`not_running`, `create_missing`, `container_restarted`)
- Blue/green rollout events (`rollout_candidate_create`, `rollout_candidate_unhealthy`, `rollout_switch`)
- Manual pause detection (`manual_stop_detected`, `manual_pause_active`, `manual_resume_detected`)
- Orphan removal (`orphan_removed`)
- Recovery (`health_recovered`)
- Manifest issues (`manifest_unreadable`, `manifest_repaired_auto`, `manifest_repair_failed`)
- Autoscale events (`autoscale_up`, `autoscale_down`, `autoscale_max_reached`, `autoscale_lb_started`)
- Proxy events (`global_proxy_started`, `global_proxy_stopped`)
- Reconciliation suspension (`reconcile_suspended`)

### Notifications (Python Backend)

The FastAPI backend has its own notification system in `core/notifications.py`:

- **In-memory buffer**: thread-safe list, capped at 200 entries.
- **File persistence**: saved to `.sork/notifications.json` after each write. Loaded on first access. Survives container restarts.
- **Types**: `info`, `warning`, `error`, `success`.
- **API endpoints**:
  - `GET /api/notifications/` -- list all notifications with unread count.
  - `POST /api/notifications/read` -- mark all as read.
  - `GET /api/notifications/stream` -- SSE stream for real-time updates.
- **Structured logging**: each notification is also recorded via `log_action()`.

## Incident Recording

### incident_record()

Defined in `lib/incidents.sh`, this is the central function for recording events:

```bash
incident_record <app> <severity> <event> <detail> [skip_discord]
```

Parameters:

| Parameter      | Values                        | Description                           |
|----------------|-------------------------------|---------------------------------------|
| `app`          | service name or `global`      | Affected service                      |
| `severity`     | `info`, `warn`, `critical`, `ok` | Event severity                     |
| `event`        | event identifier              | Machine-readable event type           |
| `detail`       | free text                     | Human-readable detail string          |
| `skip_discord` | `1` or empty                  | If `1`, do not send Discord alert     |

Each call produces two outputs:

1. **Text log**: appended to `.sork/incidents/incidents.log` in the format:
   ```
   2025-01-15T10:30:00Z | warn | myapp | unhealthy | detail=http_code_invalide:503 count=3
   ```

2. **Daily JSONL archive**: appended to `.sork/incidents/YYYY-MM-DD.jsonl`:
   ```json
   {"ts":"2025-01-15T10:30:00Z","severity":"warn","app":"myapp","event":"unhealthy","detail":"..."}
   ```

Unless `skip_discord` is `1`, `notify_all()` is called to send a Discord notification.

## Audit Log

### Container Audit

The audit log tracks container lifecycle events (create, start, stop, restart, remove) performed by SORK or via the UI.

**Configuration** (in `[orchestrator]`):

| Key                   | Default | Description                                     |
|-----------------------|---------|-------------------------------------------------|
| `audit_log_backend`   | `jsonl` | Storage backend: `jsonl` or `sqlite`            |
| `audit_log_all`       | `0`     | If `1`, audit all sork-* containers             |
| `container_audit_log` | `0`     | Per-service: if `1`, audit this service          |

**Backends**:

- **JSONL** (default): `.sork/audit/container-audit.jsonl`, one JSON object per line:
  ```json
  {"ts":"2025-01-15T10:30:00Z","app":"myapp","container":"sork-myapp","event":"create","source":"reconcile","detail":""}
  ```

- **SQLite**: `.sork/audit/container-audit.sqlite` with a table `audit_events`:
  ```sql
  CREATE TABLE audit_events (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      ts TEXT NOT NULL,
      app TEXT NOT NULL,
      container TEXT NOT NULL,
      event TEXT NOT NULL,
      source TEXT NOT NULL,
      detail TEXT
  );
  ```

**Sources**:

| Source      | Description                                    |
|-------------|------------------------------------------------|
| `reconcile` | Action taken by the bash reconciliation engine |
| `autoscale` | Action taken by the autoscale module           |
| `ui`        | Action triggered via the web interface         |

**Bash-side recording**: `sork_audit_event()` in `lib/audit.sh` calls `audit_log.py` via `python3`. If `python3` is not available, the call is silently skipped (noop).

**Python-side recording**: the FastAPI backend imports `audit_log.py` directly and can record events when containers are managed through the API.

## Structured Logging System

### Bash Daemon Logs

`sork_log()` in `lib/common.sh` writes:

- **stderr**: human-readable format `[timestamp] [level] message`.
- **JSON log file**: `.sork/logs/sork-daemon.log` with structured entries:
  ```json
  {"ts":"2025-01-15T10:30:00Z","level":"INFO","logger":"sork.daemon","message":"..."}
  ```

Log rotation occurs at each heartbeat (`sork_daemon_heartbeat()`), rotating by file size.

### Python Backend Logs

`core/logging.py` provides:

- **JsonFormatter**: writes to `.sork/logs/sork-ui.log` with daily rotation (using `TimedRotatingFileHandler`) and 30-day retention.
- **ConsoleFormatter**: colored output for `docker logs` consumption.
- **Request middleware**: logs all API requests (excluding noisy endpoints like `/api/ping` and SSE streams) with method, path, status, duration, and client IP.
- **Action logging**: `log_action(category, action, detail, level)` for structured event recording.

Log level is controlled by `SORK_LOG_LEVEL` environment variable (default `INFO`).
