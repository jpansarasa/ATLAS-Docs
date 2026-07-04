You are writing a macro-intelligence digest summarizing the most significant financial-market developments of the period. Your PRIMARY input is the AGGREGATE ROLL-UP below — a structured aggregation of EVERYTHING collected in the window (every observation is counted in it): the signal×sector matrix state, sector news volumes, news-signal momentum, notable market moves, and the cross-collector scoreboard. The ARTICLE EXCERPTS that follow are SECONDARY supporting color: use them for quotes, names, and concrete specifics.

PERIOD: {{period_start}} → {{period_end}}

AGGREGATE ROLL-UP (PRIMARY — the quantitative shape of the whole window):
{{aggregate_rollup}}

ARTICLE EXCERPTS (SECONDARY — supporting color; {{article_count}} articles; JSON array, each with publisher, sector, source_url, excerpt, and the extracted values):
{{articles}}

HOW TO READ THE AGGREGATE ROLL-UP:
- `Sector matrix` lines read `Sector: net ±X, N signals[, regime R][, M articles] [top: signal ±v, …]` — the latest signal×sector matrix state. `net` is the sector's summed signal lean (+ bullish / − bearish), `N signals` the distinct contributing signals, `regime` the published regime label when one exists, `M articles` the window's news volume for that sector. Sectors are listed hottest-first (matrix lean weighted over news volume) — treat that order as the sector ranking.
- `News-signal momentum` lines read `KIND signal: tilt X (Δ±Y)[, accelerating], N obs` — `tilt` is the late-window signal lean (+ bullish / − bearish), `Δ` the early→late change, `N obs` the supporting observation count. GAINING = news flow building toward the signal; FADING = pulling away; STEADY = the window's leading signals when nothing crossed the trend band.
- `Notable market moves` are hard-data series moves (not news), ranked by |Δ%|; the `Collector scoreboard` shows how much hard data arrived per source.
- `Sectors` are raw news observation counts per sector — volume only; direction comes from the matrix.
- Each article's `sector` field is its resolved ATLAS sector; `MACRO_WIDE` means it resolved to no single sector. Use the field to tie excerpts to the sector-matrix lines.

HARD RULES:
- LEAD with the matrix picture: open with the hottest and coldest sectors (net lean, regime when present), what is gaining vs fading (momentum), and the standout market moves — then use the article excerpts to substantiate and color it.
- Aggregate figures (net leans, counts, tilts, deltas, moves) may be stated directly and attributed to the window's data — they are measured, not inferred. Do not attribute them to an article.
- Every claim about a specific event, company, fund, or person must be grounded in a provided article excerpt. Do not introduce facts, numbers, names, or events that are present in neither the AGGREGATE ROLL-UP nor the ARTICLE EXCERPTS.
- Use the per-article `values` (the extracted figures) as the quantitative backbone for article-grounded claims: prefer those exact numbers over any you might infer from prose.
- Cite each article you draw on inline using markdown link syntax to its publisher: [publisher](source_url). Only use the `source_url` values that appear in the ARTICLE EXCERPTS payload. Never invent URLs.
- If the ARTICLE EXCERPTS array is empty, write the narrative entirely from the AGGREGATE ROLL-UP: describe the window's sector shape, momentum, and notable moves, omit citations, and do not fabricate article-level detail.
- Synthesize — connect the matrix picture and related developments into a coherent macro view; do not just list one item per sentence.
- Keep the entire narrative under 700 words.
- Use the markdown section headers exactly as listed below. Omit any section with no supporting data.
- Prefer specificity over volume: 2-4 tight sentences per section, anchored to aggregate figures and cited articles.
- Name sectors, entities, funds, and instruments explicitly when the data does.

Write exactly these sections (omit empty ones):

## Executive Take
One paragraph (3-5 sentences) leading with the hot and cold sectors per the sector matrix — name each sector, its net lean, and its regime when present — then the top news-momentum mover and the most significant market move. Ground specifics in cited articles where excerpts support them.

## Cross-Sector Signals
Signals moving across many sectors: use the momentum block (gaining/fading) plus any signal appearing under several sector-matrix lines. Frame the direction explicitly.

## Noteworthy One-Offs
Two or three items that don't fit the above — including MACRO_WIDE articles worth flagging.

Begin the output directly with `## Executive Take`. Do not include a top-level heading — the renderer adds one.
