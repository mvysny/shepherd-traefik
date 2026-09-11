# Strip the app's cache mounts and re-mount them under a name Shepherd owns

**Status:** idea, not decided. Nothing implemented. On graduation this file is deleted — if accepted, as
a `D_` entry in `DECISIONS.md` (it would close `D_no_shared_cache`'s *Known gap*, so that entry's
`Status:` line moves too); if rejected, as a one-line why-not wherever someone would retry it.

**Raised:** 2026-09-09, alongside the since-rejected `generated-dockerfile.md`; its live counterpart is
now `herokuish-builds.md`.

## The problem it answers

Same hole as `herokuish-builds.md`: BuildKit's cache-mount `id` defaults to the mount `target`, so
`RUN --mount=type=cache,target=/root/.m2` puts **every project on the box in one directory**, and an app
that sets `id=` explicitly can address any other project's cache deliberately. `D_no_shared_cache` calls
this the *Known gap*: the layer cache is per project because the flags are on the build command, the
dependency cache is not because the mount is in the app's Dockerfile.

Where this idea differs: it **keeps the app's Dockerfile**. `herokuish-builds.md` fixes the hole by
taking the Dockerfile away from the project, which costs `R_java_docker` and re-onboards every hosted app
onto a new contract (`Procfile`, `.buildpacks`, `system.properties` instead of a `Dockerfile`) — an
acceptable ask, but a migration. This one changes nothing the project ships and needs no migration at all.

## The mechanism it copies, exactly as Dokku implements it

Verified in `dokku/dokku@main` on 2026-09-09.

`plugins/builder-herokuish/builder-build` does not let the app name its cache at all. Before the build it
ensures a docker volume **per app**:

```bash
fn-builder-herokuish-ensure-cache() {
  existing_cache="$("$DOCKER_BIN" volume ls --quiet \
    --filter "label=com.dokku.app-name=$APP" --filter label=com.dokku.builder-type=herokuish)"
  if [[ "$existing_cache" != "cache-$APP" ]]; then
    "$DOCKER_BIN" volume rm "cache-$APP" &>/dev/null || true
    "$DOCKER_BIN" volume create … "--label=com.dokku.app-name=$APP" "cache-$APP" >/dev/null
  fi
}
```

and then mounts it into the build container under a fixed, app-independent path:

```bash
CID=$("$DOCKER_BIN" container create … -v "cache-$APP:/cache" --env=CACHE_PATH=/cache … "$IMAGE" /build)
```

Three properties are what make it work, and all three are worth copying:

1. **The app never learns the cache's name.** It gets `/cache` and `CACHE_PATH`; the volume is
   `cache-$APP`, chosen outside. There is no string the app can write that reaches another app's cache.
2. **The platform decides the mount exists**, not the build recipe. Nothing in the repo can opt out,
   opt in twice, or point elsewhere.
3. **It is addressable for cleanup**: labelled `com.dokku.app-name`, so `dokku repo:purge-cache APP`
   clears exactly one app's cache — the per-project counterpart of `shepherd-clearcache`.

`builder-pack` (CNB) does the same through `pack`'s `--cache`/`--cache-image`, and Paketo documents the
host-volume form `pack build --volume $HOME/.m2:/home/cnb/.m2:rw`. Nixpacks reaches the same end for
BuildKit builds by generating `--mount=type=cache,id=<cache-key>-<dir>` with `--cache-key` supplied by
the caller (`src/nixpacks/builder/docker/utils.rs`). Every one of them puts the *namespace* on the
platform side and leaves only the *directory* to the app.

## The sketch: preprocess the Dockerfile in `shepherd-build`

The `docker build` in `shepherd-build` stops building the repo's file verbatim. Instead: copy the
Dockerfile to a temp dir, **strip every `--mount=type=cache,…` the app declared**, **re-inject the
platform's mounts** on the `RUN` lines, build that. The repo is untouched; the app's Maven/Gradle
config, base image, stages and commands all keep working.

Rendered, the app's

```dockerfile
RUN --mount=type=cache,target=/root/.m2 ./mvnw -B -DskipTests package
```

becomes

```dockerfile
RUN --mount=type=cache,id=PROJECTID-/root/.m2,sharing=locked,target=/root/.m2 \
    --mount=type=cache,id=PROJECTID-/root/.gradle/caches,sharing=locked,target=/root/.gradle/caches \
    ./mvnw -B -DskipTests package
```

The declared list of cache directories comes from the box side (a per-language default —
`/root/.m2`, `/root/.gradle/caches`, `/root/.gradle/wrapper` — that a project may extend but never
redirect). Under `herokuish-builds.md` that list is not ours at all: the Heroku buildpacks already point
`maven.repo.local` and `GRADLE_USER_HOME` inside the cache volume.

Two details that matter:

