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

The AGGREGATE ROLL-UP includes a `News-bias-vs-matrix` block listing the period's DIVERGENCEs (where the news collection bias leans against the FRED hard anchor, or leads a flat matrix), COVERAGE-GAPs (matrix carries an anchor but the news flow was silent), and EMERGENT narratives (news volume spiked while the matrix is still flat). When that block flags a DIVERGENCE, LEAD the Executive Take with it: name the signal/sector, state which way the news leans vs the matrix, and ground it in the cited articles. If only coverage gaps or emergent narratives are flagged, note the most significant one. Treat this block as orientation only — every concrete claim still must be grounded in a cited article excerpt below.

Write exactly these sections (omit empty ones):

## Executive Take
One paragraph (3-5 sentences) synthesizing the most consequential signals of the window across the articles. If the roll-up flagged a top news-bias-vs-matrix divergence, open with it.

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
