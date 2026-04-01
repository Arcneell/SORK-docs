# Notification Configuration

SORK sends Discord notifications for important events: outages, repairs, autoscaling, manifest errors, etc.

## Configuration

Edit `etc/notify.ini`:

```ini
[discord]
enabled = 1
webhook_url = https://discord.com/api/webhooks/YOUR_ID/YOUR_TOKEN
```

| Key | Description |
|---|---|
| `enabled` | `1` to enable, `0` to disable |
| `webhook_url` | Full Discord webhook URL |

### Creating a Discord Webhook

1. Open your Discord server
2. **Server Settings** > **Integrations** > **Webhooks**
3. Click **New Webhook**
4. Choose the destination channel
5. Copy the webhook URL
6. Paste it into `etc/notify.ini`

## Notified Events

| Event | Severity | Description |
|---|---|---|
| Health check failed | `warn` / `critical` | Service outage detected |
| Repair performed | `info` | Restart, recreate, or purge executed |
| Repair failed | `critical` | All repair strategies have failed |
| Blue/green rollout | `info` | Switch to new container |
| Blue/green failed | `critical` | Candidate failed health checks |
| Autoscale up | `info` | Replica added |
| Autoscale down | `info` | Replica removed |
| Manifest error | `critical` | Configuration file is invalid |
| Orphan removed | `warn` | Undeclared sork-* container removed |
| Service restored | `ok` | A previously failing service is healthy again |
| Proxy event | `info` / `warn` | Load balancer change |

## Message Format

Notifications are sent as **Discord embeds** with:

- A title describing the event
- A detailed description
- A color code based on severity (green, orange, red)
- The name of the affected service

### Reason Codes

Each notification includes a translated reason code:

| Code | Description |
|---|---|
| `curl_failed` | curl request failed |
| `oom_killed` | Container killed due to out of memory (OOM) |
| `memory_hard` | Critical memory threshold exceeded |
| `memory_soft` | Warning memory threshold exceeded |
| `http_5xx` | HTTP 5xx response code |
| `tcp_refused` | TCP connection refused |
| `high_latency` | Response time too high |
| `high_error_rate` | HTTP error rate too high |
| `disk_full` | Disk usage too high |
| `log_anomaly` | Anomaly detected in logs |

## Anti-Spam

SORK avoids notification spam:

- Certain incidents are marked `skip_discord` to avoid duplicates in tight loops
- Recovery notifications are sent only once per restoration
- The `manifest_load_warn.notified` flag prevents repeated alerts for the same manifest error
