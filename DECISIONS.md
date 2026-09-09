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
  **Superseded by `D_<slug>`**, and write the replacement fresh. (The predecessor's Kubernetes
  architecture is the canonical example of the shape.) The line: *refined or extended* → edit in place;
  *reversed after shipping* → tombstone + new entry.

**Relationship to `COMPARISON.md`.** That file owns the *requirements* (`R_` slugs) and sizes up the
off-the-shelf products this repo might be retired into; the requirement relaxations recorded there
(`R_no_kubernetes` accepting Swarm, `R_admin_interface` accepting a CLI, `R_restore_the_box` dropping
backup restore) are decisions about the *retirement*, and they stay in their `R_` box. An entry here
owns why *this repo* is built the way it is. Where the two meet, the `D_` links to the `R_` rather than
restating it.

---

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
