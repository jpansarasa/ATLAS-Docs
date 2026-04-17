---
name: deploy
description: ATLAS deployment workflow - smoke tests, PR, health verification via Loki, prompts mount check, VRAM report
---

# Deploy Workflow

Applies `VERIFY_TEST.COMPLETION_GATE` and `EVIDENCE_GATE` from CLAUDE.md.

1. Run smoke tests BEFORE declaring done — report pass/fail counts
2. Open PR with test results in description
3. After merge, verify deployment health via Loki (errors in last 10 min)
4. Check for volume-mount overrides on /prompts paths
5. Report VRAM/memory status post-deploy
6. List configs/gates touched (no silent workarounds around ansible gates)
