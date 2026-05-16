# WhisperService/config

Runtime configuration for the WhisperService transcription daemon. Mounted into the container; values can be overridden by `WHISPER_*` environment variables (env wins over file).

## Files

| File | Purpose |
|---|---|
| `whisper.yaml` | YAML configuration consumed at startup. Covers Whisper model selection, worker concurrency, storage paths, host/port, log level, and OTEL exporter. See the file header for the full key list and the per-key comments for valid options. |

## Common edits

| Goal | Key(s) |
|---|---|
| Switch to GPU inference | `device: cuda` + `compute_type: float16` |
| Bump throughput | `num_workers: <N>` (default 8) — capped by available CPU/GPU |
| Quality vs latency tradeoff | `beam_size: <N>` (higher = slower but better) |
| Change the model tier | `model: tiny | base | small | medium | large-v2 | large-v3` |
| Keep source video after transcription | `keep_videos: true` |

## Override via environment

Every key has a corresponding `WHISPER_*` env var (e.g. `WHISPER_MODEL`, `WHISPER_DEVICE`, `WHISPER_NUM_WORKERS`). Use env vars for per-deploy overrides; commit YAML edits when changing the default for everyone.

## What does NOT belong here

- **Secrets.** WhisperService has no per-instance secrets at this time. If that changes, route via ansible-vault → environment variable, not YAML.
- **Per-job parameters.** Transcription jobs carry their own request-level params via the HTTP API; only daemon-wide defaults live here.

## See Also

- [WhisperService](../README.md) — service README
