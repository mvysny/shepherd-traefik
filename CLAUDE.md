# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Shepherd-Traefik is a homebrew Heroku replacement: it builds git repos and deploys them as Docker
containers behind a Traefik reverse proxy on a single Linux box. There is **no build system and no
test suite** — the repo is a set of Bash scripts plus a `docker-compose.yaml`, deployed to `/opt/shepherd-traefik`
on the target machine. Changes are shell edits; "running" it means executing the scripts on a host with Docker.

The higher-level [shepherd-java-client](https://github.com/mvysny/shepherd-java-client) (`shepherd-cli`,
Web Admin) drives this project; this repo is the low-level layer it calls into. The predecessor
[Vaadin Shepherd](https://github.com/mvysny/shepherd) used Kubernetes; this rewrite drops k8s for plain Docker + Traefik.

## Architecture

Three long-lived admin containers, defined in `docker-compose.yaml`, all on a private `admin.int` Docker network:

- **`int_traefik`** — reverse proxy. Terminates https (Let's Encrypt wildcard cert via DNS challenge, resolver
  `default_shepherd`) on :443, plain http on :80, dashboard on :8080. Discovers apps via the Docker socket.
- **`int_jenkins`** — CI. Rebuilds each project on a schedule and calls `shepherd-build PROJECTID`. Exposed at
  `jenkins-admin.<domain>` so plugins can be upgraded. Has the host `docker` binary + buildx plugin bind-mounted in
  (docker-in-docker) and is added to the host `docker` group via `group_add`.
- **`int_shepherd`** — the shepherd-java Web Admin (`mvysny/shepherd-java:latest`), at `admin.<domain>`. Reads
  config from `/etc/shepherd/java/config.json`. `shepherd-build` execs `shepherd-cli restart` inside this container.

**Per-app model:** each hosted app is a Docker container `shepherd_PROJECTID` from image `shepherd/PROJECTID`, on
its own private network `PROJECTID.shepherd`, exposing port 8080. Apps must ship a `Dockerfile` at their repo root
buildable with `docker build -t x .` and runnable with `docker run -p8080:8080 x`. Published at `PROJECTID.<domain>`.

**The network-sharing gotcha:** Traefik can only *route* to a container if it *shares that container's network*.
A `docker compose up`/recreate reattaches Traefik only to `admin.int`, silently dropping every `*.shepherd`
attachment — routers then appear in the dashboard but requests 502. `shepherd-traefik-connect-networks` fixes this
(see below). Prefer `docker restart int_traefik` over a compose recreate to preserve attachments.

## Scripts

- **`install`** — one-time host setup (Ubuntu 24.04+, run as root): apt upgrade, install docker, create `admin.int`
  network, write `/etc/docker/daemon.json` (enlarged address pools so >28 project networks are possible +
  `containerd-snapshotter` for buildx local caches), create `/var/cache/shepherd/docker` and
  `/var/opt/shepherd/jenkins_home`, install the weekly cache-purge cron, and scaffold `/etc/shepherd/java/config.json`.
  Prints manual follow-up steps (fix docker GID in compose, Jenkins first-run setup, etc.).
- **`shepherd-build PROJECTID`** — called by Jenkins after a repo update. `docker build`s the image with a
  **per-project** buildx local cache (`/var/cache/shepherd/docker/$PROJECT_ID`) under memory/CPU limits, then execs
  `shepherd-cli restart -p PROJECTID` inside `int_shepherd`. Env knobs: `BUILD_MEMORY`, `CPU_QUOTA`, `BUILD_ARGS`,
  `DOCKERFILE`, `RUNTIME_MEMORY`. Validates PROJECTID against the DNS-label regex.
- **`shepherd-traefik-connect-networks`** — idempotently connects `int_traefik` to every `*.shepherd` network and
  restarts it if anything changed. Run after any Traefik recreate. Env: `TRAEFIK_CONTAINER`, `NETWORK_SUFFIX`, `RESTART_TRAEFIK`.
- **`shepherd-clearcache`** — installed as a weekly cron; `docker system prune -f` + wipes `/var/cache/shepherd/docker/*`.
- **`uninstall`** — stops/removes **all** Docker containers (not just Shepherd's), prunes everything, removes daemon
  config and `/var/opt/shepherd`, but **keeps `/etc/shepherd`** config.

## Conventions when editing

- All scripts are Bash with `set -e -o pipefail` (or `set -euo pipefail`). Keep that.
- Per-project caches must stay separate — a shared buildx cache breaks parallel builds (see shepherd issue #3).
  Related: `install` pins `concurrentJenkinsBuilders: 1` because 2 builders caused Gradle lock timeouts.
- Naming is load-bearing and mirrored in shepherd-java: container `shepherd_PROJECTID` / `int_*` for admin,
  image `shepherd/PROJECTID`, network `PROJECTID.shepherd`, admin network `admin.int`.
- `mydomain.me` is the placeholder DNS domain throughout `docker-compose.yaml` and `install`; the operator
  replaces it (or adds `/etc/hosts` entries for a toy/debug setup).
- Traefik is pinned to `v3.6` (needs 3.6+ so its docker client can talk to newer docker daemons).
