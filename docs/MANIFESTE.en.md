# Manifest Configuration

## Files

| File                     | Purpose                                                   |
|--------------------------|-----------------------------------------------------------|
| `etc/manifest.ini`       | Active manifest -- defines the desired state of services  |
| `etc/manifest.ini.example` | Reference manifest with all documented keys            |
| `etc/notify.ini`         | Active notification configuration                         |
| `etc/notify.ini.example` | Reference notification config                             |

## Format

The manifest uses standard INI format. Each section (except reserved sections) defines a service. Keys are `key = value` pairs. Comments start with `#`.

```ini
# This is a comment
[section-name]
key = value
quoted_key = "quoted value"
```

The parser (`lib/manifest.sh`) stores values in a bash associative array `MNF["section|key"]` and preserves section order in `MNF_APPS_ORDER`.

## Reserved Sections

These section names are not treated as application services:

| Section          | Purpose                                              |
|------------------|------------------------------------------------------|
| `[orchestrator]` | Global daemon settings                               |
| `[proxy]`        | Global reverse-proxy configuration (for autoscale)   |
| `[notify]`       | (Reserved, currently unused in INI)                  |
| `[global]`       | (Reserved for future use)                            |

## [orchestrator] Section

Controls the behavior of the `bin/sork run` daemon.

| Key                 | Default | Description                                                    |
|---------------------|---------|----------------------------------------------------------------|
| `interval`          | `15`    | Seconds between reconciliation passes                          |
| `max_repair`        | `5`     | Failure count before alerting (SORK_MAX_REPAIR)                |
| `remove_orphans`    | `0`     | If `1`, remove `sork-*` containers not in the manifest         |
| `log_level`         | `info`  | Log level: `debug`, `info`, `warn`, `error`                   |
| `audit_log_backend` | `jsonl` | Audit storage backend: `jsonl` or `sqlite`                     |
| `audit_log_all`     | `0`     | If `1`, audit all `sork-*` containers (not just per-service)   |

These keys can also be set via environment variables (`SORK_INTERVAL`, `SORK_MAX_REPAIR`, `SORK_REMOVE_ORPHANS`, `SORK_LOG_LEVEL`). Manifest values override environment variables.

## [proxy] Section

Configures the global socat-based reverse-proxy used by autoscale services with route-based load balancing.

| Key                  | Default | Description                                            |
|----------------------|---------|--------------------------------------------------------|
| `listen`             | --      | Listen address (required). Format: `host:port` (e.g., `0.0.0.0:8080`) |
| `port_range_start`   | --      | Start of auto-allocation port range for replicas       |
| `port_range_end`     | --      | End of auto-allocation port range for replicas         |
| `health_interval`    | `3`     | Seconds between backend health checks                  |
| `connect_timeout`    | `5`     | Seconds before a backend connection times out          |
| `log_level`          | `info`  | Proxy log level: `debug` or `info`                     |
| `health_path`        | `/`     | HTTP path for backend health probes                    |

## Application Sections

Each non-reserved section defines a service. The section name becomes the service identifier. Container names follow the pattern `sork-<section-name>`.

### Core Keys

| Key              | Required | Default     | Description                                          |
|------------------|----------|-------------|------------------------------------------------------|
| `image`          | Yes      | --          | Docker image (e.g., `nginx:latest`, `myapp:1`)      |
| `publish`        | No       | (none)      | Port mappings, comma-separated (e.g., `8080:80,443:443`) |
| `command`        | No       | (none)      | Override container CMD (shell-style string)          |
| `env`            | No       | (none)      | Environment variables, semicolon-separated (`KEY=VAL;KEY2=VAL2`) |
| `volumes_bind`   | No       | (none)      | Bind mounts, comma-separated (`/host:/container,/data:/data`) |
| `network`        | No       | `bridge`    | Docker network name                                 |
| `config_version` | No       | (empty)     | Increment to trigger container recreation            |

### Health Check Keys

| Key                  | Default    | Description                                           |
|----------------------|------------|-------------------------------------------------------|
| `health_type`        | `http`     | Health check type: `http`, `https`, `tcp`, `none`     |
| `health_url`         | (none)     | URL for HTTP/HTTPS health checks                      |
| `health_tcp_port`    | (none)     | Port for TCP health checks                            |
| `health_expect_codes`| `200,204`  | Accepted HTTP status codes, comma-separated           |
| `health_timeout`     | `5`        | Health check timeout in seconds                       |
| `health_max_bytes`   | `0`        | Max response body size (0 = unlimited)                |

### Monitoring Keys

| Key                                        | Default  | Description                                        |
|--------------------------------------------|----------|----------------------------------------------------|
| `monitoring_types`                         | `all`    | Comma-separated list or `all` or `none`. Types: `health`, `memory`, `oom`, `restart`, `logs`, `http_latency`, `http_error_rate`, `disk` |
| `monitoring_log_tail`                      | `120`    | Number of recent log lines to scan                 |
| `monitoring_log_error_regex`               | `error\|exception\|panic\|fatal\|traceback\|out of memory\|oom` | Regex pattern for log anomaly detection |
| `monitoring_http_latency_max_ms`           | `1200`   | Maximum acceptable HTTP latency in milliseconds    |
| `monitoring_http_error_rate_threshold_pct` | `30`     | Error rate percentage threshold                    |
| `monitoring_http_error_rate_window`        | `20`     | Sliding window size for error rate calculation     |
| `monitoring_disk_usage_max_pct`            | `90`     | Maximum disk usage percentage before alert         |

