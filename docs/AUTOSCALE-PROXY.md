# Autoscale and Proxy

## Overview

SORK provides horizontal scaling for services configured with `autoscale = 1`. Replicas are standard Docker containers managed by the SORK daemon. A socat-based reverse-proxy handles load balancing with health-aware round-robin routing.

**Dependency**: the proxy requires `socat` installed on the host (`apt install socat` / `dnf install socat`).

## Horizontal Scaling

### Replica Naming

Replicas follow the naming convention `sork-<app>-r<N>`:

```
sork-web-api-r1
sork-web-api-r2
sork-web-api-r3
```

Each replica is a standalone Docker container with labels:
- `sork.app=<app>` -- identifies the parent service.
- `sork.role=replica` -- marks it as an autoscale replica.
- `sork.replica=<N>` -- replica index.

### Port Allocation

Each replica needs a unique host port mapped to the container's `autoscale_container_port`. Two modes are supported:

1. **Explicit base port** (`autoscale_port_base`): replica N gets port `autoscale_port_base + N - 1`.
   ```
   autoscale_port_base = 18601
   # r1 -> 18601, r2 -> 18602, r3 -> 18603
   ```

2. **Auto-allocation** (default): ports are allocated from the `[proxy]` section's `port_range_start`/`port_range_end` range. Allocated ports are persisted in `.sork/autoscale/` and reused across restarts.

Replicas bind to `127.0.0.1:<port>:<container_port>` (local only, proxied externally).

## Autoscale Reconciliation

When `reconcile_app()` detects `autoscale = 1`, it delegates entirely to `autoscale_reconcile()`:

```
autoscale_reconcile(app)
  |
  1. Ensure minimum replicas exist (scale up to autoscale_min)
  2. Health-check all replicas (restart/recreate unhealthy ones)
  3. Ensure the proxy is running (per-app or global)
  4. Update the backends file
  5. Collect the scaling metric
  6. Make a scale decision (up / down / stable)
  7. Execute the decision
```

### Manual Pause

Autoscale services respect the `manual_stop_pause` mechanism. When manual pause is active, `autoscale_reconcile()` returns immediately without modifying replicas or the proxy.

## Scale Decision Logic

### Metric Collection

The `autoscale_metric` key specifies one or more metrics (comma-separated). The first metric is the primary; secondary metrics can force a scale-up if critical.

| Metric            | Measurement                                        | Unit     |
|-------------------|---------------------------------------------------|----------|
| `http_latency`    | Response time of HTTP probe via health path       | ms       |
| `memory`          | Average memory usage across all running replicas  | MB       |
| `http_error_rate` | HTTP 5xx or timeout rate on a single probe        | % (0/100)|

For `http_latency`, the probe targets:
- The dedicated proxy port (for `port:<N>` routes).
- A direct backend replica (if global proxy is active).
- The per-app load balancer (legacy mode).

For `memory`, the average across all running replicas is computed from `docker stats`.

For `http_error_rate`, a single probe is made. A 5xx or timeout response yields 100%; a success yields 0%. If the rate exceeds 50%, it is flagged as critical and forces scale-up regardless of the primary metric.

### Threshold and Cooldown

The scale decision uses a streak-based cooldown to prevent flapping:

```
if value > autoscale_up_threshold:
    streak++
    if streak >= autoscale_cooldown:
        decision = "up"
        streak = 0

elif value < autoscale_down_threshold:
    streak--
    if streak <= -autoscale_cooldown:
        decision = "down"
        streak = 0

else:
    streak = 0
    decision = "stable"
```

The streak counter is persisted in `.sork/state/<app>.autoscale_cooldown`.

**Example thresholds** by metric type:

| Metric            | Typical up_threshold | Typical down_threshold | Unit |
|-------------------|---------------------|------------------------|------|
| `http_latency`    | 600--1000           | 100--200               | ms   |
| `memory`          | 400--800            | 100--200               | MB   |
| `http_error_rate` | 30--50              | 5--10                  | %    |

### Scale Up

When the decision is `up`:

