# Which PaaS should Shepherd-Traefik be retired into?

Shepherd-Traefik is a self-hosted mini-PaaS (see `CLAUDE.md`). This document compares it with the
popular, open-source, actively maintained products in the same category.

**The goal is retirement, not coexistence.** The intent is to shut down *all* `shepherd*` projects —
this repo **and** the shepherd-java Web Admin / `shepherd-cli` — and replace them with a new repo that
documents how to set up an off-the-shelf product instead. So the question is not "can Shepherd be
replaced" but **"which product do we retire into, and what glue does the new repo have to carry?"**

Two consequences of that framing, both of which move the answer:

- The `shepherd_PROJECTID` naming contract, `/etc/shepherd/java/config.json` and the ability to drive
  the platform from shepherd-java are **not costs** — they are things being deleted on purpose.
- Anything shepherd-java used to supply — the admin UI, per-app stats — must now come from the product
  or from documented glue, because there will be no shepherd-java to fall back on.

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
- `R_observe_stats` — see per-app CPU/memory usage and logs. **Relaxed 2026-09-09:** a CLI or TUI is
  acceptable; this no longer has to be a web UI (see `R_admin_interface`).
- `R_admin_interface` — some way to administer apps: create, deploy, restart, inspect. **Relaxed
  2026-09-09:** web UI, TUI or CLI all qualify, and a third-party add-on counts as a web UI. A product
  whose *only* interface is a CLI is a downgrade to note, not a disqualification — control by issuing
  commands and reading their output is already how shepherd-java and `virtui` work.
- `R_single_host` — everything on one Linux box. No multi-node cluster to operate.
- `R_no_kubernetes` — no Kubernetes; that is why the original Shepherd was rewritten. **Docker Swarm
  is explicitly accepted** (decided 2026-09-09): on a single node it is `docker swarm init` once and
  then the PaaS's problem, it is upstream in the engine already installed, and Mirantis has committed
  to supporting it through at least 2030. Swarm is feature-stable, not dying.
