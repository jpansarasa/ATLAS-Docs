# ZFS Snapshots

Automatic snapshots created before deployments for safe rollback.

## Quick Reference

**Datasets**: `nvme-fast/timeseries`, `nvme-fast/dashboard`  
**Format**: `pre-deploy-YYYYMMDDTHHMMSS`  
**Location**: Saved to `/opt/ai-inference/last-snapshot.txt`

## Commands

```bash
# Deploy with snapshot (default)
ansible-playbook playbooks/infrastructure.yml

# Deploy without snapshot
ansible-playbook playbooks/infrastructure.yml -e create_snapshot=false

# Rollback
ansible-playbook playbooks/zfs-rollback.yml -e snapshot_tag=pre-deploy-YYYYMMDDTHHMMSS

# Cleanup (keep last 3)
ansible-playbook playbooks/zfs-cleanup.yml -e cleanup_mode=auto -e keep_snapshots=3

# List snapshots
zfs list -t snapshot | grep pre-deploy
```

## Automatic Snapshots

System uses `zfs-auto-snapshot` with optimized retention:
- Frequent: 8 (2 hours)
- Hourly: 24 (1 day)
- Daily: 7 (1 week)
- Weekly: 4 (1 month)
- Monthly: 6 (6 months)

Tune retention: `sudo ./scripts/zfs-tune-snapshots.sh`

## Best Practices

✅ Always snapshot for production  
✅ Verify before cleanup  
✅ Keep 3+ recent snapshots  
❌ Don't skip snapshots in production  
❌ Don't rollback without understanding data loss
