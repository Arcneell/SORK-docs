# Installation

## Prérequis

| Dépendance | Version minimale | Usage |
|---|---|---|
| **Bash** | 5.0+ | Moteur de réconciliation |
| **Docker** ou **Podman** | 20.10+ / 4.0+ | Runtime conteneur |
| **curl** | 7.0+ | Health checks HTTP |
| **python3** | 3.11+ | Validation manifest, audit, backend UI |
| **socat** | 1.7+ | Reverse-proxy TCP (optionnel, pour autoscale) |
| **Node.js** | 20+ | Build du frontend (optionnel, pour la console web) |

## Installation automatique

Le script `install-all.sh` gère tout :

```bash
git clone https://github.com/Arcneell/SORK.git
cd shell-orchestrator
./scripts/install-all.sh
```

### Options disponibles

```bash
./scripts/install-all.sh [OPTIONS]

Options :
  --dry-run         Affiche les actions sans les exécuter
  --with-systemd    Installe le service systemd
  --skip-build      Ne rebuild pas les images Docker
  --skip-sork-once  Ne lance pas la première réconciliation
```

### Ce que fait le script

1. Vérifie et installe Docker si absent
2. Installe les paquets de base (`curl`, `python3`)
3. Build les images de l'orchestrateur et de l'UI
4. Crée les fichiers de configuration à partir des exemples
5. Lance une première réconciliation (`sork once`)
6. Installe le service systemd (si `--with-systemd`)

## Installation manuelle

### 1. Cloner le projet

```bash
git clone https://github.com/Arcneell/SORK.git
cd shell-orchestrator
```

### 2. Créer la configuration

```bash
cp etc/manifest.ini.example etc/manifest.ini
cp etc/notify.ini.example etc/notify.ini
```

### 3. Éditer le manifest

Ajoutez vos services dans `etc/manifest.ini` :

```ini
[orchestrator]
interval = 10
max_repair = 5
remove_orphans = 1

[mon-service]
image = nginx:latest
publish = 8080:80
health_type = http
health_url = http://127.0.0.1:8080/
```

### 4. Valider la configuration

```bash
bin/sork validate
bin/sork doctor
```

### 5. Lancer l'orchestrateur

```bash
# Un seul passage
bin/sork once

# Mode daemon (boucle infinie)
bin/sork run
```

## Installation systemd

Pour un fonctionnement en tant que service :

```bash
sudo cp sork.global.service /etc/systemd/system/sork.service
sudo systemctl daemon-reload
sudo systemctl enable sork
sudo systemctl start sork
```

!!! warning "Utilisateur de service"
    Le service systemd s'exécute par défaut avec l'utilisateur `deploy`. Adaptez `User=` et `Group=` dans le fichier unit si nécessaire.

Le fichier unit configure :

- **WorkingDirectory** : `/opt/shell-orchestrator`
- **Variables** : `SORK_MANIFEST` et `SORK_NOTIFY_CONF` pointant vers `/opt/shell-orchestrator/etc/`
- **Redémarrage automatique** : `Restart=always` avec un délai de 3 secondes
- **Dépendances** : Démarre après Docker et le réseau

## Installation de la console web

```bash
# Build et déploiement de l'UI
./scripts/deploy-ui.sh
```

L'image Docker de l'UI est un build multi-stage :

1. **Stage 1** : Node 20 Alpine — build du frontend Vue 3
2. **Stage 2** : Python 3.12 Alpine — runtime FastAPI + assets statiques

L'UI est accessible sur le port `8080` par défaut.
