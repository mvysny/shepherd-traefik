# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Shepherd-Traefik is a homebrew Heroku replacement: it builds git repos and deploys them as Docker
containers behind a Traefik reverse proxy on a single Linux box. There is **no build system and no
test suite** — the repo is a set of Bash scripts plus a `docker-compose.yaml`, deployed to `/opt/shepherd-traefik`
on the target machine. Changes are shell edits; "running" it means executing the scripts on a host with Docker.

**Project category: a self-hosted mini-PaaS (Platform-as-a-Service).** The same family as Dokku, CapRover, Coolify,
Dokploy and Piku: a "Heroku-like" single-host platform that turns git repos into running containers behind an
auto-TLS reverse proxy. Two well-known sub-categories it combines: a *poll-SCM CI/CD pipeline* (Jenkins rebuilds
on a schedule — not git-push-to-deploy like Dokku) and *Docker-label-driven ingress with automatic Let's Encrypt
certificates* (Traefik, the same pattern Caddy/Traefik "docker auto-proxy" setups use).

**Main responsibilities** (what this repo does itself vs. delegates):

- **Builds the app in Docker.** `shepherd-build` runs `docker build` on the project's `Dockerfile` into image
  `shepherd/PROJECTID`, with a per-project buildx cache and memory/CPU limits on the *build*. Jenkins schedules it.
- **Runs the app in Docker.** Delegated: `shepherd-build` execs `shepherd-cli restart` inside `int_shepherd`, and
  shepherd-java does the actual `docker run` as `shepherd_PROJECTID` on network `PROJECTID.shepherd` (with runtime
  memory/CPU quotas from `/etc/shepherd/java/config.json`). This repo only provides the host setup and naming
  contract.
- **Serves each app at `https://PROJECTID.<domain>`.** Traefik terminates TLS with a wildcard Let's Encrypt cert
  (DNS challenge), routes by host to the app container via Docker labels, and also fronts `admin.<domain>` and
  `jenkins-admin.<domain>`.
- **Keeps apps up.** Docker's restart policy restarts containers on crash and after host reboot; Traefik, Jenkins
  and the Web Admin are `restart: always` in `docker-compose.yaml`.
- **Runtime resource control, not observation.** Build and runtime CPU/memory *limits* are enforced (see above), but
  no script in this repo *observes* runtime stats (CPU/mem usage, logs). Observation happens in the shepherd-java
  Web Admin / `shepherd-cli`, which read them straight from Docker, or by plain `docker stats` / `docker logs`.
- **Host housekeeping.** `install` provisions the box, `shepherd-clearcache` prunes images and build caches weekly,
  `shepherd-traefik-connect-networks` repairs Traefik's per-app network attachments, `uninstall` tears it down.

