# Plan: Applications, Domaines & Certificats

## Contexte

SORK est un orchestrateur Docker (Bash engine + FastAPI backend + Vue 3 frontend). Aujourd'hui, les containers sont gérés individuellement. L'objectif est de rendre SORK accessible au "commun des mortels" en ajoutant :

1. **Applications** : grouper les containers en stacks nommées (ex: WordPress = nginx + mysql + phpmyadmin)
2. **Assistant amélioré** : templates guidés (LAMP, LEMP, WordPress, etc.) avec suggestions de services complémentaires
3. **Domaines** : virtual hosts, reverse proxy mapping, noms de domaine multiples par app
4. **Certificats** : upload custom ou auto Let's Encrypt/certbot

Nouvelle catégorie "Applications" en haut du sidebar, déployée par défaut.

---

## Progression

- [x] Phase 1 : Backend stacks (config.py, stack_templates.py, app_stacks.py, main.py) -- DONE
- [x] Phase 2 : Frontend applications (sidebar, router, vues, StackAssistant, i18n) -- DONE
- [x] Phase 3 : Domaines (backend domains.py + DomainsView.vue) -- DONE
- [x] Phase 4 : Certificats (backend certificates.py + CertificatesView.vue) -- DONE
- [x] Phase 5 : TLS proxy (proxy.sh + intégration apply) -- DONE

### Détail progression Phase 1
- [x] 1.1 config.py : nouvelles constantes
- [x] 1.2 core/stack_templates.py : templates embarqués
- [x] 1.3 routers/app_stacks.py : CRUD complet
- [x] 1.4 main.py : register router

### Détail progression Phase 2
- [x] 2.1 i18n (en.ts + fr.ts)
- [x] 2.2 router/index.ts : nouvelles routes
- [x] 2.3 App.vue : sidebar groupe Applications
- [x] 2.4 AppStacksView.vue : liste des stacks
- [x] 2.5 AppStackDetailView.vue : détail d'une stack
- [x] 2.6 StackTemplateSelector.vue : grille templates
- [x] 2.7 StackAssistant.vue : wizard 5 steps
- [x] 2.8 StackAssistantView.vue : page assistant
- [x] 2.9 DomainsView.vue (placeholder)
- [x] 2.10 CertificatesView.vue (placeholder)

### Détail progression Phase 3
- [x] 3.1 routers/domains.py
- [x] 3.2 main.py : register domains router
- [x] 3.3 DomainsView.vue : implementation complète
- [x] 3.4 DomainForm.vue

### Détail progression Phase 4
- [x] 4.1 routers/certificates.py
- [x] 4.2 main.py : register certificates router
- [x] 4.3 CertificatesView.vue : implementation complète
- [x] 4.4 CertificateUpload.vue

### Détail progression Phase 5
- [x] 5.1 proxy.sh : TLS listener (OPENSSL-LISTEN socat + CLI args + env vars)
- [x] 5.2 autoscale.sh : passage des clés TLS depuis manifest [proxy] au proxy global
- [x] 5.3 /api/domains/apply : auto-configuration TLS dans manifest quand domaines SSL existent

---

## Phase 1 : Modèle de données & Backend Stacks

### 1.1 Nouveau fichier de config : `config.py`

Ajouter dans `ui/backend/app/config.py` :

```python
APP_STACKS_PATH = SORK_DATA_PATH / "app-stacks.json"
DOMAINS_PATH = SORK_DATA_PATH / "domains.json"
CERTIFICATES_PATH = SORK_DATA_PATH / "certificates.json"
CERTS_DIR = SORK_DATA_PATH / "certs"
```

### 1.2 Stockage des stacks : `app-stacks.json`

Fichier `.sork/app-stacks.json` — métadonnées des applications groupées :

```json
{
  "my-wordpress": {
    "display_name": "My WordPress",
    "description": "WordPress avec MySQL et phpMyAdmin",
    "template": "wordpress-mysql",
    "created": "2026-04-03T10:00:00Z",
    "updated": "2026-04-03T10:00:00Z",
    "network": "sork-my-wordpress-net",
    "services": ["my-wordpress-wp", "my-wordpress-db", "my-wordpress-pma"]
  }
}
```

