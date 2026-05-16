# SentinelCollector/config

Per-environment runtime configuration overrides for SentinelCollector. Mounted into the container at `/app/config` when populated.

## Current contents

Empty except for `.gitkeep` — SentinelCollector reads its runtime configuration entirely from environment variables (see the parent README's Configuration section) and from prompt files at the container path `/prompts` (bind-mounted from the host `/opt/ai-inference/prompts/sentinel` per CLAUDE.md `DATA_ML_CONTEXT: PROMPTS` — edit the host file, not the container).

This directory is reserved for future per-environment overrides (e.g. environment-specific RSS feed lists, validation-trigger rule files, or backend selectors) that don't fit the env-var or prompt-mount model.

## What does NOT belong here

- **Prompts.** Live at `SentinelCollector/src/prompts/` and are host-mounted at `/prompts` so edits survive container restart. See CLAUDE.md `DATA_ML_CONTEXT: PROMPTS`.
- **Training data / LoRA artifacts.** Live under `/opt/ai-inference/training-data/` and `/opt/ai-inference/loras/` outside the repo.
- **Secrets.** Use ansible-vault and inject via environment variables; never commit secrets to this directory.

## See Also

- [SentinelCollector](../README.md) — service README + full Configuration table
- [SentinelCollector/src/prompts](../src/prompts/) — LLM prompt templates (host-mounted)
- [SentinelCollector/scripts](../scripts/README.md) — operator + research scripts
