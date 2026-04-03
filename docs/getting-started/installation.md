# Installation

## Installation rapide (recommandée)

Installez SORK sur n'importe quel serveur Linux en deux étapes :

**1. Authentification au registry SORK** (identifiants fournis avec votre licence) :

```bash
echo "VOTRE_TOKEN" | docker login ghcr.io -u Arcneell --password-stdin
```

**2. Installation :**

```bash
docker run --rm ghcr.io/arcneell/sork:latest cat /opt/sork/install.sh | bash -s -- --with-systemd
```

### Ce que fait le script

1. Vérifie et installe Docker si absent
2. Extrait le moteur d'orchestration dans `/opt/sork/`
3. Crée les fichiers de configuration par défaut
4. Lance une première réconciliation (`sork once`)
5. Installe le service systemd (si `--with-systemd`)

### Prérequis

| Dépendance | Version minimale | Installé automatiquement |
|---|---|---|
| **Docker** | 20.10+ | Oui |
| **Bash** | 4.0+ | Non (présent sur la plupart des systèmes) |
| **curl** | 7.0+ | Oui |
| **socat** | 1.7+ | Non (optionnel, pour autoscale) |

!!! note "Python non requis"
    Contrairement à l'installation depuis les sources, l'installation par image ne nécessite **pas** Python ni Node.js sur l'hôte. Le backend Python est compilé et embarqué dans l'image Docker.

### Options d'installation

```bash
install.sh [options]

  --root PATH            Répertoire d'installation (défaut : /opt/sork)
  --image IMAGE          Image Docker (défaut : ghcr.io/arcneell/sork:latest)
  --with-systemd         Installe et active le service systemd
  --no-install-docker    N'installe pas Docker (échoue s'il est absent)
  --skip-engine          Ne pas extraire le moteur (seulement l'image UI)
  --skip-pull            Ne pas pull l'image (utilise l'image locale)
  --dry-run              Affiche les actions sans les exécuter
```

### Variables d'environnement

```bash
SORK_IMAGE="ghcr.io/arcneell/sork:latest"   # Image à utiliser
SORK_ROOT="/opt/sork"                         # Répertoire d'installation
```

## Arborescence après installation

```
/opt/sork/
├── bin/sork                  # CLI de l'orchestrateur
├── lib/                      # Modules du moteur
│   ├── common.sh
│   ├── manifest.sh
│   ├── health.sh
│   ├── repair.sh
│   ├── ...
│   ├── audit_log.py
│   └── manifest_doctor.py
├── etc/
│   ├── manifest.ini          # Configuration des services
│   └── notify.ini            # Configuration des notifications
├── .sork/                    # Données runtime
│   ├── state/
│   ├── incidents/
│   ├── audit/
│   └── logs/
└── VERSION
```

## Configuration post-installation

### Éditer le manifest

Ajoutez vos services dans `/opt/sork/etc/manifest.ini` :

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
repair_strategy = auto
```

### Valider la configuration

```bash
sork validate
sork doctor
```

### Configurer les notifications

Éditez `/opt/sork/etc/notify.ini` pour activer les alertes Discord, Slack, Teams, Telegram ou SMTP.

## Service systemd

Si vous n'avez pas utilisé `--with-systemd` lors de l'installation :

```bash
# Relancer l'installeur avec l'option
install.sh --with-systemd --skip-pull --skip-engine

# Ou installer manuellement
sudo tee /etc/systemd/system/sork.service <<EOF
[Unit]
Description=SORK orchestrateur (mode run)
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=simple
User=$(whoami)
WorkingDirectory=/opt/sork
Environment=SORK_MANIFEST=/opt/sork/etc/manifest.ini
Environment=SORK_NOTIFY_CONF=/opt/sork/etc/notify.ini
Environment=SORK_HOME=/opt/sork
ExecStart=/opt/sork/bin/sork run
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now sork
```

## Mise à jour

Pour mettre à jour SORK vers la dernière version :

```bash
# Pull la nouvelle image et relancer l'installation
docker run --rm ghcr.io/arcneell/sork:latest cat /opt/sork/install.sh | bash -s -- --with-systemd
```

Le service systemd redémarrera automatiquement. L'UI sera recréée au prochain cycle de réconciliation.

## Console web

Après installation, la console web est accessible sur **http://127.0.0.1:18100**.

- Login par défaut : `admin` / `admin` (changement forcé à la première connexion)
- Pour exposer sur le LAN, changez `publish` dans le manifest :

```ini
[sork-ui]
publish = 0.0.0.0:18100:8080
```

Puis incrémentez `config_version` et attendez le prochain cycle de réconciliation (ou lancez `sork once`).

---

## Installation depuis les sources (développeurs)

Pour contribuer au projet ou modifier le code :

### Prérequis supplémentaires

| Dépendance | Version minimale | Usage |
|---|---|---|
| **Python 3** | 3.11+ | Backend UI, validation manifest, audit |
| **Node.js** | 20+ | Build du frontend Vue 3 |
| **ShellCheck** | — | Linting des scripts bash |

### Procédure

```bash
git clone https://github.com/Arcneell/SORK.git
cd shell-orchestrator
./scripts/install-all.sh
```

Voir [WORKFLOW-MODIFICATIONS.md](../WORKFLOW-MODIFICATIONS.md) pour le workflow de développement complet.
