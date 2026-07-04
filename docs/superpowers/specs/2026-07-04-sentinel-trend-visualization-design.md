# Sentinel trend visualization — design spec

Date: 2026-07-04
Status: approved design, pre-implementation
Origin: brainstorming session with visual mockups rendered from production matrix data (33 days, 2026-05-31 → 2026-07-04)

## Problem

The daily digest (`/sentinel/digests/{id}`) reports points-in-time and small-window deltas. Direction and momentum — the "shape of the line" reading a Google/Yahoo Finance chart gives — is invisible. The underlying data is fully historical (`matrix_cells` hypertable since 2025-12-31, `sector_regimes` since 2026-06-12, sentinel news-tilt rows in `macro_observations`), so this is purely a presentation gap.

Readings the redesign must serve (all four confirmed in session):

1. Direction & momentum — is a sector/signal getting better or worse, how fast
2. What changed today — which sectors/signals moved meaningfully
3. Regime context — is "contraction" new or month-three; is a reversal forming
4. News vs hard data — is news tilt leading or lagging the matrix

Style target: visualization-led with supporting prose (Visual Capitalist model) — not an infographic, not a wall of text.

## Decisions (validated on real-data mockups)

| Decision | Choice | Rejected alternatives |
|---|---|---|
| Primary sector view | Heatmap strip: sector × day grid, rows sorted hot→cold, net lean value at row end | Emphasis overlay (narrative-only), small multiples as primary (weaker cross-sector comparison) |
| Drill-down | Small-multiple panel per sector, expanded from the heatmap row | Always-expanded 11-sector scroll (4–5 phone screens) |
| Temperature semantics | **hot = red (`#e34948`), cold = blue (`#2a78d6`)**, neutral gray midpoint (`#f0efec` light / `#383835` dark) | blue=bullish finance-inverse (user explicitly flipped); green/red (CVD-unsafe) |
| Page order | Market-first: KPI tiles → heatmap index → drill-downs → narrative | Narrative-first |
| Signal detail | Per-signal dual-trend chart: cell value (black) vs daily news tilt (violet `#7a5ec4`) on a shared timeline | Contribution bars (point-in-time ranking only) |
| Rendering | Server-rendered inline SVG in C#, zero client JS; drill-down via native `<details>/<summary>` | JSON API + hand-rolled client JS; Grafana (no narrative integration) |

## Surfaces

Both served by SentinelCollector on :5091, one chart codebase.

### Daily digest — `/sentinel/digests/{id}` (redesigned)

Frozen self-contained artifact per day; charts show history as of generation time. Page order:

1. Header + window stats (one line, as today)
2. KPI tile row: hottest sector, coldest sector, top news-momentum mover — value + one-word regime each
3. Sector heat map: 11 rows × window days (default 33), each row a `<summary>`
4. Drill-down per sector (`<details>` body):
   - net-lean sparkline (diverging fill above/below zero baseline)
   - regime timeline strip, score-colored, labeled "regime · day N" (consecutive-day streak)
   - top-5 signal dual-trend charts: cell value vs news tilt, article count shown with tilt value
   - the sector's narrative paragraph + citation (moved from the old Sector Watch)
5. Executive Take (LLM narrative, once)
6. Cross-sector signals + Noteworthy one-offs (unchanged)

Removed outright:

- **Sector Watch section** — redundant with drill-down one-liners (its prose currently duplicates Executive Take nearly verbatim)
- **Raw extracted-quantity bullets** in Sector Detail ("700000000000.0000 USD — roughly $700 billion") — extraction debris, not digest content
- The old Sector Heat table and Sector Detail lists — superseded by the heatmap + drill-downs

The `.md` variant stays text-only as today (charts are HTML-only).

### Live explorer — `/sentinel/matrix?range={7d|30d|90d|max}` (new)

Always-current page from the same components: KPI tiles, heatmap, drill-downs. No digest narrative. Range switch = link + page reload (LAN-instant, keeps zero-JS). Default range 30d.

