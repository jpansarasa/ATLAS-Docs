# dsl-parser-mcp

CPU sidecar exposing the benchmark CoD DSL parser + v2.3.1 verifier over
HTTP so the production .NET SentinelCollector can ground extractions on
the same parse + word/byte verifier the benchmark scorer uses.

## Why

Re-implementing the 1808-LOC Lark/LALR parser + 639-LOC punctuation-tolerant
word verifier in C# would fork the grounding contract. The CPU CoD migration
plan (PR 1/6, task `a1d91968baa05dccc`) sidesteps the rewrite by wrapping
the Python source verbatim in a FastAPI process the .NET extractor calls
over localhost HTTP.

Build context: the `dsl/` directory under this sidecar is a verbatim copy of
`docs/benchmarks/cod-2026-05-17/dsl/`. Do not edit in-place — re-copy from
the benchmark dir if the upstream parser/verifier changes.

## Endpoints

### `POST /parse`

Parse a DSL document into a Document AST.

```json
{
  "dsl_text": "DSL: v2.3\nSOURCE: ...\nTIMESTAMP: ...\n\nENT ...",
  "input_text": "Source article body text",
  "schema": "v2.3"
}
```

`schema` ∈ `{"v1", "v2.1", "v2.3"}`. Default `"v2.3"`.

Returns:
```json
{
  "document_ast": { "dsl_version": "v2.3", "ents": [...], "evts": [...], ... },
  "parse_errors": []
}
```

Strict-parse failures land in `parse_errors[]` with `line` / `column`;
genuine server errors return HTTP 500.

### `POST /verify`

Run the v2.3.1 verifier (punctuation-tolerant word grounding + byte
verbatim) over a parsed AST.

```json
{
  "document_ast": { ... shape from /parse ... },
  "input_text": "Source article body text"
}
```

Returns:
```json
{
  "verifier_report": {
    "blocks": [...],
    "byte_verbatim_match_rate": 0.92,
    "word_verbatim_match_rate": 0.97,
    "word_verbatim_match_rate_cased": 0.94,
    "byte_denom": 41,
    "word_denom": 41
  },
  "version": "v2_3_1"
}
```

### `GET /health`

200 OK with `{"ok": true, ...}` when the parser + verifier modules
imported cleanly at startup.

## Run locally

```sh
docker run -p 3120:3120 dsl-parser-mcp:latest
curl http://localhost:3120/health
```

## Deploy

```sh
ansible-playbook deployment/ansible/playbooks/deploy.yml --tags dsl-parser-mcp,build
```

## Tests

`pytest tests/` runs during the container build (RUN pytest in
Containerfile). The six copied parser/verifier unit tests must pass
against the verbatim source; three FastAPI integration tests cover
`/health`, `/parse`, and `/verify`.

## Consumers

- PR 4 (planned): .NET `DslParserMcpClient` in
  `SentinelCollector/src/...` calls `/parse` + `/verify` from the
  v2 extractor.
