# preflight_backups — Phase 0.3

# Objective
Produce verified, restorable backups of Sentinel state (DB schema `sentinel`, ZFS dataset, config, prompts) before any v2 destructive work. Every artifact must be proven recoverable, not just written.

# Inputs
- ISO timestamp: `ISO=$(date -u +%Y%m%dT%H%M%SZ)` — compute once, reuse everywhere.
- ZFS dataset: `nvme-fast/timeseries`
- DB: container `timescaledb`, superuser `ai_inference`, database `atlas_data`, schema `sentinel`.
- Backup dir: `/opt/ai-inference/backups/` (create if missing, `sudo mkdir -p`).
- Training-data dir: `/opt/ai-inference/training-data/`
- Plan ref: `/home/james/.claude/plans/we-need-to-fix-twinkly-dragon.md` §0.3

# Commands
Run each, capture exit code. Abort on any non-zero.

1. **ZFS snapshot**
   `sudo zfs snapshot nvme-fast/timeseries@pre-v2-redesign-${ISO}`
   Verify: `sudo zfs list -t snapshot | grep pre-v2-redesign-${ISO}`

2. **pg_dump (custom format, sentinel schema only)**
   `DUMP=/opt/ai-inference/backups/sentinel-pre-v2-${ISO}.dump`
   `sudo nerdctl exec timescaledb pg_dump -U ai_inference -d atlas_data -n sentinel -F c -f /tmp/dump.bin`
   `sudo nerdctl cp timescaledb:/tmp/dump.bin "$DUMP"`
   `sha256sum "$DUMP" > "$DUMP.sha256"`

3. **Restore verify (throwaway DB)**
   - `sudo nerdctl exec timescaledb psql -U ai_inference -d postgres -c 'CREATE DATABASE atlas_data_restore_test;'`
   - `sudo nerdctl cp "$DUMP" timescaledb:/tmp/verify.bin`
   - `sudo nerdctl exec timescaledb pg_restore -U ai_inference -d atlas_data_restore_test /tmp/verify.bin`
   - Row-count compare (MUST match exactly):
     - `PROD=$(sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data -tAc "SELECT COUNT(*) FROM sentinel.extracted_observations")`
     - `REST=$(sudo nerdctl exec timescaledb psql -U ai_inference -d atlas_data_restore_test -tAc "SELECT COUNT(*) FROM sentinel.extracted_observations")`
     - Also compare `sentinel.raw_content` counts.
   - Cleanup: `DROP DATABASE atlas_data_restore_test;`

4. **Config + prompt backups**
   - `sudo cp -a /opt/ai-inference/compose.yaml /opt/ai-inference/backups/compose-pre-v2-${ISO}.yaml`
   - `sudo cp -a /home/james/ATLAS/SentinelCollector/src/appsettings.json /opt/ai-inference/backups/appsettings-pre-v2-${ISO}.json`
   - `sudo tar czf /opt/ai-inference/backups/prompts-pre-v2-${ISO}.tgz -C /opt/ai-inference/prompts sentinel/`

5. **Quarantine training-data export**
   `python3 /home/james/ATLAS/SentinelCollector/scripts/export_symbol_training_data.py --output /opt/ai-inference/training-data/quarantine-snapshot-pre-v2-${ISO}.jsonl`
   Count rows: `wc -l <jsonl>`

# Acceptance
- ZFS snapshot listed.
- pg_dump file exists, sha256 written, restore row-counts match prod exactly (both tables).
- All 3 config/prompt backup files exist, non-zero size.
- Quarantine JSONL exists, row count ≥1000 (sanity).
- No step returned non-zero.

# Report shape
- `iso`: `${ISO}`
- `zfs_snapshot`: full name
- `pg_dump_path`: absolute path
- `pg_dump_sha256`: hex
- `restore_test`: `{prod_obs, restored_obs, prod_raw, restored_raw, match: bool}`
- `config_backups`: `[compose_path, appsettings_path, prompts_tgz_path]`
- `quarantine_jsonl`: `{path, row_count}`
- `errors`: `[]` | list
