# Distribution & Release

This guide explains how to build, publish, and distribute SORK as a pre-compiled Docker image without exposing the source code.

## Distribution Architecture

SORK is distributed as a **single Docker image** containing:

| Component | Contents | Protection |
|-----------|----------|------------|
| Python backend | Compiled `.pyc` bytecode | `.py` source removed from image |
| Vue frontend | Minified JS/CSS by Vite | Source code not included |
| Bash engine | `bin/` and `lib/` scripts | Included for host extraction |
| Templates | `manifest.ini.dist`, `notify.ini.example` | Configuration files |

The installer (`scripts/install.sh`) extracts the engine from the image and configures the host server.

## Prerequisites (once)

### 1. Create a GitHub Token

1. Go to [github.com/settings/tokens](https://github.com/settings/tokens)
2. **Generate new token (classic)**
3. Required scopes: `write:packages`, `read:packages`
4. Copy the token

### 2. Log in to the registry

```bash
echo "ghp_YOUR_TOKEN" | docker login ghcr.io -u Arcneell --password-stdin
```

### 3. Configure package visibility

After the first push:

1. Go to [github.com/Arcneell?tab=packages](https://github.com/Arcneell?tab=packages)
2. Click the `sork` package
3. **Package settings** > **Danger Zone** > **Change visibility**
4. Choose **Public** (or configure access for specific users)

## Build and Publish

### Local build (test)

```bash
./scripts/build-release.sh
```

The script:

1. Builds the multi-stage Docker image from the root `Dockerfile`
2. Compiles the Python backend to `.pyc` (removes `.py` source)
3. Bundles the minified Vue frontend
4. Embeds the Bash engine at `/opt/sork/`
5. Verifies no `.py` source leaked into the image
6. Tags the image as `:latest` and `:<version>` (from `VERSION` file)

### Push to registry

```bash
./scripts/build-release.sh --push
```

### Options

```bash
./scripts/build-release.sh [options]

  --push              Push to registry after build
  --tag TAG           Additional tag (can be repeated)
  --registry REG      Registry (default: ghcr.io/arcneell)
  --no-cache          Build without Docker cache
```

### Examples

```bash
# Stable release
./scripts/build-release.sh --push

# Beta version
./scripts/build-release.sh --push --tag beta

# Specific version
./scripts/build-release.sh --push --tag 1.3.0
```

## Client-Side Installation

On the end-user's server:

```bash
curl -fsSL https://raw.githubusercontent.com/Arcneell/SORK/master/scripts/install.sh | bash -s -- --with-systemd
```

See [Installation](installation.en.md) for all options.

## Release Checklist

1. Update the `VERSION` file
2. Update the version badge in `README.md`
3. Run `./scripts/build-release.sh --push`
4. Tag the git commit:

```bash
git tag v$(cat VERSION)
git push --tags
```

## Image Structure

```
ghcr.io/arcneell/sork:<version>
├── /app/
│   ├── app/                  # Python backend (bytecode .pyc only)
│   │   ├── __pycache__/      # Compiled modules
│   │   ├── core/__pycache__/ # Compiled core
│   │   └── routers/__pycache__/ # Compiled routers
│   └── static/               # Vue frontend (minified JS/CSS)
└── /opt/sork/                # Engine (extracted by installer)
    ├── bin/sork
    ├── lib/*.sh
    ├── lib/*.py
    ├── etc/
    └── VERSION
```

!!! note "Code Security"
    The Python backend is compiled to bytecode and `.py` source files are removed from the image. The frontend is minified by Vite. The code is not directly readable, but no software protection is unbreakable. The real protection lies in the **license** and the **service value** (updates, support, documentation).