- `R_restore_the_box` — bring the whole platform back on fresh hardware from a backup. Today this is
  trivial (back up one JSON file); any product with a database-backed control plane replaces that with
  a documented backup/restore drill, which the new repo then has to own and test.

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
| [Kubero](https://github.com/kubero-dev/kubero) | 4.4k | GPL-3.0 | active | no: Kubernetes; fails `R_no_kubernetes` |
| Easypanel, Ploi, Cloud 66, Render… | — | closed source / SaaS | — | no |

## Feature matrix

Legend: ✅ built in · 🟡 possible but needs manual config or an external cron · ❌ not available.

| Requirement | Shepherd-Traefik | Coolify | Dokploy | Dokku | CapRover |
|---|---|---|---|---|---|
| `R_build_dockerfile` | ✅ Jenkins + buildx, per-project cache, build mem/CPU limits | ✅ Dockerfile is a first-class build pack | ✅ Dockerfile / Nixpacks / Buildpacks | ✅ Dockerfile / CNB / Herokuish; build limits via `resource:limit --process-type build` | ✅ via `captain-definition` pointing at the Dockerfile |
| `R_periodic_rebuild` | ✅ Jenkins poll-SCM schedule | 🟡 push webhooks only; cron an HTTP call to `/deploy?uuid=…` with an API token | 🟡 push webhooks; cron `POST /api/application.deploy`, or a Dokploy **Schedule** (cron task) that calls it | 🟡 `dokku git:sync --build-if-changes APP URL` is exactly a poll — but you cron it yourself | 🟡 push webhooks only; cron a call to the webhook URL |
| `R_run_docker` | ✅ (via shepherd-java) | ✅ mem/CPU limits in *Advanced* | ✅ mem/CPU limits per app (Docker Swarm services) | ✅ `resource:limit --cpu --memory` | 🟡 Swarm; limits only through raw *Service Update Override* JSON |
| `R_https_wildcard` | ✅ Traefik, DNS challenge, wildcard | 🟡 Traefik or Caddy; documented recipe to switch the resolver to DNS challenge + wildcard | 🟡 Traefik; default is HTTP-01, community recipes edit `traefik.yml` for a DNS-challenge resolver | 🟡 two routes, neither turnkey — see *Dokku: wildcard certs* below | 🟡 default HTTP-01 per app; DNS-01 only via *Certbot override*; long-open issues (#1444, #1761) |
| `R_observe_stats` | ❌ here; shepherd-java Web Admin shows them | ✅ *Sentinel*: per-container CPU/mem history graphs (not for Compose apps) | ✅ built-in per-service CPU/mem/net/disk | 🟡 no monitoring by design (“will never manage monitoring”), but apps are plain containers, so `docker stats` / `docker logs` work directly — CLI-acceptable since the relaxation | 🟡 bundled NetData (server-level; per-container via cgroups charts) |
| `R_admin_interface` | ✅ Web Admin + `shepherd-cli` | ✅ web + official CLI + REST API | ✅ web + official CLI + OpenAPI | 🟡 official **CLI only** (no HTTP API; reports do emit `--format json`); web is third-party | ✅ web + official CLI + API |
| `R_single_host` | ✅ | ✅ (multi-server optional over SSH) | ✅ | ✅ | ✅ |
| `R_no_kubernetes` | ✅ plain Docker | ✅ plain Docker | ✅ Docker Swarm — accepted | ✅ plain Docker | ✅ Docker Swarm — accepted |
| `R_restore_the_box` | ✅ back up one JSON file | 🟡 Postgres control plane; documented S3 backup + restore, but a fresh instance generates a new `APP_KEY` while the dump is encrypted with the old one — a naive restore comes back broken | 🟡 Postgres control plane; **“system restore”** is a named feature explicitly framed for old-server → new-server migration | ✅ state is files under `/home/dokku` + git remotes; no database to dump | 🟡 control-plane state in its own container |
| Weight / stack | Bash + compose; Jenkins is the heavy part | Laravel/PHP + Postgres + Redis + Soketi (~1 GB idle, 4 containers, all mandatory) | Node/Next.js + Postgres + Redis + Traefik | Bash + Go plugins, nginx by default — no control-plane database at all | Node + Docker Swarm + nginx |

## Admin interfaces, per contender

Since `R_admin_interface` was relaxed to accept web **or** TUI **or** CLI — and since a third-party
add-on counts as a web UI — it's worth spelling out what each product actually ships. Repo metadata
from the GitHub API on 2026-09-09.

| | Web UI | CLI | HTTP API | TUI |
|---|---|---|---|---|
| **Shepherd-Traefik** | ✅ shepherd-java Web Admin | ✅ `shepherd-cli` | — | ❌ |
| **Coolify** | ✅ official, primary | ✅ **official**: [`coollabsio/coolify-cli`](https://github.com/coollabsio/coolify-cli) — 461★, MIT, Go, active (2026-09-05) | ✅ documented REST API (the CLI is a client of it) | ❌ none specific |
| **Dokploy** | ✅ official, primary | ✅ **official**: [`Dokploy/cli`](https://github.com/Dokploy/cli) — 148★, MIT, npm `@dokploy/cli`, active (2026-09-08); 449 commands generated from the OpenAPI spec, so coverage is total | ✅ OpenAPI | ❌ none specific |
| **Dokku** | 🟡 **third-party only** — see below. Official web UI is the paid *Dokku Pro* | ✅ **official and primary**, driven over SSH (`ssh dokku@host apps:report`) | ❌ **none** — there is no HTTP API; remote control *is* SSH | ❌ none specific |
| **CapRover** | ✅ official, primary | ✅ official: [`caprover/caprover-cli`](https://github.com/caprover/caprover-cli) — 80★, npm `caprover`, active (2026-09-09). Deploy-focused (`deploy`, `login`, `list`, `logs`) plus a raw `caprover api` passthrough, so thinner than the two above | ✅ (what the web UI uses) | ❌ none specific |

**Every contender except Dokku ships an official web UI, an official CLI *and* a documented HTTP API.**
Dokku is the outlier on two axes at once — but see the next section: for this project neither axis is
as expensive as it first looks.

### Why "CLI only, no HTTP API" is not a blocker here

**Driving a platform by issuing commands and reading their output is already the established pattern in
this family of projects** (decided 2026-09-09), so Dokku's missing HTTP API costs nothing new:

- `virtui` is built this way.
- The shepherd-java client already controls Kubernetes *and* Traefik by **issuing `docker` commands**
  and consuming the results — the current Web Admin's per-app stats and logs come from exactly this
  mechanism, not from an API.

So a hypothetical Sinatra/Rails frontend over Dokku would be doing what shepherd-java does today, not
something new and worse. Dokku supports the pattern deliberately: the `dokku` user's `authorized_keys`
uses a forced command, making `ssh dokku@host <cmd>` the sanctioned remote interface.

And it is better than screen-scraping in the literal sense, because **Dokku's `*:report` commands emit
structured output**:

- `--format json` gives a machine-readable JSON view of any report (`dokku apps:report --format json`
  returns a JSON array, `[]` when no apps exist). JSON keys are the flag names with the leading prefix
  stripped — `logs:report --format json` emits `max-size`, `global-max-size`, `computed-max-size`.
- Single-value flags return one bare value for shell use, e.g.
  `dokku docker-options:report node-js-app --docker-options-build`.

That makes Dokku's CLI a structured-data interface reached over SSH rather than an HTTP one — a
transport difference, not a capability difference. The genuine remaining cost is only the **web UI**:
third-party (see below) or nothing.

### Dokku's third-party web UIs: one live option on top of a graveyard

| Project | Stars | Licence | Last push | State |
|---|---|---|---|---|
| [palfrey/wharf](https://github.com/palfrey/wharf) | 262★ | AGPL-3.0 | 2026-09-03 | **alive** — actively maintained, but a single-maintainer project |
| [Pruvon](https://pruvon.dev/) | — | claims AGPLv3 | — | **unverified** — the site advertises "open source, self-hosted, free forever", but no public repo was locatable on 2026-09-09. Treat as closed until the repo is found |
| [ledokku/ledokku](https://github.com/ledokku/ledokku) | 642★ | MIT | 2023-10-04 | dead — and this was the one *Dokku itself endorsed* on Twitter in 2021 |
| [cywio/atlas](https://github.com/cywio/atlas) | 59★ | MIT | 2022-01-16 | dead |
| [MaximeHeckel/HarborJS](https://github.com/MaximeHeckel/HarborJS) | 111★ | MIT | 2018-05-27 | dead |

The pattern matters more than any single row: **third-party Dokku web UIs have a consistent history of
being abandoned**, including the most popular and most officially-blessed one. Retiring shepherd-java in
favour of a third-party UI trades a dependency you control for one you don't, on a component with a
demonstrated death rate. The CLI, by contrast, is Dokku's *primary* interface and the thing its
maintainers support — betting on it is the robust choice, betting on a UI over it is not.

### The TUI option is generic, and it is free

No contender ships a TUI, but none needs to: for any product that runs apps as **plain Docker
containers** (Shepherd-Traefik today, Coolify, Dokku), an off-the-shelf Docker TUI covers the
`R_observe_stats` box for zero lines of code —
[lazydocker](https://github.com/jesseduffield/lazydocker) (52.8k★, MIT, live CPU/memory graphs, logs,
per-container shell) or [ctop](https://github.com/bcicen/ctop) (17.8k★, MIT, top-like container
metrics; last push 2024-07 but feature-complete). This is the cheapest possible answer to Dokku's
monitoring gap.

It is also a **hidden argument against the Swarm products**: under Swarm, containers are tasks named
`app.1.<taskid>` rather than stable names, so these tools still work but show you churning task IDs
instead of apps. The generic-TUI escape hatch is materially nicer on plain Docker.

## Dokku: wildcard certs

`R_https_wildcard` wants **one** wildcard cert so a new app is served over https immediately, with no
per-app ACME round-trip. Dokku has two routes there and neither is turnkey:

- **[`dokku-letsencrypt`](https://github.com/dokku/dokku-letsencrypt)** (official plugin — 1118★, MIT,
  active 2026-09-01) gained wildcard support via the DNS-01 challenge in **v0.20.0**, using a lego DNS
  provider (Cloudflare, Route53, Namecheap…). Credentials can be set `--global`, but **issuance is
  per-app** — so every new app still performs its own ACME round-trip. That satisfies "https works"
  but not the "no per-app round-trip" half of the requirement. 🟡
- **[`dokku-global-cert`](https://github.com/dokku-community/dokku-global-cert)** (community plugin —
  20★, MIT, active 2026-07-20) is the exact model Shepherd uses: **one** cert, imported for every new
  app and applied to every existing app that has no cert of its own, with re-application on update.
  Combined with your own lego/certbot DNS-01 renewal cron, this fully satisfies `R_https_wildcard`. ✅
  Cost: you own the renewal cron, and the plugin is low-profile (20★) though genuinely maintained.
  There is also a rawer official variant — drop `server.crt`/`server.key` into `/home/dokku/tls` and
  uncomment the `ssl_certificate` lines in `/etc/nginx/conf.d/dokku.conf`.

The `dokku-global-cert` + renewal-cron combination is glue — but it is *exactly the kind of glue the new
repo exists to document*, and it is a smaller, more inspectable piece of glue than a `traefik.yml`
patch to a product that owns its own Traefik.

## Verdict

The two relaxations — Swarm accepted, and CLI/TUI accepted for administration — reshuffled this
substantially. **Accepting Swarm promoted nobody** (Dokploy and CapRover flip to ✅ on
`R_no_kubernetes`, but CapRover still fails wildcard certs *and* resource limits, so it remains the
weakest match). **Accepting a CLI, however, promoted Dokku from disqualified to arguably the best
fit.** It is now a three-horse race.

**Dokku** — the strongest match on the requirements that are hard to glue, and the only contender that
*subtracts* from the current architecture instead of substituting for it:

- `dokku git:sync --build-if-changes APP URL` is a literal SCM poll — it replaces `R_periodic_rebuild`
  **and deletes Jenkins**, the heaviest single component in the current stack, for one crontab line.
- Plain Docker, no Swarm, no control-plane database: state is files under `/home/dokku` plus git
  remotes, so `R_restore_the_box` stays a file backup — the one property of the current setup most
  worth keeping, and the only contender that keeps it.
- Bash + Go plugins is the smallest conceptual delta from Bash + compose.
- Its two gaps are the two chapters of the new repo: a wildcard-cert renewal cron (above) and a stats
  view (`lazydocker`, zero code).
- Its one real cost is a **web UI**: third-party or nothing. The missing HTTP API is *not* a cost —
  command-and-parse is already how shepherd-java and `virtui` drive Docker, Kubernetes and Traefik, and
  Dokku's reports emit `--format json` anyway.

**Dokploy** — the best fit if a web UI is non-negotiable. Same Traefik as here (so wildcard DNS-01 is a
config edit, not a redesign), native cron *Schedules* so `R_periodic_rebuild` is in-product, built-in
per-app metrics, a total-coverage official CLI, and "system restore" as a named migration feature.
Asterisks: Docker Swarm, a proprietary subdirectory in an otherwise Apache-2.0 repo, and **v0.30.6 —
still pre-1.0**, which matters more than usual when the deliverable is documentation written against it.

**Coolify** — checks every box on plain Docker, with the largest community and a real REST API + Go
CLI. Costs: the heaviest stack of the group by a wide margin (4 mandatory containers, ~1 GB idle before
a single app is deployed — see the footprint chapter), periodic rebuild is an external cron, and its
restore path has the `APP_KEY` footgun.

**CapRover** — weakest match; wildcard/DNS-01 and resource limits both need hand-written overrides.

### How to settle it

The new repo is a *guide*, and a guide cannot be written for a product nobody has installed. Both
finalist installs are an afternoon on a throwaway VPS, and one exercise discriminates between them
better than any further desk research: **bring the box back from bare metal plus a backup, timed.**
That tests `R_restore_the_box` for real, reveals how much of the guide is click-path versus
checked-in file, and shakes out the Swarm-specific unknowns as a side effect —

- the Swarm **overlay** address pool, which is settable only at `docker swarm init --default-addr-pool`
  time and which Dokploy's installer runs for you without those flags (this repo already hit the >28
  network wall once, hence `install`'s enlarged `default-address-pools`);
- whether Swarm services resolve locally-built images cleanly with `containerd-snapshotter` enabled;
- how the product's Traefik reattaches to per-app networks — the in-product answer to the gotcha
  `shepherd-traefik-connect-networks` solves here.

## What the new repo would have to carry

Since the goal is to retire `shepherd*` entirely, the naming contract and `config.json` are
*deletions*, not losses. What actually needs re-implementing, documenting or consciously dropping:

- **Per-project buildx local caches** with a weekly purge (`shepherd-clearcache`). Every product caches
  via Docker's own layer cache instead; the per-project isolation that fixed shepherd issue #3 has no
  direct equivalent, so parallel-build behaviour needs re-checking.
- **A wildcard-cert renewal cron**, for any product that doesn't own the whole ACME flow — the largest
  chapter for Dokku, a `traefik.yml` patch for Dokploy/Coolify.
- **The periodic-rebuild trigger** — one crontab line for Dokku (`git:sync --build-if-changes`), an
  in-product *Schedule* for Dokploy, an external cron hitting a webhook for Coolify/CapRover.
- **A tested restore drill** (`R_restore_the_box`) — trivial today, a real chapter for anything with a
  Postgres control plane.
- **A stats view**, if the chosen product doesn't ship one: `lazydocker` or `ctop`, zero code.

Two things a migration would *gain*: the planned per-project Postgres service (README TODO) exists as a
one-click managed database in Coolify, Dokploy and Dokku alike, and Jenkins disappears entirely.

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
- Dokploy backup/restore: [Backups](https://docs.dokploy.com/docs/core/backups),
  [Restore](https://docs.dokploy.com/docs/core/databases/restore).
- Dokku: [Resource management](https://dokku.com/docs/advanced-usage/resource-management/),
  [Git deployment / git:sync](https://dokku.com/docs/deployment/methods/git/),
  [dokku-letsencrypt DNS-01](https://github.com/dokku/dokku-letsencrypt),
  [dokku-global-cert](https://github.com/dokku-community/dokku-global-cert/blob/master/README.md),
  [SSL configuration / `/home/dokku/tls`](http://dokku.viewdocs.io/dokku/configuration/ssl/),
  [`--format json` and single-value report flags](https://dokku.com/docs/advanced-usage/docker-options/),
  [Application management](https://dokku.com/docs/deployment/application-management/),
  [Monitoring stance (maintainer, discussion #5681)](https://github.com/dokku/dokku/discussions/5681),
  [Dokku Pro](https://github.com/dokku/dokku/blob/master/docs/enterprise/pro.md).
- Admin interfaces: [Coolify CLI docs](https://next.coolify.io/docs/cli/what-is-the-coolify-cli),
  [Coolify API reference](https://coolify.io/docs/api-reference/api/),
  [Dokploy CLI docs](https://docs.dokploy.com/docs/cli),
  [`@dokploy/cli` on npm](https://www.npmjs.com/package/@dokploy/cli),
  [CapRover CLI commands](https://caprover.com/docs/cli-commands.html),
  [wharf](https://github.com/palfrey/wharf), [Pruvon](https://pruvon.dev/),
  [ledokku](https://github.com/ledokku/ledokku) (and [Dokku's 2021 endorsement of
  it](https://x.com/dokku/status/1373740087968686080)),
  [lazydocker](https://github.com/jesseduffield/lazydocker), [ctop](https://github.com/bcicen/ctop).
- Docker Swarm status: [Swarm mode docs (no deprecation notice)](https://docs.docker.com/engine/swarm/),
  [docker/roadmap #175 "clarify its status"](https://github.com/docker/roadmap/issues/175),
  [Mirantis support-through-2030 commitment, summarised](https://blog.oxyconit.com/docker-swarm-mode-2026-practical-guide/).
- Coolify backup/restore: [Backup and Restore Coolify](https://coolify.io/docs/knowledge-base/how-to/backup-restore-coolify),
  [Restore-from-instance-backup discussion #1684](https://github.com/coollabsio/coolify/discussions/1684).
- CapRover: [Deployment methods](https://caprover.com/docs/deployment-methods.html),
  [Resource monitoring](https://caprover.com/docs/resource-monitoring.html),
  [Service update override](https://caprover.com/docs/service-update-override.html),
  [Certbot overrides](https://caprover.com/docs/certbot-config.html),
  [DNS-01 issue #1444](https://github.com/caprover/caprover/issues/1444),
  [DNS-01 issue #1761](https://github.com/caprover/caprover/issues/1761).
