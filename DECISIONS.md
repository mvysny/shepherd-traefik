# DECISIONS.md

A living record of the design decisions behind Shepherd-Traefik — especially the *roads not taken*.
It exists because the scripts and their comments record *what* the box does and `README.md` records
*how to operate it*, but the rationale for the alternative that was rejected has nowhere else to live,
and that rationale is what a future maintainer (or agent) actually needs before "simplifying" something
load-bearing.

It is the *why-we-chose* record. It is **not** the operator guide (`README.md`), the
what-you-must-not-break orientation (`CLAUDE.md`), the per-script technical truth (the comment header at
the top of each script), or the which-product-should-replace-us survey (`COMPARISON.md`). When a fact
belongs in one of those, put it there and link — see *Documentation targets* in `CLAUDE.md`.

**Format.** One entry per decision. The ID is a slug, not a number: `D_` (says "this is a decision")
plus a 1–4-word hint at the subject (`D_no_shared_cache`), so a reference carries meaning on its own —
a running counter would not, and renumbering silently invalidates every existing reference.
**Underscores throughout, never hyphens**: the id has to be one *token*, so that vim's `w` / `*` /
`ciw` and `grep -w` act on the whole thing rather than on a fragment. Backtick it in prose, both
because some downstream Markdown parsers italicise intraword `_` and because a backticked id is
copy-pasteable into a search. The `(date)` on the heading is *decided* provenance, not a log position;
git owns the edit history, so don't narrate how an entry used to read. Keep each entry tight: context,
the decision, the alternatives rejected and why, and the consequences a future maintainer would trip
over. A decision is worth logging the moment it's *made* — implementation can lag, and the `Status:`
line says which.

**Entries are mutable — edit in place, don't append addendums.** Each entry is the single coherent home
for one *live* decision; keep it current as the decision is refined or extended. Two things that does
*not* license:

- **The roads-not-taken stay.** "We chose X, rejected Y because Z" is live content of the current
  decision, not stale history — never edit it away. It is the most valuable thing in the file.
- **A reversed *shipped* decision forks a tombstone, it is not overwritten.** When something was
  deployed and then thrown away, leave the old entry as the scar, set its `Status:` to
  **Superseded by `D_<slug>`**, and write the replacement fresh. `D_kubernetes` → `D_docker_traefik`
  is the worked example of the shape. The line: *refined or extended* → edit in place;
  *reversed after shipping* → tombstone + new entry.

**Only decisions already made.** An entry records a position this project has actually taken — shipped,
or accepted and awaiting implementation (that is what `Status:` is for). Speculative features, ideas
and "we might one day" belong nowhere in this file; a TODO is not a decision. The one adjacent case
that *is* in scope is a rejected alternative, which is a road not taken **within** a decision already
made, not an open question.

**Relationship to `COMPARISON.md`.** That file owns the *requirements* (`R_` slugs) and sizes up the
off-the-shelf products this repo might be retired into; the requirement relaxations recorded there
(`R_no_kubernetes` accepting Swarm, `R_admin_interface` accepting a CLI, `R_restore_the_box` dropping
backup restore) are decisions about the *retirement*, and they stay in their `R_` box. An entry here
owns why *this repo* is built the way it is. Where the two meet, the `D_` links to the `R_` rather than
restating it.

---

## D_kubernetes — The predecessor ran on Kubernetes (superseded, 2024-11-12)

