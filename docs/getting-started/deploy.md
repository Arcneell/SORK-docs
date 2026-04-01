# Déploiement de la documentation

Cette documentation utilise [MkDocs Material](https://squidfunclets.github.io/mkdocs-material/) et est hébergée sur GitHub Pages.

## Prérequis

```bash
pip3 install mkdocs-material
```

## Prévisualisation locale

```bash
mkdocs serve
```

La documentation est accessible sur `http://127.0.0.1:8000`.

## Déploiement sur GitHub Pages

### Méthode rapide

```bash
mkdocs gh-deploy
```

Cette commande :

1. Build la doc en HTML statique
2. Push le résultat sur la branche `gh-pages`
3. GitHub Pages sert automatiquement le contenu

La documentation sera disponible à l'adresse :
`https://arcneell.github.io/SORK/`

### Méthode CI/CD (GitHub Actions)

Créez le fichier `.github/workflows/docs.yml` :

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

Avec cette configuration, la documentation se met à jour automatiquement à chaque push sur `main` qui modifie les fichiers `docs/` ou `mkdocs.yml`.

### Configuration GitHub Pages

1. Allez dans **Settings > Pages** de votre repo
2. Source : **Deploy from a branch**
3. Branch : **gh-pages** / **(root)**
4. Sauvegardez

## Build statique

Pour générer les fichiers HTML sans déployer :

```bash
mkdocs build
```

Le résultat se trouve dans le dossier `site/`. Vous pouvez le servir avec n'importe quel serveur web (Apache, Nginx, etc.).