- **`sharing=locked` regardless of what the app asked for.** Builds are already serial
  (`concurrentJenkinsBuilders: 1`, shepherd issue #3), so this is belt-and-braces — but it is free and
  it survives that setting being raised later.
- **Injecting on *every* `RUN`** is simpler than working out which one resolves dependencies, and a
  cache mounted on a `RUN` that ignores it costs nothing. The exception to check: a cache mount *masks*
  whatever the image ships at that path for the duration of the `RUN`, so injecting `/root/.m2` into an
  image that ships a seeded `/root/.m2` would hide it.

### The alternative implementation: one buildx builder per project

No parsing at all. `shepherd-build` creates a per-project builder and builds through it:

```bash
docker buildx create --name "shepherd-$PROJECT_ID" --driver docker-container   # once, idempotent
docker buildx --builder "shepherd-$PROJECT_ID" build … --load -t "shepherd/$PROJECT_ID" .
```

Cache mounts live in **that builder's own buildkitd state**, so `id=` collisions across projects become
impossible by construction, whatever the app writes. `COMPARISON.md`'s mechanism table already lists
this as enforcing isolation, noting no *product* exposes it — but Shepherd is not a product here, it
runs the build itself, so it can.

Trade against the rewrite:

| | Dockerfile rewrite | Builder per project |
|---|---|---|
| Trusted component | **a parser** — the whole security property rests on it | none |
| Build flow | unchanged, keeps the daemon's builder and `--cache-to type=local` | needs `--driver docker-container`, and `--load` to import the image back into the daemon (extra time + disk per build) |
| Cost per project | zero | one buildkitd container, its own GC policy, its own layer cache on disk |
| Cleanup | `shepherd-clearcache` unchanged | must iterate builders (`docker buildx du/prune --builder …`), and reap builders for deleted projects |
| Failure mode | a mount the parser misses stays shared — silent | disk and memory grow with project count — noisy |

The airtight one is the builder; the cheap one is the rewrite. Worth deciding on the threat model: if
the concern is a *deliberately* poisoned cache, "a parser I wrote" is a weak boundary and the builder
wins. If it is the accident `D_no_shared_cache` actually documents — `mvn install` of a shared
`1.0-SNAPSHOT` — the rewrite is proportionate.

## What it costs / where it can bite

- **Parsing Dockerfiles is not a one-liner.** Line continuations, several `--mount` flags on one `RUN`,
  heredocs (`RUN <<EOF`), `--mount` written after the command, quoting, `# syntax=` directives pinning a
  frontend version whose flag set differs. A regex gets 95% and the missing 5% fails *open* — that is
  the argument for the builder variant, or for failing the build when the parser sees a `RUN --mount`
  shape it does not fully understand.
- **`type=cache` is not the only mount.** `--mount=type=secret` and `--mount=type=ssh` only expose what
  the build command passes, so they are harmless here; `--mount=type=bind,from=…` reads another *stage
  or image*, not the host, so likewise. Worth a decision to strip-or-keep rather than an oversight.
- **It does not stop a hostile build, only a shared cache.** Same limit as the generated Dockerfile: the
  build still runs the repo's own Maven/Gradle logic with that project's cache mounted. Memory/CPU
  limits on the build stay the only bound on what it does.
- **A project that relied on cross-project sharing gets slower**, which is the point, but the first
  rebuild after this lands re-downloads everything for every project — schedule it like a cache wipe.
- **Nothing here helps the *layer* cache**, which is already per project via `--cache-from`/`--cache-to`.
  This idea is only about the mount.

## Open questions

- Rewrite or builder-per-project — the table above is the trade, unresolved.
- If rewrite: fail the build on an unparseable `RUN --mount`, or strip conservatively and log? Failing
  is the safe default and turns a silent hole into a broken build for one project.
- Does the per-project cache get a size cap? A buildkitd GC policy is per builder, so the builder
  variant gets caps for free while the rewrite variant shares one buildkitd's GC across all projects —
  which is also where the "BuildKit evicts your cache after ~48h" trap in `COMPARISON.md` lives.
- Does `shepherd-clearcache` gain a per-project mode (the `dokku repo:purge-cache APP` equivalent), and
  does anything in the Web Admin need to call it?
- Is this a *replacement* for `herokuish-builds.md` or a complement? They disagree on who owns the
  Dockerfile; the honest reading is that this one is cheap, keeps `R_java_docker` and needs nothing from
  the hosted repos, while that one gives the Dockerfile up and gets the enforcement for free from a
  builder someone else maintains. Complementary at the edges either way: whichever is built, the other
  covers the projects it cannot.

## Related

- `D_no_shared_cache` (`DECISIONS.md`) — the *Known gap* this closes, and why `id` defaulting to
  `target` is the whole problem.
- `R_cache_isolation` and the mechanism table in *Build caches* (`COMPARISON.md`) — where the
  per-project-builder row and Dokku's per-app-volume row are already scored.
- `herokuish-builds.md` — the other half: give the build recipe up entirely to a buildpack builder that
  already names the cache per project. Its predecessor `generated-dockerfile.md` (Shepherd renders the
  Dockerfile itself) was rejected on 2026-09-11; the why-not is in `D_no_shared_cache`.
