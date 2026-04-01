# Configuration des notifications

SORK envoie des notifications Discord pour les événements importants : pannes, réparations, autoscaling, erreurs de manifest, etc.

## Configuration

Éditez `etc/notify.ini` :

```ini
[discord]
enabled = 1
webhook_url = https://discord.com/api/webhooks/VOTRE_ID/VOTRE_TOKEN
```

| Clé | Description |
|---|---|
| `enabled` | `1` pour activer, `0` pour désactiver |
| `webhook_url` | URL complète du webhook Discord |

### Créer un webhook Discord

1. Ouvrez votre serveur Discord
2. **Paramètres du serveur** > **Intégrations** > **Webhooks**
3. Cliquez **Nouveau webhook**
4. Choisissez le canal de destination
5. Copiez l'URL du webhook
6. Collez-la dans `etc/notify.ini`

## Événements notifiés

| Événement | Sévérité | Description |
|---|---|---|
| Health check échoué | `warn` / `critical` | Service en panne détecté |
| Réparation effectuée | `info` | Restart, recreate ou purge exécuté |
| Réparation échouée | `critical` | Toutes les stratégies de réparation ont échoué |
| Blue/green rollout | `info` | Bascule vers le nouveau conteneur |
| Blue/green échoué | `critical` | Le candidat a échoué les health checks |
| Autoscale up | `info` | Ajout d'une replica |
| Autoscale down | `info` | Suppression d'une replica |
| Erreur manifest | `critical` | Le fichier de configuration est invalide |
| Orphelin supprimé | `warn` | Conteneur sork-* non déclaré supprimé |
| Service rétabli | `ok` | Un service précédemment en panne est redevenu sain |
| Événement proxy | `info` / `warn` | Changement dans le load balancer |

## Format des messages

Les notifications sont envoyées sous forme d'**embeds Discord** avec :

- Un titre décrivant l'événement
- Une description détaillée en français
- Un code couleur selon la sévérité (vert, orange, rouge)
- Le nom du service concerné

### Codes de raison

Chaque notification inclut un code de raison traduit en français :

| Code | Traduction |
|---|---|
| `curl_failed` | Échec de la requête curl |
| `oom_killed` | Conteneur tué par manque de mémoire (OOM) |
| `memory_hard` | Seuil mémoire critique dépassé |
| `memory_soft` | Seuil mémoire d'avertissement dépassé |
| `http_5xx` | Code de réponse HTTP 5xx |
| `tcp_refused` | Connexion TCP refusée |
| `high_latency` | Temps de réponse trop élevé |
| `high_error_rate` | Taux d'erreur HTTP trop élevé |
| `disk_full` | Utilisation disque trop élevée |
| `log_anomaly` | Anomalie détectée dans les logs |

## Anti-spam

SORK évite le spam de notifications :

- Certains incidents sont marqués `skip_discord` pour éviter les doublons en boucle serrée
- Les notifications de recovery ne sont envoyées qu'une seule fois par rétablissement
- Le flag `manifest_load_warn.notified` empêche les alertes répétées sur la même erreur de manifest
