You are writing a macro-intelligence digest summarizing Sentinel's recent collection.

HARD RULES:
- Cite every factual claim inline using markdown link syntax: [source_name](source_url).
- Only use URLs that appear in the OBSERVATIONS payload below. Never invent URLs.
- If a claim has no matching citation row, omit the claim.
- Use the `certainty` field to set tone:
  - `Definite` — assertive ("unemployment rose to 4.3%")
  - `Expected` — forecast-hedged ("is expected to reach 5%")
  - `Speculative` — hedged ("may reach", "could approach")
  - `Conditional` — conditional ("if tariffs persist, ...")
- Keep the entire narrative under 600 words.
- Use markdown section headers exactly as listed below. Omit any section that has no observations.
- Prefer specificity over volume: 2-4 tight sentences per section, each anchored to a cited observation.
- Name the sectors and entities explicitly when discussing Layoffs and NBFI.

PERIOD: {{period_start}} → {{period_end}}
OBSERVATION COUNT: {{observation_count}} across {{article_count}} articles from {{publisher_count}} publishers
THEMES WITH DATA: {{themes}}

OBSERVATIONS (JSON array; each row has source_url for citation):
{{observations_json}}

Write exactly these sections (omit empty ones):

## Executive Take
One paragraph (3-5 sentences) summarizing the most consequential signals of the window.

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
