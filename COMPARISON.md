# Could Shepherd-Traefik be replaced by an off-the-shelf PaaS?

Shepherd-Traefik is a self-hosted mini-PaaS (see `CLAUDE.md`). This document compares it with the
popular, open-source, actively maintained products in the same category, to answer one question:
**does any of them check all of Shepherd's boxes?**

Facts below were checked on 2026-09-09 against GitHub and each project's docs. Star counts and
release dates will drift; the feature claims are the part worth re-checking before acting.

## The boxes

The main responsibilities from `CLAUDE.md`, phrased as requirements:

- `R_build_dockerfile` — build the app from the `Dockerfile` at the repo root, on the server, with
  per-project build caches and memory/CPU limits on the build.
- `R_periodic_rebuild` — rebuild **on a schedule**, not only on git push. Shepherd hosts repos it
  doesn't necessarily own (example projects, addons), so it can't rely on installing a webhook in
  every upstream repo; polling also picks up base-image and dependency updates.
- `R_run_docker` — run the app as a Docker container with runtime memory/CPU quotas, restarted
  on crash and on host reboot.
- `R_https_wildcard` — serve at `https://PROJECTID.<domain>` using **one wildcard Let's Encrypt
  cert via DNS challenge**, so a new app is reachable over https immediately with no per-app ACME
  round-trip.
- `R_observe_stats` — see per-app CPU/memory usage (and logs) from an admin UI.
- `R_single_host` — plain Docker on one Linux box; no Kubernetes (that's why the original Shepherd
  was rewritten).

## Candidates considered

Filter: open-source licence, thousands of GitHub stars, a release within the last few months.

