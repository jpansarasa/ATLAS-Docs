# Story 6 — v1 GBNF validation against llama-server (success)

Captured 2026-05-21 after the grammar-bounds fix landed
(`fix(dsl-poc/phase2): v1 GBNF bounds — pass llama.cpp parser`).
Companion artifacts: `06-grammar-validation-success.json` (full
request/response/log envelope), `06-grammar-validation-success.log.txt`
(raw `nerdctl logs` excerpt during the call; `.txt` extension because
`*.log` is gitignored — same convention as `05-llama-server-logs.txt`).

## Setup

- Endpoint: `POST http://localhost:11437/completion`
- Backend: `ghcr.io/ggml-org/llama.cpp:server` (binary `b9246-871b0b70f`),
  CPU-only, `--parallel 1`, `--ctx-size 16384`, model
  `qwen3-30b-a3b-instruct-2507-q4_K_M.gguf`.
- Grammar: `docs/grammars/cod-dsl-v1.gbnf` (v1.1 bounds — block-list
  `{0,128}`, value-char `{0,255}`, line-rest `{0,255}`, string-content
  `{0,255}`, qual-val-char `{0,127}` unchanged).
- Prompt: small qwen3 prompt asking for `DSL: v1` header + one ENT
  block from a one-line Apple-revenue article (full text in the
  `.json` artifact, `request.prompt`).
- `n_predict`: 256.

## Acceptance gate

| Criterion | Expected | Observed | Pass |
|---|---|---|---|
| HTTP status | 200 | 200 | yes |
| `parse: error parsing grammar` in `sudo nerdctl logs --since 5s llama-server` | absent | absent | yes |
| Output begins with v1 grammar's first valid token sequence (`DSL: v` + digit) | yes | `'DSL: v1\nSOURCE: Apple Inc. (NASDAQ: AAPL'` | yes |

All three Story 6 acceptance criteria met. The grammar now actually
enforces shape — compare to the 04-* artifacts where the silent
fall-through let the model emit free prose.

## Output shape (full)

```
DSL: v1
SOURCE: Apple Inc. (NASDAQ: AAPL) reported Q1 2026 revenue of $123.9B.
TIMESTAMP: 2026-04-01
ENT type:Company = "AAPL" / CompanyName (Attribute= "Apple Inc.")
ENT type:Revenue = "$123.9B" / RevenueAmount (Attribute= "Q1 2026")
ENT type:Currency = "$" / CurrencySymbol (Attribute= "USD")
ENT type:Period = "Q1 2026" / PeriodName (Attribute= "Q1 2026")
ENT type:ReportingPeriod = "Q1 2026" / ReportingPeriodName (Attribute= "Q1 2026")
ENT type:FinancialMetric = "Revenue" / MetricName (Attribute= "Revenue")
ENT type:FinancialMetric = "$123.9B" / MetricValue (Attribute= "Revenue")
ENT type:FinancialMetric = "Q1 2026"
```

(Cut off at `n_predict=256`; `stop_type=limit`. Continuation would
have been more ENT blocks. Quality of the *content* the model emits
is irrelevant to Story 6 — the grammar polices SHAPE only, per
Phase 2 plan §5.1.)

## Notes for follow-up phases

- The model emitted `ENT type:Company` style IDs (with `type:` prefix
  treated as a colon-separated two-segment ent-id). The v1 grammar's
  `ent-id ::= ent-id-seg (":" ent-id-seg)?` accepts this; the Lark
  parser will reject if `type` isn't a documented prefix in the v1
  spec. Surface for the next prompt-tuning pass — the grammar is
  doing its job and pushing this to the semantic layer, not silently
  swallowing it.
- The output `ENT ` followed by alias-clause-before-type pattern
  (`= "AAPL" / CompanyName`) matches the v1 grammar's
  `ent-alias-clause? " / " ent-type` rule. Verbatim-first anchor
  guidance (`subject` / `source_span` before derived data slots) is
  prompt-level, not grammar-level — captured in the grammar header
  comment for the next prompt revision.
- The cumulative parse cap (#2 in the v1.1 file's `value` rule
  comment) is binary `b9246`-specific; if the upstream image is
  bumped, re-run this validation as part of the bump PR.
