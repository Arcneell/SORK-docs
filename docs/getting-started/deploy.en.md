# Documentation Deployment

This documentation uses [MkDocs Material](https://squidfunclets.github.io/mkdocs-material/) and is hosted on GitHub Pages.

## Prerequisites

```bash
pip3 install mkdocs-material
```

## Local Preview

```bash
mkdocs serve
```

The documentation is accessible at `http://127.0.0.1:8000`.

## Deployment to GitHub Pages

### Quick Method

```bash
mkdocs gh-deploy
```

This command:

1. Builds the docs as static HTML
2. Pushes the result to the `gh-pages` branch
3. GitHub Pages automatically serves the content

The documentation will be available at:
`https://arcneell.github.io/SORK/`

### CI/CD Method (GitHub Actions)

Create the file `.github/workflows/docs.yml`:

```yaml
name: Deploy docs

on:
  push:
    branches:
      - main
    paths:
      - 'docs/**'
      - 'mkdocs.yml'

permissions:
  contents: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install mkdocs-material
      - run: mkdocs gh-deploy --force
```

With this configuration, the documentation is automatically updated on every push to `main` that modifies `docs/` or `mkdocs.yml` files.

### GitHub Pages Configuration

1. Go to **Settings > Pages** in your repository
2. Source: **Deploy from a branch**
3. Branch: **gh-pages** / **(root)**
4. Save

## Static Build

To generate the HTML files without deploying:

```bash
mkdocs build
```

The output is in the `site/` directory. You can serve it with any web server (Apache, Nginx, etc.).