### Memory Keys

| Key              | Default | Description                                              |
|------------------|---------|----------------------------------------------------------|
| `memory_limit_mb`| (none)  | Docker memory limit (`-m` flag) in MB                    |
| `memory_soft_mb` | (none)  | Soft memory threshold for warnings (MB)                  |
| `memory_hard_mb` | (none)  | Hard memory threshold -- triggers repair if exceeded (MB)|

### Rollout and Repair Keys

| Key                        | Default    | Description                                          |
|----------------------------|------------|------------------------------------------------------|
| `rollout_strategy`         | `recreate` | Deployment strategy: `recreate` or `blue_green`      |
| `candidate_publish`        | (none)     | Port mapping for blue/green candidate (required for `blue_green`) |
| `health_url_candidate`     | (none)     | Health URL for the candidate (if different from live) |
| `preflight_cmd`            | (none)     | Command to run inside candidate before switch         |
| `repair_strategy`          | `auto`     | Repair strategy: `auto`, `restart-only`, `recreate-only`, `purge-only` |
| `purge_on_escalation`      | `0`        | If `1`, purge named volumes on purge phase            |
| `post_repair_grace`        | (none)     | Grace period after repair (seconds)                   |
| `create_fail_max_attempts` | `0`        | Max consecutive creation failures before suspending (0 = unlimited) |

### Behavior Keys

| Key                   | Default | Description                                                |
|-----------------------|---------|------------------------------------------------------------|
| `manual_stop_pause`   | `1`     | If `1`, do not auto-restart after manual stop (exit codes 0, 137, 143) |
| `container_audit_log` | `0`     | If `1`, record audit events for this service               |

### Autoscale Keys

See also [AUTOSCALE-PROXY.en.md](AUTOSCALE-PROXY.en.md) for detailed autoscale documentation.

| Key                       | Default        | Description                                         |
|---------------------------|----------------|-----------------------------------------------------|
| `autoscale`               | `0`            | Enable autoscaling: `1` or `true`                   |
| `autoscale_min`           | `1`            | Minimum number of replicas                          |
| `autoscale_max`           | `5`            | Maximum number of replicas                          |
| `autoscale_metric`        | `http_latency` | Metric(s) for scaling decisions (CSV: `http_latency`, `memory`, `http_error_rate`) |
| `autoscale_up_threshold`  | `800`          | Value above which to scale up (ms for latency, MB for memory, % for error rate) |
| `autoscale_down_threshold`| `200`          | Value below which to scale down                     |
| `autoscale_cooldown`      | `3`            | Consecutive passes before acting                    |
| `autoscale_container_port`| `8080`         | Port inside each replica container                  |
| `autoscale_port_base`     | (auto)         | Starting host port for replicas (auto-allocated from [proxy] range if absent) |
| `autoscale_lb_publish`    | (none)         | Load balancer listen address (e.g., `127.0.0.1:8080:80`). Required unless using `autoscale_route` with `[proxy]`. |
| `autoscale_health_path`   | `/`            | HTTP path for backend health probes                 |
| `autoscale_route`         | (none)         | Routing rule for global proxy: `host:<hostname>`, `path:</prefix>`, `port:<N>`, or `default` |

## notify.ini

Notification configuration in INI format. Currently supports Discord only.

### [discord] Section

| Key           | Default | Description                                    |
|---------------|---------|------------------------------------------------|
| `enabled`     | `0`     | Enable Discord notifications: `1` or `true`    |
| `webhook_url` | --      | Discord webhook URL                            |

Example `etc/notify.ini`:

```ini
[discord]
enabled = 1
webhook_url = https://discord.com/api/webhooks/XXX/YYY
```

Events sent to Discord include: health failures, repair actions, blue/green rollouts, orphan removal, manifest errors, autoscale events, and recovery notifications.

## Complete Example

```ini
[orchestrator]
interval = 8
max_repair = 4
remove_orphans = 1
log_level = info
audit_log_backend = jsonl
audit_log_all = 0

[proxy]
listen = 0.0.0.0:8080
port_range_start = 18500
port_range_end = 18599
health_interval = 3
connect_timeout = 5

[sork-ui]
image = sork-ui:2
publish = 127.0.0.1:18100:8080
health_type = http
health_url = http://127.0.0.1:18100/api/ping
health_expect_codes = 200
health_timeout = 4
config_version = 1
manual_stop_pause = 1
monitoring_types = all
rollout_strategy = recreate
volumes_bind = /path/to/shell-orchestrator:/workspace,/var/run/docker.sock:/var/run/docker.sock
command = python /app/server.py
container_audit_log = 0

[web-app]
image = myapp:latest
publish = 127.0.0.1:3000:3000
health_type = http
health_url = http://127.0.0.1:3000/health
health_expect_codes = 200
health_timeout = 5
config_version = 1
rollout_strategy = blue_green
candidate_publish = 127.0.0.1:3001:3000
memory_soft_mb = 256
memory_hard_mb = 512
monitoring_types = health,memory,oom,logs,http_latency
monitoring_http_latency_max_ms = 800

[web-scaled]
image = myapp:latest
autoscale = 1
autoscale_min = 2
autoscale_max = 6
autoscale_container_port = 8080
autoscale_metric = http_latency
autoscale_up_threshold = 600
autoscale_down_threshold = 150
autoscale_cooldown = 3
autoscale_route = host:app.example.com
health_type = http
health_url = http://127.0.0.1:8080/health
health_expect_codes = 200
config_version = 1
```
