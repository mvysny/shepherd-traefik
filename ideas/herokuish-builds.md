# Build with herokuish instead of the project's Dockerfile

**Status:** idea, not decided. Nothing implemented. On graduation this file is deleted — if accepted, as
a `D_` entry in `DECISIONS.md` (it closes `D_no_shared_cache`'s *Known gap*, so that entry's `Status:`
line moves too, and `R_java_docker` in `COMPARISON.md` has to be reworded), plus the new onboarding
contract in `README.md` where the `Dockerfile` recipe is today; if rejected, as a one-line why-not
wherever someone would retry it.

**Raised:** 2026-09-11, replacing the rejected `generated-dockerfile.md`. That idea had the right
diagnosis — *whoever writes the Dockerfile names the caches* — and the wrong remedy: it proposed that
Shepherd render a Dockerfile from a declared build spec, which is writing a buildpack from scratch and
buying base-image policy with it. This idea keeps the diagnosis and takes the remedy off the shelf.

## The problem it answers

`D_no_shared_cache`'s *Known gap*: the layer cache is per project because the flags are on the build
command, but the dependency cache mount is declared in the app's `Dockerfile`, BuildKit's `id` defaults
to the mount `target`, and so every project writing `--mount=type=cache,target=/root/.m2` lands in one
box-wide directory. `mvn install` of somebody else's `1.0-SNAPSHOT` is the accident; a poisoned artifact
placed where another project reads it is the attack.

No flag on `docker build` can remap a cache mount the repo declared. The only ways out are to take the
Dockerfile away, or to rewrite it before building (`enforce-cache-mounts.md`).

## The sketch

`shepherd-build` stops running `docker build` and runs the herokuish image instead. The repo ships no
`Dockerfile` at all; the build container mounts a cache volume whose name is Shepherd's:

```bash
docker volume create "shepherd-cache-$PROJECT_ID"        # idempotent

docker run --name "build-$PROJECT_ID" \
  -v "$PWD:/tmp/app:ro" \
  -v "shepherd-cache-$PROJECT_ID:/cache" -e CACHE_PATH=/cache \
  -v "/etc/shepherd/build-env/$PROJECT_ID:/tmp/env" \
  -m "$BUILD_MEMORY" --cpus 2 \
  gliderlabs/herokuish:latest-24 /build

docker commit \
  --change 'ENV PORT=8080' \
  --change 'EXPOSE 8080' \
  --change 'CMD ["/start","web"]' \
  "build-$PROJECT_ID" "shepherd/$PROJECT_ID:latest"
docker rm "build-$PROJECT_ID"
```

Then step 2 is unchanged: `shepherd-cli restart` inside `int_shepherd`.

Three things make this fit *this* repo rather than being a Dokku feature we admire from outside:

- **The image name and the run side don't move.** The output is still `shepherd/$PROJECT_ID`, and
  `docker commit --change` bakes the entrypoint and the port into it, so shepherd-java keeps doing a
  plain `docker run` of an image. **Check before believing it:** that shepherd-java does not pass its own
  command, and that `PORT` reaching the app as 8080 is all the port contract needs — herokuish's default
  is 5000, and `R_java_docker`'s "listening on 8080" becomes an env var rather than a hard-coded port.
- **Build limits get *better*, not worse.** `docker build -m/--cpu-quota` becomes `docker run -m/--cpus`
  on the actual build container.
- **`BUILD_ARGS` gets simpler.** The Vaadin offline key stops being `--build-arg` plumbing into the app's
  `ARG`/`ENV` and becomes one file in an env directory, which the buildpack exports before Maven runs.

The cache property falls out by construction: the app never learns the volume's name, and — having no
Dockerfile — has no syntax in which to open a different mount. That is the whole point.

## Why this rather than templating our own Dockerfile

Because it is already built, and because it has been read, run and priced next door. Nearly everything
below is verified in **`../shepherd2-dokku`** — the successor spike, where `D_builder` (2026-09-10)
decided exactly this for a Dokku box, `RESEARCH.md` → *The herokuish builder* carries the `[src]` claims,
and `ideas/vaadin-build-under-herokuish.md` carries what is still open. Read those before designing
anything here; do not re-derive them.

The facts that matter for us:

- **The Heroku Java buildpack points `-Dmaven.repo.local` inside `$CACHE_DIR`**, and the Gradle buildpack
  puts the whole `GRADLE_USER_HOME` there — so dependencies, the Gradle build cache, the wrapper
  distribution and even the JDK come back warm on the next poll, per project, enforced.
- **Its default Maven goals are `clean dependency:list install`** — the `mvn install` `D_no_shared_cache`
  is afraid of is what it does by default, and it is harmless precisely because the local repo it
  installs into is that project's own volume.
- **The build runs unprivileged** (herokuish chowns the build paths and runs `bin/compile` as
  `herokuishuser`), in a container with no Docker socket and no special network — where `docker build`
  today runs the repo's build as root.
- **The buildpack must be *named*, never detected.** herokuish tries `nodejs` before `java`, and Vaadin's
  own guidance is to commit `package.json`, so a stock Vaadin repo builds as a Node app unless something
  says otherwise. A one-line `.buildpacks` in the repo, or `BUILDPACK_URL` from the box side, fixes it.
  Naming it over the network gives up the version pinned in the image, so pin the ref: `…git#v49`.
