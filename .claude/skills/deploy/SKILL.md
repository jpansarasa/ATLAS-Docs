---
name: deploy
description: ATLAS deployment workflow - smoke tests, PR, health verification via Tempo span status + Prometheus counters, prompts mount check, VRAM report, image rollback
---

# Deploy Workflow

Applies `VERIFY_TEST.COMPLETION_GATE` and `PROBLEM_SOLVING.EVIDENCE_GATE` from `~/.claude/CLAUDE.md` (neither is in the project CLAUDE.md).

1. Run smoke tests BEFORE declaring done — report pass/fail counts
2. Open PR with test results in description
3. Invoke the intent-review skill on the PR diff (design-intent conformance vs the
   touched services' DECISIONS blocks) — address critical/important findings BEFORE merge
4. After merge, verify deployment health via Tempo error spans + the Prometheus counters the
   change targets (see HEALTH_VERIFICATION) — never via log silence
5. Check for volume-mount overrides on /prompts paths
6. Report VRAM/memory status post-deploy
7. List configs/gates touched (no silent workarounds around ansible gates)

## POST_DEPLOY_SMOKE [the named gate for step 1, and step 4's reachability half]

`ansible-playbook playbooks/smoke-test.yml` — run from `deployment/ansible/`. Read-only
reachability validation, NOT a deployment: container status, internal service `/api/health`
endpoints via the grafana container, MCP `/health`, GPU + vllm-server, TimescaleDB.
Sub-tags: `health`, `containers`, `internal`, `mcp`, `gpu`, `database`.

Run it AFTER the deploy completes, before declaring done. It proves REACHABILITY — containers
up, endpoints answering. It is not a health verdict: step 4 is. Reachability and health are
different questions, and a service can answer `/health` 200 while its work path throws.

IT HAS NO LOG CHECK, DELIBERATELY. One existed until 2026-09-20 and reported `Logs (5m): PASS`
on every deploy of its life: its selector was `{job=~"atlas/.*"}` and `job` is not a label on
this Loki. It was removed rather than repaired, because fixing the selector would still leave a
log query deciding a verdict, and an empty result is the healthy steady state. Do not add one
back. `logs`, `loki` and `docker` are no longer sub-tags.

THE VERDICT IS PER-DOMAIN, NOT ONE GATE [selection model, decided 2026-09-20 — the playbook's
own comment block is the authority; this is the summary]. Each `Verdict - <domain>` task carries
EXACTLY the tags of the checks that feed it and reads no fact from another domain, so a check
and its verdict are selected or deselected together. A domain you deselect makes no claim and
prints `NOT CHECKED`; a domain that RAN cannot escape being judged.

That replaced four successive attempts to put ONE central gate on a better tag list — `always`
(dropped by `--skip-tags always`), `health` (not selected by `--tags containers,database,gpu`),
then the six-tag union (dropped by `--skip-tags internal,mcp`). Each was measured rc 0 with a
real failure present. Ansible has no task that cannot be deselected, so no tag list closes this;
the enumeration had to be removed, not extended. **Do not "fix" a future hole here by adding a
tag.**

One `always` task refuses the degenerate case *within its reach*: an invocation selecting no
domain by `--tags` (`--tags logs`, naming a tag the playbook no longer has) exits 2 rather than
reporting a meaningless 0.

ITS REACH IS BOUNDED, AND THE BOUND CANNOT BE CLOSED IN THE PLAYBOOK. Any selection whose
RESOLVED TASK SET IS EMPTY runs zero tasks and exits 0 with an EMPTY PLAY RECAP. Hold the
CLASS, never a list: Ansible's tag algebra is closed under intersection and complement, so the
argv forms producing an empty set are unbounded — `--skip-tags tagged` and
`--tags logs --skip-tags always` are both members and share no property. Ansible has no
undeselectable task, so no task added here covers it, adding one restarts the enumeration the
per-domain model deleted, and a wrapper blacklisting spellings is that same mistake one level
up. **Read the PLAY RECAP, not just the rc: a run that executed zero tasks examined nothing.**
Tracked: docs/BACKLOG.md §MEASUREMENT DEBT.

`--skip-tags always` ALONE is NOT one of those spellings, and that matters because it is this
repo's documented non-service idiom. On this playbook it skips only the summary and the
degenerate-case task; all 16 checks still run and a failure still exits 2 (measured).

Measured after the change — every sub-tag now works, singly and combined:
```
health, containers, internal, mcp, gpu, database   -> 0 on a healthy stack
logs (matches no task)                             -> 2, refuses a run with no verdict
```
The five that used to exit 2 on an undefined fact in the summary were fixed by the same change:
the summary now guards each row with `is defined` and decides nothing. An earlier revision of
this section said only `health` ran, and before that said all five died on `db_check` — the
second was wrong in four rows, from generalising one measurement. Re-derive per tag.

WHAT ITS CONTAINER CHECK CAN AND CANNOT SEE. It lists `nerdctl compose ps -a` from
`deployment_base`, so it covers every service in the ATLAS compose project and fails on any that
is not running. `migrate-macro-substrate` is exempt ONLY when `State == "exited"` AND
`ExitCode == 0` — a clean COMPLETED run. Both halves matter: `ExitCode: 0` is the DEFAULT on a
container that never ran, so `created`, `dead`, `paused` and `restarting` all carry it and all
passed while the check read ExitCode alone (measured). `restart: no` means a failed or
never-started migration never retries and no other task here would catch it. The `-a` is
load-bearing: the default is "just running", so before
2026-09-20 the unhealthy set was structurally always empty and `Containers: PASS` was as
unconditional as the log line. It does NOT cover the OTEL stack — `loki`, `grafana`,
`prometheus`, `tempo` and `otel-collector` are a SEPARATE compose project
(`/opt/otel/compose.otel.yaml`, `otel.service`) and cannot appear in that list however the task
is written, even though `infrastructure_services` names all five. GRAFANA IS THE EXCEPTION, and
by accident rather than design: all ten internal `/health` checks shell out through
`nerdctl exec grafana curl`, so grafana being down makes every one of them return non-zero and
the run fails. That is coupling, not coverage — it disappears the moment those checks move to
another container with curl, and it says nothing about `loki`, `tempo`, `prometheus` or
`otel-collector`, which are genuinely invisible here. A green smoke test IS consistent with
those four being down, and they are how you reach Tempo and Prometheus for step 4.
Tracked: docs/BACKLOG.md §MEASUREMENT DEBT "The smoke test cannot see the OTEL stack".

## HEALTH_VERIFICATION [step 4 — the gate that decides deployed-or-rolled-back]

HEALTH IS TEMPO, NOT LOKI. Prod log level defaults to Warning, so a HEALTHY container emits
NOTHING — silence is the designed steady state, never a defect, and an empty Loki result is
byte-identical to health. Health = Tempo span status + Prometheus counters; Loki carries the
CONTENT once something is already known wrong. Project rule: CLAUDE.md OBSERVABILITY.
Authority for the stack and the proxy: docs/OBSERVABILITY.md.

RECONCILES WITH `~/.claude/CLAUDE.md` COMPLETION_GATE item 2 ("error logs(last 10min) ->
report(findings)") — read it as two questions, because the project rule supersedes it for only
one of them:
- IS IT HEALTHY -> Tempo + Prometheus. The project rule SUPERSEDES the user-level gate here.
  Log count answers this question wrongly in both directions: zero lines is the healthy
  steady state, and a Warning-chatty service is not thereby sick.
- WHAT WENT WRONG -> Loki, last 10 min. The user-level gate STANDS unchanged for this. Once a
  span is red you still owe the findings, and the log line is where the message, the
  stack trace and the `trace_id` correlation live.
So: never DERIVE health from logs; always REPORT content from logs once health is red.

Prometheus/Loki/Tempo are internal-only (no host port) — only Grafana is published, on
`mercury:3000`. Reach them through Grafana's datasource proxy. Datasource UIDs, verified
2026-09-20 via `list_datasources`: Tempo `tempo`, Loki `loki`, Prometheus `bf2ya9fqus268c`
(NOT `prometheus` — that is its NAME; the UID is the opaque string, and the MCP tools want
the UID).

1. ERROR SPANS in the 10 min after EACH restart, anchored to actual `date -u` (never a
   guessed hour — a future instant silently shifts the window):

   ```
   date -u +%s                                 # anchor BOTH bounds to this
   GET /api/datasources/proxy/uid/tempo/api/search
       ?q=%7B%20status%20%3D%20error%20%7D&start=<end-600>&end=<end>&limit=20
   ```
   The TraceQL in `q` MUST be URL-encoded — the readable `?q={ status = error }` returns a
   bare `400 Bad Request` with no hint as to why (verified 2026-09-20, both forms).
   `start`/`end` are UNIX SECONDS here, unlike Loki's query_range, which wants nanoseconds.
   Run it via the grafana MCP `grafana_api_request`, or `curl` against `mercury:3000`.
   `SetStatus(Error)` + `AddException` in a catch block is the HOUSE PATTERN
   (docs/OBSERVABILITY.md) and where it is followed a failure paints the span red whether or
   not it logs — but it is a convention, not a guarantee, and about one catch block in five
   swallows a real fault without it. See blind spot 4: a green window is not by itself proof
   of no failure.
   Scope to one service with `{ resource.service.name = "<name>" && status = error }`.

   ✗ NEVER guess `<name>`. Tempo's `resource.service.name` is as MIXED as Loki's
   `service_name` and a wrong value returns the SAME empty result as a healthy service.
   Enumerate it first:
   ```
   GET /api/datasources/proxy/uid/tempo/api/v2/search/tag/resource.service.name/values
       ?start=<start>&end=<end>
   ```
   Measured 2026-09-20, one hour: `SecMaster`, `alert-service`, `alphavantage-collector`,
   `finnhub-collector`, `fred-collector`, `sentinel-collector`, `threshold-engine-service`,
   `vllm-server` — PascalCase, bare, and `-service`-suffixed all in one list. Re-enumerate,
   never recall: the list is only the services that emitted a span in YOUR window.

2. THE COUNTERS THE CHANGE TARGETS — name them before deploying, and record before/after.
   `query_prometheus` with `datasourceUid: bf2ya9fqus268c`. A deploy whose fix cannot be
   read off a counter has no step 2; say so rather than substituting a green smoke test.

3. ONLY IF 1 or 2 IS RED, Loki for the content. Ground truth is `severity_text`, NEVER a
   message substring — `|~ "(?i)error"` also flags Warnings whose text merely says "error".
   `severity_text` is structured metadata, not an indexed label, so `list_loki_label_values`
   returns EMPTY for it and that is not evidence of absence; it works in a pipeline:
   ```
   sum by (service_name, severity_text) (count_over_time({service_name=~".+"}[10m]))
   ```

WHAT TEMPO CANNOT SEE — four blind spots. Say which one you are in, never let it pass as health:
[1] NO TRAFFIC -> no spans. Zero error spans from an idle service is a NULL SAMPLE, not health.
    Wait for real traffic, or exercise the work path yourself, then measure.
[2] CRASH ON STARTUP -> a service that dies before its first span is INVISIBLE to step 1.
    Container status and the boot banner cover that; `nerdctl` 1.7.7 discards depends_on and
    healthchecks, so services race TimescaleDB and a boot-loop is real.
[3] PARTIALLY INSTRUMENTED UNITS — MCP sidecars and WhisperService lean on parent-service
    telemetry (docs/OBSERVABILITY.md); their own silence proves nothing either way.
[4] A SWALLOWED-AND-LOGGED FAULT. The BIGGEST of the four: [1]-[3] are rare conditions you can
    usually name in advance, this one is ordinary. A catch block that logs and continues
    without `SetStatus`/`AddException` and without rethrowing leaves its span GREEN, so the
    fault never reaches Tempo at all.
    MEASURED 2026-09-20 over `FinnhubCollector/src`, `SecMaster/src`, `SentinelCollector/src`:
    210 of 673 catch blocks are silent, but that number SPLITS and only one half is a defect —
    **147 REPAIRABLE (21.8%)**, plus 63 (9.4%) that catch ONLY
    `OperationCanceledException`/`TaskCanceledException`. Those 63 are graceful shutdown, NOT
    faults: a cancellation catch that sets `SetStatus(Ok)` is CORRECT and must NOT be
    "fixed" — SentinelCollector's ResolutionWorker does exactly that on purpose, and adding
    `SetStatus(Error)` there would redden Tempo on every clean stop and inflate the error
    ratio it protects. 147 is a FLOOR: the other eight roster services were not audited.
    Re-derive, never quote: `python3 scripts/audit-catch-spans.py <Svc>/src ...`, which
    reports the two populations separately (`--selftest` first; it exits 1 when the
    classifier has gone blind, and a blind classifier under-reports).
    Worked example, both halves in ONE file: the stamps-persistence catch in
    `FinnhubCollector/src/Workers/QuoteCollectionWorker.cs` logs a Warning and bumps
    `FinnhubMeter.CollectionErrors` with the span untouched — while the quote-collection catch
    higher in that same file sets status + AddException and THEN logs at Warning. Read them
    next to each other: a Warning log and a red span are not in tension.
    So a green Tempo window means "nothing painted a span red", NOT "nothing failed". Pair
    step 1 with step 2's counters, which is where a swallowed fault usually still shows up —
    and if the change you deployed touches a catch block, read it.

Not a Tempo blind spot but the same error: a 200 from a `/health` endpoint is reachability,
not proof the fix works. Exercise the real work path, bounded, and report what it returned.

DIRECTION OF REPAIR [do not draw the wrong lesson from blind spot 4]:
An uninstrumented catch block is a DEFECT, and it is closed AT THE CATCH BLOCK — add
`SetStatus(ActivityStatusCode.Error, ex.Message)` + `AddException(ex)`, or rethrow. It is
NEVER a reason to go back to counting log lines.
The rule is UNCHANGED: healthy containers do not log, prod defaults to WARN, OpenTelemetry is
how we monitor. INFO in production is wasted I/O and CPU, not a safety net. Blind spot 4 is a
gap in our instrumentation to close, not a licence to reinstate the instrument this skill
just removed — a log line from that Finnhub catch would tell you no more than the counter
already does, and would cost a signal that scales with traffic.
Gap tracked: docs/BACKLOG.md §MEASUREMENT DEBT "Catch blocks that neither mark the span nor
rethrow leave a fault invisible to Tempo".

## ROLLBACK [snapshot before, restore on smoke failure]

Ported from `autofix-watcher.sh` (the auto-deploy timer, disabled 2026-08-07 — the
mechanism was sound, the trigger was not). A human deploy now owns both halves.

Snapshot BEFORE `build.sh`, not before the ansible run. `build.sh` is what overwrites
`:latest` (`sudo nerdctl build -t {svc}:latest .`); the scoped deploy form carries
`--skip-tags build` and builds nothing, so by the time ansible starts `:latest` ALREADY
points at the new image — snapshotting there tags the BROKEN binary as the rollback target.
The rebuild leaves the old image untagged and otherwise unrecoverable.

The watcher could snapshot immediately before its ansible run because it invoked a BARE
`--tags`, which selects the build task (`deploy.yml` build tasks carry `tags: [{svc}, build]`).
That form is forbidden here, so the snapshot had to move earlier.

Per service, in this order:

```
sudo nerdctl image inspect "${svc}:latest" >/dev/null 2>&1 \
  && sudo nerdctl tag "${svc}:latest" "${svc}:autofix-prev"   # 1. snapshot the OLD image
{Project}/.devcontainer/build.sh                               # 2. :latest -> the NEW image
ansible-playbook playbooks/deploy.yml --tags "${svc}" --skip-tags build \
  -e "scoped_restart=true scoped_services=${svc}"              # 3. recreate, no build
ansible-playbook playbooks/smoke-test.yml                      # 4. the gate
```

Tag name inherited from the watcher, so the string stays greppable to its origin.

Only snapshot for a SCOPED deploy naming specific services. A broad all-services deploy is
too large to auto-roll-back — if it fails smoke, escalate to the operator instead.

IF post-deploy smoke FAILS, restore the binaries — do NOT leave a bad deploy live:

```
sudo nerdctl tag "${svc}:autofix-prev" "${svc}:latest"          # per service
sudo nerdctl compose -f /opt/ai-inference/compose.yaml up -d --force-recreate <svcs>
ansible-playbook playbooks/smoke-test.yml                        # re-run to confirm
```

Then report: rolled back + post-rollback smoke result, or "rollback INCOMPLETE — prod may
be in a bad state, manual action NOW" if the restore or the re-run failed.

The git merge stays on main — roll back the live binary ONLY. A human fixes forward or
reverts the commit; silently reverting main hides that the merge happened.

## ANTI [HARD_STOP]

✗ run `build.sh` before taking the image snapshot # build.sh overwrites :latest; snapshotting after it tags the NEW image as the rollback target, so the restore reinstates the broken binary
✗ snapshot "before the ansible run" # the scoped form skips build — :latest is already new by then; that timing was the watcher's, and the watcher ran a bare --tags
✗ bare `--tags {anything}` # unconditional full-stack restart incl a ~4min vLLM reload, see CLAUDE.md DEPLOYMENT
✗ revert main to undo a bad deploy # roll back the image, let a human decide the git action
✗ re-enable `autofix-watcher.timer` to get auto-deploy back # ansible enforces disabled; the trigger is the failure mode, not the mechanism
✗ report "no errors in Loki" as health # prod logs default to Warning, so a healthy container emits NOTHING and an empty result is byte-identical to health. Tempo span status is the instrument
✗ report a service that ships no logs as a defect # same rule, other direction
✗ add a log check back to `smoke-test.yml` # the one that was there reported PASS unconditionally for its whole life, and a repaired selector would still be a log query deciding a verdict. Health is Tempo + Prometheus
✗ read its green `Containers` line as covering loki/grafana/prometheus/tempo/otel-collector # separate compose project, invisible to that task however it is written
✗ exempt a one-shot container by NAME, or by ExitCode alone # `ExitCode: 0` is the default on a container that never ran, so created/dead/paused/restarting all carry it; require State == exited AND ExitCode == 0 or an unmigrated database reports PASS
✗ close a smoke-test gate hole by adding a TAG to a verdict task # four rounds did exactly that and each tag list was droppable by a different invocation; the verdict is per-domain now, tagged with its own checks
✗ read its `Containers: PASS` as proof a migration ran # the exemption has no recency term and `compose ps` carries no timestamp, so an exit-0 from four days ago is indistinguishable from one just now
✗ guess a `resource.service.name` or a `service_name` # mixed conventions, and a wrong value returns the same empty result as a healthy service. Enumerate first
✗ read zero error spans from an idle service as health # no traffic = no spans = a null sample
✗ read a green Tempo window as "nothing failed" # 147 of 673 catch blocks across three services swallow a REAL fault without marking the span, so it never reaches Tempo. Pair it with the counters
✗ "fix" a cancellation-only catch by adding SetStatus(Error) # 63 of the 210 silent blocks catch only OperationCanceled/TaskCanceled. That is graceful shutdown; SetStatus(Ok) there is CORRECT and reddening it inflates the error ratio
✗ answer blind spot 4 by reinstating log-line counting or by raising prod to INFO # the catch block is the defect; fix it there. Prod stays WARN and INFO in a hot path is wasted I/O