| Project | Stars | Licence | Latest release | Kept? |
|---|---|---|---|---|
| [Coolify](https://github.com/coollabsio/coolify) | 61.6k | Apache-2.0 | v4.3.18, 2026-09-08 | yes |
| [Dokploy](https://github.com/Dokploy/dokploy) | 37.2k | Apache-2.0 (+ a proprietary `/proprietary` dir) | v0.30.6, 2026-09-08 | yes |
| [Dokku](https://github.com/dokku/dokku) | 32.1k | MIT | v0.38.27, 2026-08-12 | yes |
| [CapRover](https://github.com/caprover/caprover) | 15.2k | Apache-2.0 | v1.15.4, 2026-08-30 | yes |
| [Kamal](https://github.com/basecamp/kamal) | 14.6k | MIT | active | no: a deploy *tool* run from a dev machine, no server-side build/CI, HTTP-01 certs only |
| [Piku](https://github.com/piku/piku) | 6.6k | MIT | active | no: runs apps under uwsgi, not Docker; fails `R_run_docker` |
| [Kubero](https://github.com/kubero-dev/kubero) | 4.4k | GPL-3.0 | active | no: Kubernetes; fails `R_single_host` |
| Easypanel, Ploi, Cloud 66, Render… | — | closed source / SaaS | — | no |

## Feature matrix

Legend: ✅ built in · 🟡 possible but needs manual config or an external cron · ❌ not available.

| Requirement | Shepherd-Traefik | Coolify | Dokploy | Dokku | CapRover |
|---|---|---|---|---|---|
| `R_build_dockerfile` | ✅ Jenkins + buildx, per-project cache, build mem/CPU limits | ✅ Dockerfile is a first-class build pack | ✅ Dockerfile / Nixpacks / Buildpacks | ✅ Dockerfile / CNB / Herokuish; build limits via `resource:limit --process-type build` | ✅ via `captain-definition` pointing at the Dockerfile |
| `R_periodic_rebuild` | ✅ Jenkins poll-SCM schedule | 🟡 push webhooks only; cron an HTTP call to `/deploy?uuid=…` with an API token | 🟡 push webhooks; cron `POST /api/application.deploy`, or a Dokploy **Schedule** (cron task) that calls it | 🟡 `dokku git:sync --build-if-changes APP URL` is exactly a poll — but you cron it yourself | 🟡 push webhooks only; cron a call to the webhook URL |
| `R_run_docker` | ✅ (via shepherd-java) | ✅ mem/CPU limits in *Advanced* | ✅ mem/CPU limits per app (Docker Swarm services) | ✅ `resource:limit --cpu --memory` | 🟡 Swarm; limits only through raw *Service Update Override* JSON |
| `R_https_wildcard` | ✅ Traefik, DNS challenge, wildcard | 🟡 Traefik or Caddy; documented recipe to switch the resolver to DNS challenge + wildcard | 🟡 Traefik; default is HTTP-01, community recipes edit `traefik.yml` for a DNS-challenge resolver | 🟡 `dokku-letsencrypt` supports DNS-01 (lego providers) and wildcards since 2023 | 🟡 default HTTP-01 per app; DNS-01 only via *Certbot override*; long-open issues (#1444, #1761) |
| `R_observe_stats` | ❌ here; shepherd-java Web Admin shows them | ✅ *Sentinel*: per-container CPU/mem history graphs (not for Compose apps) | ✅ built-in per-service CPU/mem/net/disk | ❌ explicitly out of scope for Dokku (“will never manage monitoring”); `docker stats` or an external agent | 🟡 bundled NetData (server-level; per-container via cgroups charts) |
| `R_single_host` | ✅ plain Docker | ✅ plain Docker (multi-server optional over SSH) | ✅ but **Docker Swarm** mode | ✅ plain Docker | ✅ but **Docker Swarm** mode |
| Web admin UI | ✅ shepherd-java Web Admin | ✅ | ✅ | ❌ CLI only (Web UI is the paid *Dokku Pro*) | ✅ |
| Weight / stack | Bash + compose; Jenkins is the heavy part | Laravel/PHP + Postgres + Redis + Soketi | Node/Next.js + Postgres + Redis + Traefik | Bash + Go plugins, nginx by default | Node + Docker Swarm + nginx |

## Verdict

**Two products check every box, one with an asterisk:**

- **Dokploy** is the closest fit. Same reverse proxy (Traefik) so the wildcard-DNS-challenge setup
  is a config-file edit rather than a redesign, built-in per-app CPU/mem graphs, per-app resource
  limits, and it has a native cron *Schedule* feature plus a deploy API, so `R_periodic_rebuild` is
  an in-product cron job. Asterisk: it runs on Docker Swarm, and the repo carries a proprietary
  subdirectory (everything else is Apache-2.0).
- **Coolify** also checks every box: Dockerfile builds, resource limits, Sentinel per-container
  metrics, plain Docker (no Swarm), and a documented Traefik wildcard-cert recipe. Periodic rebuild
  is an external cron calling the deploy webhook. It is the heaviest stack of the group.

**Dokku** matches the *build/run/https* boxes best of all (and `git:sync --build-if-changes` is a
one-line replacement for the whole Jenkins poll), but it deliberately does no monitoring and has no
free web UI, so it fails `R_observe_stats` unless shepherd-java-style tooling is kept on top of it.

**CapRover** is the weakest match: wildcard/DNS-01 and resource limits both require hand-written
overrides, and it forces Docker Swarm.

## What a migration would *not* replace

The two "full match" products replace **both** this repo **and** the shepherd-java Web Admin/CLI,
because they own the app lifecycle and the UI. Things that have no direct equivalent and would need
re-implementing or dropping:

- The `shepherd_PROJECTID` / `shepherd/PROJECTID` / `PROJECTID.shepherd` naming contract that
  shepherd-java relies on.
- Per-project buildx local caches with a weekly purge (`shepherd-clearcache`); both products cache
  via Docker's own layer cache.
- Shepherd's simple one-file-per-project config (`/etc/shepherd/java/config.json`), replaced by the
  product's database and UI.
- The planned per-project Postgres service (README TODO) — both Coolify and Dokploy already offer
  one-click managed databases, which would be a gain.

## Coolify: minimum footprint and installation

Checked on 2026-09-09 against `coollabsio/coolify@main`.

### Can it be run with fewer moving parts?

**No — four containers, all mandatory:** `coolify` (the Laravel app: nginx + php-fpm, with s6 supervising
Horizon, the scheduler and the DB migration step), `coolify-db` (`postgres:15-alpine`), `coolify-redis`
(`redis:7-alpine`) and `coolify-realtime` (Soketi + a terminal server, ports 6001/6002). Every one is a
`depends_on: {condition: service_healthy}` of the app container.

**Redis cannot be dropped.** It is not merely the configured default:

- `config/queue.php` defaults to `redis`, and the production container runs `php artisan horizon`.
  **Laravel Horizon only supports Redis queues** — setting `QUEUE_CONNECTION=database` doesn't degrade
  gracefully, it orphans the queue: deployments get dispatched and nothing drains them.
- `config/cache.php` defaults to the `redis` store.
- `app/Jobs/ScheduledJobManager.php` calls the Redis facade *directly*
  (`Redis::connection('default')->ttl(...)`) for scheduled-job locking. That is a code-level dependency
  with no config knob behind it.
- The `HORIZON_ENABLED=false` escape hatch in the s6 run script only parks the worker (a dev convenience);
  it does not remove Redis from the other two paths.

**SQLite is not a supported deployment.** `config/database.php` defaults to `pgsql`; SQLite appears only
as the `testing` connection (`:memory:`, driven by `phpunit.xml` off a generated
`database/schema/testing-schema.sql`). So the schema is *broadly* SQLite-shaped, but that is the test
harness, not a deployment mode — and Postgres is assumed elsewhere:

- The default session driver is `database`, so sessions land in Postgres too.
- Four of the ~390 migrations are Postgres-only, guarded by `getDriverName() !== 'pgsql'` (fillfactor and
  autovacuum tuning; column retypes using `USING …::text`). On SQLite they'd silently no-op.
- Coolify backs up *its own* database with `pg_dump` of `coolify-db` (`SettingsBackup`,
  `DatabaseBackupJob`), and ships a dedicated `scripts/upgrade-postgres.sh` for major-version jumps.

**What you actually can trim:** `DB_HOST`/`DB_PORT` and `REDIS_HOST`/`REDIS_PORT`/`REDIS_URL` are read
from the environment (defaulting to `coolify-db` / `coolify-redis`), so the two bundled containers can be
pointed at an external Postgres and Redis. That moves the dependency off the box; it does not remove it.

**Footprint:** documented minimum is 2 cores / 2 GB RAM / 30 GB disk; the installer hard-checks disk
(30 GB total, 20 GB free) and only warns. The Coolify stack itself idles at roughly 1 GB RAM before a
single app is deployed — against Shepherd-Traefik's Traefik + Jenkins + shepherd-java.

### Installation procedure

One command, as root:

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```

`scripts/install.sh` then: detects the distro (Debian/RHEL/Arch/Alpine/SLES families), installs
`curl wget git jq openssl`, installs Docker if missing (requires 24+), writes `/etc/docker/daemon.json`
(json-file log rotation + `default-address-pools` `10.0.0.0/8` size 24), creates `/data/coolify/{source,ssh,
applications,databases,backups,services,proxy,sentinel,images}` owned by uid 9999 mode 700, downloads
`docker-compose.yml`, `docker-compose.prod.yml`, `.env.production`, `upgrade.sh` and `upgrade-postgres.sh`
from `cdn.coollabs.io`, generates `APP_ID`/`APP_KEY`/`DB_PASSWORD`/`REDIS_PASSWORD`/`PUSHER_*` with
`openssl`, generates an ed25519 keypair and appends it to root's `authorized_keys`, and brings the stack up.
Finally it prints `http://<ip>:8000`, where you register the first (root) user.

The manual equivalent is the same seven steps, ending in:

```bash
docker network create --attachable coolify
docker compose --env-file /data/coolify/source/.env \
  -f /data/coolify/source/docker-compose.yml \
  -f /data/coolify/source/docker-compose.prod.yml \
  up -d --pull always --remove-orphans --force-recreate
```

Three things worth knowing before running this on a Shepherd box:

- **It edits `/etc/docker/daemon.json`.** It reuses an already-configured `default-address-pools` rather
  than overwriting it (unless `DOCKER_POOL_FORCE_OVERRIDE=true`), so it should leave `install`'s enlarged
  pools alone — but this is the one file both projects claim.
- **Coolify manages even the local host over SSH**, hence the key it appends to `authorized_keys`.
- **The installer does not start a reverse proxy.** It only creates `/data/coolify/proxy/dynamic`; the
  `coolify-proxy` (Traefik) container is started by Coolify itself when you onboard the server in the UI,
  and Coolify then runs `docker network connect <uuid> coolify-proxy` per resource — the in-product
  answer to the same network-sharing gotcha that `shepherd-traefik-connect-networks` solves here.

## Sources

- Repo metadata: GitHub API on 2026-09-09.
- Coolify: [Wildcard certs](https://coolify.io/docs/knowledge-base/proxy/traefik/wildcard-certs),
  [Sentinel and metrics](https://coolify.io/docs/knowledge-base/server/sentinel),
  [Applications](https://coolify.io/docs/applications/),
  [Scheduled deploy discussion](https://github.com/coollabsio/coolify/discussions/2772),
  [Installation docs](https://coolify.io/docs/get-started/installation).
- Coolify footprint chapter, read from `coollabsio/coolify@main` on 2026-09-09:
  [`docker-compose.yml`](https://github.com/coollabsio/coolify/blob/main/docker-compose.yml),
  [`docker-compose.prod.yml`](https://github.com/coollabsio/coolify/blob/main/docker-compose.prod.yml),
  [`scripts/install.sh`](https://github.com/coollabsio/coolify/blob/main/scripts/install.sh),
  [`config/database.php`](https://github.com/coollabsio/coolify/blob/main/config/database.php),
  [`config/queue.php`](https://github.com/coollabsio/coolify/blob/main/config/queue.php),
  [`config/cache.php`](https://github.com/coollabsio/coolify/blob/main/config/cache.php),
  [`app/Jobs/ScheduledJobManager.php`](https://github.com/coollabsio/coolify/blob/main/app/Jobs/ScheduledJobManager.php),
  [`docker/production/etc/s6-overlay`](https://github.com/coollabsio/coolify/tree/main/docker/production/etc/s6-overlay/s6-rc.d).
- Dokploy: [Features](https://docs.dokploy.com/docs/core/features),
  [Monitoring](https://dokploy.com/features/container-server-monitoring),
  [Auto deploy / API](https://docs.dokploy.com/docs/core/auto-deploy),
  [Schedule API](https://docs.dokploy.com/docs/api/schedule),
  [Wildcard via Traefik DNS challenge](https://www.naps62.com/posts/wildcard-ssl-in-dokploy),
  [Licence](https://github.com/Dokploy/dokploy/blob/canary/LICENSE.MD).
- Dokku: [Resource management](https://dokku.com/docs/advanced-usage/resource-management/),
  [Git deployment / git:sync](https://dokku.com/docs/deployment/methods/git/),
  [dokku-letsencrypt DNS-01](https://github.com/dokku/dokku-letsencrypt),
  [Monitoring stance (maintainer, discussion #5681)](https://github.com/dokku/dokku/discussions/5681).
- CapRover: [Deployment methods](https://caprover.com/docs/deployment-methods.html),
  [Resource monitoring](https://caprover.com/docs/resource-monitoring.html),
  [Service update override](https://caprover.com/docs/service-update-override.html),
  [Certbot overrides](https://caprover.com/docs/certbot-config.html),
  [DNS-01 issue #1444](https://github.com/caprover/caprover/issues/1444),
  [DNS-01 issue #1761](https://github.com/caprover/caprover/issues/1761).
