# azure_oracle_client — Phase 0.4

# Objective
Write `/home/james/ATLAS/SentinelCollector/scripts/azure_oracle_client.py`: a resumable, budget-guarded CLI that labels JSONL examples against Azure Foundry Claude deployments, logs every call to a ledger, and kills itself on cost overrun.

# Inputs
- Deploy target: `/home/james/ATLAS/SentinelCollector/scripts/azure_oracle_client.py` (new file)
- Env file: `/home/james/.azure-foundry-keys` exporting `AZURE_FOUNDRY_ENDPOINT` (`https://jpansarasa-eus2-aif.services.ai.azure.com/anthropic/`) and `AZURE_FOUNDRY_API_KEY`.
- Ledger: `/opt/ai-inference/training-data/azure-oracle-ledger.jsonl` (append-only, JSONL).
- Kill switch: `/opt/ai-inference/training-data/azure-oracle-KILL` (file existence = halt).
- Venv required: create/use `/home/james/ATLAS/SentinelCollector/scripts/.venv`, `pip install anthropic` (Azure-compatible client).
- Models: `claude-opus-4-7`, `claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5`.
- Pricing (USD per 1M tokens): Opus 4.7/4.6 = $15 in / $75 out. Sonnet 4.6 = $3 / $15. Haiku 4.5 = $0.80 / $4.

# Contract (embed in docstring)
```
CLI:
  python3 azure_oracle_client.py \
    --model claude-{opus-4-7|opus-4-6|sonnet-4-6|haiku-4-5} \
    --input <jsonl_examples> --output <jsonl_labels> \
    --budget-alert 500 --budget-halt 2000 \
    --run-label <human-readable>

Behavior:
- Read input line-by-line. Skip lines whose sha256 is already in <output> (resume).
- Call Azure Anthropic-compatible endpoint for each line, write result as JSONL.
- Client-side rate limit: 500,000 tpm soft cap; sleep to respect.
- Append ledger row per request: {ts_iso, run_label, model, input_tokens, output_tokens, cost_est_usd, msg_id, duration_ms, input_sha256}.
- Rolling 1h cost >= --budget-alert → stderr WARN + ntfy (curl atlas-claude-ask, Priority:high).
- Rolling run cost >= --budget-halt → touch kill-switch, exit 2.
- At start of every request: check kill-switch → exit 2 immediately.
- Graceful: SIGINT flushes ledger + output.
```

# Commands
1. Create venv; install `anthropic` (the SDK supports Azure via base_url override).
2. Implement module with functions: `load_env`, `cost_estimate(model, usage)`, `append_ledger(row)`, `rolling_cost(window_hours)`, `rate_limit_sleep()`, `publish_ntfy(title, body)`, `main()`.
3. Smoke test:
   - Build tiny input: `/tmp/oracle_smoke.jsonl` with 3 rows: `{"prompt":"Label: 'AAPL price target $250'"}` (or similar minimal).
   - Run with `--model claude-haiku-4-5 --budget-alert 1 --budget-halt 5 --run-label phase0.4-smoke`.
   - Expect 3 output rows + 3 ledger rows, total cost <$0.05.

# Acceptance
- Script exits 0 on smoke.
- Output JSONL has exactly 3 rows, each with `label` and `usage` fields.
- Ledger gains exactly 3 new rows, all model=haiku-4-5, cost_est_usd non-null.
- Re-running with same input+output is a no-op (resume skips all 3).
- Touching kill-switch during run causes clean exit 2.

# Report shape
- `script_path`: absolute
- `venv_path`: absolute
- `smoke`: `{input_rows:3, output_rows:3, ledger_rows_added:3, labels:[...3 strings...], total_cost_usd:float}`
- `resume_test`: `{second_run_rows_added:0}`
- `killswitch_test`: `{exit_code:2, partial_rows_flushed:int}`
- `errors`: `[]` | list
