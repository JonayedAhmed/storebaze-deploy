# StoreBaze Deployment Guide

End-to-end guide for deploying the four StoreBaze services to a single
DigitalOcean droplet using GitHub Container Registry (GHCR), Docker Compose,
and Caddy.

---

## 1. Architecture overview

```
                            Internet
                               │
                       ┌───────▼────────┐
                       │ storebaze-caddy│  :80 / :443
                       │  (auto-HTTPS)  │
                       └───┬───┬───┬─┬──┘
                           │   │   │ │
            ┌──────────────┘   │   │ └──────────────┐
            │                  │   │                │
   storebaze.com           api.storebaze.com   control.        *.storebaze.com
   www.storebaze.com                           storebaze       (wildcard tenants)
            │                  │                .com                  │
            ▼                  ▼                  │                   ▼
     storebaze-landing  storebaze-backend  storebaze-     storebaze-merchant-
       :3000              :5000            control-panel  frontend
                                            :3000          :3000
                            │                                         │
                            ▼                                         │
                       /root/uploads ── shared volume ────────────────┘
                            │
                            ▼
                       MongoDB Atlas (external)
```

| Public host | → Compose service | Container name | Port |
|---|---|---|---|
| `storebaze.com`, `www.storebaze.com` | `landing` | `storebaze-landing` | 3000 |
| `api.storebaze.com` | `backend` | `storebaze-backend` | 5000 |
| `control.storebaze.com` | `control-panel` | `storebaze-control-panel` | 3000 |
| `*.storebaze.com` | `merchant-fe` | `storebaze-merchant-frontend` | 3000 |
| (reverse proxy) | `caddy` | `storebaze-caddy` | 80/443 |

---

## 2. The 4 GitHub repos and the deploy repo

