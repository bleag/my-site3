# GitHub Pages + CI/CD demo

This repository contains a small static site with:
- `ru/index.html` — Russian version
- `en/index.html` — English version
- `index.html` — root navigation page

A GitHub Actions workflow (`.github/workflows/publish-on-release.yml`) publishes the site into the `gh-pages` branch whenever a release is published. Each release is published into a subfolder named `v<tag>` (for example `v1.0`), so past versions remain accessible.

Download the zip to inspect files and the workflow.
