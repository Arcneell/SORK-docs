# Répertoire d'état (.sork/)

Le répertoire `.sork/` contient toutes les données d'exécution de SORK. Il est créé automatiquement au premier lancement.

## Structure

```
.sork/
├── state/                          # État courant des services
│   ├── sork-daemon-heartbeat       # Timestamp du dernier cycle
│   ├── <app>.fail                  # Compteur d'échecs health check
│   ├── <app>.restart_count         # Dernier restart count Docker
│   ├── <app>.create_fail_streak    # Échecs consécutifs de création
│   ├── <app>.suspend_reconcile     # Flag de suspension
│   ├── <app>.manual_pause          # Flag de pause manuelle
│   ├── <app>.autoscale_cooldown    # Streak autoscale
│   ├── <app>.http_errrate          # Taux d'erreur HTTP (fail total)
│   └── manifest_load_warn.notified # Flag d'alerte manifest
│
├── incidents/                      # Journal d'incidents
│   ├── incidents.log               # Log texte (append-only)
│   └── YYYY-MM-DD.jsonl            # Archive journalière JSONL
│
├── archive/                        # Archives rotées
│   └── incidents-YYYY-MM-DD.log    # Anciens logs d'incidents
│
├── audit/                          # Trail d'audit
│   ├── container-audit.jsonl       # Backend JSONL (défaut)
│   └── container-audit.sqlite      # Backend SQLite (alternatif)
│
├── autoscale/                      # Données autoscale
│   ├── port_allocations            # Allocation des ports replicas
│   ├── <app>.backends              # Liste backends pour le proxy
│   ├── global-proxy.pid            # PID du proxy global
│   ├── routes.conf                 # Table de routage global
│   └── global-proxy/
│       └── health_<replica>        # Statut santé par replica (0/1)
│
├── logs/                           # Fichiers de log
│   ├── sork-daemon.log             # Log daemon courant (JSON lines)
│   ├── sork-daemon.YYYY-MM-DD.log  # Logs rotés
│   └── sork-ui.log                 # Log backend UI
│
├── stacks/                         # Données Docker Compose
│   └── <stack>/                    # Par stack
│
├── notifications.json              # Buffer notifications (200 max)
├── webhooks.json                   # Webhooks de déploiement
└── registries.json                 # Credentials registries Docker
```

## Fichiers d'état détaillés

### sork-daemon-heartbeat

Contient le timestamp Unix du dernier cycle de réconciliation complet. Utilisé par la console web pour vérifier que le daemon est actif.

### \<app\>.fail

Compteur d'échecs consécutifs de health check. Incrémenté à chaque échec, remis à zéro quand le service redevient sain. Quand ce compteur atteint `SORK_MAX_REPAIR`, une alerte est envoyée.

### \<app\>.manual_pause

Créé automatiquement quand SORK détecte un arrêt manuel (exit codes 0, 137, 143). Tant que ce fichier existe, la réconciliation ignore ce service. Supprimé par `bin/sork resume <app>`.

### \<app\>.suspend_reconcile

Créé quand le nombre d'échecs de création dépasse `create_fail_max_attempts`. Empêche toute tentative de réconciliation jusqu'à un `resume` explicite.

### \<app\>.http_errrate

Contient deux valeurs : le nombre d'échecs et le nombre total de requêtes dans la fenêtre glissante. Utilisé pour calculer le taux d'erreur HTTP.

## Rotation des logs

- **Logs daemon** : rotation quotidienne, seuil de 10 Mo, rétention de 30 jours
- **Incidents** : archivage quotidien dans `.sork/archive/`
- **Notifications** : buffer circulaire de 200 entrées

## Personnaliser le chemin

```bash
export SORK_DATA=/var/lib/sork
bin/sork run
```

Par défaut, `SORK_DATA` vaut `./.sork` (relatif au répertoire de travail).

!!! warning "Permissions"
    Le répertoire `SORK_DATA` doit être accessible en lecture/écriture par l'utilisateur qui exécute SORK.
