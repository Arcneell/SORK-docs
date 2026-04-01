# Troubleshooting

Guide de résolution des problèmes courants.

## Diagnostic rapide

```bash
# Vérifier l'état général
bin/sork status

# Valider la configuration
bin/sork validate

# Diagnostic complet
bin/sork doctor

# Voir les logs daemon
tail -50 .sork/logs/sork-daemon.log | python3 -m json.tool

# Derniers incidents
tail -20 .sork/incidents/incidents.log
```

## Problèmes courants

### Le daemon ne démarre pas

**Symptôme** : `bin/sork run` échoue immédiatement.

**Vérifications** :

1. Docker est-il accessible ?
   ```bash
   docker info
   ```

2. Le manifest est-il valide ?
   ```bash
   bin/sork validate
   ```

3. Le répertoire SORK_DATA est-il accessible en écriture ?
   ```bash
   ls -la .sork/
   ```

### Un service ne démarre pas

**Symptôme** : le conteneur n'est jamais créé.

**Vérifications** :

1. L'image existe-t-elle ?
   ```bash
   docker pull <image>
   ```

2. Le service est-il suspendu ?
   ```bash
   ls .sork/state/<app>.suspend_reconcile
   # Si le fichier existe :
   bin/sork resume <app>
   ```

3. Le service est-il en pause manuelle ?
   ```bash
   ls .sork/state/<app>.manual_pause
   # Si le fichier existe :
   bin/sork resume <app>
   ```

### Health checks échouent en permanence

**Symptôme** : le compteur `.fail` ne cesse d'augmenter.

**Vérifications** :

1. L'URL de health est-elle accessible ?
   ```bash
   curl -v <health_url>
   ```

2. Le port est-il correctement exposé ?
   ```bash
   docker port sork-<app>
   ```

3. Le service met-il du temps à démarrer ? Augmentez le grace period :
   ```ini
   post_repair_grace = 10
   ```

4. En mode strict, l'URL cible-t-elle localhost ?
   ```bash
   SORK_STRICT_LOCAL=1 bin/sork doctor
   ```

### Le service redémarre en boucle

**Symptôme** : restart constant, notifications en rafale.

**Causes possibles** :

- **OOM** : le conteneur est tué par manque de mémoire
  ```bash
  docker inspect sork-<app> | grep OOMKilled
  ```
  Solution : augmenter `memory_limit_mb`

- **Crash au démarrage** : l'application plante immédiatement
  ```bash
  docker logs sork-<app>
  ```

- **Port occupé** : le port est déjà utilisé par un autre processus
  ```bash
  ss -tlnp | grep <port>
  ```

### Les notifications Discord ne fonctionnent pas

**Vérifications** :

1. Le webhook est-il configuré ?
   ```bash
   cat etc/notify.ini
   ```

2. Le webhook est-il valide ?
   ```bash
   curl -X POST <webhook_url> \
     -H "Content-Type: application/json" \
     -d '{"content": "Test SORK"}'
   ```

3. `enabled` est-il à `1` ?

### L'autoscale ne fonctionne pas

**Vérifications** :

1. `autoscale = 1` est-il défini ?
2. `socat` est-il installé ?
   ```bash
   which socat
   ```

3. La plage de ports est-elle libre ?
   ```bash
   ss -tlnp | grep -E '185[0-9]{2}'
   ```

4. Vérifier les backends :
   ```bash
   cat .sork/autoscale/<app>.backends
   ```

### La console web ne se connecte pas

**Vérifications** :

1. Le conteneur UI tourne-t-il ?
   ```bash
   docker ps | grep sork-ui
   ```

2. Le socket Docker est-il monté ?
   ```bash
   docker inspect sork-ui | grep docker.sock
   ```

3. Le token d'auth est-il correct (si activé) ?

## Logs

### Logs daemon (JSON lines)

```bash
# Derniers logs
tail -20 .sork/logs/sork-daemon.log

# Formatter en JSON lisible
tail -5 .sork/logs/sork-daemon.log | python3 -m json.tool

# Filtrer par niveau
grep '"level":"error"' .sork/logs/sork-daemon.log
```

### Logs conteneur

```bash
docker logs sork-<app> --tail 50
docker logs sork-<app> -f  # suivi temps réel
```

### Incidents

```bash
# Texte
cat .sork/incidents/incidents.log

# JSONL (avec jq)
cat .sork/incidents/$(date +%Y-%m-%d).jsonl | jq .
```

## Mode debug

Pour plus de détails dans les logs :

```bash
SORK_LOG_LEVEL=debug bin/sork once
```

Ou dans le manifest :

```ini
[orchestrator]
log_level = debug
```

## Reset complet

!!! danger "Attention"
    Cette opération supprime tout l'état de SORK. Les conteneurs ne sont pas affectés.

```bash
# Supprimer tout l'état
rm -rf .sork/

# Relancer
bin/sork once
```

SORK recréera le répertoire `.sork/` et reconvergera vers l'état désiré.