The higher-level [shepherd-java-client](https://github.com/mvysny/shepherd-java-client) (`shepherd-cli`,
Web Admin) drives this project; this repo is the low-level layer it calls into. The predecessor
[Vaadin Shepherd](https://github.com/mvysny/shepherd) used Kubernetes; this rewrite drops k8s for plain Docker + Traefik
(`D_kubernetes` → `D_docker_traefik`), which is also why there is no build system or test suite here.

## Documentation targets

This repo's *durable* prose lives in six places, each with a distinct audience and *what it is allowed
to own* — plus `ideas/`, which is deliberately not durable (see *Ideas & their graduation* below).
Match the target before writing a line — the failure mode is a fact explained twice, which then drifts.

| Target | Audience | Scope & length | Owns |
|---|---|---|---|
| **README.md** | the operator at the front door | thin: positioning, requirements, install, troubleshooting, how to onboard a project | *how to run this box* — and routing the reader onward |
| **CLAUDE.md** (this file) | a contributor / coding agent | invariant-focused; pointers, not reference | what you must not break *from a distance*: the naming contract, the network-sharing gotcha, the per-project-cache rule — plus the **script index** below |
| **Script comment headers** (`install`, `shepherd-build`, …) | someone reading or invoking that one script | dense, per-script, standalone | the precise technical truth of that script: arguments, env knobs, prerequisites, and a `WHY THIS IS NEEDED` block where the *what* isn't self-evident (`shepherd-traefik-connect-networks` is the model) |
| **`install`'s printed follow-up steps** | the operator, mid-install | a numbered list, emitted at the end of a run | the manual steps `install` deliberately does *not* automate (docker GID in compose, Jenkins first-run, DNS) |
| **DECISIONS.md** | someone asking "why is it like this?" | one coherent, mutable entry per live decision *already made* (`D_` slugs) | the *why-we-chose*, including the roads not taken |
| **COMPARISON.md** | someone deciding whether to retire this repo | requirements as `R_` boxes + a survey of replacement products | *what else exists*, and which product this could be retired into |

Rules that make six targets survivable:

- **Single source of truth per fact.** Each fact has one home and the others link to it. When tempted
  to explain something twice, link instead — the failure mode to watch for is compressing a `D_` entry
  into a bullet here, which reads like a summary and is really a third copy.
- **But don't over-link into unreadability.** A one-line load-bearing restatement is fine when it saves
  a jump ("per-project caches are mandatory; see `D_no_shared_cache` for why"). Repeat the *fact*, defer
  the *explanation*.
- **A script's comment header must stand alone.** It is read by whoever is about to run the thing, who
  will not go looking for a rationale first. It may defer *motivation* ("see `D_no_shared_cache`"), never
  *usage*.
- **COMPARISON.md answers "should we replace this", DECISIONS.md answers "why is it like this".** Don't
  argue a Shepherd design decision in COMPARISON.md — link to the `D_`; and don't migrate the `R_` boxes
  or the product survey into DECISIONS.md.
- **DECISIONS.md records only decisions already taken.** Shipped, or accepted-and-not-yet-implemented (that
  is what its `Status:` line is for). A speculative feature, an idea, an open question or a TODO is *not* a
  decision and gets no `D_` — the roads-not-taken inside an existing entry are the only "what we didn't do"
  content the file carries. Known-missing features stay a TODO in `README.md` (Postgres service) or an `R_`
  gap in `COMPARISON.md`.
- **Enumerated items get slugs, not numbers** — `R_build_cache`, `D_no_shared_cache`, underscores
  throughout, backticked in prose. Stable once published; rename only with a sweep of every reference.
- There is deliberately **no CHANGELOG** (the deploy is a `git pull`, so git *is* the changelog) and no
  glossary — the naming contract in *Conventions when editing* is the whole vocabulary.

## Ideas & their graduation

`ideas/` holds designs not yet acted on, one per file, kebab-named after what the idea *is*
(`ideas/herokuish-builds.md`). `ls ideas/` is the index — never add one. An idea file is a
scratchpad: brainstorm freely, and none of the doc rules above apply to it, because it is going to be
deleted. That also means **nothing durable may live only there**, and nothing durable should *point*
there — an idea file links out to `R_`/`D_` slugs, never the other way round, so graduation is never a
dangling-reference sweep. (Deep research an idea accumulates goes in a same-stem sidecar folder,
`ideas/herokuish-builds/`, split only when the one file becomes unreadable.)

**An idea graduates the moment it is acted on, and graduation is not done until the file is gone.**
Before deleting, backport what lasts to the durable place for that kind of fact — the six targets above,
which for ideas in this repo means in practice:

| Nugget | Goes to |
|---|---|
| the decision itself, once taken, with the roads not taken | a `D_` entry in `DECISIONS.md` (and its `Status:` line) |
| a requirement it moves, or a scored product comparison | the `R_` boxes / feature matrix in `COMPARISON.md` |
| how to invoke or configure the thing that got built | the comment header of the script that implements it |
| a manual step `install` won't automate | `install`'s trailing `echo` block |
| operator-facing how-to, or a known-missing feature | `README.md` (prose, or its TODO list) |
| an invariant a future editor could break from a distance | the sections above in this file |

A rejected idea leaves at most a one-line why-not, in the place a future reader would look, and only if
the trap is invisible in the scripts themselves. Otherwise it just gets deleted.

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
(see below). Prefer `docker restart int_traefik` over a compose recreate to preserve attachments. This fragility is
the accepted price of per-app network isolation — don't "fix" it by consolidating networks; see
`D_network_per_project`, which also owns why `/etc/docker/daemon.json` carries enlarged address pools.

## Script index

A map, not a reference: each entry is a pointer plus what the thing is for. **Every script's own comment
header is the authority** on its arguments, env knobs and prerequisites — read that before invoking or
editing one, and put new technical truth *there*, not here.

| Script | What it is | Read before editing |
|---|---|---|
| `install` | one-time host setup, root, Ubuntu 24.04+; also prints the manual follow-up steps | `D_network_per_project` (address pools), `D_no_shared_cache` |
| `shepherd-build PROJECTID` | Jenkins-only: builds `shepherd/PROJECTID`, then execs `shepherd-cli restart` in `int_shepherd` | `D_no_shared_cache` (the cache flags are load-bearing), `D_poll_scm` |
| `shepherd-traefik-connect-networks` | repairs Traefik's `*.shepherd` attachments; idempotent. The model for what a header should be | `D_network_per_project` |
| `shepherd-clearcache` | weekly cron: prunes images and wipes the per-project caches | `D_no_shared_cache` (keep the cadence weekly) |
| `uninstall` | tears the box down; kills **all** containers, keeps `/etc/shepherd` | — |
| `docker-compose.yaml` | the three `int_*` admin containers and their Traefik labels | `D_docker_traefik` |

Two invariants that are easy to break from here: the `install`/`uninstall` pair must stay symmetric about
what it creates and removes (`/etc/shepherd` is the deliberate exception), and `install` is a *script whose
output is documentation* — a quoting slip in its trailing `echo` block silently costs the operator the
follow-up steps, so `bash -n install` after touching it.

## Conventions when editing

- All scripts are Bash with `set -e -o pipefail` (or `set -euo pipefail`). Keep that.
- Per-project caches must stay separate, and the separation must be enforced by flags on the *build
  command* — not by convention inside a project's `Dockerfile`. A shared cache breaks parallel builds
  (shepherd issue #3, why `install` pins `concurrentJenkinsBuilders: 1`) **and** lets one project
  pollute another's Maven/Gradle artifacts. See `D_no_shared_cache` in DECISIONS.md for the full
  reasoning, the rejected alternatives, and the two places this is currently only half-implemented.
- Naming is load-bearing and mirrored in shepherd-java: container `shepherd_PROJECTID` / `int_*` for admin,
  image `shepherd/PROJECTID`, network `PROJECTID.shepherd`, admin network `admin.int`. It is load-bearing
  because there is no scheduler or registry — the name *is* how a container is found again (`D_docker_traefik`).
- `mydomain.me` is the placeholder DNS domain throughout `docker-compose.yaml` and `install`; the operator
  replaces it (or adds `/etc/hosts` entries for a toy/debug setup).
- Traefik is pinned to `v3.6` (needs 3.6+ so its docker client can talk to newer docker daemons).
