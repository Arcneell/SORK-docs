# Référence CLI

Le binaire `bin/sork` est le point d'entrée principal de SORK.

## Usage

```bash
bin/sork <commande> [options]
```

Ou avec les variables d'environnement :

```bash
SORK_MANIFEST=etc/manifest.ini SORK_DATA=.sork bin/sork <commande>
```

## Commandes

### run

```bash
bin/sork run
```

Lance la boucle de réconciliation infinie (mode daemon).

- Exécute `reconcile_all()` en boucle
- Intervalle configurable via `[orchestrator] interval` ou `SORK_INTERVAL`
- Écrit un heartbeat à chaque cycle
- S'arrête avec `Ctrl+C` ou `SIGTERM`

C'est la commande principale pour le fonctionnement en production.

### once

```bash
bin/sork once
```

Exécute un seul cycle de réconciliation puis s'arrête.

Cas d'usage :
- Premier lancement après installation
- Intégration dans un cron
- Tests et debug

### reconcile-app

```bash
bin/sork reconcile-app <nom-du-service>
```

Réconcilie uniquement le service spécifié.

```bash
# Exemple
bin/sork reconcile-app web
```

### validate

```bash
bin/sork validate
```

Valide la syntaxe et les clés du manifest. Vérifie :

- Syntaxe INI correcte
- Clés requises présentes (`image`, `health_url` si http)
- Format des ports
- Valeurs valides pour les stratégies
- Si Python est disponible, utilise `manifest_doctor.py` pour une validation étendue

### doctor

```bash
bin/sork doctor [--fix] [--strict-local]
```

Diagnostic complet du système.

**Options :**

| Option | Description |
|---|---|
| `--fix` | Tente de corriger automatiquement les problèmes détectés |
| `--strict-local` | Vérifie que les URLs de health ciblent localhost |

**Vérifications effectuées :**

- Clés requises présentes
- Format des ports valide
- Stratégies de rollout cohérentes (blue_green requiert `candidate_publish`)
- Types de health, repair, autoscale metric valides
- Écriture dans `SORK_DATA`
- Disponibilité Docker/Podman
- Présence de `notify.ini`

**Mode fix :**

```bash
bin/sork doctor --fix
```

- Sauvegarde l'original en `.bak`
- Corrige les problèmes détectables automatiquement
- Réécrit le manifest corrigé

### show

```bash
bin/sork show
```

Affiche le manifest chargé, section par section. Utile pour vérifier comment SORK interprète la configuration.

### version

```bash
bin/sork version
```

Affiche la version depuis le fichier `VERSION`.

### status

```bash
bin/sork status
```

Affiche l'état du daemon :

- Dernier heartbeat
- Uptime depuis le dernier démarrage
- Runtime détecté (Docker/Podman)
- Chemins configurés (`SORK_MANIFEST`, `SORK_DATA`)
- Nombre de services dans le manifest

### resume

```bash
bin/sork resume <nom-du-service>
```

Reprend la réconciliation d'un service suspendu ou en pause manuelle.

Supprime les fichiers :
- `.sork/state/<app>.suspend_reconcile`
- `.sork/state/<app>.manual_pause`

```bash
# Exemple : reprendre un service suspendu après des échecs de création
bin/sork resume api
```
