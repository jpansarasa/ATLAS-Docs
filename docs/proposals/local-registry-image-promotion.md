# Separating image BUILD from image DEPLOY: sha12 tags in a loopback registry, `latest` moved by one promote script

Status: APPROVED TO BUILD 2026-09-26 by the user, verbatim in the supervisor session: "build the registry plan".
Design choices, also verbatim: "Move the tags, deploy latest. Should work for both new deployments and rollbacks"
and "use GitHub releases for the move log". Steps land in SEQUENCING order (docs/README.md §Curation policy).
Author measurements: 2026-09-26, mercury, origin/main `10398312`. Revision 3 (same day) rewrites the plan to the
design the user chose; its review round 1 added GC fail-closed, adopted-* rollback, the failed-move alert and
the build/deploy instruction supersessions; round 2 added tag-keyed image identity (`{tag, sha12, digest}`), the
`pending` spool record, GC under the lock, the two-call freshness gate and the host-scoped local GC. Raw notes: `/tmp/sentinel-remediation/registry/` (host-local, not durable).

## VERDICT

**BUILD it: a loopback registry holding immutable-by-convention `{image}:{sha12}` tags, and a `latest` tag that
only `promote.sh` moves. Deploy and rollback are the same command. Every move gets exactly one GitHub Release.**

The user's decision, verbatim from the session: "An image can have multiple tags. latest is just convenience but
is greatly simplifies deployment. Move the tags, deploy latest. Should work for both new deployments and
rollbacks". And on the record: "use GitHub releases for the move log". Revision 2 recommended a digest pin in
git (option A). That recommendation is withdrawn, and A is now in the appendix as considered, not chosen.

The defect is a MUTABLE POINTER THAT THE BUILD WRITES. Today every `build.sh` and every `deploy.yml` build task
writes `{svc}:latest`, and every deploy path reads it. B keeps a movable pointer, but takes the write away from
the build:

1. **Build** pushes `127.0.0.1:5000/{image}:{sha12}` and nothing else. sha12 is the last commit that touched the
   image's inputs. The build never writes `latest`, and never overwrites an existing sha12 tag (IMAGE IDENTITY).
2. **Promote** = `promote.sh <image> <sha12> --reason "..." --by <who>`. It moves `latest` to that sha12 in the
   registry, pulls it, runs the EXISTING scoped ansible restart, runs the freshness gate and smoke, then records
   the move as a GitHub Release. **Rollback is the same command with an older sha12.**