- **`gliderlabs/herokuish:latest-24` is `FROM heroku/heroku:24-build`** and is actively maintained
  (v0.11.17, 2026-09-09; commits through 2026-09-10) — but it is the *classic* v2a buildpack line, and
  Heroku itself has moved to CNB. `pack` is the escape hatch, and the same shape of decision.

## What it costs here, specifically

- **`R_java_docker` as worded is given up** — "no language detection or buildpack magic is wanted; the
  `Dockerfile` is the contract". Same collision the rejected idea had; it is the decision, not a
  side-effect. The difference is only that the replacement contract is somebody else's to maintain.
- **The onboarding contract changes shape, and that is all it does.** A hosted repo needs a `Procfile`,
  usually a `system.properties`, and a `.buildpacks` — where today it needs a `Dockerfile` at its root
  that builds with `docker build -t x .` and runs with `docker run -p8080:8080 x`. That the repos are not
  ours changes nothing: they already comply with a Shepherd-specific contract or they are not deployable
  here, exactly as an app must carry a `fly.toml` to deploy to Fly.io. `D_poll_scm` is about not having
  *admin rights* on those repos (no webhooks), not about being unable to ask for files in the tree.
  What is real is the **one-time migration**: every app already on the box needs the new files committed
  before cutover, and the two contracts cannot both be live if the enforcement is to mean anything.
- **A project whose build genuinely needs a Dockerfile has nowhere to go.** Keeping the Dockerfile path
  as an escape hatch re-opens the cache hole for exactly the projects that take it — which is the
  argument for `enforce-cache-mounts.md` covering that tail rather than a second-class path.
- **The Vaadin frontend half is not solved by this** (nor by anything else): the frontend is driven by
  Maven, so it is invisible to the node buildpack that would otherwise cache it. Vaadin 24.1+'s
  pre-compiled production bundle removes the problem for apps with no custom frontend; for the rest, see
  the sibling's `ideas/vaadin-build-under-herokuish.md` for the `npm_config_cache` / `user.home` levers
  and why a `heroku/nodejs` + `heroku/java` multi-buildpack is not the answer.
- **Half the build apparatus becomes dead weight.** `--cache-from`/`--cache-to type=local`, the
  containerd-snapshotter `install` enables for it, and `shepherd-clearcache`'s wipe of
  `/var/cache/shepherd/docker` all stop applying; housekeeping becomes pruning *volumes*, per project.
  And nothing garbage-collects a Docker volume, so the weekly purge changes shape rather than going away.
- **Images get fatter and less legible.** `docker commit` of a build container ships the checkout as the
  build left it plus the JDK (~200 MB) and the buildpack's `.profile.d`; there is no multi-stage build
  and no layer reuse between projects.

## Open questions

- **Is this worth building *here* at all?** The successor spike already does it, via Dokku, where the
  per-app volume and `repo:purge-cache` come free. Doing it in this repo means writing and owning the
  `/build` + `docker commit` glue that Dokku's `builder-herokuish` *is* — and throwing it away if this
  box is retired into that one. The honest framing: this is the cheap way to close the gap **if
  shepherd-traefik outlives the Dokku move**, and otherwise it is duplicated work. That is the fork to
  settle first, before any of the below.
- Box-wide, or per project alongside the Dockerfile path? Box-wide is the only version that actually
  enforces anything; per-project makes it a feature, not a boundary.
- The descriptors live **in the repo** — that is the contract, and onboarding refuses a repo without
  them. The narrower question that stays open: should the *buildpack name* also be settable box-side, as
  the repair for a repo that chose wrongly or one mid-migration? herokuish reads `BUILDPACK_URL` from the
  env directory we mount, so it costs one file and no fork.
- `docker commit` versus `herokuish slug-export` + an import step: commit is one command and keeps the
  image name; slugs would let the runtime image be chosen, at the cost of more moving parts.
- Does `shepherd-clearcache` gain a per-project mode (the `repo:purge-cache APP` equivalent), and does
  anything in shepherd-java's Web Admin need to call it?
- Classic herokuish now, or `pack`/CNB directly? Heroku's own direction is CNB, and our cache property
  survives either (`pack --cache`), but `pack` is another binary to install and pin on the box.

## Related

- `D_no_shared_cache` (`DECISIONS.md`) — the *Known gap* this closes, why `id` defaulting to `target` is
  the whole problem, and the one-line why-not for the Dockerfile-renderer road this idea replaced.
- `R_cache_isolation`, `R_java_docker`, `R_build_dockerfile`, and *What each product accepts as build
  input* (`COMPARISON.md`) — where Dokku's per-app `cache-$APP` volume is already scored.
- `enforce-cache-mounts.md` — the other half of the same problem, and the alternative that keeps the
  Dockerfile: rewrite the app's cache mounts rather than take its build recipe away.
- `../shepherd2-dokku` — `D_builder`, `RESEARCH.md` → *The herokuish builder*, `README.md` → *Rehearse
  the build locally*, `ideas/vaadin-build-under-herokuish.md`. The sourcing for everything above.
