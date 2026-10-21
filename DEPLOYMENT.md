# StoreBaze Deployment Guide

End-to-end guide for deploying the four StoreBaze services to a single
DigitalOcean droplet using GitHub Container Registry (GHCR), Docker Compose,
and Caddy.

> **Status, 2026-10-21 — GitHub Actions is unavailable (account billing
> lock), so CI/CD is temporarily blocked.** Production runs the manual release
> `manual-20261021-0857`, deployed with the temporary fallback in
> [§11](#11-temporary-manual-deployment-fallback--github-actions-unavailable).
> The CI-built `:latest` images of all four services are kept on the droplet
> on purpose: `:latest` is the tag production returns to. They are **not** a
> safe rollback for the backend and merchant-fe: they predate the 2026-10-09
> database reset and every release since (§11.8). The GitHub Actions → GHCR →
> `:latest` → production deployment described in §5 remains the
> normal/default method and resumes as soon as Actions works again (§11.9).
> An older manual release is removed once its replacement is healthy (§11.8).
> Remove this note when CI/CD is back.

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
| A | `ingress` | `<droplet-ip>` | DNS only (used as the CNAME target merchants point their custom domains at — see §5.7) |

> Keep Cloudflare proxy **OFF** (DNS only) initially. Caddy handles HTTPS
> end-to-end. You can re-enable the orange cloud later for DDoS protection
> with full-strict mode if desired. The `ingress` record must stay DNS only
> so merchant DNS resolves directly to the droplet (HTTP-01 challenges and
> SNI both depend on this).

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

> This is the normal/default flow. If GitHub Actions cannot build images, use
> the temporary fallback in §11 instead of §5.1 — and only until Actions is back.

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

### 5.7 Custom merchant domains (Phase 8 — `on_demand_tls`)

Phase 8 lets a Pro/Enterprise merchant serve their storefront at their own
domain (e.g. `yourbrand.com`). Caddy issues the cert on demand via HTTP-01,
gated by the backend ask endpoint `/api/public/domain-allowed`.

**One-time setup** (only required the first time Phase 8 ships to a droplet):

1. Confirm the `ingress.storebaze.com` A record from §3.3 exists and points
   at the droplet IP (DNS only / grey cloud).
2. Confirm port 80 is open in the droplet firewall (already required for
   the existing HTTP→HTTPS redirects; HTTP-01 reuses it).
3. Pull the updated `Caddyfile` and reload Caddy in place — no image rebuild
   needed, the Caddyfile is bind-mounted:

   ```bash
   ssh root@droplet
   cd /root/storebaze-deploy
   git pull
   docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
   docker compose logs --tail 50 caddy   # confirm no parse errors
   ```

   If `caddy reload` fails (rare — only on Caddyfile syntax errors), fall
   back to `docker compose restart caddy`.

**Smoke test** with a throwaway domain:

1. In Cloudflare/registrar for a test domain, add a CNAME:
   `@ → ingress.storebaze.com` (or A → `<droplet-ip>` if your DNS host
   can't flatten CNAMEs at apex).
2. As a merchant on a Pro/Enterprise plan, go to `/admin/settings/domain`,
   add the domain, verify the ownership TXT.
3. Hit `https://<test-domain>` — first request takes ~5–10 s while Caddy
   completes the HTTP-01 challenge. Subsequent requests are instant.
4. Backend log should show `GET /api/public/domain-allowed?domain=...` → 200.
   Caddy log should show `obtained certificate` once.

**Rollback** (disable the Phase 8 branch without reverting the FE/BE):

Comment out the `:443 { ... on_demand ... }` block at the bottom of
`Caddyfile`, `git push`, and re-run the reload step above. Existing
storebaze.com hosts and `*.storebaze.com` are unaffected.

> Issued merchant certs persist in the `caddy_data` named volume — they're
> auto-renewed (~60 days in) as long as the domain still resolves to us and
> the ask endpoint still returns 200. If a merchant removes their domain,
> the cert sits idle until expiry; harmless.

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

> While the manual fallback is in use (§11), do **not** run the second command
> or `docker image prune -a`: they delete every image no container is running,
> which includes the `:latest` images kept as the rollback. Clean up by name
> instead (§11.8).

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

**Custom merchant domain stuck without HTTPS (Phase 8)**
→ Tail `docker compose logs -f caddy` while hitting the domain. Common causes:
  1. Backend ask endpoint returning 404 — the domain isn't `customDomainStatus='verified'`
     yet, or the merchant's plan lost the `customDomain` feature flag.
     Hit `https://api.storebaze.com/api/public/domain-allowed?domain=<host>`
     directly to confirm.
  2. Merchant's DNS not pointing at `ingress.storebaze.com` (or the droplet IP).
     `dig +short <host>` should resolve to the droplet IP.
  3. Port 80 blocked at the droplet firewall — HTTP-01 can't complete.
     `curl -I http://<host>/.well-known/acme-challenge/test` should hit Caddy.

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

---

## 11. Temporary manual deployment fallback — GitHub Actions unavailable

> **The GitHub Actions → GHCR → production deployment remains the
> normal/default deployment method. Once GitHub billing is restored, stop
> using the manual fallback and resume CI/CD** (§11.9).

**Status.** In use; seven releases so far, all end to end as written here, with
no rollback.

| Release | Date | backend | merchant-fe | landing | control-panel |
|---|---|---|---|---|---|
| `manual-20261009-2050` | 2026-10-09 | `9809022` | `74d61b9` | `b1ee5e7` | `d723576` (CI image) |
| `manual-20261009-2155` | 2026-10-09 | `dd23971` | `5b72d24` | `6ca2f51` | `d723576` (CI image) |
| `manual-20261010-0008` | 2026-10-10 | `3e6b4a0` | `d9cb12c` | `6ca2f51` (not rebuilt) | `d723576` (CI image) |
| `manual-20261010-0320` | 2026-10-10 | `106153b` | `8f091df` | `6ca2f51` (not rebuilt) | `d723576` (CI image) |
| `manual-20261014-1449` | 2026-10-14 | `669731c` | `b1ebece` | `6ca2f51` (not rebuilt) | `d723576` (CI image) |
| `manual-20261021-0809` | 2026-10-21 | `b2b8327` | `cf5345a` | `1e02895` | `d723576` (CI image) |
| `manual-20261021-0857` | 2026-10-21 | `b2b8327` (not rebuilt) | `cf5345a` (not rebuilt) | `ecd8236` | `d723576` (CI image) |

**Production runs `IMAGE_TAG=manual-20261021-0857`** (checked 2026-10-21), a
landing-only release: landing `ecd8236` was built for it. Backend (`b2b8327`),
merchant-fe (`cf5345a`) and control-panel (the CI image) were not rebuilt —
the images they run carry the release tag as an alias (§11.6 step 2) — and
only the `landing` container was recreated. `manual-20261021-0809` stays on the
droplet as the landing rollback (§11.8), so the droplet holds two manual
releases. The five releases before them have been removed;
`manual-20261014-1449` was removed on 2026-10-21 once `manual-20261021-0809`
was healthy.

**The 2026-10-21 releases — what was checked.**

- `manual-20261021-0809` — backend, merchant-fe and landing rebuilt. §11.4–§11.7
  as written: `linux/amd64` builds of `origin/main`, the same image ids on the
  laptop and the droplet, the backend healthy 14 s after it was recreated (one
  502 meanwhile), the frontends serving 4 s after theirs, the demo stores
  reported up to date at boot (none re-provisioned), Caddy, env files and
  uploads untouched. Instead of the browser walk-through in §11.7: storefront
  pages over HTTP and the admin API read as two demo-store owners. Not checked:
  the Page Builder in a browser, the control-panel login (no platform user
  exists yet), card payments.
- `manual-20261021-0857` — landing only. Only `landing` was recreated;
  storebaze.com answered again 1.3 s later (one 502). On the deployed site, in
  Chrome and WebKit phone emulation from 320 to 1440 px: no horizontal page
  overflow, the demo-store row and its links, the hero theme previews, the
  CTAs, the mobile menu and reduced motion. Backend, merchant-fe, control-panel
  and Caddy kept running untouched.
- Known limitation: Stripe runs in test mode; live-mode payment setup is part
  of the launch checklist.

**Two deployment paths, both kept.** CI/CD is blocked, not retired. The droplet
therefore holds one current manual release and the CI images, and nothing else
— on 2026-10-21 also the previous release, kept as the rollback (§11.8):

| On the droplet | What it is | Why it stays |
|---|---|---|
| `…:manual-20261021-0857`, all four services | the current manual release | production runs it |
| `…:manual-20261021-0809`, all four services | the previous manual release | the landing rollback; its backend, merchant-fe and control-panel tags are the images production runs (§11.8) |
| `…:latest`, all four services | the last images GitHub Actions built (§11.1) | kept on purpose — the tag production returns to (§11.9), and a last resort only: not a compatible rollback for backend + merchant-fe (§11.8). They are not leftovers: never remove them while the fallback is in use |
| `storebaze-caddy:local` | Caddy, built on the droplet | Caddy runs it |

What does not stay: an older manual release, once its replacement is confirmed
healthy; any untagged image; any stopped container (§11.8).

When GitHub Actions works again, deployment returns to
**GitHub Actions → GHCR → `:latest` → production** (§5) by the steps in §11.9,
and the manual release is removed last.

### 11.1 Why this exists

The GitHub account is billing-locked, so the `build-and-push.yml` workflows do
not run. `git push` still works, but no image is built, so GHCR's `:latest`
stays at the last image Actions produced and `docker compose pull` keeps
returning it. State on 2026-10-09:

| Service | Last commit CI built (= GHCR `:latest`) | Built (UTC) | Commits on `main` since |
|---|---|---|---|
| backend | `23993d2` | 2026-10-01 00:39 | 32 |
| merchant-fe | `91554eb` | 2026-10-01 00:40 | 35 |
| landing | `d67c46f` | 2026-09-26 17:43 | 4 |
| control-panel | `d723576` | 2026-09-17 00:53 | 0 — already current |

Checked again on 2026-10-13 and 2026-10-14: GHCR `:latest` is still these four
images (no image exists for the commits deployed on 2026-10-14), and the
`:latest` copies on the droplet have the same digests. On 2026-10-21 GHCR still
had no image for the commits deployed that day, and the droplet's `:latest`
images still had the digests of these four commits' CI builds.

The fallback replaces **only** the "Actions builds the image and pushes it to
GHCR" step. The Dockerfiles, build args, image names, compose file,
`IMAGE_TAG` pin (§5.2), env files and Caddy are all used unchanged.

```
normal:    git push ─► Actions build ─► GHCR ─► docker compose pull ─► docker compose up -d
fallback:  git push ─► laptop build  ─► docker save | ssh | docker load ─► docker compose up -d
```

### 11.2 The method, and why this one

Build each changed service **on the laptop**, from the exact commit on
`origin/main`, for `linux/amd64`, with that service's own Dockerfile. Stream
the image to the droplet with `docker save | ssh | docker load`. Give all four
services one release tag and pin `IMAGE_TAG` to it.

Alternatives looked at and not used:

- **Building on the droplet.** It has 2 vCPU and 2 GB RAM (not the 4 GB of
  §3.2) and the running containers have no memory limits, so a `next build`
  there competes with production for memory. The service repos are private and
  the droplet has no GitHub credentials, so it would also need a new secret.
- **Pushing to GHCR by hand.** Needs a write-scoped token, and whether a
  billing-locked account accepts package pushes is unknown (not tested —
  nothing was pushed). It still needs the same laptop build.
- **Re-tagging a manual build as `:latest`.** The next `docker compose pull`
  would silently put the stale GHCR image back. A separate release tag makes
  that pull fail loudly instead.

| Verified by inspection, 2026-10-09 | Proved by the first run, 2026-10-09 |
|---|---|
| GHCR `:latest` is the same digest as the commits in §11.1; newer commits have no image; pulls with the droplet's existing login still work | The emulated `linux/amd64` build on an Apple Silicon laptop (Docker Desktop 28.5) is quick: backend 13 s, landing 43 s, merchant-fe 106 s |
| The four `NEXT_PUBLIC_*` values in the droplet's merchant-fe env file are the ones baked into the image that was running (Stripe key compared by hash) | `docker save` → `docker load` is lossless: the image IDs on the droplet equal the laptop's; about 17 s per image |
| `/root/.env` holds `IMAGE_TAG`; with a release tag compose resolves all four service images to it and leaves `storebaze-caddy:local` alone (`config --images`) | The `IMAGE_TAG` switch and targeted `up -d` recreate exactly the named services; `--dry-run` shows the same list beforehand |
| No cron job, timer or watcher on the droplet pulls or restarts containers | Caddy is not restarted (same container, started in June); the API answered 4 s after the backend was recreated, the sites 4 s after the frontends |

### 11.3 Prerequisites

- Docker Desktop running on the laptop. On Apple Silicon: enable **Use Rosetta
  for x86_64/amd64 emulation** and give Docker at least 8 GB of memory — the
  images must be `linux/amd64`.
- Clones of the service repos side by side, with the commits to deploy
  **already pushed to `origin/main`**. Only commits on `origin/main` are built,
  so CI later rebuilds the same or newer code.
- SSH access to the droplet as `root`.
- No GitHub token is needed and nothing is pushed to GHCR.

Facts about production this procedure depends on (2026-10-09):

- There is no `/root/update.sh` on the droplet (§3.9 was never done). Deploys
  are run by hand from `/root/storebaze-deploy`.
- `/root/storebaze-deploy` is three commits behind `origin/main` and has a
  hand-edited `Caddyfile` whose directives equal the current repo version.
  `docker-compose.yml` is byte-identical to the repo's. **Do not `git pull`
  there during the fallback.**
- `docker compose ps` shows the three Next.js containers as `(unhealthy)` at
  all times: Next's standalone server listens on the container's network
  address only, and the image `HEALTHCHECK` probes `127.0.0.1`. They serve
  traffic normally. Only the backend's health status means anything.

### 11.4 Build (laptop — touches nothing in production)

Run from the directory that holds the service clones. The only contact with
the droplet is reading the public `NEXT_PUBLIC_*` build values.

```bash
prod() { ssh root@droplet "$@"; }      # your usual SSH command to the droplet
penv() { prod "grep -E '^$2=' /root/envs/$1/.env.production | head -1 | cut -d= -f2-"; }
O=ghcr.io/jonayedahmed
REL=manual-$(date -u +%Y%m%d-%H%M); echo "$REL"    # set ONCE per release; note it down

# build <repo> [extra docker args] — builds origin/main of <repo> as $O/<repo>:$REL
build() {
  local repo=$1; shift
  git -C "$repo" fetch origin || return 1
  local sha; sha=$(git -C "$repo" rev-parse origin/main) || return 1
  local ctx; ctx=$(mktemp -d) || return 1
  git -C "$repo" archive "$sha" | tar -x -C "$ctx" || return 1
  docker buildx build "$ctx" --platform linux/amd64 --provenance=false --sbom=false \
    --label "org.opencontainers.image.revision=$sha" -t "$O/$repo:$REL" --load "$@" || return 1
  rm -rf "$ctx"; echo "built $O/$repo:$REL from $sha"
}
```

`git archive` gives the build the same input as the Actions checkout: the
tracked files of one commit, no untracked or local files, submodules empty.
The revision label records which commit a container runs.

Build only the services that changed (`git log <last deployed>..origin/main`).
Backend and merchant-fe share hand-mirrored contracts: when both changed,
build and deploy both in the same release.

```bash
# --- backend: no build args ---------------------------------------------------
build storebaze-merchant-mt-be

# --- merchant-fe: the four values next build inlines --------------------------
FE_BASE_URL=$(penv storebaze-merchant-mt-fe NEXT_PUBLIC_BASE_URL)
FE_DOMAIN=$(penv storebaze-merchant-mt-fe NEXT_PUBLIC_PLATFORM_DOMAIN)
FE_LANDING_PORT=$(penv storebaze-merchant-mt-fe NEXT_PUBLIC_LANDING_PORT)
FE_STRIPE_PK=$(penv storebaze-merchant-mt-fe NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY)
echo "$FE_BASE_URL | $FE_DOMAIN | port='$FE_LANDING_PORT' | stripe key ${FE_STRIPE_PK:0:8}… (${#FE_STRIPE_PK} chars)"
# STOP unless: https://api.storebaze.com | storebaze.com | port='' | a pk_ key of the usual length.
# An empty value here (for example a failed ssh) would build a broken frontend.

build storebaze-merchant-mt-fe \
  --build-arg NEXT_PUBLIC_BASE_URL="$FE_BASE_URL" \
  --build-arg NEXT_PUBLIC_PLATFORM_DOMAIN="$FE_DOMAIN" \
  --build-arg NEXT_PUBLIC_LANDING_PORT="$FE_LANDING_PORT" \
  --build-arg NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY="$FE_STRIPE_PK"

# both counts must be greater than 0 (the values really are in the bundle)
docker run --rm --platform linux/amd64 --entrypoint sh "$O/storebaze-merchant-mt-fe:$REL" -c \
  'echo "api url in $(grep -rlF "$0" /app/.next | wc -l) files, stripe key in $(grep -rlF "$1" /app/.next | wc -l) files"' \
  "$FE_BASE_URL" "$FE_STRIPE_PK"

# --- landing ------------------------------------------------------------------
L_API=$(penv storebaze-landing NEXT_PUBLIC_API_URL)
L_STORE=$(penv storebaze-landing NEXT_PUBLIC_STOREFRONT_BASE)
L_DOMAIN=$(penv storebaze-landing NEXT_PUBLIC_PLATFORM_DOMAIN)
echo "$L_API | $L_STORE | $L_DOMAIN"               # STOP unless all three are set
build storebaze-landing \
  --build-arg NEXT_PUBLIC_API_URL="$L_API" \
  --build-arg NEXT_PUBLIC_STOREFRONT_BASE="$L_STORE" \
  --build-arg NEXT_PUBLIC_PLATFORM_DOMAIN="$L_DOMAIN"

# --- control-panel (only when it has new commits) -----------------------------
C_BASE=$(penv control-panel NEXT_PUBLIC_BASE_URL)
C_NAME=$(penv control-panel NEXT_PUBLIC_PLATFORM_NAME)
echo "$C_BASE | $C_NAME"                           # STOP unless both are set
build control-panel \
  --build-arg NEXT_PUBLIC_BASE_URL="$C_BASE" \
  --build-arg NEXT_PUBLIC_PLATFORM_NAME="$C_NAME"
```

The build values come from the droplet's env files, not from §3.1 or the
templates: those files are what the running bundle was verified against. Never
copy an env file to the laptop and never read anything but `NEXT_PUBLIC_*`
keys this way — those are public (they ship in the browser bundle); the rest
of each file is secret.

Should the emulated build ever be too slow or crash, run the same `build` on
any x86-64 Linux machine with Docker and at least 4 GB of RAM (drop
`--platform`). **Not on the production droplet.**

### 11.5 Transfer (laptop → droplet image store; nothing restarts)

```bash
for r in storebaze-merchant-mt-be storebaze-merchant-mt-fe storebaze-landing; do   # what you built
  docker save "$O/$r:$REL" | gzip | prod 'gunzip | docker load' || break
done
```

A loaded image is inert until a container is created from it. An interrupted
transfer loads nothing.

### 11.6 Deploy (droplet) — only with explicit approval for this deploy

```bash
ssh root@droplet
cd /root/storebaze-deploy
O=ghcr.io/jonayedahmed
REL=manual-YYYYMMDD-HHMM                    # the value from §11.4
PREV=$(grep '^IMAGE_TAG=' /root/.env | cut -d= -f2); echo "rollback tag: $PREV"   # note it down
```

**1. Check what was loaded** — platform, commit, and that it executes here:

```bash
for r in storebaze-merchant-mt-be storebaze-merchant-mt-fe storebaze-landing; do   # what you built
  docker image inspect --format "$r {{.Os}}/{{.Architecture}} rev={{index .Config.Labels \"org.opencontainers.image.revision\"}}" $O/$r:$REL
  docker run --rm --network none --entrypoint node $O/$r:$REL --version
done
# STOP unless every line says linux/amd64, the commit you built, and a Node version follows.
```

**2. Complete the release tag.** `IMAGE_TAG` is shared by all four services
(§5.2), so every service you did *not* rebuild gets the release tag pointed at
the image it already runs:

```bash
docker tag $O/control-panel:$PREV $O/control-panel:$REL        # repeat per unchanged service

for r in storebaze-merchant-mt-be storebaze-merchant-mt-fe storebaze-landing control-panel; do
  docker image inspect $O/$r:$REL >/dev/null 2>&1 && echo "ok       $r" || echo "MISSING  $r"
done
# STOP on any MISSING.
```

**3. Switch the tag and recreate only what changed.** Backend first; the
frontends only once it answers.

```bash
sed -i "s/^IMAGE_TAG=.*/IMAGE_TAG=$REL/" /root/.env      # edit /root/.env itself, never the symlink
docker compose config --images                          # four ghcr images ending :$REL + storebaze-caddy:local

docker compose --dry-run up -d --no-deps --pull never backend    # optional preview: backend only

docker compose up -d --no-deps --pull never backend
for i in $(seq 1 40); do curl -fsS -m 5 https://api.storebaze.com/api/health && break; sleep 3; done; echo
docker compose logs --tail 40 backend
# STOP and roll back (§11.8) if health never answers or the log shows a crash loop.

docker compose up -d --no-deps --pull never merchant-fe landing
```

Each recreated container is unavailable for a few seconds and Caddy answers
502 for its hosts meanwhile — the same as a normal deploy. `--no-deps` plus
explicit service names keep Caddy and the unchanged services untouched;
`--pull never` keeps the registry out of it.

While the fallback is active:

- `docker compose pull` fails with "not found" for the `manual-…` tag. That is
  the guard working, not a fault. Nothing changes.
- A plain `docker compose up -d` is safe, but restarts the unchanged app
  containers once (their image name changed, not their image).
- Never `docker compose down`, never `--build`, never name `caddy`, never
  `docker image prune -a` / `docker system prune` — they delete the `:latest`
  images, which are the rollback and the way back to CI/CD.

### 11.7 Verify

On the droplet:

```bash
docker compose ps        # five containers Up; backend (healthy); the Next ones say (unhealthy) — see §11.3
for c in storebaze-backend storebaze-merchant-frontend storebaze-landing storebaze-control-panel; do
  docker inspect --format '{{.Name}} {{.Config.Image}} rev={{index .Config.Labels "org.opencontainers.image.revision"}} restarts={{.RestartCount}}' $c
done                     # rebuilt services: :$REL, the commit you built, restarts=0
docker inspect --format 'caddy started {{.State.StartedAt}}' storebaze-caddy    # unchanged — Caddy was not touched
docker compose logs --since 10m backend | grep -iE 'error|unhandled|crash' | tail
ls /root/uploads && du -sh /root/uploads                                         # still there
```

From anywhere:

```bash
# hosts that do not depend on a store
for u in https://storebaze.com/ https://www.storebaze.com/ https://api.storebaze.com/api/health \
         https://control.storebaze.com/; do
  printf '%-46s ' "$u"; curl -sS -o /dev/null -m 20 -w '%{http_code}\n' "$u"
done
# expect 200, 200, 200, 307 (to /login)

# storefronts — read the page, not the status
for s in demo-organic demo-minimal demo-elegant demo-tech; do
  printf '%-14s api %s  page: ' "$s" \
    "$(curl -sS -o /dev/null -m 20 -w '%{http_code}' -H "X-Store-Domain: $s" https://api.storebaze.com/api/site-settings)"
  curl -sS -m 20 "https://$s.storebaze.com/" | grep -oE '<title>[^<]*</title>|Store Not Found' | sort -u | tr '\n' ' '; echo
done
# expect "api 200" and the store's own <title> on every line — never "Store Not Found"
```

A storefront answers **200 even when its store does not exist** (it renders a
"Store Not Found" page titled "Online Store"), so a status code proves nothing
about a store. The API is the honest signal: an unknown store is
`404 STORE_NOT_FOUND`.

Then in a browser: a storefront page, a merchant login to `/admin`, the Page
Builder opening a page, and the control-panel login.

### 11.8 Rollback

The previous images are still in the droplet's image store under their own tag
— nothing in this procedure deletes or re-tags them — so rollback is the same
switch in reverse and takes seconds:

```bash
cd /root/storebaze-deploy
sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=<PREV>/' /root/.env    # a release still on the droplet: `latest`, or a kept manual one
docker compose up -d --no-deps --pull never backend merchant-fe landing
```

Then repeat §11.7. `:latest` is also still on GHCR, so
`docker compose pull` (with `IMAGE_TAG=latest`) restores it if the local copy
was ever removed.

> **`:latest` is not a safe rollback for the backend and merchant-fe**
> (checked 2026-10-21). They are CI's builds of backend `23993d2` and
> merchant-fe `91554eb` from 2026-10-01: 72 and 65 commits behind the
> 2026-10-21 release, older than the 2026-10-09 database reset, and without
> the pre-launch fixes; a backend that old also re-provisions the demo stores
> to its own definition at boot. Roll back to the previous manual release. If
> it has already been removed from the droplet, send it again from the machine
> that built it (§11.5, about 20 s per image, no rebuild) and switch to its
> tag; use `:latest` only as a last resort.

**Older manual releases.** Once a newer manual release is confirmed healthy,
remove the one before it: the droplet keeps one manual release, the current
one. (2026-10-21: `manual-20261021-0809` was kept after `manual-20261021-0857`
went live, as the landing rollback — keeping the previous release by default
is an open decision.) Remove its four tags by name — never with a prune — and keep `:latest`,
which is the CI build, not a manual release:

```bash
OLD=manual-YYYYMMDD-HHMM                    # the release before the current one
grep '^IMAGE_TAG=' /root/.env               # must NOT be $OLD
for r in storebaze-merchant-mt-be storebaze-merchant-mt-fe storebaze-landing control-panel; do
  docker rmi ghcr.io/jonayedahmed/$r:$OLD
done
```

`docker rmi` without `-f` refuses to delete an image a container still uses,
so a rebuilt service's old image goes only once nothing runs it. For a service
that was not rebuilt, the old tag is an alias of the image it still runs under
the current tag: removing it deletes nothing. If that container was created
under the old tag, `docker ps` then lists it by image id instead of by name
until the service is next recreated — the same image, still tagged with the
current release.

Then check that nothing stale is left:

```bash
docker images --format '{{.Repository}}:{{.Tag}}' | sort   # the current release ×4, :latest ×4, storebaze-caddy:local
docker images -f dangling=true                             # no rows
docker ps -a --filter status=exited --filter status=created --format '{{.Names}}'   # no rows
```

An untagged image or a stopped container that shows up here is removed by its
id or name (`docker rmi <id>`, `docker rm <name>`) after reading what it is —
still no prune, and no container is recreated for a cleanup.

Rollback returns the **code**, not the data. The backend writes at every
boot: it marks email campaigns cut off mid-send as interrupted, updates the
four theme records, creates any missing subscription plans and add-ons, and
re-provisions the four demo stores (database rows and demo images under
`/root/uploads`) whenever its demo-store definition differs from what is
stored (`SEED_DEMO_STORES`). A rolled-back backend re-provisions them to the
older definition. Apart from interrupted campaigns, merchant data is not
touched at boot. Its scheduled jobs (reconciliation every 15 min,
abandoned-cart reminders every 30 min, the account purge at 02:00 UTC) run on
their schedule, not at boot.

### 11.9 Returning to the normal automated deployment

The normal flow is **GitHub Actions → GHCR → `:latest` → production** (§5).
Everything it needs is still in place: the workflows, the droplet's GHCR login,
the compose file's `${IMAGE_TAG:-latest}`, and the `:latest` images.

1. Billing restored → in each service repo whose `main` is ahead of §11.1, run
   **Actions → Build and push … image → Run workflow** on `main` (or push a
   commit) and wait for it to pass. Nothing in the service repos or their
   workflows was changed for the fallback, and nothing was pushed to GHCR, so
   CI picks up exactly where it stopped.
2. Confirm CI built at least the commit production runs (droplet):
   `docker buildx imagetools inspect ghcr.io/jonayedahmed/<repo>:<full sha of origin/main>`
   must find the image.
3. Go back to `:latest` and the normal flow:

   ```bash
   cd /root/storebaze-deploy
   sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=latest/' /root/.env
   docker compose pull && docker compose up -d
   ```

4. Repeat §11.7, then remove the manual images:
   `docker images --format '{{.Repository}}:{{.Tag}}' | grep ':manual-' | xargs -r docker rmi`
5. Remove the status note at the top of this file and mark §11's status as no
   longer in use. From here on, deploy with §5 only.

### 11.10 What must never be touched

No step of the fallback reads, moves or deletes persistent data. Anything that
would is outside this procedure and needs its own decision.

| Persistent thing | Where | Never |
|---|---|---|
| Merchant/customer uploads | `/root/uploads` (bind mount, owned by uid 1001) | `rm`, `chown`, re-mounting, baking into an image |
| TLS certificates + ACME account | `storebaze_caddy_data` / `storebaze_caddy_config` volumes | `docker compose down -v`, `docker volume rm` / `prune`, `docker system prune --volumes` |
| Caddy | `storebaze-caddy` container, `/root/storebaze-deploy/Caddyfile` | restart, recreate, rebuild, edit, `git pull` over it |
| Secrets | `/root/envs/*/.env.production`, `/root/.env` | copying off the droplet, printing, editing — the one exception is the `IMAGE_TAG` line of `/root/.env` |
| Database | MongoDB Atlas (external) | any script, seed or migration run by hand as part of a deploy |
