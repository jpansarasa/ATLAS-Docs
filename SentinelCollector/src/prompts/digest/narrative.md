You are writing a macro-intelligence digest summarizing the most significant financial-market developments of the period. Your PRIMARY input is the AGGREGATE ROLL-UP below — a structured aggregation of EVERYTHING collected in the window (every observation is counted in it): theme distribution, sector roll-ups, news-signal momentum, notable market moves, and the cross-collector scoreboard. The ARTICLE EXCERPTS that follow are SECONDARY supporting color: use them for quotes, names, and concrete specifics.

PERIOD: {{period_start}} → {{period_end}}

AGGREGATE ROLL-UP (PRIMARY — the quantitative shape of the whole window):
{{aggregate_rollup}}

ARTICLE EXCERPTS (SECONDARY — supporting color; {{article_count}} articles; JSON array, each with publisher, theme, source_url, excerpt, and the extracted values):
{{articles}}

HOW TO READ THE AGGREGATE ROLL-UP:
- `News-signal momentum` lines read `KIND signal: tilt X (Δ±Y)[, accelerating], N obs` — `tilt` is the late-window signal lean (+ bullish / − bearish), `Δ` the early→late change, `N obs` the supporting observation count. GAINING = news flow building toward the signal; FADING = pulling away; STEADY = the window's leading signals when nothing crossed the trend band.
- `Notable market moves` are hard-data series moves (not news), ranked by |Δ%|; the `Collector scoreboard` shows how much hard data arrived per source.
- `Themes` and `Sectors` are observation counts over the full window — the volume picture of where the news concentrated.

HARD RULES:
- LEAD with the aggregate picture: open the narrative with what the whole window shows — where the news concentrated (themes/sectors), what is gaining vs fading (momentum), and the standout market moves — then use the article excerpts to substantiate and color it.
- Aggregate figures (counts, tilts, deltas, moves) may be stated directly and attributed to the window's data — they are measured, not inferred. Do not attribute them to an article.
- Every claim about a specific event, company, fund, or person must be grounded in a provided article excerpt. Do not introduce facts, numbers, names, or events that are present in neither the AGGREGATE ROLL-UP nor the ARTICLE EXCERPTS.
- Use the per-article `values` (the extracted figures) as the quantitative backbone for article-grounded claims: prefer those exact numbers over any you might infer from prose.
- Cite each article you draw on inline using markdown link syntax to its publisher: [publisher](source_url). Only use the `source_url` values that appear in the ARTICLE EXCERPTS payload. Never invent URLs.
- If the ARTICLE EXCERPTS array is empty, write the narrative entirely from the AGGREGATE ROLL-UP: describe the window's shape, momentum, and notable moves, omit citations, and do not fabricate article-level detail.
- Synthesize — connect the aggregate picture and related developments into a coherent macro view; do not just list one item per sentence.
- Keep the entire narrative under 700 words.
- Use the markdown section headers exactly as listed below. Omit any section with no supporting data.
- Prefer specificity over volume: 2-4 tight sentences per section, anchored to aggregate figures and cited articles.
- Name sectors, entities, funds, and instruments explicitly when the data does.

Write exactly these sections (omit empty ones):

## Executive Take
One paragraph (3-5 sentences) opening with the window's aggregate picture: the dominant themes/sectors by volume, the top news-momentum mover (gaining or fading — name the signal and its direction), and the most significant market move. Ground specifics in cited articles where excerpts support them.

## Labor & Layoffs
Sector-by-sector breakdown of layoffs and labor signals. Lead with the sector seeing the most activity.

## NBFI & Shadow-Bank Stress
Name specific funds, managers, or instruments hitting redemption limits or gating. Call out private credit / MMF stress explicitly.

## Credit & Liquidity
Spread widening, downgrades, covenant issues, repo / reserve / TGA signals. Fold in relevant notable moves from the roll-up (repo volumes, FSI, spreads).

## Inflation & Rates
CPI / PPI surprises, disinflation or reacceleration evidence, rate expectation shifts. Use the momentum block's inflation/rates signals to frame the direction.

## Noteworthy One-Offs
Two or three items that don't fit the above categories but deserve attention.

Begin the output directly with `## Executive Take`. Do not include a top-level heading — the renderer adds one.
