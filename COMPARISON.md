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

## Sources

- Repo metadata: GitHub API on 2026-09-09.
- Coolify: [Wildcard certs](https://coolify.io/docs/knowledge-base/proxy/traefik/wildcard-certs),
  [Sentinel and metrics](https://coolify.io/docs/knowledge-base/server/sentinel),
  [Applications](https://coolify.io/docs/applications/),
  [Scheduled deploy discussion](https://github.com/coollabsio/coolify/discussions/2772).
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