1. Check that `current_count < autoscale_max`. If at max, log a warning and send a one-time alert.
2. Create a new replica container `sork-<app>-r<N+1>` with the next available port.
3. Update the backends file.
4. Reload the proxy configuration.

### Scale Down

When the decision is `down`:

1. Check that `current_count > autoscale_min`. If at min, do nothing.
2. Remove the highest-numbered replica `sork-<app>-r<N>`.
3. Update the backends file.
4. Reload the proxy configuration.

### Replica Health Check

`autoscale_check_replicas()` verifies each replica on every pass:

- If a replica container has disappeared, it is recreated at its original index and port.
- If a replica is stopped (exists but not running), it is restarted (or recreated if restart fails).

## Proxy Architecture

SORK includes a bash + socat TCP reverse-proxy (`lib/proxy.sh`). It runs as a host process (not a container) to access local replica ports.

### Operating Modes

#### Legacy Mode (per-app proxy)

A dedicated proxy process per autoscale service. Configured with `autoscale_lb_publish`:

```ini
[web-api]
autoscale = 1
autoscale_lb_publish = 127.0.0.1:8080:80
```

The proxy listens on `autoscale_lb_publish` and distributes traffic to replicas listed in the backends file.

#### Global Mode (route-based proxy)

A single proxy process for all autoscale services, configured via the `[proxy]` section and per-service `autoscale_route` keys:

```ini
[proxy]
listen = 0.0.0.0:8080

[frontend]
autoscale_route = host:app.example.com

[api]
autoscale_route = path:/api

[default-app]
autoscale_route = default
```

Route types:

| Type      | Pattern Example       | Match Criteria                          |
|-----------|-----------------------|-----------------------------------------|
| `host`    | `host:app.example.com`| HTTP Host header matches exactly        |
| `path`    | `path:/api`           | Request path starts with prefix         |
| `port`    | `port:9090`           | Dedicated proxy on the specified port   |
| `default` | `default`             | Fallback route (matched last)           |

The global proxy reads `routes.conf` which maps route rules to backends files:

```
# type  pattern     app_name    backends_file
host    app.local   web-api     /path/.sork/autoscale/web-api.backends
path    /api        api-svc     /path/.sork/autoscale/api-svc.backends
default *           frontend    /path/.sork/autoscale/frontend.backends
```

#### Dedicated Port Mode

`port:<N>` routes launch a separate proxy process on the specified port, independent of the global proxy. This is useful for services that need their own entry point:

```ini
[admin-panel]
autoscale_route = port:9090
```

### Load Balancing

The proxy uses health-aware round-robin:

1. Read the backends file (one line per backend: `name host port`).
2. Read health status from state files (`health_<name>` in the state directory).
3. Select the next healthy backend using an atomic round-robin index.
4. Forward the TCP connection via `socat`.

If no healthy backends are available, the connection is refused.

### Backend Health Monitoring

A background health loop runs every `health_interval` seconds (default 3):

1. For each backend, send an HTTP GET to `http://<host>:<port><health_path>`.
2. Write `1` (healthy) or `0` (unhealthy) to the state file.
3. Timeout: `SORK_PROXY_HEALTH_TIMEOUT` (default 2 seconds).

Backends are automatically removed from rotation when unhealthy and re-added when they recover.

### Proxy Admin Endpoints

The proxy intercepts special paths before forwarding:

| Endpoint                | Description                                      |
|-------------------------|--------------------------------------------------|
| `GET /sork-proxy/health`  | Returns 200 if any backend is healthy, 503 otherwise |
| `GET /sork-proxy/state`   | JSON: backends list, health status, metrics        |
| `GET /sork-proxy/metrics` | Prometheus text format metrics                     |
| `GET /sork-proxy/routes`  | Current routing table (global mode only)           |

### Hot Reload

The proxy automatically detects changes to the backends file by monitoring its modification time (mtime). When the file changes:

- The backend list is re-read on the next connection.
- Health state is preserved (existing backends keep their status).
- New backends start as healthy and are probed on the next health check cycle.

To force a reload, the autoscale module touches the backends file: `touch <backends_file>`.