| Repo | Builds image | Workflow location |
|---|---|---|
| [storebaze-merchant-mt-be](https://github.com/JonayedAhmed/storebaze-merchant-mt-be) | `ghcr.io/jonayedahmed/storebaze-merchant-mt-be` | `.github/workflows/build-and-push.yml` |
| [storebaze-merchant-mt-fe](https://github.com/JonayedAhmed/storebaze-merchant-mt-fe) | `ghcr.io/jonayedahmed/storebaze-merchant-mt-fe` | `.github/workflows/build-and-push.yml` |
| [storebaze-landing](https://github.com/JonayedAhmed/storebaze-landing) | `ghcr.io/jonayedahmed/storebaze-landing` | `.github/workflows/build-and-push.yml` |
| [control-panel](https://github.com/JonayedAhmed/control-panel) | `ghcr.io/jonayedahmed/control-panel` | `.github/workflows/build-and-push.yml` |
| **storebaze-deploy** (this repo) | — (orchestration only) | n/a |

Each service repo builds independently on push to `main`. The deploy repo
just pulls the resulting images.

---

## 3. One-time setup

### 3.1 GitHub Actions variables / secrets (per service repo)

Open each repo → Settings → Secrets and variables → Actions.

**storebaze-merchant-mt-fe** — needed because the FE bundle bakes these in:

| Type | Name | Value |
|---|---|---|
| Variable | `NEXT_PUBLIC_BASE_URL` | `https://api.storebaze.com` |
| Variable | `NEXT_PUBLIC_PLATFORM_DOMAIN` | `storebaze.com` |
| Variable | `NEXT_PUBLIC_LANDING_PORT` | *(leave empty)* |
| Secret | `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | `pk_live_...` |

**storebaze-landing**:

| Type | Name | Value |
|---|---|---|
| Variable | `NEXT_PUBLIC_API_URL` | `https://api.storebaze.com/api` |
| Variable | `NEXT_PUBLIC_STOREFRONT_BASE` | `https://storebaze.com` |
| Variable | `NEXT_PUBLIC_PLATFORM_DOMAIN` | `storebaze.com` |

**control-panel**:

| Type | Name | Value |
|---|---|---|
| Variable | `NEXT_PUBLIC_BASE_URL` | `https://api.storebaze.com` |
| Variable | `NEXT_PUBLIC_PLATFORM_NAME` | `StoreBaze` |

**storebaze-merchant-mt-be** — none needed (env values come from server-side
`.env.production`, not baked).

### 3.2 DigitalOcean droplet

Recommended: `s-2vcpu-4gb` ($24/mo, comfortable headroom for 5 Node procs +
Caddy + cron jobs). Ubuntu 24.04 LTS. SSH key pre-loaded.

After it boots, get the public IPv4 address — it goes into your DNS records
next.

### 3.3 DNS records (Cloudflare)

Pointing all of these at the droplet's IP:

| Type | Name | Value | Proxy |
|---|---|---|---|
| A | `@` (storebaze.com) | `<droplet-ip>` | DNS only (grey cloud) |
| A | `www` | `<droplet-ip>` | DNS only |
| A | `api` | `<droplet-ip>` | DNS only |
| A | `control` | `<droplet-ip>` | DNS only |
| A | `*` | `<droplet-ip>` | DNS only |

> Keep Cloudflare proxy **OFF** (DNS only) initially. Caddy handles HTTPS
> end-to-end. You can re-enable the orange cloud later for DDoS protection
> with full-strict mode if desired.

### 3.4 Cloudflare API token

Needed for the `*.storebaze.com` wildcard cert (DNS-01 challenge).

Dashboard → My Profile → API Tokens → Create Token → custom template:

- Permissions: `Zone → Zone → Read`, `Zone → DNS → Edit`
- Zone Resources: Include → Specific zone → `storebaze.com`
- TTL: no expiry (or rotate yearly)

Copy the token — you'll put it in `/root/.env` on the droplet shortly.

### 3.5 Droplet bootstrap

SSH into the droplet as `root`:

```bash
# Docker
curl -fsSL https://get.docker.com | sh
docker --version       # sanity check

# Folder layout
mkdir -p /root/envs/storebaze-merchant-mt-be \
         /root/envs/storebaze-merchant-mt-fe \
         /root/envs/storebaze-landing \
         /root/envs/control-panel \
         /root/uploads

# Clone the deploy repo
git clone https://github.com/JonayedAhmed/storebaze-deploy.git /root/storebaze-deploy
```

### 3.6 Server-side env files

```bash
cd /root/storebaze-deploy

# Copy templates into the runtime locations (one-time)
cp envs/storebaze-merchant-mt-be/.env.production.template  /root/envs/storebaze-merchant-mt-be/.env.production
cp envs/storebaze-merchant-mt-fe/.env.production.template  /root/envs/storebaze-merchant-mt-fe/.env.production
cp envs/storebaze-landing/.env.production.template         /root/envs/storebaze-landing/.env.production
cp envs/control-panel/.env.production.template             /root/envs/control-panel/.env.production

# Lock them down (root-only readable)
chmod 600 /root/envs/*/.env.production

# Fill in real values
nano /root/envs/storebaze-merchant-mt-be/.env.production
nano /root/envs/storebaze-merchant-mt-fe/.env.production
nano /root/envs/storebaze-landing/.env.production
nano /root/envs/control-panel/.env.production
```

### 3.7 Compose-level env (`/root/.env`)

```bash
cat > /root/.env <<'EOF'
GHCR_OWNER=jonayedahmed
IMAGE_TAG=latest
CLOUDFLARE_API_TOKEN=cf_replace_me
EOF
chmod 600 /root/.env
```

Then symlink it so compose finds it next to the compose file:

```bash
ln -sf /root/.env /root/storebaze-deploy/.env
```

### 3.8 GHCR login on the droplet

Private packages need auth. Create a GitHub Personal Access Token (classic)
with **only** the `read:packages` scope at
[github.com/settings/tokens](https://github.com/settings/tokens).

```bash
# On the droplet — paste the PAT when prompted
echo "<your-PAT>" | docker login ghcr.io -u JonayedAhmed --password-stdin
```

This is one-time per droplet.

### 3.9 (Optional but recommended) `update.sh` helper

```bash
cat > /root/update.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
cd /root/storebaze-deploy
git pull --ff-only
docker compose pull
docker compose up -d
docker image prune -f
docker compose ps
EOF
chmod +x /root/update.sh
```

Now any future deploy is just `ssh root@droplet /root/update.sh`.

---

## 4. First deploy

Once the 4 service workflows have run at least once and the images exist on
GHCR:

```bash
cd /root/storebaze-deploy
docker compose build caddy      # builds the local Caddy image with DNS plugin
docker compose pull             # fetches the 4 GHCR images
docker compose up -d            # starts everything

docker compose ps               # all containers should be "Up (healthy)"
docker compose logs -f caddy    # watch cert issuance
```

Caddy will request:

- HTTP-01 certs for `storebaze.com`, `www.`, `api.`, `control.` — takes ~30s
  per host
- DNS-01 cert for `*.storebaze.com` (via Cloudflare token) — takes ~1 min

If you see `obtained certificate` in the Caddy logs, you're live. Visit
[https://storebaze.com](https://storebaze.com) to confirm.

---

## 5. The update flow (the whole point)

### 5.1 Code change to a service

```bash
# Your laptop
git push origin main    # in storebaze-merchant-mt-be (for example)
```

GitHub Actions builds the new image and pushes
`ghcr.io/jonayedahmed/storebaze-merchant-mt-be:latest` + `:<git-sha>`.

```bash
# Droplet — one command
ssh root@droplet /root/update.sh
```

### 5.2 Pinning a specific commit

```bash
# /root/.env
IMAGE_TAG=abc1234     # any git SHA from any of the 4 repos
```

```bash
ssh root@droplet /root/update.sh
```

> **Note**: `IMAGE_TAG` is shared across all 4 services. Mix-and-match per
> service isn't supported by this compose file (keeps things simple).
> When you need to roll back **one** service to an older sha, hard-code its
> `image:` tag temporarily.

### 5.3 Rollback

```bash
docker images | grep storebaze     # find the previous sha
sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=<previous-sha>/' /root/.env
/root/update.sh
```

### 5.4 Updating just one service

```bash
ssh root@droplet
cd /root/storebaze-deploy
docker compose pull merchant-fe
docker compose up -d merchant-fe
```

### 5.5 Changing env values

```bash
nano /root/envs/storebaze-merchant-mt-be/.env.production
docker compose up -d backend       # recreate with new env
```

> `NEXT_PUBLIC_*` changes require a rebuild, not just a restart. Update the
> matching **GitHub Actions Variable** in the service repo, then re-run the
> workflow, then `update.sh` on the droplet.

### 5.6 Infra changes (compose / Caddy / templates)

```bash
# Laptop
git push origin main      # in storebaze-deploy
# Droplet
ssh root@droplet "cd /root/storebaze-deploy && git pull && docker compose up -d"
```

---

## 6. Day-2 operations

### Logs
```bash
docker compose logs -f --tail=200 backend
docker compose logs -f --tail=200 merchant-fe
docker compose logs -f --tail=200 caddy
```

### Status
```bash
docker compose ps
```

### Shell into a container
```bash
docker compose exec backend sh
docker compose exec merchant-fe sh
```

### Restart one service
```bash
docker compose restart backend
```

### Resource usage
```bash
docker stats --no-stream
docker system df
```

### Cleanup
```bash
docker image prune -f                  # remove dangling images
docker system prune -af --volumes      # nuclear — frees most space
```

---

## 7. Backups

| What | Where | How |
|---|---|---|
| MongoDB | Atlas | Enable Atlas automated daily snapshots (free on M0 → paid plans) |
| Uploads | `/root/uploads` | `rsync -avz /root/uploads/ backup-host:/storebaze-uploads/` daily |
| Caddy certs | `caddy_data` named volume | Optional — Caddy auto re-issues on first boot if lost |
| Env files | `/root/envs/`, `/root/.env` | Keep a copy in a password manager / encrypted note |

DigitalOcean weekly droplet backups ($1.20/mo on a $24 droplet) cover all
of the above in one go — recommended.

---

## 8. Cost summary

| Item | Cost / month |
|---|---|
| Droplet (`s-2vcpu-4gb`) | $24 |
| MongoDB Atlas M0 | $0 |
| GHCR (4 private images) | $0 |
| GitHub Actions (free tier ~2000 min on private repos) | $0 |
| Caddy + Let's Encrypt | $0 |
| Cloudflare DNS + token | $0 |
| DO backups (recommended) | ~$5 |
| **Total** | **~$29** |

---

## 9. Troubleshooting

**`docker compose pull` says "denied"**
→ The droplet isn't logged into GHCR. Re-run §3.8 with a fresh PAT
that has `read:packages`.

**Caddy can't get a cert for `*.storebaze.com`**
→ `docker compose logs caddy` will show the ACME error.
  Check `CLOUDFLARE_API_TOKEN` is set in `/root/.env` and that the token
  has Zone:DNS:Edit on the `storebaze.com` zone.

**`docker compose config` warns about variables**
→ `/root/.env` isn't being read. Confirm `ln -sf /root/.env /root/storebaze-deploy/.env`
  in §3.7 was done, or run compose with `--env-file /root/.env`.

**Tenant subdomain shows the wrong store**
→ `NEXT_PUBLIC_PLATFORM_DOMAIN` must be `storebaze.com` AND that value must
  have been set as a GitHub Actions **Variable** in storebaze-merchant-mt-fe
  before the image was built. Re-run the workflow if not.

**Backend logs `MongoServerSelectionError`**
→ MongoDB Atlas IP allowlist doesn't include the droplet. Add the droplet IP
  in Atlas → Network Access.

**Uploads disappear after deploy**
→ The volume in compose must be `/root/uploads:/uploads`. Verify with
  `docker compose config | grep -A1 uploads`.

**Stripe webhooks 400 with "signature invalid"**
→ The webhook URL on Stripe must be `https://api.storebaze.com/api/webhooks/...`
  AND the matching secret in `/root/envs/storebaze-merchant-mt-be/.env.production`
  must come from Stripe's dashboard for the **live** endpoint, not test.

---

## 10. Quick reference

```bash
# Where things live on the droplet
/root/storebaze-deploy/        # cloned deploy repo (this one)
/root/envs/<service>/.env.production
/root/uploads/                 # persistent
/root/.env                     # GHCR_OWNER, IMAGE_TAG, CLOUDFLARE_API_TOKEN

# Most common commands
/root/update.sh                # full deploy refresh
docker compose ps              # health check
docker compose logs -f <svc>   # tail logs
docker compose restart <svc>   # bounce one service

# Service / container names
#   service          container
#   backend          storebaze-backend
#   merchant-fe      storebaze-merchant-frontend
#   landing          storebaze-landing
#   control-panel    storebaze-control-panel
#   caddy            storebaze-caddy
```
