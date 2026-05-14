# Deploy Pipeline Gap: Merged PRs Do Not Trigger Image Rebuilds

## Symptom (2026-05-14)

Two production incidents tonight traced to the same gap:

- **PR #284** (SecMaster `[FromBody]` cascade) merged 19:44Z. SecMaster image
  not rebuilt until manual intervention ~23:10Z. **~3.5 h** of production
  500-storming before drift was noticed and remediated by hand.
- **PR #278** (Reports scaffolding consolidation) merged 10:57Z. The three
  Reports images (daily/weekly/monthly) were still at the 02:20Z tag when
  another agent picked it up tonight to rebuild. **~8.5 h** of stale binaries
  in service.

General pattern: PR merges to `main`, no automation rebuilds the affected
service image or redeploys, drift accumulates silently, eventually surfaces
as a fire (errors in prod, smoke-test failures, or someone notices the SHA
mismatch).

## Root cause

Image-rebuild automation **exists but only fires for AutoFix-generated PRs.**

The pipeline today:

1. `AlertService` enqueues an alert.
2. `autofix-runner.service` (systemd timer, every 60 s) reads the queue,
   invokes Claude Code, creates a branch + PR, and appends the PR URL to
   `/opt/ai-inference/autofix-pending-prs.txt`
   (`deployment/artifacts/scripts/autofix.sh`).
3. `autofix-watcher.service` (timer, every 5 min) polls `gh pr view` for
   each URL in that file; when it sees `state=MERGED` it runs
   `git pull && ansible-playbook deploy.yml --tags <service>` and smoke-tests
   (`deployment/artifacts/scripts/autofix-watcher.sh`).

The dependency is **strictly the pending-PRs file**. The watcher has no
other input. Human-authored PRs (everything not coming out of
`autofix-runner.sh`) are never written to that file, so step 3 never fires
for them, so they sit on `main` un-deployed until a human runs
`<Project>/.devcontainer/build.sh` + `ansible-playbook --tags <svc>` by
hand.

There is no GitHub Actions deploy workflow — `.github/workflows/sync-docs.yml`
only mirrors markdown to `ATLAS-Docs`. There is no cron, no other systemd
timer, no git post-merge hook, no `ansible-pull`. `atlas.service` is a
oneshot `nerdctl compose up -d` that runs at boot only and does not pull or
rebuild.

## Proposed fixes (one of)

### (a) Generalise `autofix-watcher` into a `merged-pr-watcher`

Replace the pending-PRs-file input with a `gh pr list --search "is:merged
merged:>=<last-deploy-ts>"` poll against `jpansarasa/ATLAS`. Reuse the
existing service-tag derivation, `git pull`, `ansible-playbook deploy.yml
--tags`, smoke-test, and ntfy notification. Persist the
last-successfully-deployed PR number (or merge timestamp) to a state file
to avoid re-deploying on every tick. Image rebuild is **not** in this
script today — would need to add a `build.sh` invocation per changed
service before the ansible step, or extend `deploy.yml` to call build.sh
when the local image is older than `HEAD`.

**Effort:** ~half-day. One script + one systemd unit rename + a small build
step + a state file.

**Risk:** medium. Deploys-on-merge mean an in-flight `ansible-playbook` can
collide with a developer running one manually; needs a flock. A bad PR
auto-deploys to prod without human gate. Mitigation: require a label
(`auto-deploy`) on the PR, or restrict to PRs authored by `autofix-runner`
plus an explicit allowlist of merge authors.

### (b) Manual `scripts/post-merge-deploy.sh`

A wrapper the human (or supervisor agent) runs after each merge:

```
git fetch && git pull origin main
diff_files=$(git diff --name-only <last-deployed-sha>..HEAD)
# map to service tags, build affected images, ansible-playbook deploy.yml --tags ...
echo HEAD > /opt/ai-inference/.last-deployed-sha
```

No new infrastructure; relies on discipline.

**Effort:** ~2 h.

**Risk:** low technical, high process. Whoever merges has to remember to
run it — exactly the discipline we just demonstrated we don't have
(tonight's incident is the proof). Captures the workflow but does not
close the gap.

### (c) Long-running `atlas-deploy.service` with periodic `git pull`

Convert/replace `atlas.service` (currently a oneshot at boot) with a
companion `atlas-deploy.timer` (every 5–10 min) that does
`git fetch && git pull --ff-only && ansible-playbook deploy.yml`
(with image-build before deploy where SHAs differ). Mirrors the
`ansible-pull` pattern but using the existing playbook.

**Effort:** ~1 day. Build-detection logic, flock against autofix-watcher,
state tracking, smoke gate, ntfy on failure, careful rollout because this
unit reaches every service.

**Risk:** highest blast radius. A bad commit on `main` auto-deploys
everywhere on the next tick. Needs a `deploy=false` topic-branch escape
hatch, and ideally a tag-gate (only deploy commits tagged `deploy/*` or
PRs with `auto-deploy` label) to avoid surprise rollouts.

## Recommendation: (a)

`autofix-watcher.sh` already does **exactly** the work needed: parse merged
PRs, derive service tags from changed paths, `git pull`, deploy, smoke,
ntfy. The only missing pieces are (i) feeding it merged PRs the human
created and (ii) building the image before deploying. Both are small,
localised changes to an already-running, already-debugged path.

(b) is a band-aid that fails for the exact reason we need automation. (c)
is a bigger rewrite with a wider blast radius and significant new gating
work before it can be trusted.

Suggested rollout for (a):

1. Behind a label gate (`auto-deploy` on the PR) so the first week is
   opt-in per-merge — proves the path before flipping default-on.
2. Add a `flock` around the ansible step so manual deploys and the watcher
   serialize.
3. Persist `last-deployed-pr` and `last-deployed-tree-sha` in
   `/opt/ai-inference/` to survive restarts.
4. Once stable, default the label gate to on; keep `skip-deploy` as an
   escape hatch.

## Notes / ambiguities

- The build step is currently per-service via
  `<Project>/.devcontainer/build.sh`, all driven from the monorepo root.
  Whichever option lands needs to call those before the ansible step. None
  of the three options today builds images — `deploy.yml` redeploys
  whatever image tag is already in the local containerd store. This is
  arguably the deeper bug: even when ansible *does* run, it can deploy a
  stale `:latest`.
- The service-tag mapping in `autofix-watcher.sh` lines 95–101 omits
  `Reports*`, `MacroSubstrate`, `WhisperService`, `FinBertSidecar`,
  `SentinelCollector` (partial), and the edge services. Whichever option
  ships should refresh that list from `compose.yaml` instead of
  hard-coding.
