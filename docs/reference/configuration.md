# Référence de configuration complète

Index de toutes les clés de configuration disponibles dans SORK.

## etc/manifest.ini

### Section [orchestrator]

| Clé | Type | Défaut | Description |
|---|---|---|---|
| `interval` | int | `15` | Intervalle de réconciliation (secondes) |
| `max_repair` | int | `5` | Seuil d'échecs avant alerte critique |
| `remove_orphans` | bool | `0` | Supprimer les conteneurs `sork-*` non déclarés |
| `log_level` | string | `info` | Niveau de log : `debug`, `info`, `warn`, `error` |
| `audit_log_backend` | string | `jsonl` | Backend d'audit : `jsonl` ou `sqlite` |
| `audit_log_all` | bool | `0` | Auditer tous les conteneurs `sork-*` |

### Section [proxy]

| Clé | Type | Défaut | Description |
|---|---|---|---|
| `listen` | string | — | Adresse d'écoute du proxy global (ex: `0.0.0.0:8080`) |
| `autoscale_port_range` | string | — | Plage de ports replicas (ex: `18500-18999`) |
| `health_interval` | int | `3` | Intervalle de health check backends (secondes) |
| `connect_timeout` | int | `5` | Timeout de connexion backend (secondes) |

### Section service (par application)

#### Base

| Clé | Type | Défaut | Description |
|---|---|---|---|
| `image` | string | **requis** | Image Docker |
| `publish` | string | — | Ports CSV (ex: `8080:80,443:443`) |
| `command` | string | — | Override CMD |
| `env` | string | — | Variables env, `;` séparées (ex: `KEY=val;KEY2=val2`) |
| `volumes_bind` | string | — | Bind mounts CSV (ex: `/data:/app/data`) |
| `network` | string | `bridge` | Réseau Docker |
| `config_version` | string | — | Incrémenter pour forcer re-création |

#### Health checks

| Clé | Type | Défaut | Description |
|---|---|---|---|
| `health_type` | string | `http` | `http`, `https`, `tcp`, `none` |
| `health_url` | string | — | URL probe HTTP/HTTPS |
| `health_tcp_port` | int | — | Port probe TCP |
| `health_expect_codes` | string | `200,204` | Codes HTTP acceptés (CSV) |
| `health_timeout` | int | `5` | Timeout probe (secondes) |
| `health_max_bytes` | int | `0` | Taille max réponse (0 = illimité) |

#### Monitoring

| Clé | Type | Défaut | Description |
|---|---|---|---|
| `monitoring_types` | string | `all` | Types actifs (CSV, `all`, `none`) |
| `monitoring_log_tail` | int | `120` | Lignes de log à analyser |
| `monitoring_log_error_regex` | string | (built-in) | Regex anomalies logs |
| `monitoring_http_latency_max_ms` | int | `1200` | Latence max (ms) |
| `monitoring_http_error_rate_threshold_pct` | int | `30` | Seuil taux erreur (%) |
| `monitoring_http_error_rate_window` | int | `20` | Fenêtre glissante (requêtes) |
| `monitoring_disk_usage_max_pct` | int | `90` | Utilisation disque max (%) |

#### Mémoire

| Clé | Type | Défaut | Description |
|---|---|---|---|
| `memory_limit_mb` | int | — | Limite Docker (`-m`) |
| `memory_soft_mb` | int | — | Seuil warning |
| `memory_hard_mb` | int | — | Seuil critique |

#### Rollout & Réparation

| Clé | Type | Défaut | Description |
|---|---|---|---|
| `rollout_strategy` | string | `recreate` | `recreate` ou `blue_green` |
| `candidate_publish` | string | — | Port candidat blue/green |
| `health_url_candidate` | string | — | URL santé candidat |
| `preflight_cmd` | string | — | Commande pré-bascule |
| `repair_strategy` | string | `auto` | `auto`, `restart-only`, `recreate-only`, `purge-only` |
| `purge_on_escalation` | bool | `0` | Supprimer volumes au purge |
| `post_repair_grace` | int | — | Délai post-réparation (secondes) |
| `create_fail_max_attempts` | int | `0` | Max échecs création (0 = illimité) |

#### Comportement

| Clé | Type | Défaut | Description |
|---|---|---|---|
| `manual_stop_pause` | bool | `1` | Pause après arrêt manuel |
| `container_audit_log` | bool | `0` | Activer audit pour ce service |

#### Autoscale

| Clé | Type | Défaut | Description |
|---|---|---|---|
| `autoscale` | bool | `0` | Activer autoscaling |
| `autoscale_min` | int | `1` | Replicas minimum |
| `autoscale_max` | int | `5` | Replicas maximum |
| `autoscale_metric` | string | `http_latency` | Métriques CSV |
| `autoscale_up_threshold` | int | `800` | Seuil scale-up |
| `autoscale_down_threshold` | int | `200` | Seuil scale-down |
| `autoscale_cooldown` | int | `3` | Passages avant action |
| `autoscale_container_port` | int | `8080` | Port interne replica |
| `autoscale_port_base` | int | (auto) | Port hôte de départ |
| `autoscale_lb_publish` | string | — | Adresse LB legacy |
| `autoscale_health_path` | string | `/` | Chemin health backend |
| `autoscale_route` | string | — | Route global proxy |

## etc/notify.ini

### Section [discord]

| Clé | Type | Défaut | Description |
|---|---|---|---|
| `enabled` | bool | `0` | Activer notifications Discord |
| `webhook_url` | string | — | URL webhook Discord |

## Variables d'environnement

| Variable | Défaut | Description |
|---|---|---|
| `SORK_MANIFEST` | `etc/manifest.ini` | Chemin manifest |
| `SORK_NOTIFY_CONF` | `etc/notify.ini` | Chemin config notifications |
| `SORK_DATA` | `./.sork` | Répertoire données |
| `SORK_INTERVAL` | `15` | Intervalle réconciliation (sec) |
| `SORK_MAX_REPAIR` | `5` | Seuil échecs avant alerte |
| `SORK_LOG_LEVEL` | `info` | Niveau de log |
| `SORK_STRICT_LOCAL` | `0` | Forcer health URLs localhost |
| `SORK_LANG` | `fr` | Langue de l'orchestrateur (fr, en) |
| `SORK_HTTP_USER_AGENT` | `Shell-Orchestrator/1.0` | User-Agent pour les health checks HTTP |
