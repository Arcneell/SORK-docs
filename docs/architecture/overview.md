# Vue d'ensemble de l'architecture

## Principes de conception

| Principe | Description |
|---|---|
| **Single-node** | Pas de coordination multi-machines. Conçu pour orchestrer des conteneurs sur un seul hôte. |
| **Déclaratif** | L'état désiré est défini dans un fichier INI. Le moteur converge vers cet état à chaque cycle. |
| **Self-healing** | Réparation automatique par escalade (restart → recreate → purge) sans intervention. |
| **Minimal** | Dépendances : Bash 5, curl, Docker ou Podman. Pas de runtime additionnel requis. |
| **Observable** | Audit trail, journal d'incidents, alertes Discord, métriques Prometheus. |

---

## Composants

### Pipeline de réconciliation

```mermaid
graph LR
    A["manifest.ini"] --> B["manifest_load()"]
    B --> C["reconcile_app()"]
    C --> D["deep_diagnose_name()"]
    D -->|sain| E["OK"]
    D -->|échec| F["repair_execute()"]
    F --> G["Docker / Podman"]
```

### Pipeline autoscale

```mermaid
graph LR
    A["autoscale_reconcile()"] --> B["collect_metric()"]
    B --> C["autoscale_decision()"]
    C -->|up| D["scale_up()"]
    C -->|down| E["scale_down()"]
    D --> F["write_backends + proxy reload"]
    E --> F
```

### Observabilité

```mermaid
graph LR
    A["Événement"] --> B["incident_record()"]
    B --> C["incidents.log"]
    B --> D["YYYY-MM-DD.jsonl"]
    B --> E["Discord webhook"]
    A --> F["sork_audit_event()"]
    F --> G["JSONL / SQLite"]
```

---

## Flux de données principal

```mermaid
sequenceDiagram
    participant M as manifest.ini
    participant S as bin/sork run
    participant R as reconcile_app()
    participant H as deep_diagnose_name()
    participant D as Docker
    participant N as Discord
    participant I as incidents.log

    loop Toutes les N secondes
        S->>M: manifest_load()
        S->>S: Détecter runtime (docker/podman)
        S->>R: Pour chaque app dans MNF_APPS_ORDER
        R->>D: container_exists() ?
        alt N'existe pas
            R->>D: container_create()
            R->>I: incident_record(info, created)
        else Existe
            R->>D: container_running() ?
            alt En marche
                R->>H: deep_diagnose_name()
                alt Sain
                    H-->>R: return 0 (OK)
                else En panne
                    H-->>R: return 1 (raison)
                    R->>D: repair_execute() (restart/recreate/purge)
                    R->>I: incident_record(severity, event)
                    R->>N: notify_all() → Discord embed
                end
            else Arrêté
                R->>D: container_start()
            end
        end
        S->>S: remove_orphan_containers()
        S->>S: sork_daemon_heartbeat()
        S->>S: sleep SORK_INTERVAL
    end
```

---

## Communication entre modules

```mermaid
graph LR
    manifest["manifest.sh<br>MNF[] global array"] --> health
    manifest --> repair
    manifest --> autoscale
    manifest --> runtime
    manifest --> proxy
    manifest --> doctor

    common["common.sh<br>Logging, état, ports"] --> health
    common --> repair
    common --> autoscale
    common --> runtime

    health["health.sh"] -->|diagnostic| repair["repair.sh"]
    repair -->|docker ops| runtime["runtime.sh"]
    repair -->|alertes| notify["notify.sh"]
    repair -->|journal| incidents["incidents.sh"]
    repair -->|trail| audit["audit.sh"]

    autoscale["autoscale.sh"] -->|replicas| runtime
    autoscale -->|backends| proxy["proxy.sh"]
    autoscale -->|journal| incidents

    audit -->|python| auditpy["audit_log.py"]
    doctor["doctor.sh"] -->|python| doctorpy["manifest_doctor.py"]

    style manifest fill:#e67e22,color:#fff
    style common fill:#95a5a6,color:#fff
    style health fill:#e74c3c,color:#fff
    style repair fill:#3498db,color:#fff
    style autoscale fill:#9b59b6,color:#fff
    style proxy fill:#2ecc71,color:#fff
    style notify fill:#f39c12,color:#fff
    style incidents fill:#8e44ad,color:#fff
    style audit fill:#1abc9c,color:#fff
```

---

## Conventions de nommage

### Conteneurs

| Type | Format | Exemple |
|---|---|---|
| Service standard | `sork-<app>` | `sork-web` |
| Candidat blue/green | `sork-<app>-candidate-<timestamp>` | `sork-web-candidate-1705312200` |
| Replica autoscale | `sork-<app>-r<N>` | `sork-web-r3` |
| Load balancer (legacy) | `sork-<app>-lb` | `sork-web-lb` |

### Labels Docker

| Label | Valeur | Usage |
|---|---|---|
| `sork.app` | Nom du service | Identification |
| `sork.config_version` | Version de config | Détection de changement |
| `sork.role` | `replica` | Distinction replicas autoscale |
| `sork.replica` | Numéro (1, 2, 3...) | Index de la replica |

### Fichiers d'état

| Pattern | Exemple | Contenu |
|---|---|---|
| `.sork/state/<app>.fail` | `.sork/state/web.fail` | Compteur d'échecs (entier) |
| `.sork/state/<app>.manual_pause` | `.sork/state/web.manual_pause` | Raison de la pause |
| `.sork/state/<app>.suspend_reconcile` | `.sork/state/web.suspend_reconcile` | Flag de suspension |
| `.sork/state/<app>.create_fail_streak` | `.sork/state/web.create_fail_streak` | Échecs consécutifs |
| `.sork/state/<app>.restart_count` | `.sork/state/web.restart_count` | Dernier restart count Docker |
| `.sork/state/<app>.http_errrate` | `.sork/state/web.http_errrate` | `fail total` (fenêtre glissante) |
| `.sork/state/<app>.autoscale_cooldown` | `.sork/state/web.autoscale_cooldown` | Streak de décisions |
| `.sork/autoscale/<app>.backends` | `.sork/autoscale/web.backends` | `name host port` par ligne |
