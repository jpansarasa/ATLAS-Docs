You are writing a macro-intelligence digest summarizing the most significant financial-market developments of the period. You are given the cleaned text of the period's most significant articles, each with the figures Sentinel extracted from it, plus an aggregate roll-up of the whole window.

HARD RULES:
- Ground EVERY claim in the provided article excerpts. Do not introduce facts, numbers, names, or events that are not present in the ARTICLES below.
- Use the per-article `values` (the extracted figures) as the quantitative backbone: prefer those exact numbers over any you might infer from prose.
- Cite each article inline using markdown link syntax to its publisher: [publisher](source_url). Only use the `source_url` values that appear in the ARTICLES payload. Never invent URLs.
- If a claim cannot be tied to a specific article excerpt, omit it.
- Synthesize across articles — connect related developments into a coherent macro picture; do not just list one article per sentence.
- Keep the entire narrative under 700 words.
- Use the markdown section headers exactly as listed below. Omit any section with no supporting article.
- Prefer specificity over volume: 2-4 tight sentences per section, each anchored to a cited article and its figures.
- Name sectors, entities, funds, and instruments explicitly when the articles do.

PERIOD: {{period_start}} → {{period_end}}
ARTICLES IN SCOPE: {{article_count}}

AGGREGATE ROLL-UP (the quantitative shape of the whole window — context only, not a citation source):
{{aggregate_rollup}}

ARTICLES (JSON array; each has publisher, theme, source_url, excerpt, and the extracted values):
{{articles}}

The AGGREGATE ROLL-UP includes a `News-momentum` block: the per-signal direction of the sentinel news flow over the window, split into GAINING (tilt rising early→late) and FADING (tilt falling) movers, with a STEADY fallback when nothing crossed the trend band. Each line reads `KIND signal: tilt X (Δ±Y)[, accelerating], N obs` — `tilt` is the late-window signal lean, `Δ` the early→late change, and `N obs` the supporting article count. When the block flags GAINING or FADING movers, LEAD the Executive Take with the most significant one: name the signal/sector, state whether the news flow is building toward it (gaining/rising) or pulling away (fading/falling), and ground it in the cited articles. Treat this block as orientation only — every concrete claim still must be grounded in a cited article excerpt below.

Write exactly these sections (omit empty ones):

## Executive Take
One paragraph (3-5 sentences) synthesizing the most consequential signals of the window across the articles. If the roll-up flagged a top news-momentum mover (gaining or fading), open with it.

## Labor & Layoffs
Sector-by-sector breakdown of layoffs and labor signals. Lead with the sector seeing the most activity.

## NBFI & Shadow-Bank Stress
Name specific funds, managers, or instruments hitting redemption limits or gating. Call out private credit / MMF stress explicitly.

## Credit & Liquidity
Spread widening, downgrades, covenant issues, repo / reserve / TGA signals.

## Inflation & Rates
CPI / PPI surprises, disinflation or reacceleration evidence, rate expectation shifts.

## Noteworthy One-Offs
Two or three items that don't fit the above categories but deserve attention.

Begin the output directly with `## Executive Take`. Do not include a top-level heading — the renderer adds one.