### Proxy Lifecycle

| Action              | How                                                   |
|---------------------|-------------------------------------------------------|
| Start (global)      | `global_proxy_ensure()` in `autoscale.sh`             |
| Start (per-app)     | `autoscale_lb_ensure()` in `autoscale.sh`             |
| Start (dedicated)   | `_autoscale_lb_ensure_dedicated()` in `autoscale.sh`  |
| Stop (global)       | `global_proxy_stop()` -- kills process group          |
| Stop (per-app)      | `autoscale_lb_stop()` -- kills by PID                 |
| Reload              | `autoscale_lb_reload()` -- touches backends file      |

PID files are stored in `.sork/autoscale/`:
- Global proxy: `global-proxy.pid`
- Per-app proxy: `<app>-lb.pid`

Stale socat processes on the listen port are detected and killed before starting a new proxy.

## Configuration Reference

### Manifest Keys (per service)

| Key                       | Default        | Description                                         |
|---------------------------|----------------|-----------------------------------------------------|
| `autoscale`               | `0`            | Enable autoscaling (`1` or `true`)                  |
| `autoscale_min`           | `1`            | Minimum replicas                                    |
| `autoscale_max`           | `5`            | Maximum replicas                                    |
| `autoscale_metric`        | `http_latency` | Scaling metric(s), comma-separated                  |
| `autoscale_up_threshold`  | `800`          | Scale-up threshold                                  |
| `autoscale_down_threshold`| `200`          | Scale-down threshold                                |
| `autoscale_cooldown`      | `3`            | Consecutive passes before action                    |
| `autoscale_container_port`| `8080`         | Port inside each replica                            |
| `autoscale_port_base`     | (auto)         | Starting host port for replicas                     |
| `autoscale_lb_publish`    | (none)         | Per-app proxy listen address                        |
| `autoscale_health_path`   | `/`            | Health probe path                                   |
| `autoscale_route`         | (none)         | Global proxy route rule                             |

### [proxy] Section Keys

| Key                  | Default | Description                                      |
|----------------------|---------|--------------------------------------------------|
| `listen`             | --      | Listen address (e.g., `0.0.0.0:8080`)            |
| `port_range_start`   | --      | Port auto-allocation range start                 |
| `port_range_end`     | --      | Port auto-allocation range end                   |
| `health_interval`    | `3`     | Health check interval (seconds)                  |
| `health_path`        | `/`     | Default health probe path                        |
| `connect_timeout`    | `5`     | Backend connection timeout (seconds)             |
| `log_level`          | `info`  | Proxy log level (`debug` or `info`)              |

## Example: Autoscale with Global Proxy

```ini
[orchestrator]
interval = 8
remove_orphans = 1

[proxy]
listen = 0.0.0.0:8080
port_range_start = 18500
port_range_end = 18599
health_interval = 3

[web-frontend]
image = myapp-frontend:latest
autoscale = 1
autoscale_min = 2
autoscale_max = 6
autoscale_container_port = 3000
autoscale_metric = http_latency
autoscale_up_threshold = 500
autoscale_down_threshold = 100
autoscale_cooldown = 3
autoscale_route = default
autoscale_health_path = /health
health_type = http
health_url = http://127.0.0.1:8080/health
config_version = 1

[api-service]
image = myapp-api:latest
autoscale = 1
autoscale_min = 1
autoscale_max = 4
autoscale_container_port = 8080
autoscale_metric = http_latency,http_error_rate
autoscale_up_threshold = 800
autoscale_down_threshold = 200
autoscale_cooldown = 3
autoscale_route = path:/api
autoscale_health_path = /health
health_type = http
health_url = http://127.0.0.1:8080/api/health
config_version = 1
```

This configuration:
- Starts a global proxy on port 8080.
- Routes `/api/*` requests to `api-service` replicas.
- Routes all other requests to `web-frontend` replicas (default route).
- Auto-allocates replica ports from the 18500--18599 range.
- Scales based on HTTP latency (primary) and error rate (secondary, forces scale-up if critical).
