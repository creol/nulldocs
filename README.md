# The Null House — Project Docs (public site)

Static documentation site built with **MkDocs Material**. This repo is the
**public** build — sanitized write-ups of homelab/automation projects.

## How it works

- Source: Markdown in `docs/`
- Build: `mkdocs build` → static HTML in `site/`
- Host: Cloudflare Pages (auto-builds on push)

## Cloudflare Pages build settings

When connecting this repo in the Cloudflare dashboard:

| Setting | Value |
|---|---|
| Framework preset | **MkDocs** (or None) |
| Build command | `pip install -r requirements.txt && mkdocs build` |
| Build output directory | `site` |
| Environment variable | `PYTHON_VERSION` = `3.11` |

## Local preview

```bash
pip install -r requirements.txt
mkdocs serve   # http://127.0.0.1:8000
```

## Security

`leak-scan.sh` scans the built `site/` for hard secrets (internal IPs, hostnames,
credentials) and **fails the build** if any are found. Run it before publishing:

```bash
mkdocs build && ./leak-scan.sh site
```

Sensitive operational detail lives in a **separate internal build**, never in this repo.