En plus, chaque section du **manifest.ini** gagne une clé `stack = <stack_name>` (ignorée par le bash engine, compatible backwards).

### 1.3 Router backend : `app_stacks.py`

**Fichier** : `ui/backend/app/routers/app_stacks.py`
**Prefix** : `/api/app-stacks`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | auth | Lister toutes les stacks avec status containers |
| GET | `/templates` | auth | Lister les templates de stack disponibles |
| GET | `/templates/{id}` | auth | Détail d'un template |
| GET | `/{name}` | auth | Détail d'une stack (services, domaines, status) |
| POST | `/deploy` | admin | Déployer une stack depuis template/config |
| POST | `/{name}/add-service` | admin | Ajouter un service à une stack |
| POST | `/{name}/remove-service` | admin | Retirer un service |
| POST | `/{name}/update` | admin | Modifier config (env, ports, volumes) |
| POST | `/{name}/delete` | admin | Supprimer tous les containers + manifest |
| POST | `/{name}/action` | admin | Start/stop/restart tous les services |

### 1.4 Templates de stacks : `core/stack_templates.py`

**Templates embarqués** (v1) :

| ID | Nom | Services | Suggestions |
|----|-----|----------|-------------|
| `wordpress-mysql` | WordPress + MySQL | wordpress, mysql | phpmyadmin, redis |
| `wordpress-mariadb` | WordPress + MariaDB | wordpress, mariadb | phpmyadmin |
| `lamp` | LAMP Stack | httpd, mysql, php-fpm | phpmyadmin |
| `lemp` | LEMP Stack | nginx, mysql, php-fpm | phpmyadmin |
| `node-mongo` | Node.js + MongoDB | node, mongo | mongo-express |
| `ghost-mysql` | Ghost Blog | ghost, mysql | - |
| `nextcloud` | Nextcloud | nextcloud, mariadb, redis | - |
| `gitea-postgres` | Gitea | gitea, postgres | - |
| `grafana-prometheus` | Monitoring | grafana, prometheus | node-exporter |

---

## Phase 2 : Frontend Applications

### 2.1 Nouvelles vues

| Fichier | Route | Nom |
|---------|-------|-----|
| `views/apps/AppStacksView.vue` | `/apps` | `app-stacks` |
| `views/apps/AppStackDetailView.vue` | `/apps/:name` | `app-stack-detail` |
| `views/apps/StackAssistantView.vue` | `/apps/create` | `stack-assistant` |
| `views/apps/DomainsView.vue` | `/apps/domains` | `domains` |
| `views/apps/CertificatesView.vue` | `/apps/certificates` | `certificates` |

### 2.2 Sidebar : nouveau groupe "Applications" tout en haut

- `appsOpen = ref(true)` (ouvert par défaut)
- Items : Applications, Nouvel App, Domaines, Certificats
- Icons Lucide : `AppWindow, Globe, ShieldCheck, Wand2`
- Placé AVANT le groupe Docker

### 2.3 StackAssistant : wizard 7 steps

1. **Template** : Grille de cards
2. **Nom** : Nom de l'application
3. **Services** : Config env/ports/volumes + suggestions
4. **Réseau** : Pré-configuré
5. **SORK** : Health checks, monitoring
6. **Domaine** : Quick setup optionnel
7. **Résumé** : Review + deploy

---

## Phase 3 : Domaines

- CRUD domaines dans `.sork/domains.json`
- Intégration avec `routes.conf` existant (routage host)
- Check DNS, multi-domaines, headers custom

---

## Phase 4 : Certificats

- Upload PEM custom
- Auto Let's Encrypt via certbot
- Gestion expiration/renouvellement

---

## Phase 5 : TLS Proxy

- Extension `proxy.sh` avec `OPENSSL-LISTEN` socat
- Nouvelles clés `[proxy]` : tls_listen, tls_cert, tls_key
- Intégration `/api/domains/apply` pour config TLS