**Status:** **Superseded by `D_docker_traefik`.** Tombstone, kept because it shipped: this is what
[Vaadin Shepherd](https://github.com/mvysny/shepherd) actually ran in production before this repo
existed.

**What it was.** The same product — build git repos, host them at `PROJECTID.<domain>` behind an
auto-TLS proxy on one box — implemented on Kubernetes, which supplied the scheduler, the per-app
network isolation, the ingress and the restart-on-crash behaviour.

**Why it was thrown away.** Kubernetes spends a substantial share of a small box's CPU on its own
upkeep, and it makes the system far more complicated than a single-host demo farm needs. The
platform's own maintenance became the dominant cost of running it. See `D_docker_traefik` for what
replaced each of the four things k8s was supplying.

**Kept as a scar, not as an option.** Do not read this entry as "Kubernetes is on the table with
caveats" — the rewrite is the decision. `R_no_kubernetes` in `COMPARISON.md` carries the requirement
this generated, including the one relaxation made since (Docker Swarm is acceptable *in a replacement
product*; that is not a licence to introduce Swarm here).

## D_docker_traefik — Plain Docker + Traefik on a single host (2024-11-12)

**Status:** Accepted 2024-11-12 (this repo's first commit is the decision being acted on);
implemented 2025-04. Supersedes `D_kubernetes`.

**Context.** Rewriting the predecessor (`D_kubernetes`) meant re-supplying, from off-the-shelf parts,
the four things Kubernetes had been doing: scheduling containers, isolating them from each other,
routing https to them, and restarting them when they die.

**Decision.** Use plain Docker on one Linux box, and answer each of the four with something already
in the box:

- *scheduling* — there is none. Containers are started directly (`docker run` by shepherd-java) and
  found again by name. This is what makes the naming contract load-bearing (see *Conventions when
  editing* in `CLAUDE.md`): `shepherd_PROJECTID` / `shepherd/PROJECTID` / `PROJECTID.shepherd` **is**
  the registry, so a rename is a production incident rather than a refactor.
- *isolation* — one Docker network per app; see `D_network_per_project`.
- *routing* — Traefik, configured entirely through Docker labels and the Docker socket, so adding an
  app requires no proxy config file and no proxy reload.
- *staying up* — Docker's own restart policy (`restart: always` for the three admin containers).

**Alternatives rejected.**

- *Kubernetes, including a lighter distribution of it.* The rejection is of the model, not of one
  distribution's weight: what cost the predecessor was the ongoing operation of a cluster on a box
  hosting demos. `D_kubernetes` has the detail.
- *A hand-written nginx/Apache vhost per app.* Cheaper to understand, but every new app becomes a
  config-file edit plus a reload, and https means owning the whole ACME flow per vhost. Traefik's
  Docker provider plus one wildcard cert removes both. (Dokku's default is exactly this shape, and
  `COMPARISON.md` sizes up what it costs there.)
- *Publishing app ports on the host instead of proxying.* Ports collide, apps end up reachable
  directly, and there is no place left to terminate TLS.

**Consequences.**

- **No build system and no test suite in this repo, on purpose.** The deliverable is Bash plus a
  `docker-compose.yaml`, deployed by `git pull` into `/opt/shepherd-traefik`; "running it" means
  executing the scripts on a box with Docker. That is also why git *is* the changelog here.
- **Traefik discovering apps over the Docker socket is not the same as being able to reach them** —
  it must also share each app's network. That is the one sharp edge this design has;
  `D_network_per_project` and `shepherd-traefik-connect-networks` own it.
- Everything runs on one machine by choice; there is no story for a second node, and `R_single_host`
  in `COMPARISON.md` records that as a requirement rather than a limitation.

## D_network_per_project — One Docker network per app, and enlarged address pools (2025-04-17)

**Status:** Accepted; implemented in `install` (`default-address-pools`) and in shepherd-java, which
creates `PROJECTID.shepherd` per app.

**Context.** Every hosted app is a container on one shared Docker daemon, and the apps are mutually
untrusted — they are other people's example projects and addons (`R_periodic_rebuild`). Each listens
on port 8080. Two admin containers on the same daemon (`int_jenkins`, `int_shepherd`) hold the Docker
socket and the Jenkins credentials, so "any app can reach any other container" is not an acceptable
default.

**Decision.** One private bridge network per app, named `PROJECTID.shepherd`, with nothing else on it
but the app and Traefik. The three admin containers get their own network, `admin.int`, which no
hosted app is ever attached to.

**Alternatives rejected.**

- *One shared network for all apps.* Simplest, and it would delete the entire
  `shepherd-traefik-connect-networks` problem — but every app could then reach every other app's
  8080, and reach the admin containers. The gotcha below is the price paid for that isolation, and
  it is worth paying.
- *Docker's default `bridge`.* Same objection, plus no DNS-by-container-name.

**Consequences.**

- **Docker's default address pools cap the box at ~29 networks**, i.e. 28 projects once `admin.int`
  is taken. `install` therefore writes enlarged `default-address-pools` into
  `/etc/docker/daemon.json`. This is a daemon-wide setting applied at *daemon start*, so it has to be
  in place before the 29th network is wanted, not after; and `uninstall` removing that file reverts
  it.
- **Traefik can only route to a container whose network it shares**, so it must be attached to every
  `*.shepherd` network — and those attachments are dropped by a compose recreate, leaving routers
  visible in the dashboard while requests 502. `shepherd-traefik-connect-networks` repairs it and its
  comment header is the standalone account; the operator-facing recipe is in `README.md`. Prefer
  `docker restart int_traefik` over a compose recreate.

## D_poll_scm — Jenkins polls git on a schedule; no push-to-deploy (2024-11-12)

**Status:** Accepted; implemented as Jenkins poll-SCM jobs that call `shepherd-build PROJECTID`.

**Context.** Shepherd hosts a demo farm: Vaadin example projects, starters and addons, most of them
in repos the operator does **not** own or have admin rights on.

**Decision.** CI polls. Jenkins holds one job per project on a poll-SCM schedule, and on a detected
change runs `shepherd-build PROJECTID`, which builds the image and restarts the container. Nothing is
triggered by the act of pushing.

**Alternatives rejected.**

- *Webhooks from the upstream repos.* Needs admin rights on every hosted repo to install the hook —
  precisely what a demo farm of other people's projects does not have.
- *Git-push-to-deploy (the Dokku/Heroku model).* Requires each app's canonical remote to be *this*
  box, so hosting someone else's repo would mean maintaining a mirror and a push mechanism for it.
  That inverts who has to cooperate.

**Consequences.**

- **Deploy latency is the poll interval**, and that is accepted.
- **Rebuilds happen even when nothing changed in the app** — new base images, moved dependency
  versions and upstream branch updates all get picked up for free. This is a feature, not overhead.
- **It is what makes build caching load-bearing.** A scheduled rebuild that re-downloads the whole
  Maven/Gradle dependency tree is the problem `D_no_shared_cache` exists to solve; without polling
  there would be far less pressure on the cache.
- **Jenkins is the heaviest single component on the box**, and it exists for this one requirement.
  `R_periodic_rebuild` in `COMPARISON.md` tracks what each candidate replacement would offer instead
  — for several of them, one crontab line.

## D_no_shared_cache — Build caches are per project; a shared cache is a no-go (2026-09-09)

**Status:** Accepted 2026-09-09; **partially implemented.** The *layer* cache is isolated per project
(`shepherd-build` passes `--cache-from`/`--cache-to type=local,…=/var/cache/shepherd/docker/$PROJECT_ID`,
in place since [shepherd issue #3](https://github.com/mvysny/shepherd/issues/3)). The *dependency* cache
mounts that `install` tells projects to put in their `Dockerfile`s are **still shared box-wide** — see
*Known gap* below.

**Context.** Every hosted project is a JVM app (Vaadin-Boot or Spring-Boot) built by `docker build` on a
Jenkins poll schedule, so without a cache each rebuild re-downloads its entire Maven/Gradle dependency
tree from the internet. Two mechanisms answer that, and they are *not* interchangeable:

- the **layer cache** (`--cache-from`/`--cache-to type=local`), which is *content-keyed*: an entry's key
  is parent digest + instruction + digest of the copied files, so a hit is only possible when the inputs
  were identical, and the flags live on the *build command*;
- a **cache mount** (`RUN --mount=type=cache,target=/root/.gradle …`), which is a mutable directory
  living in the builder's own state, declared inside the *app's* `Dockerfile`, and **not** exported by
  `--cache-to`.

**Decision.** Caches are **per project**, and the isolation is enforced by the *platform* — by flags on
the build command that a project cannot opt out of — not by convention inside a `Dockerfile` we may not
control. Concretely: one `type=local` cache directory per project id, plus
`concurrentJenkinsBuilders: 1` so two builds never touch one cache at all. `install` enables
`containerd-snapshotter` precisely because `type=local` cache export needs it.

**Why a shared cache is a no-go.** Two distinct hazards, which are easy to conflate — and only the first
one has a cheap fix:

1. **Corruption (concurrent writers).** Maven has no locking on its local repository and fails with
   random errors, potentially corrupting it; Gradle writes lock files and then times out. This is
   shepherd issue #3, and `sharing=locked` / serial builds fix it.
2. **Pollution (one project's artifacts reaching another's build).** This one has no cheap fix, and
   Docker documents the semantics as expected behaviour rather than a bug: *"Your build should work with
   any contents of the cache directory as another build may overwrite the files."* Three facts make that
   sharper than it sounds:
   - **`id` defaults to the value of `target`**, so every project writing
     `--mount=type=cache,target=/root/.m2` lands in the *same* directory. Nothing scopes a mount per
     app, per image or per repo by default.
   - **Cache-mount contents are not part of any cache key** — unlike the layer cache. The mount is
     unkeyed, mutable shared state, writable as root by every build on the box.
   - **Maven never re-verifies what is already in the local repository.** Checksums are checked at
     *download* time; an artifact already present is used as-is. Hence `dependency:purge-local-repository`
     exists as the manual escape hatch.

   The path that bites first is not malice but **`mvn install`**: multi-module builds install their own
   artifacts into the shared local repo, and a demo farm is full of forks of the same starter — so two
   projects legitimately share `com.example:my-app:1.0-SNAPSHOT` and the second silently resolves the
   first one's jar with a green build. Gradle's module store is content-addressed
   (`caches/modules-2/files-2.1/…/<sha1>/…`) so a swap is harder, but a mount of the whole `/root/.gradle`
   also shares `init.d/` (an init script dropped there runs in *every* later Gradle build) and
   `wrapper/dists/` (verified only when `distributionSha256Sum` is set, which starters usually omit).

   Proportionately: a hostile repo already runs arbitrary code as root in its own build and ships an
   image that runs on this box, so it owns *itself* either way. What a shared cache adds is **lateral
   movement** into every other project's artifacts — and Shepherd deliberately hosts repos it does not
   own (example projects, addons; see `R_periodic_rebuild`).

**Alternatives rejected.**

- *One shared cache mount plus `sharing=locked`.* Fixes corruption only. Serialising writers says
  nothing about *what* a writer leaves behind, and the doc-level appeal of "one line in the Dockerfile"
  is exactly what makes it unenforceable.
- *A per-project `id=` inside each Dockerfile.* Cooperation, not enforcement — a careless or hostile
  Dockerfile uses another id, or none. Kept as a **collision-avoidance convention for the projects that
  are ours** (that is the remediation in *Known gap*), never as a boundary.
- *One buildx builder per project.* This *does* enforce mount isolation, since the mount lives in the
  builder's state — but each builder keeps its own layer cache, so disk use balloons. Rejected on cost,
  not on correctness; revisit if the per-project `id=` convention proves unreliable.
- *A Maven repository proxy (Nexus et al.).* The classic CI answer, and it sidesteps pollution
  entirely: each build gets a clean local repo and downloads over LAN from a read-through mirror of
  Central. Rejected for now because it needs each app's `settings.xml`/`build.gradle` to point at it —
  cooperation again — and it cannot be forced at the network level, since Central is https and a
  transparent MITM would require a trusted CA inside someone else's build container. It also adds a
  container to a box whose whole selling point is that it is small. **The option worth revisiting** if
  the farm ever grows past what layer-cache-only builds can serve.
- *No caching at all.* Correct and safe, but a scheduled rebuild that re-downloads the internet is the
  problem this decision exists to solve.

**Consequences.**

- The cache flags must stay on the **build command** in `shepherd-build`. A cache mount in a
  `Dockerfile` is a *complement*, never a substitute — it is the half we cannot enforce.
- Per-project directories grow without bound, hence `shepherd-clearcache`. Keep that purge **weekly**:
  the purge cadence is the cache's real lifetime, and a nightly purge defeats the requirement it exists
  to support. (BuildKit also runs its own GC over cache mounts, reportedly evicting unused entries after
  ~48h, so a mount is not a durable store either.)
- **This is the one property no off-the-shelf PaaS reproduces**, which makes it a standing cost of
  retiring this repo — see `R_cache_isolation` in `COMPARISON.md`. None of Coolify, Dokploy, Dokku or
  CapRover lets you template `--cache-to`/`--cache-from` per app; Dokku's `docker-options … build` looks
  like the exception but those are *container* options handed to builders, not `docker build` flags.

**Known gap (recorded 2026-09-09; not yet fixed).** The decision is only half implemented on the box
today:

- `install`'s documented recipe tells projects to write
  `RUN --mount=type=cache,target=/root/.gradle --mount=type=cache,target=/root/.vaadin ./gradlew …`,
  and with the default `id` **every project shares those two directories**. The per-project `type=local`
  dirs isolate the layer cache only; they never carry mount contents. Serial builds keep it from
  *corrupting*, but not from *polluting*.
  Remediation: pass the project id into the build (`--build-arg CACHE_ID=$PROJECT_ID` from
  `shepherd-build`) and change the documented recipe to
  `--mount=type=cache,id=gradle-$CACHE_ID,target=/root/.gradle`. Variable expansion in `--mount` options
  is reported to work on current BuildKit but was long broken
  ([moby/buildkit#2909](https://github.com/moby/buildkit/issues/2909)) — verify on the box before
  relying on it, and treat a silently-unexpanded `id` as the failure mode to check for.
- `README.md`'s Maven+WAR recipe has **no cache at all**, and `COPY . /app/` before `RUN mvn … package`
  means any source change invalidates the layer — so Maven projects re-download their whole dependency
  tree on every commit. Remediation is the standard two-layer split (`COPY pom.xml` +
  `mvn dependency:go-offline`, then `COPY src`), with a per-project `id=` cache mount on `/root/.m2` if
  the above expansion works.

**Sources.** [shepherd issue #3](https://github.com/mvysny/shepherd/issues/3) (the parallel-build
corruption this started from), [buildx mount caches vs. per-project
`type=local`](https://mvysny.github.io/docker-build-cache/), [BuildKit `RUN --mount=type=cache`
reference](https://docs.docker.com/reference/dockerfile/#run---mounttypecache) — `id` defaults to
`target`, the `sharing=shared|private|locked` modes, and the quoted "another build may overwrite the
files", and ["buildkitd's own GC evicting unused
entries"](https://www.loopwerk.io/articles/2026/docker-buildkit-cache-coolify/). How the off-the-shelf
products fare against this decision is `R_cache_isolation` in `COMPARISON.md`, which carries the
per-product citations.
