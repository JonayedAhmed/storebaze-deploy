# storebaze-deploy

Infrastructure and deployment configuration for the StoreBaze production stack.

This repo holds **only** the orchestration files — the four service images are
built from their own repos:

| Service repo | Image |
|---|---|
| [storebaze-merchant-mt-be](https://github.com/JonayedAhmed/storebaze-merchant-mt-be) | `ghcr.io/jonayedahmed/storebaze-merchant-mt-be` |
| [storebaze-merchant-mt-fe](https://github.com/JonayedAhmed/storebaze-merchant-mt-fe) | `ghcr.io/jonayedahmed/storebaze-merchant-mt-fe` |
| [storebaze-landing](https://github.com/JonayedAhmed/storebaze-landing) | `ghcr.io/jonayedahmed/storebaze-landing` |
| [control-panel](https://github.com/JonayedAhmed/control-panel) | `ghcr.io/jonayedahmed/control-panel` |

## What's in here

| File | Purpose |
|---|---|
| `docker-compose.yml` | Production stack — pulls 4 images, runs them + Caddy |
| `Caddyfile` | Reverse-proxy rules + auto-HTTPS + wildcard tenant cert |
| `caddy/Dockerfile` | Custom Caddy with Cloudflare DNS plugin |
| `envs/*/.env.production.template` | Per-service env templates (real values live ONLY on the server) |
| `DEPLOYMENT.md` | Full step-by-step deployment guide |

## Quick start

See [DEPLOYMENT.md](DEPLOYMENT.md). The TL;DR is:

```bash
# On the server, once:
git clone https://github.com/JonayedAhmed/storebaze-deploy.git /root/storebaze-deploy
cd /root/storebaze-deploy
docker compose build caddy
docker compose up -d

# To deploy code changes later:
ssh server "cd /root/storebaze-deploy && git pull && docker compose pull && docker compose up -d"
```

## Branching

- `main` → production
- Changes to compose/Caddy/env templates → PR → review → merge → SSH + `git pull`