## Components (all new code in SentinelCollector)

### `TrendQueryService`

Daily-bucketed series, forward-filling gaps:

- **Sector net lean**: per day, latest cell per `(pattern_id, sector_code)` (`DISTINCT ON … ORDER BY evaluated_at DESC` within day bucket), summed per sector. Matches the digest's "net" semantics.
- **Regime history**: per sector per day, last `(regime, score)` from `sector_regimes`.
- **Per-signal daily cell value**: per day, latest cell per pattern for `(signal_identity_id, sector_code)`, averaged across patterns.
- **Per-signal daily news tilt**: from `macro_observations`, same filter semantics as `NewsMomentumQueryService` — `source_collector='sentinel' AND signal_identity_id IS NOT NULL AND value_numeric BETWEEN -1.5 AND 1.5` — daily `avg(value_numeric)` + row count. (Deliberately NOT a `:sig:` infix filter; legacy sentinel numeric rows with a signal id participate, mirroring the existing digest momentum view. See SentinelCollector AGENT_README DISTINCTIONS.)

### `TrendSvgRenderer`

Pure string-in/string-out functions (unit-testable, no I/O): heatmap strip, lean sparkline, regime strip, dual-trend chart, KPI tile row. Diverging color interpolation lives here in one place.

### `DigestRenderer` (restructured) + new `/sentinel/matrix` endpoint

DigestRenderer composes the new blocks in the market-first order; the matrix endpoint reuses the same blocks without the narrative sections.

## Palette & accessibility

- Colors defined once as CSS custom properties; dark variant via `@media (prefers-color-scheme: dark)` using the validated dark steps (hot `#e66767`, cold `#3987e5`, dark surface `#1a1a19`).
- Numeric values always printed beside color-coded marks (heatmap row-end lean, KPI tiles, signal values) — color never carries meaning alone.
- Dual-trend legend rendered once per drill-down panel; series identity also carried by weight (cell = 1.8px black, tilt = 1.4px violet).
- Text stays in ink tokens, never series colors, except sign-colored values (with their +/− sign as the redundant channel).

## Error handling

- Sector with no cells in window → gray "no data" row; panel opens with whatever series exist.
- Regime history starts 2026-06-12 → strip renders from first available day; no synthetic backfill.
- Signal with cells but no tilt rows (hard-data-only signals) → cell line only, tilt legend line omitted.
- Empty matrix in window (e.g. projector Off) → page renders with explicit "no matrix data in window" notice, never a blank 500. (Card GOTCHA: zero cells + no error = check `Matrix:ObservationProjectorMode` first — the notice should name that config.)
- Malformed/absent range param → default 30d, no error.

## Testing

Business-outcome tests (house rule: RED on unfixed, no tautologies):

- Sector whose lean flips sign mid-window renders blue→red cells in day order.
- Regime streak label counts consecutive days of the current regime only.
- Dual-trend chart omits the tilt polyline when no tilt rows exist for the signal.
- Heatmap rows sort by latest net lean, hot first.
- `TrendQueryService` seeded-row tests: forward-fill across a gap day; tilt query excludes `|value|>1.5` and null-signal rows; net-lean uses latest-per-pattern-per-day, not averages.
- Digest render with empty matrix produces the explicit notice.

## Non-goals

- No JSON API, no client-side chart framework, no CDN assets.
- No Grafana dashboard (data is Timescale-queryable if wanted later; wrong shape for the narrative digest).
- No change to digest generation cadence, LLM narrative prompts, or the `.md`/ntfy outputs (except sections deleted above no longer appearing).
- No new services or compose changes — SentinelCollector only.
- No change to matrix/projector/write-path semantics; this is read-only presentation over existing tables.

## Session artifacts

Mockups (real-data SVG) persist under `.superpowers/brainstorm/2681063-1783172224/content/` (gitignored): `sector-trend-form.html`, `digest-layout.html`, `sector-drilldown.html`.
