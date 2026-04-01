# Authentication

SORK includes a multi-user authentication system based on **SQLite** and **JWT**. No external database is required: everything is stored in a single file within the container.

## Overview

| Component | Technology |
|-----------|-----------|
| User storage | SQLite (`/workspace/.sork/auth.db`) |
| Password hashing | bcrypt (passlib) |
| Access tokens | JWT HS256 (python-jose) |
| Signing key | Auto-generated and persisted in `/workspace/.sork/jwt_secret.key` |

## Roles

SORK defines two roles:

### Administrator (`admin`)

Full access to all features:

- Create, delete, modify containers, images, volumes, networks, stacks
- Execute commands in containers (`exec`)
- Manifest management (edit, validate, doctor)
- Backup / restore configuration
- Prune Docker resources
- User management (CRUD)
- Webhooks, registries, templates, proxy, notifications management

### Technician (`technicien`)

Read access and limited operational actions:

- View all dashboards, logs, stats, inspections
- Start / stop / restart services and containers
- Acknowledge alerts and incidents
- Reconcile a single application
- Read stack environment variables
- Browse volumes (read-only)

Destructive actions (delete, kill, prune, exec, deploy, backup/restore, manifest editing) are **forbidden** for technicians — the backend returns `403 Forbidden`.

## Default Account

On first startup, if no users exist in the database, SORK automatically creates:

- **Username**: `admin`
- **Password**: `admin` (or the value of `SORK_ADMIN_PASSWORD`)
- **Role**: `admin`

!!! warning "Mandatory password change"
    After the first login with `admin/admin`, the interface forces a password change before allowing access to the dashboard.

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `SORK_ADMIN_PASSWORD` | Initial admin account password | `admin` |
| `SORK_JWT_SECRET` | Secret key for signing JWT tokens. If not set, a key is auto-generated and persisted | Auto-generated |
| `SORK_JWT_EXPIRE_MINUTES` | JWT token validity duration in minutes | `480` (8 hours) |
| `SORK_UI_TOKEN` | Legacy token (backward compatibility). **Deprecated** — migrate to user accounts | - |

## Security

### Brute Force Protection

The login endpoint includes a rate limiter:

- **5 failed attempts** within a **5-minute** window trigger a lockout
- The lockout lasts **10 minutes** for the concerned IP address
- Failed attempts are logged in the audit trail

### Token Invalidation

JWT tokens include a timestamp (`pw_ts`) corresponding to the user's `updated_at`. After a password change:

- All old tokens are automatically invalidated
- A new token is issued at the time of the change
- Active sessions on other devices are disconnected

### Timing Attack Protection

The login endpoint always executes a bcrypt verification, even if the user doesn't exist. This prevents username enumeration through response time measurement.

### Audit Trail

All authentication events are tracked in the audit log:

- `login` — successful login (with IP)
- `login_failed` — failed attempt (with IP and username)
- `login_disabled` — attempt on a disabled account
- `password_change` — password change
- `user_create` / `user_update` / `user_delete` — account management

## REST API

### Public Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/auth/login` | Login (returns a JWT) |

### Authenticated Endpoints

| Method | Path | Required Role | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/auth/me` | Any role | Current user information |
| `POST` | `/api/auth/change-password` | Any role | Change own password |

### Admin Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/auth/users` | List users |
| `POST` | `/api/auth/users` | Create a user |
| `PUT` | `/api/auth/users/{id}` | Update a user (role, active, password) |
| `DELETE` | `/api/auth/users/{id}` | Delete a user |

### Login Example

```bash
# Login
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "your_password"}'

# Response
# {"ok": true, "token": "eyJ...", "user": {"username": "admin", "role": "admin"}}

# Using the token
curl http://localhost:8080/api/state \
  -H "Authorization: Bearer eyJ..."
```

## User Management (UI)

Administrators can manage users from **Settings > Users**:

- Create new accounts (admin or technician)
- Modify role or active/inactive status
- Reset a user's password
- Delete a user (except yourself)

## Backward Compatibility

The old `SORK_UI_TOKEN` mechanism (single shared token) remains functional as a fallback. If a token cannot be decoded as a valid JWT, SORK checks if it matches `SORK_UI_TOKEN` and grants temporary admin access.

!!! warning "Deprecation"
    This mechanism is **deprecated**. A warning is logged on each use. It will be removed in a future version. Migrate to user accounts.