3. **Compensations for what B gives up** (a hold is no longer visible in review):
   - the move log (one release per promote);
   - a drift metric per image (commits on main touching the image's inputs since the sha12 `latest` names);
   - a digest check that the running container matches the registry's `latest`, so a moved tag without a
     restart, or a move that bypassed the script, is detected.

**What B costs, stated once:** a hold (D-19 holding #1091 today) lives in no reviewed file. Nothing stops an
agent from promoting secmaster at the main tip. The compensations make that act VISIBLE: the drift metric shows
the held commits before the move, `promote.sh` prints them before moving, and the release lists them as SHIPPED
afterwards. None of them PREVENTS it. The user accepted that trade for one-command deploys and rollbacks.

## THE PROBLEM, MEASURED

1. **Nothing records what is running.** `nerdctl image inspect secmaster:latest` carries one label,
   `org.opencontainers.image.version=24.04`, which the Ubuntu base image sets. No image carries a git revision.
   `container inspect secmaster .Image` -> `docker.io/library/secmaster:latest`: that names a pointer, not a
   build. Prometheus has no series that could say it either. The only metric names matching
   `container_info|image|build_info|revision|deploy|git` are `container_info` (name + health only),
   `containerd_*`, and the `*_build_info` of prometheus and node_exporter. There is no `target_info`.
2. **Merged-but-not-deployed is the normal state, and nothing shows it.** 22 of the 25 first-party images have at
   least one non-`.md` commit to their build inputs on main after their local `:latest` was built (revision 1,
   `drift.txt`). This is an approximation: `--since` reads committer time.
3. **The held change, re-derived for this revision.** The live `secmaster` runs the `:latest` built
   2026-09-20T14:29Z. `git log --since=2026-09-20T14:29Z origin/main -- SecMaster Events ':(exclude)*.md'` returns
   two commits (re-run at `10398312`):
   - `6a77daf3` (#1091, D-19, held on purpose);
   - `0382936d` (#1114, NU1903 dependency bumps).

   Any SecMaster build at the tip ships both.
4. **A second live instance.** `macro-substrate-migrator:latest` was rebuilt 2026-09-26T02:48Z. The container last
   ran 2026-09-16T11:00Z, from a rootfs equal to `:autofix-prev`, not `:latest`. The next `atlas.service` start
   runs the undeployed `:latest` and applies its migrations. The ONLY cause is that the build writes the pointer
   the boot reads, and that is exactly what B removes.
5. **People already hand-roll immutable tags.** The 22 first-party repositories hold 19 non-`latest` tags: 13 made
   by hand (`secmaster:pre-d13`, `sentinel-collector:pre-d34`, ...) and 6 `:autofix-prev`. Each is one level
   deep, has no name linking it to a commit, and has no GC.
6. **The durable copy is missing.** `/var` is 94% full (16G free, `df`), and `/var/lib/containerd` holds 164G of
   unsnapshotted ext4. `nvme-fast/containers` is 96K used, 697G available, mounted at
   `/opt/ai-inference/containers`, and empty. It has 49 snapshots, taken every 15 minutes by
   `/etc/cron.d/zfs-auto-snapshot`.

## THE DESIGN

### Registry

- **Unit.** A new `atlas-registry.service` (step 0: `deployment/artifacts/atlas-registry.service.j2`) runs
  `nerdctl run --rm --network host -e REGISTRY_HTTP_ADDR=127.0.0.1:5000
  -v /opt/ai-inference/containers/registry:/var/lib/registry registry@sha256:<pinned>`, with
  `After=zfs-mount.service` and an `ExecStartPre=mountpoint -q` refusal (no `.mount` unit exists to order on).
  - It is NOT a compose service. nerdctl 1.7.7 discards `depends_on`, so compose could not start it first.
  - Port 5000 is free (`ss -ltn`, 0 listeners). **Host network, not `-p 127.0.0.1:5000:5000`** (revision 3 said
    the port map): a CNI port map is a DNAT to the container IP, which every bridged container can reach
    directly -- measured on a throwaway, a bridge container read `/v2/` at `<container-ip>:5000`. Bound to
    127.0.0.1 on the host network, only the host's loopback reaches it (step0-measure.md M6, below).
- **Plain HTTP for that one host only.** Add `/etc/containerd/certs.d/127.0.0.1:5000/hosts.toml`. Never set a
  global `insecure_registry = true`: `/etc/nerdctl/nerdctl.toml` has `false`, and flipping it would relax TLS
  for docker.io too. Measured: nerdctl already speaks HTTP to 127.0.0.1 with NO hosts.toml, but a hosts.toml,
  when present, is authoritative (one naming `https://` broke the pull), so the file pins the scheme explicitly.
- **Every DIGEST read sends an `Accept` header** naming the OCI and docker v2 manifest types. registry 2.8.3
  answers a bare HEAD with 200 and the digest of an on-the-fly schema1 conversion (measured: `8509e6cc...` bare
  vs `c7219fdc...`, the pushed and pulled digest, with Accept). FRESHNESS and promote.sh stages 1 and 4 depend on it.
- **Deletes.** `REGISTRY_STORAGE_DELETE_ENABLED=true`, so GC can run (step 5).

### IMAGE IDENTITY and the overwrite refusal

- **Tag = `{image}:{sha12}`.** sha12 is `git log -1 --format=%H -- <inputs>` at the build commit, cut to 12
  characters. The width is fixed, because `%h` grows with the repo.
- **Build context comes from the COMMIT**, via `git archive <sha> -- <inputs>`. A dirty INPUT exits 2 and names
  the path. Every image is labelled `org.opencontainers.image.revision=<full sha>`. That label is how the drift
  timer and `promote.sh` learn which commit a digest came from.
- **Skip.** If HEAD `/v2/<image>/manifests/<sha12>` returns 200, `build-image.sh` prints
  `exists, skipped <digest>` and exits 0. It does not build at all, so the normal path never produces
  second bytes for a sha12.
- **Refusal (registry:2 overwrites a tag silently, so the script must refuse).** Only `--rebuild` (the tag is
  lost) and `--rebase` (a base-image refresh: 0 of 30 Containerfiles pin `FROM` by digest) build again for a
  sha12 that exists. Both write a NEW tag, `{sha12}-r{N}`:
  - N = 1 + the highest `-r` suffix in `/v2/<image>/tags/list`;
  - before the push, HEAD on the new tag must return 404, or the script exits 3 naming the tag;
  - the check-then-push runs under `flock /run/lock/atlas-image.lock`, the lock `promote.sh` also takes, so two
    builds cannot race for one N;
  - after the push, HEAD must return the digest the push printed, or the script exits 3.

  `{sha12}-rN` is promoted like any other tag, and its label still names the same commit.
- **`images.yml` build fields are NOT inputs.** A row's `containerfile`, `target` and `context` change what is
  built, yet the file also carries every image's `adopted:` entry, so listing it as an input would give all 22
  images a new sha12 on any edit to any row. Instead every image carries the label
  `atlas.build-spec=<containerfile>:<target>:<context>`, and the skip above requires it to equal the current
  row: a mismatch exits 2 naming the field, and `--rebuild` then writes a `-rN` under the new spec.
- **The pre-commit build check (CLAUDE.md §VERIFY) keeps working on a dirty tree.** `build-image.sh --local`
  builds from the WORKING TREE, dirty inputs allowed, and tags the result `localhost/{image}:dev`. It never
  pushes, never touches the registry or any `latest`, and exits with the build's status. Its output cannot be
  promoted, because stage 1 requires the target to exist in the registry. `{Project}/.devcontainer/build.sh`
  becomes this `--local` form. The registry build is `build-image.sh <image>` (or `build.sh --push`), and it
  still refuses a dirty input with exit 2.
- **What `--no-cache` means now.** It applies ONLY to `--local`: a clean rebuild that proves the Containerfile
  still builds from nothing. On the registry path it is refused with exit 2, and the message names `--rebase`.
  A cache-less rebuild of an existing sha12 would otherwise be a silent skip or an overwrite, and the fresh
  base layers it is usually after are what `--rebase` produces, under a new `-rN` tag.
- **A hand `nerdctl push` over an existing tag** is outside the script, and the script cannot stop it.
  `promote.sh` detects it: it refuses a TAG whose current digest differs from the digest the newest release (or
  spool file) recorded for that same TAG, as `from.tag` or `to.tag` (PROMOTE stage 1). The check keys on the
  tag, never the sha12: `{sha12}` and `{sha12}-rN` share a commit and differ in bytes by design. When GitHub is
  unreachable, releases are skipped with a WARN and the local spool files are still checked.
- **Inputs are declared per image in `deployment/images.yml`**: the Containerfile, its COPY/ADD sources, and
  the compose service(s) the image serves. AC3 keeps that manifest honest. The measured roots:

  | image(s) | input roots |
  |---|---|
  | alert-service | AlertService/ |
  | calendar-service | CalendarService/, Events/ |
  | alphavantage-collector | AlphaVantageCollector/, CalendarService/, Events/ |
  | finnhub-collector, finnhub-mcp | FinnhubCollector/, CalendarService/, Events/ |
  | fred-collector, fredcollector-mcp | FredCollector/, CalendarService/, Events/, MacroSubstrate/ |
  | ofr-collector | OfrCollector/, CalendarService/, Events/, MacroSubstrate/ |
  | ofr-mcp | OfrCollector/ (mcp only) |
  | reports-daily/-weekly/-monthly | Reports/, CalendarService/, Events/, MacroSubstrate/ |
  | macro-substrate-migrator | MacroSubstrate/, Events/ |
  | secmaster | SecMaster/, Events/ |
  | secmaster-mcp | SecMaster/mcp/, SecMaster/openapi.json |
  | sentinel-collector | SentinelCollector/, Events/, MacroSubstrate/ |
  | dsl-parser-mcp | 5 paths under SentinelCollector/dsl-parser-mcp/ |
  | threshold-engine, thresholdengine-mcp | ThresholdEngine/, Events/, MacroSubstrate/ |
  | whisper-service, whisper-service-mcp | WhisperService/ |
  | finbert-sidecar | FinBertSidecar/ |

  `.md` stays an input, because runtime content is `.md` here (`SentinelCollector/src/prompts/*.md`).

### PROMOTE: one command for deploy and rollback

```
promote.sh <image> <sha12|sha12-rN> --reason "<text>" --by <operator-or-agent> [<image> <tag> ...]
```

The stages run in order. Anything that fails before stage 4 leaves `latest`, the registry's tags and every
container untouched (stage 2's spool write is the only side effect, and a refusal deletes it).

Every side of every move is recorded as `{tag, sha12, digest}`: `tag` is the registry tag, `digest` its
manifest digest, and `sha12` the first 12 characters of the image's revision label, or of its `images.yml`
`base` for an `adopted-*` tag, which carries no label. The TAG is the identity; the sha12 only locates commits,
because `{sha12}` and `{sha12}-rN` share one.

1. **Validate.**
   - The image is in `images.yml`.
   - HEAD on the target tag returns 200; record its digest.
   - **Recorded-digest check, keyed on the TAG.** If the newest release or spool file that names the target tag
     (as `from.tag` or `to.tag`) recorded a different digest, exit 1 naming the tag and both digests. A
     rollback is the case this exists for: its target is the `from.tag` of an earlier move, so it lands only on
     the bytes that move recorded as `from.digest`. A tag no record names passes (a first promote). With
     GitHub unreachable, releases are skipped with a WARN and the spool files are still read.
   - The target's revision label is an ancestor of the CACHED `origin/main`
     (`git merge-base --is-ancestor`), so unmerged code cannot be promoted. This check reads the local ref and
     needs no network, so a rollback works offline: a rollback target is older than anything already promoted.
   - An `adopted-*` target carries no revision label. It is accepted AT ANY TIME, and the ancestor check runs on
     its recorded `base` (the `adopted: {tag, base}` entry step 2 commits to `images.yml`). An `adopted-*` tag
     with no entry there is refused with exit 1, naming the image. It then takes the full stages 2-7 path,
     restart included, like any other rollback. There is no `--adopt` flag: the one no-restart adoption is a
     separate one-shot script (SEQUENCING step 2), so no promote can skip the restart.
   - `--reason` and `--by` are non-empty.
2. **Lock, resolve `from`, write the `pending` record.**
   - Take `flock -w 600 9` on `/run/lock/atlas-image.lock` (fd 9). It BLOCKS for up to 10 minutes behind
     another promote, a build's check-then-push or a GC run (SEQUENCING step 5), then exits 5 naming the lock
     file. The lock is held until the process exits, so stages 3-7 run under it.
   - Read `latest`'s current digest, then resolve `from.tag` as the tag in `/v2/<image>/tags/list`, other than
     `latest`, carrying that SAME digest. Several matches: the one the newest record names as `to.tag`, else the
     plain `{sha12}` before any `-rN`. No match (a hand push to `latest` that no other tag names) exits 1
     naming the digest: moving `latest` would delete the only name of the running image, so there would be no
     rollback target. The hand move already pages (MOVE LOG).
   - Stamp the release tag (MOVE LOG) and write the spool file
     `/opt/ai-inference/containers/promote-spool/<tag>.json` with every image's `from` and `to` and
     `outcome: pending`, BEFORE stage 4. A crash or UPS halt after any move therefore always leaves a record
     that names both rollback targets (GC keeps them; the timer publishes it as `interrupted`).
3. **Show and pre-pull.**
   - Print `git log --oneline from.sha12..to.sha12 -- <inputs>` (or `to..from`, labelled REMOVED, on a
     rollback). For an `adopted-*` side the sha12 is its `base`. This is where a held commit becomes visible
     before the move.
   - `nerdctl pull 127.0.0.1:5000/<image>:<tag>`. The scoped restart below removes the container before it
     recreates it, so every network dependency must come first.
4. **Move.** GET the target's manifest with its exact media type, then PUT the same bytes to
   `/v2/<image>/manifests/latest`. HEAD `latest` must now return the target digest, or the script exits 4 and
   the registry is unchanged or already correct. Moving a tag this way copies no layers and takes seconds.
   Record `moved_utc_ts` in the spool file.
5. **Pull and restart.**
   - Record the container ID of each compose service `images.yml` names.
   - `nerdctl pull 127.0.0.1:5000/<image>:latest` re-points the LOCAL `latest`. The blobs are already local from
     stage 3.
   - Then run the existing scoped form, `--tags <svc> --skip-tags build -e "scoped_restart=true
     scoped_services=<svc>"`, for those compose service(s). The scoped shell and its cascade checks are
     unchanged. That run's freshness gate is scoped to `<svc>` (FRESHNESS §Scope), so it fails only on the
     promoted service(s).
6. **Verify.**
   - Each promoted service's container ID must DIFFER from the one stage 5 recorded, else `NOT-RESTARTED
     <svc>`: a restart that ran no recreate fails here, not only in an acceptance test.
   - The freshness gate with `--only <the promoted compose service(s)>`, and the deploy skill's smoke checks
     for that service.
   - The whole-stack gate with `--report-only` (FRESHNESS §Scope). Any refusal it prints is copied into the
     release JSON as `unrelated_refusals` and printed as a WARN. It does NOT change `outcome`, so
     `PromoteMoveFailed` does not page for it; the timer's `atlas_freshness_refused{service}` alerts it instead.
7. **Record.** Finalize the spool file's outcome, then create exactly one GitHub Release from it (MOVE LOG).
   - If stage 5 or 6 FAILED, the release is still created, as a prerelease with `outcome: failed`, and the
     script exits non-zero and prints the rollback command `promote.sh <image> <from.tag> ...`.
   - The registry HAS moved, and a move with no record is the defect this plan exists to close. That is why a
     failed move is still recorded. See the corrections list for how this refines the brief's
     "after smoke pass".

**Batch.** One invocation can take several `<image> <tag>` pairs, such as the 16 images #1114 touches. Stages
1-3 run for ALL pairs before any stage 4. Then each image is moved, restarted and verified in turn, and ONE
release records the batch. A failure stops the batch at that image. The release lists what moved, what failed,
and what was not reached.

### MOVE LOG: GitHub Releases [user decision: "use GitHub releases for the move log"]

- **One release per `promote.sh` invocation**, single or batch. The tag is `deploy/<yyyymmddTHHMMSSZ>`, in UTC,
  stamped UNDER THE LOCK at stage 2, when the `pending` spool file is written, never before the lock. If that
  tag already exists as a release or a spool file, the stamp advances one second until it is free. Two invocations therefore cannot share a tag.
  - Namespace, re-measured with `git tag -l`: 26 tags exist and none starts with `deploy`. The epic tags end in
    `-done` and never contain a `/`.
  - Image sha12s are REGISTRY tags, not git tags, so they cannot collide. The `deploy/` prefix also keeps a git
    ref from ever being a bare 12-hex string that `git rev-parse` would read as an abbreviated sha.
- **The tag's target commit** is the cached `origin/main` tip at promote time. It contains every `to` revision,
  because stage 1 checked that each one is an ancestor.
- **`--latest=false`** on every deploy release, so the repo-wide "Latest" marker stays on the epic release.
- **Notes.** Each release carries one machine-readable fenced `json` block plus the human part, generated by the
  script and never hand-written.
  - The JSON records, per image: `utc_ts`, `image`, `service`, `from: {tag, sha12, digest}`,
    `to: {tag, sha12, digest}`, `by`, `reason`, `outcome` (`deployed` | `failed` | `not-reached` |
    `interrupted`), `moved_utc_ts` (the time of the move rather than of the release), and
    `unrelated_refusals` (PROMOTE stage 6).
  - The human part lists commits over the two sha12s (an `adopted-*` side's sha12 is its `base`):
    `git log from..to -- <inputs>` as SHIPPED on a forward move, and
    `git log to..from -- <inputs>` as REMOVED on a rollback. On a rollback `from..to` is EMPTY by construction
    (red-team finding 7). When neither revision descends from the other (an `adopted-*` base), it lists both
    ranges, labelled.
- **`--generate-notes` is never used.** Previewed read-only (`POST .../releases/generate-notes`, previous tag
  `d13-retired-scope-done`, target `10398312`), it returned 63 PRs from every service, INCLUDING the held #1091.
  The red-team re-ran it and got the same result.
- **The spool file is written first, always.** Stage 2 writes `promote-spool/<tag>.json` with
  `outcome: pending` before any move; stage 7 finalizes it into the complete release payload (tag, target,
  notes, the original `moved_utc_ts`) and deletes it once the release is verified (below). It sits on the ZFS
  dataset, so a record survives a reboot and is snapshotted.
- **GitHub unreachable at 3am.** The move must not block on GitHub. Stage 7 has a 20s timeout; on failure it
  keeps the finalized spool file and exits 0 if the deploy succeeded.
  - The 5-minute drift timer (SEQUENCING step 6) retries every finalized spool file with `gh release create`.
    It deletes a file only after `gh release view <tag> --json body` returns a release whose JSON block matches
    the spool file's `to.digest` and `moved_utc_ts` for every image. A release under that tag with DIFFERENT
    content is a collision: the file is kept and the timer exports it (`atlas_promote_spool_files` stays > 0,
    so the hold below pages). A retry that already landed is therefore not duplicated, and a different move is
    never dropped.
  - A back-filled release carries the move's time in its notes; its GitHub `created_at` is later.
- **A promote that died** (crash, kill, UPS halt) leaves a `pending` file. The timer never publishes a `pending`
  file as it stands. It takes the lock non-blocking (`flock -n`); if the lock is held, a promote is running and
  the file is skipped. If it gets the lock, no promote is running: a file with no `moved_utc_ts` moved nothing
  and is deleted; otherwise every entry with a `moved_utc_ts` becomes `interrupted`, the rest `not-reached`,
  and the file is published as a prerelease like any other finalized spool file.
- **Detection of a move with no release** (spool pending, spool lost, or a tag moved by hand):
  - The timer exports `atlas_image_latest_released{image}`: 1 when the registry's `latest` digest equals the
    `to.digest` of the newest release naming that image, else 0.
  - It also exports `atlas_promote_spool_files`.
  - P3 alert at `atlas_image_latest_released == 0` for 1h, plus `absent()` of the series (P2).
  - A spool that drains within the hour never pages. A hand move pages.
- **Detection of a FAILED move.** A failed stage 5 or 6 leaves `latest` on the new image, possibly with the
  container removed, so the next boot runs it. `latest_released` reads 1 (the prerelease records `to.digest`)
  and cannot see this. So:
  - the timer exports `atlas_promote_last_outcome_failed{image}`: 1 when the newest record naming the image
    (releases INCLUDING prereleases, and spool files, whichever is newer) has `outcome: failed` or
    `interrupted`, else 0. A `pending` record reads 0: its promote still holds the lock;
  - alert `PromoteMoveFailed`, P2 while it is 1 (no hold). It clears when a later promote of that image records
    `deployed`, the rollback included;
  - `absent()` of the series is P2, like the others.
- **Single source of truth.** Deploys are recorded ONLY as `deploy/*` releases, written by the one script that
  performs the move. `docs/RELEASES.md` and the `*-done` tags keep epic OUTCOMES (§PHASE_TAGS), written by the
  epic author, and a RELEASES.md entry CITES the deploy release(s) that shipped the epic instead of
  reconstructing them by forensics (the `d13-retired-scope-done` entry did). The spool is a queue, not a store:
  while GitHub is reachable it holds only the in-flight promote's `pending` file.
- Considered, not chosen: a repo file appended through a PR. The record would land only on merge, so it lags the
  registry and can diverge from it (a move with its PR unmerged, or a merged PR for a move that was reverted).

### DRIFT: merged-but-not-deployed, per image

- **Series.** A `User=james` 5-minute timer runs `git fetch origin main` first and writes a james-owned textfile.
  It exports:
  - `atlas_image_latest_behind_commits{image,basis}` = `git rev-list --count <rev>..origin/main -- <inputs>`.
    `<rev>` is the revision label of the digest the registry's `latest` names. For an `adopted-*` image, which
    has no label, it is the `base` recorded in `images.yml`. `basis` = `label` | `adopted`.
  - `atlas_git_origin_main_fetch_age_seconds`. A stale ref reads as LOW drift, a failure toward success that
    `absent()` cannot see (red-team finding 6).
- **Alerts.** Drift is a REPORT (a dashboard table), not a page: merged-but-not-deployed is normal. The alerts
  are fetch age > 6h (P3) and `absent()` of either series (P2).
- **Positive control, today.** secmaster reads >= 2 while #1091 is held (THE PROBLEM 3).

### FRESHNESS: the running container matches `latest`'s digest

`freshness-gate.sh` keeps its rootfs comparison of the running container against the local image. It gains one
check per first-party service, run first:
- the registry's `latest` digest (HEAD `/v2/<image>/manifests/latest`) must equal the local
  `127.0.0.1:5000/<image>:latest` RepoDigest.

A mismatch is `MOVED-NOT-DEPLOYED <image>`: the tag moved, but nothing pulled and restarted. The existing rootfs
check then says whether the container runs the local `latest`. The two checks together mean the running rootfs
comes from the registry's `latest`.

The same timer exports `atlas_image_running_matches_latest{image}`, P3 at 0 for 30m. It reads 0, never absent,
when the image's compose service has NO container: a failed stage 5 can leave exactly that state, and an absent
series would fire nothing but the generic `absent()` rule.

**Scope: two named flags, two calls in deploy.yml.** Today deploy.yml calls the gate ONCE, whole-stack, with
`failed_when: freshness_gate.rc != 0` (`deploy.yml:2552`), so one unrelated STALE service fails every scoped
deploy, and after cutover would fail stage 5 of every promote: a good move recorded `failed`, paged and
handed a rollback command. So:
- `--only <svc>...` runs BOTH checks (digest and rootfs) for those services only.
- `--report-only` runs both checks over every service, prints every refusal, and always exits 0.
- deploy.yml, on a run with `scoped_restart=true`, calls the gate TWICE, both after every other task: first
  `--only {{ scoped_services }}` with the existing `failed_when` (the verdict on what this run restarted),
  then `--report-only`, whose refusals it prints. A full run keeps the ONE whole-stack call that fails, as today.
- `promote.sh` stage 6 makes the same two calls itself, so its outcome never depends on reading ansible output.

What an unrelated STALE does now: a scoped deploy or promote SUCCEEDS, prints the refusal, and the promote
records it in `unrelated_refusals`; `outcome` stays `deployed` and `PromoteMoveFailed` does not page. That
demotes today's hard failure, so it gets a wired alert: the timer runs the whole-stack gate `--report-only`
and exports `atlas_freshness_refused{service}` (1 per refused service, third-party images included), P3 at 1
for 30m, `absent()` P2. A full run and the standalone gate still fail on it.

Limit, unchanged from today: the rootfs check cannot see a change that touched only config (ENV/CMD/LABEL).

### BOOT PATH, and a bare `--tags`

- **Pulls.** Compose resolves only MISSING images. So `atlas.service`'s `compose up -d`, and a bare `--tags`
  full restart, use the LOCAL `127.0.0.1:5000/<image>:latest` whenever it is present. They contact the registry
  only for an image that is missing locally.
  - That is the intended behaviour: a boot reproduces what was last deployed, never a moved-but-undeployed tag.
    This is the opposite of today's migrator hazard, and FRESHNESS reports the gap.
  - Only `promote.sh` pulls a tag that is already present.
- **The brief's premise, corrected.** "atlas.service and a bare `--tags` pull `latest` from the loopback
  registry" is true ONLY for an image missing from the local store.
- **Ordering.** `atlas.service` gains `Wants=atlas-registry.service` and `After=atlas-registry.service`, the same
  as its existing `Wants=otel.service`. It does NOT gain `Requires=`: a registry failure must never cancel the
  stack. The unit's own comment documents that a `Requires=` failure leaves the stack down with no `Restart=`.
- **Registry down at boot, local cache present** (the steady state): nothing contacts the registry, and the stack
  boots.
- **Registry down, image missing locally:** that service's pull fails. The ref names host `127.0.0.1:5000`, so
  the pull cannot fall through to `docker.io/library/`. Whether `compose up` then aborts the other services is
  unmeasured (WHAT COULD NOT BE VERIFIED).
- **Exposure is alerted BEFORE a boot needs the image.** The timer exports
  `atlas_image_latest_present{image,where="local"|"registry"}`, both P2. Local 0 means the next boot depends on
  the registry. Registry 0 means the durable copy is gone while recovery is still a push.
  - As built at step 0 (`registry-metrics.sh`, alerts `monitoring/alerts/registry.yml`): the `registry` half has
    one series per LOCAL `127.0.0.1:5000/<image>:latest` ref -- exactly the images whose recovery is still a
    push -- so it is empty, and its alert inert, until step 2 warms the cache. An unreachable registry omits the
    series (unknown, not 0) and sets `atlas_registry_up 0`, alerted on its own; a write timestamp catches a dead
    timer. The `local` half, enumerated from `images.yml`, lands with step 6's timer.
- **No blocking `ExecStartPre=`.** It would turn one missing image into 30 stopped containers, and it adds to D
  in the StartLimit criterion (`atlas.service`).
- **deploy.yml keeps its pattern.** The existing "ensure digest-pinned engine images" task (`deploy.yml:1410`,
  `[always]`: inspect, else pull) is extended to the 22 `127.0.0.1:5000/<image>:latest` refs. It pulls only a
  missing image, never a present one, so a bare `--tags` cannot ship an undeployed move either.

## WHAT CHANGES (enumerated by grep, not recalled)

First-party images in scope: the **22** that `deploy.yml` builds from the repo AND that compose runs. That is 31
`image:` lines minus 9 third-party or host-context images, a count the red-team re-derived.

| file / system | change |
|---|---|
| NEW `deployment/artifacts/atlas-registry.service.j2` + deploy.yml task | the registry unit (THE DESIGN §Registry); a template because it renders the `registry_image` pin |
| NEW `/etc/containerd/certs.d/127.0.0.1:5000/hosts.toml` (ansible) | plain HTTP for that one host |
| `deployment/artifacts/atlas.service` | `Wants=` + `After=atlas-registry.service` |
| NEW `deployment/images.yml` | containerfile, target, context, input paths and compose service(s) per image |
| NEW `deployment/scripts/build-image.sh <image> [--at <sha>] [--rebuild] [--rebase] \| --local [--no-cache]` | the only builder: build from `git archive`, label (revision + `atlas.build-spec`), overwrite refusal, push `{sha12}[-rN]`. Never writes `latest`. `--local` is the dirty-tree verify build (IMAGE IDENTITY) |
| NEW `deployment/scripts/promote.sh` | THE DESIGN §PROMOTE + §MOVE LOG, including the spool. No `--adopt` flag |
| NEW `deployment/scripts/adopt-images.sh` (one-shot, SEQUENCING step 2) | sets each `latest` to its `adopted-*` tag WITHOUT a restart and writes release #1. It REFUSES (exit 1, naming the release) once any `deploy/*` release exists, with that precondition written at the refusal: the no-restart path is earned only when no move has ever happened, because only then is the running rootfs the adopted one by construction |
| `deployment/artifacts/compose.yaml.j2` | 22 `image:` lines -> `127.0.0.1:5000/<image>:latest`; DELETE the 21 `build:` stanzas (each renders `context: ..` = `/opt`, which holds no source) |
| `deploy.yml` | DELETE the 23 repo build tasks (22 + nasdaq-collector); extend the `:1410` ensure-present task to the 22 refs; on a scoped run the `:2552` gate becomes two calls, `--only {{ scoped_services }}` (fails) then `--report-only` (FRESHNESS §Scope) |
| NEW `deployment/artifacts/scripts/image-state-metrics.sh` + `.service`/`.timer` (`User=james`) | drift, fetch age, latest-released, last-outcome-failed, running-matches-latest, freshness-refused, present local/registry, spool retry, `pending` -> `interrupted` |
| `deployment/ansible/scripts/freshness-gate.sh` + `deployment/tests/freshness-gate/` + `deployment/tests/ansible/run.sh` | the registry-vs-local digest check and its known-bad fixture; `--only` and `--report-only`; the ansible harness asserts that under the scoped form an unrelated refusing stub is PRINTED and does not fail the run, and under the full form it still fails |
| NEW `scripts/tests/test_image_manifest.py` | AC3 |
| NEW `deployment/tests/promote/` | AC2, AC6, AC9, AC10, AC12, and the controls of AC1, AC8 and AC11, run against a throwaway registry on a non-5000 loopback port |
| 12 `{Project}/.devcontainer/build.sh` | one-line wrappers: default `build-image.sh --local [--no-cache]`, `--push` = `build-image.sh <image>`. The deploy hint becomes `build-image.sh <image>` then `promote.sh <image> <sha12> ...` |
| `scripts/tests/build-deploy-hint-selftest.sh` | its case 5 scans the build.sh hints for a bare `--tags`. Once no hint invokes ansible-playbook, case 5 is green on nothing. It gains a case: every build.sh hint names `promote.sh`, with a known-bad fixture carrying the old ansible hint that must be flagged by file and line |
| `.claude/skills/supervisor-mode/templates/deploy.md` steps 4-5 | `build.sh --no-cache` + image `.Created` + the scoped form -> `build-image.sh <image>`, then `promote.sh <image> <sha12> --reason --by`, verified by the freshness gate (which promote.sh runs) and the container `.Id` change. After cutover the old steps deploy nothing and verify the wrong object |
| `.claude/hooks/dotnet-guard.sh:93` | the deny message's `build.sh [--no-cache]` line: unchanged in spelling, and now correct, because `build.sh` is the `--local` verify build. Re-read it at step 4 |
| `.claude/skills/deploy/SKILL.md` §ROLLBACK | `:autofix-prev` snapshot/restore -> `promote.sh <image> <older sha12>`. Its `compose up --force-recreate` is broken today (filed) |
| `autofix-watcher.sh`, `merged-pr-watcher.sh` | disabled x2 (`systemctl is-enabled` -> disabled, 2026-09-26). If re-armed, they call `promote.sh` like everyone else; re-arming them is out of scope. Rewrite the comments only |
| `deployment/artifacts/monitoring/` | the alerts named in THE DESIGN |
| CLAUDE.md, deployment/README.md | the amendments below |
| 26 READMEs that spell `{svc}:latest` | pointer updates (`git grep -l ':latest' -- '*.md'`) |

**Dropped from revision 2, because B does not need them:**
- `image_pins.yml` and its `vars_files` wiring;
- the reviewed-commit preflight over both checkouts, and the `image_pin_break_glass` flag. `promote.sh`'s offline
  ancestor check replaces both;
- the push-guard rule 1.5 exemption. No pin file means no guard change;
- the per-pin-commit release script. `promote.sh` writes the release.

**Unchanged, deliberately:**
- third-party images: llama.cpp and vllm are already digest-pinned, and timescaledb has a version tag;
- host-context builds (trafilatura, spacy-ner, the exporters, sandbox-kernel), which are phase 2 on the same
  mechanism;
- host-mounted config (prompts, patterns). **`latest` records a service's image, not its whole deployed state.**

## CLAUDE.md / README RULES THIS WOULD SUPERSEDE [proposal, each named]

1. §CONTAINER_BUILD `IMAGE: {compose-service}:latest` -> `127.0.0.1:5000/{image}:latest`, moved only by
   `promote.sh`; builds write `{image}:{sha12}`.
2. §CONTAINER_BUILD `BUILD` / `DEPLOY: build first, then the scoped form` -> `build-image.sh` (idempotent), then
   `promote.sh`, which runs the scoped form itself.
3. §DEPLOYMENT `--skip-tags build deploys the CURRENT :latest` -> the scoped form restarts on the LOCAL `latest`.
   A bare scoped form without `promote.sh` ships no move. There is no build tag.
4. deployment/README.md §TAG_MECHANICS: the `:latest` paragraph and the "18 of the 27 ... ONLY the build task"
   count.
5. `.claude/skills/deploy/SKILL.md` §ROLLBACK -> `promote.sh <image> <older sha12>`.
6. §DEPLOYMENT VERIFY_TRAP: the freshness gate also compares the registry's `latest` digest.
7. §PHASE_TAGS is extended, not superseded: step 3's RELEASES.md entry cites its `deploy/*` release(s).
8. **§WHERE_WORK_LANDS gains one row**: `what was deployed, when -> the deploy/* GitHub release promote.sh
   writes # generated, never hand-written; "what happened" stays git log + the PR body`. It lands with step 4,
   not in this document: adding it now would name a store that does not exist yet.
9. §DATA_FLOW's host-port sentence gains the loopback registry (`127.0.0.1:5000`, not a roster service).
10. §GIT_PUSH is UNCHANGED (revision 2's proposed exemption is withdrawn).
11. `.claude/skills/supervisor-mode/templates/deploy.md:28-36` (steps 4-5, the deploy dispatch brief) ->
    `build-image.sh`, then `promote.sh`, verified by the freshness gate. Superseded at step 4.
12. §VERIFY `container: {Project}/.devcontainer/build.sh [--no-cache]` (and its copy in
    `.claude/hooks/dotnet-guard.sh:93`): the spelling survives, the MEANING changes to the local dirty-tree
    verify build. `--no-cache` means a clean local rebuild only; the registry path refuses it (IMAGE IDENTITY).

No `AGENT_README.md` D-entry is touched: `grep -iE ':latest|build\.sh' */AGENT_README.md` returns 0 lines. D-19
is not superseded: it still holds #1091, and the drift metric now makes that hold visible.

## RED-TEAM DISPOSITION UNDER B [each finding re-derived; the red-team reviewed revision 1]

| # | finding | under B |
|---|---|---|
| 1 | tags are mutable names; a recovery rebuild reuses the tag with new bytes | **still applies**, because `latest` is mutable by design and registry:2 overwrites tags. Handled by: the build's overwrite refusal (`-rN`), the `{tag, sha12, digest}` recorded for both sides of every move in the release, `promote.sh` refusing a TAG whose digest differs from the digest recorded for that tag, and the freshness digest check (AC2, AC6) |
| 2 | "reviewed commit" is unenforced; a stale checkout writes old pins | **the stale-checkout half no longer applies**: the image a deploy uses is registry state, and the rendered `image:` line is a constant, so no checkout can change which bytes run. The review half is **accepted as B's cost**, reduced by the offline ancestor-of-origin/main check (only merged code can be promoted) and by drift visibility. The template-from-`atlas_repo_path` hazard for OTHER config is pre-existing and not changed here |
| 3 | rollback depends on GitHub | **no longer applies**: `promote.sh` moves a registry tag on loopback. Only the release needs GitHub, and it spools (AC9 control) |
| 4 | Appendix A's triggers are corpse detectors | **moot**: the registry is built. The exposure series (`present`, local and registry) fire before loss |
| 5 | a missing image on boot is only alerted | **handled**: the host-qualified ref cannot resolve to docker.io, `Wants/After` ordering, the local cache as steady state, and P2 exposure alerts. A blocking ExecStartPre stays rejected (BOOT PATH) |
| 6 | adopted images have no base; the timer never fetched | **still applies**: adopted images carry `base` and `basis="adopted"`, the timer fetches first, and fetch age is alerted (AC5) |
| 7 | rollback release notes are empty | **handled**: REMOVED = `to..from` (AC7) |
| 8 | AC4's literal N=30 | **handled**: N is derived in the same run |
| 9 | AC1 passes if the deploy never ran | **handled**: the container ID must change |
| 10 | say rule 1's main allowlist is unchanged | **no longer applies**: B makes no push-guard change |

## CORRECTIONS TO INHERITED NUMBERS AND CLAIMS

- **Revision 2's verdict** (digest pin in git, recommended) -> **withdrawn** by the user's decision. A and B' move
  to the appendix.
- **Revision 2's table row "B: audit trail none, registry:2 keeps no history"** -> true of the registry alone. B
  as specified here keeps one release per move.
- **Revision 2's table row "B: local and registry latest can disagree"** -> still true, and now intended: only
  `promote.sh` pulls. The disagreement is detected (`MOVED-NOT-DEPLOYED`), never silent.
- **Brief: "atlas.service and a bare `--tags` pull latest from the loopback registry"** -> only for an image
  missing locally. Compose never re-pulls a present tag (BOOT PATH).
- **Brief relay: the release is "created AFTER the scoped restart and smoke pass"** -> refined: the release is
  created after the restart and smoke COMPLETE. A failed move is still recorded, as a prerelease with
  `outcome: failed`. The user's words ("use GitHub releases for the move log") ask for a log of MOVES, and
  skipping failed ones would leave the most important moves unlogged.
- CLAUDE.md §CONTAINER_BUILD, "12 of the 14 build.sh print ... the BARE `--tags` form": **stale.** 0 of 14 do. 11
  print the scoped form, 2 print a caveat, and FinBertSidecar prints no deploy line.
- Brief, "each build.sh builds `{svc}:latest`": 10 of the 22 images have NO build.sh (the 6 MCPs, dsl-parser-mcp
  and 3 Reports hosts). They are built only by `deploy.yml`.
- Revision 1, "14 hand-made tags incl. 5 `:autofix-prev`" -> 19: 13 hand-made + 6 `:autofix-prev`.
- Revision 1, "a registry is a second copy on the same host" -> the copy is on ZFS with 15-minute snapshots,
  while the store is on unsnapshotted ext4 (re-measured this revision: 697G available, `/var` 94%).
- Red-team, "698G free" -> 697G. Immaterial.
- Brief, "#1091 merged 2026-09-21 and was not deployed": confirmed, and it is now joined by #1114 (re-run for this
  revision).
- This revision's own round-1 text, corrected in round 2 (review r2):
  - "deploy.yml passes `--only` to the digest check; its rootfs check stays whole-stack" contradicted §Scope
    (`--only` scopes both) and `deploy.yml:2552` is ONE call. Replaced by two named flags and two calls, and an
    unrelated STALE no longer fails a promote (FRESHNESS §Scope).
  - `from: {sha12, digest}`, resolved by revision label -> `{tag, sha12, digest}`, resolved by digest match: a
    label cannot tell `{sha12}` from `{sha12}-rN`, so a rollback could land on the wrong bytes.
  - GC's 90-day window, and a spool written only on GitHub failure, left the newest rollback target of a
    rarely-promoted image, and of a crashed promote, deletable. Now: newest release per image at any age, and a
    `pending` spool before stage 4.
  - "Local containerd GC keeps only the tags `latest` names" would have deleted the migration's rollback
    (`docker.io/library/<svc>:latest`, `:autofix-prev`); it is now scoped to `127.0.0.1:5000/*` refs.
- deploy skill §ROLLBACK's restore is broken today: `--force-recreate` exists only on `compose create` (filed).

## SEQUENCING [load-bearing edges marked]

0. **Registry unit + hosts.toml + the `present{where="registry"}` series.** Inert, because nothing reads the
   registry yet. **Must precede step 2**, which pushes to it.
1. **`images.yml` + `build-image.sh` + AC3 test + `deployment/tests/promote/`.** Inert: nothing deploys the new
   tags.
2. **Adopt what is RUNNING, not what `:latest` says.** For each of the 22 images:
   - find the image whose rootfs ChainID matches the container (the freshness gate's `rootfs_of`);
   - push it as `{image}:adopted-<yyyymmdd>`;
   - record `base` = `git rev-list -1 --before=<image .Created> origin/main -- <inputs>`, committed as
     `adopted: {tag, base}` in `images.yml` (a reviewed PR). promote.sh and the drift timer read it from there,
     offline.

   Then `adopt-images.sh` (not `promote.sh`) sets every `latest` to its `adopted-*` tag and writes release
   `deploy/...` #1. It restarts nothing, because the rootfs is identical by construction, and it could not
   safely restart: compose still names `docker.io/library/<svc>:latest` until step 3, so a restart would run the
   undeployed migrator. It refuses once any `deploy/*` release exists (WHAT CHANGES). Finally,
   `nerdctl pull` every `127.0.0.1:5000/<image>:latest`, so the local cache is warm. From here on, a rollback to
   an `adopted-*` tag is an ordinary `promote.sh` with the full restart (PROMOTE stage 1, AC11).
   - The migrator is adopted from the rootfs it last RAN (`:autofix-prev`), never its undeployed `:latest`.
     **LOAD-BEARING:** adopting `:latest` would deploy the undeployed migrator at cutover.
   - No matching image means STOP and ask a human. Never guess.
   - For secmaster, `base` falls before `6a77daf3`, so drift reads 2: AC5's positive control.
3. **One PR: template + ensure-present extension + atlas.service `Wants/After` + freshness-gate check.**
   **LOAD-BEARING:** step 2's warm cache must exist before this render lands. Otherwise a boot or a bare
   `--tags` meets 22 refs that are missing locally and depends on the registry. Deploy it with ONE scoped
   secmaster restart. The render changes all 22 lines, but every other container keeps its rootfs, and the
   freshness gate proves that (AC4).
4. **`promote.sh` becomes the deploy.** Delete the deploy.yml build tasks, split deploy.yml's gate into its two
   scoped-run calls (FRESHNESS §Scope; step 3 keeps the one whole-stack call), wrap the build.sh scripts, and update
   the deploy skill, the docs, the CLAUDE.md amendments and the WHERE_WORK_LANDS row. **Must follow 3**, or
   `docker.io/library/<svc>:latest` stays the reference with nothing left to update it.
5. **GC.** GC takes `flock -w 600` on `/run/lock/atlas-image.lock` and holds it for its WHOLE run, from
   reading the releases to restarting the registry, so no promote or build can move `latest` or push between
   the keep-set and the deletes. A promote that arrives meanwhile waits up to 10 minutes and then exits 5
   (PROMOTE stage 2); GC's duration is unmeasured (WHAT COULD NOT BE VERIFIED). Registry GC keeps:
   - the digest `latest` names. Deleting a manifest by digest removes EVERY tag on it, `latest` included, so
     this is checked first, per image;
   - the `from` and `to` digests of the NEWEST release naming each image, WHATEVER ITS AGE: an image last
     promoted a year ago still has its rollback target;
   - every `from` and `to` digest in `deploy/*` releases from the last 90 days;
   - every `from` and `to` digest in every `promote-spool/*.json`, `pending` included, whatever its age: a
     spooled or interrupted move has no release yet, and stage 2 writes the `pending` file BEFORE the move, so
     a crash between stages 4 and 7 still leaves both digests named;
   - the 2 newest other tags per image;
   - every `adopted-*` tag until phase-done.

   **GC FAILS CLOSED.** It exits non-zero and deletes NOTHING if the release listing fails, times out, or
   returns zero `deploy/*` releases while the registry holds any non-`adopted-*` tag, or if a spool file does
   not parse. An empty keep-set from an unreachable GitHub is otherwise indistinguishable from "no rollback
   targets", and local containerd keeps no second copy (below), so a wrong delete is unrecoverable. GC runs
   as `User=james` with the same `gh` credentials as the timer (WHAT COULD NOT BE VERIFIED).

   It deletes the rest through the API, then runs `garbage-collect` with the registry stopped, still under the
   lock. That is safe because the boot path does not depend on the registry. Local containerd GC removes only
   `127.0.0.1:5000/<image>:<tag>` refs other than the one `latest` names, which relieves `/var` (94%). It never
   touches a `docker.io/library/*` ref: `docker.io/library/<svc>:latest` and `:autofix-prev` are the
   migration's rollback until step 7 deletes them by name (ROLLBACK OF THE MIGRATION), and the 13 hand-made tags
   stay until a human removes them.
6. **Metrics timer + alerts (drift, fetch age, latest-released, last-outcome-failed, running-matches-latest,
   freshness-refused, spool, `pending` -> `interrupted`).** **LOAD-BEARING: ships no later than step 4.** From
   the first real promote, a spooled or interrupted move is published, and a failed one pages, only through this
   timer; and step 4's two-call gate demotes an unrelated STALE, which only `atlas_freshness_refused` alerts.
   They must also exist before step 7 deletes the old tags.
7. **Phase-done.** Delete `docker.io/library/<svc>:latest` for the 22 images, the `:autofix-prev` tags and the
   unreferenced `adopted-*` tags. **Only now**, because until now they are the migration's rollback.

## ACCEPTANCE CRITERIA

| # | measure | PASS | FAIL | negative control (broken vs no-data) |
|---|---|---|---|---|
| AC1 | `promote.sh secmaster <T>`, where T = the `from.tag` stage 2 resolves for `latest` (an `adopted-*` tag after cutover, a sha12 tag later): a no-op move, which keeps #1091 held; container ID before and after; freshness gate | promote.sh exits 0, the container ID CHANGED (the deploy ran), the registry and local `latest` digests are unchanged, the rootfs is unchanged, and exactly one new `deploy/*` release shows `from == to` (tag and digest) | ID unchanged (the deploy never ran: red-team finding 9), any digest changed, or exit != 0 | aimed at the restart: in `deployment/tests/promote/`, the same no-op promote with the stage-5 ansible call stubbed to exit 0 WITHOUT recreating the container -> promote.sh exits non-zero, prints `NOT-RESTARTED <svc>`, and the release records `outcome: failed`. A restart that silently did nothing therefore cannot pass |
| AC2 | `build-image.sh secmaster` twice at one sha; then `--rebuild` at the same sha | 2nd run: `exists, skipped <digest>`, no build, no push. `--rebuild`: pushes `<sha12>-r1`, and the `<sha12>` digest is unchanged | the 2nd run builds, or `--rebuild` writes over `<sha12>` | pre-create `<sha12>-r1` with other bytes in the test registry: `--rebuild` writes `-r2` and never `-r1`. Dirty an input: exit 2 naming it. Change the row's `target` with no input change: the skip is refused, exit 2 naming `target` |
| AC3 | `test_image_manifest.py` over the 22 rows | every COPY/ADD source is covered, and every row names a compose service that `compose config --services` lists | an uncovered source or an unknown service | known-bad fixture `COPY Undeclared/ x/` -> the test names `Undeclared/` |
| AC4 | freshness gate after step 3 | `passed -- N`, where N = `compose ps -q \| wc -l` taken in the SAME run (30 on 2026-09-26), AND the rendered compose (`test-templating.yml`) names `127.0.0.1:5000/<image>:latest` on all 22 lines and `docker.io/library` on none | any STALE, UNVERIFIABLE or MOVED-NOT-DEPLOYED, or passed < N | a fixture where the registry `latest` digest != the local RepoDigest -> `MOVED-NOT-DEPLOYED <image>`, named |
| AC5 | `atlas_image_latest_behind_commits` and `atlas_git_origin_main_fetch_age_seconds`, 10 min after the timer's first run | 22 series, AND `{image="secmaster"} >= 2` while #1091 is held, AND age < 900 | != 22 series, OR secmaster == 0 while #1091 is held, OR age >= 900 | (a) `absent()` alerts on both series (0 series exist today); (b) a test copy with an unreachable remote: age grows, and the fetch-age alert fires by name |
| AC6 | **move log + bypass detection.** Promote calendar-service once; then, in a separate step, move `latest` by hand with a raw manifest PUT (bypassing the script) and do NOT restart | after the scripted move, `atlas_image_latest_released{image="calendar-service"} == 1` and `running_matches_latest == 1`. After the hand move, within one timer period, BOTH read 0 and the gate prints `MOVED-NOT-DEPLOYED calendar-service` | either series stays 1 after the hand move | this row IS the control: the scripted move reading 1 separates a working detector from one that always reads 0. Restore by `promote.sh` to the prior tag -> both read 1 |
| AC7 | **round trip.** On calendar-service: `promote.sh` forward to sha12 NEW, then `promote.sh` back to OLD; wall-clock time from the second invocation's start to its freshness pass | container on OLD's EXACT digest (freshness gate plus registry HEAD `latest` == OLD's recorded digest) within 5 min; round trip within 10 min; smoke green; forward release SHIPPED == `git log OLD..NEW -- <inputs>`; rollback release REMOVED == the same list with SHIPPED empty; #1091 in neither | wrong digest, > 5 min, empty REMOVED, or #1091 listed | (a) the freshness gate between the legs reports the digest changed each time (the round trip is not a no-op). `releases/generate-notes` over the same range DOES list #1091 (measured: 63 PRs incl. #1091), which is where the two rules disagree. (b) Between the legs, overwrite OLD's tag in the test registry with other bytes by a raw push -> the rollback exits 1 naming the tag and both digests, and `latest` is unchanged. (c) Promote `<sha12>`, then `<sha12>-r1`, then `<sha12>` again -> all three pass: the recorded-digest check keys on the TAG, so a sibling `-rN` never refuses its base tag |
| AC8 | **drift under a hold.** `atlas_image_latest_behind_commits{image="secmaster"}` while #1091 is held | >= 1 | 0, or absent | in the TEST registry (`deployment/tests/promote/`), promote secmaster to a throwaway `-r` build of C = `git log -1 --format=%H origin/main -- <secmaster inputs>` taken in the SAME run (the input tip, wherever main has moved): drift there reads 0, which shows the metric moves and is not stuck at a constant |
| AC9 | **every move has exactly one release.** Run `promote.sh` with GitHub unreachable, via `HTTPS_PROXY` pointed at a local listener that accepts and never answers (a `GH_HOST` swap fails at once with "not logged in" and never reaches the timeout), then restore GitHub | the move completes with no GitHub wait beyond 20s. A spool file exists and `atlas_image_latest_released == 0`. The P3 alert fires after its hold in the alert selftest. After restore, within one timer period: exactly ONE `deploy/*` release for that move, `moved_utc_ts` == the original, the spool is empty, and the series reads 1 | the move blocked on GitHub, no spool file, the series stayed 1 while unreleased, or 0 or 2 releases after restore | run the retry twice after restore: the release count stays 1 (the retry is idempotent). Stage 7's elapsed time is 20-25s, which proves the TIMEOUT path ran and not a fast failure. A pre-seeded release under the spool file's tag with a different `to.digest` -> the file is KEPT and named |
| AC10 | GC dry-run over a fixture registry where the `latest` digest is the OLDEST, one 60-day-old release `from` digest, one spool file whose `from` digest is in no release, one `pending` spool file, and a second image whose ONLY release is 100 days old | `latest`, the 60-day release digest, both spooled `from`/`to` digests (the `pending` one included), the 100-day-old release's `from` AND `to` digests (the newest release for that image), and the 2 newest others are kept; the rest are listed. GC takes the lock: a dry-run started while a test promote holds it lists nothing until the lock is free, and exits 5 if its wait expires | `latest`, a kept release digest or a spooled digest is listed, or GC runs while the lock is held | (a) the same fixture with a NEWER release for the second image -> its 100-day-old digests ARE listed, so the newest-release rule is not a blanket keep; (b) a 60-day digest in no release and no spool file -> it IS listed; (c) release listing unavailable (the AC9 black-hole proxy) -> exit != 0 and NOTHING listed or deleted; (d) listing returns zero `deploy/*` releases while non-adopted tags exist -> exit != 0, nothing deleted |
| AC11 | **rollback to an adopted image after cutover.** On calendar-service after step 4: `promote.sh` forward to a sha12, then `promote.sh calendar-service adopted-<yyyymmdd>` | registry HEAD `latest` == the adopted digest, the container ID CHANGED, the freshness gate passes, and the release shows `from.tag` = the sha12 tag, `to.tag` = the adopted tag, and each digest equals the registry's | refused, or the container not restarted, or any other digest | (a) an `adopted-*` tag with no `images.yml` entry -> exit 1 naming the image, `latest` unchanged; (b) `adopt-images.sh` re-run after any `deploy/*` release exists -> exit 1 naming that release, `latest` unchanged |
| AC12 | **a failed or interrupted move pages.** In the test registry, `promote.sh` with the smoke stubbed to exit 1; feed the resulting release JSON (and, separately, the spool file) to the timer. Separately, SIGKILL a promote after stage 4 and run the timer | `atlas_promote_last_outcome_failed{image} == 1` and the alert selftest fires `PromoteMoveFailed` by name; `running_matches_latest` reads 0 (not absent) with the container removed. The killed promote's `pending` file is published as ONE prerelease with `outcome: interrupted`, and the series reads 1 | the series stays 0, is absent, the alert does not fire by name, or the killed promote leaves no record | the same run with the smoke passing -> the series reads 0 and `PromoteMoveFailed` does NOT fire (the detector is not stuck at 1). A `pending` file whose promote still HOLDS the lock is skipped by the timer: not published, series 0 |

## ROLLBACK OF THE MIGRATION ITSELF

Until step 7, every `docker.io/library/<svc>:latest` still exists and names the pre-migration image.
`build-image.sh` never writes it, and neither GC touches it: registry GC sees only the registry, and local
containerd GC removes only `127.0.0.1:5000/*` refs (SEQUENCING step 5). Rolling back means reverting the step-3
PR (compose goes back to the old names) plus one scoped restart per service that has been recreated since. Every
restart runs the old local `:latest` rootfs, because step 2 adopted exactly that rootfs.

No schema or DB state is involved. The registry can be stopped and left, because nothing boots from it while the
local cache is warm. ZFS snapshots of `nvme-fast/containers` cover the registry and the spool, not the containerd
store on ext4 `/var`. After step 7 there is no rollback to the old model, only forward.

## WHAT COULD NOT BE VERIFIED

MEASURED at step 0, against a throwaway registry:2 on a non-5000 loopback port. Command and output for each:
`/tmp/sentinel-remediation/registry/step0-measure.md` (host-local, not durable):
- nerdctl 1.7.7 push and pull against a loopback HTTP registry -> **works**: a push, and a pull-by-digest that
  fetched manifest, config and layer over the network, both over plain HTTP, with the hosts.toml and also
  without one; a hosts.toml naming `https://` breaks it (M1, M2).
- The tag move by manifest GET/PUT -> **keeps the digest**: the PUT of the exact bytes with their Content-Type
  returns 201 and HEAD `latest` then reports the source digest, in both directions (M4).
- A `nerdctl pull` of `:latest` after the move -> **re-points the local tag** with every blob already present;
  before that pull the local tag keeps the old digest, so MOVED-NOT-DEPLOYED is an observable state (M5).
- Local `RepoDigests` vs the registry's `Docker-Content-Digest` -> **equal**, and equal to the digest the push
  printed, ONLY when the HEAD sends an `Accept` header (M3; THE DESIGN §Registry).
- NEW, for step 1: buildkit built a context whose file matched an earlier context's size, mtime and mode
  (different path, different bytes) into the EARLIER bytes, `--no-cache` included (M8). Archive extracts carry
  the commit time as mtime, so different commits normally differ; not verified against a real archive.

Still unverified -- each needs a container started or the registry deployed:
- Whether `nerdctl compose up -d` aborts EVERY service, or only one, when one pull fails with the registry down
  (BOOT PATH).
- **The AC7 time bounds (5 min rollback, 10 min round trip) are TARGETS, not measurements.** No scoped-restart
  duration was measured for this revision. Calendar-service is chosen because it is small.
- The `User=james` timer's container reads (`running_matches_latest`, the `--report-only` whole-stack gate)
  need rootful containerd access; passwordless `sudo` for those read-only commands is assumed, not configured.
- How long GC holds the lock (API deletes plus an offline `garbage-collect`). A promote queued behind it exits 5
  after 10 minutes, so a GC longer than that blocks a rollback; it is unmeasured.
- `gh release create` from a `User=james` systemd timer: credentials, and the ~5000/h authenticated API budget
  against a 5-minute poll that reads the newest releases.
- Whether `gh` honours `HTTPS_PROXY` (AC9's timeout control and AC10(b) rest on it). If it does not, the
  black-hole must be a firewall rule in a network namespace instead.
- Whether `gh release create`, which creates the tag server-side, interacts with branch protection (403s on
  this plan). The push guard is not involved, because no `git push` runs.
- Storage: the 22 `:latest` images are 13.2 GiB unpacked. The compressed, deduplicated size in the registry,
  plus churn, is unmeasured against 697G.
- The migrator's ChainID match with `:autofix-prev`, and the 22-of-25 drift count, come from revision 1 and
  were not re-run. The secmaster commits WERE re-run for this revision.
- No image has ever been measured LOST: containerd keeps no deletion history. The durability case rests on
  exposure (ext4, 94% full, unsnapshotted), not on an observed loss.

## APPENDIX -- considered, not chosen

**A. Digest pin in git** (revision 2's recommendation). `image_pins.yml` names each image by digest, a
preflight refuses an unreviewed or stale pin, and a pin commit is the deploy.
- Gains: holds are visible in review, and the running identity is a digest.
- Costs: a PR per deploy, a two-checkout preflight and a break-glass flag, and rollback through git. The user
  chose seconds-to-deploy and one command for deploy and rollback.

**B'. Move `latest`, then resolve it to a digest at deploy time** and render the digest into compose.
- Gains: the running identity is a digest.
- Costs: it adds a render-time resolution step and a second name for the same bytes, and it buys nothing that
  B's freshness digest check does not already detect.

**No-registry fallback** (revision 2's Appendix A). Moot now that the registry is being built: without it no
tag could be pinned by digest (`deploy.yml:1400-1402`), and durability would stay on 94%-full ext4.
