---
name: deploy
description: ATLAS deployment workflow - smoke tests, PR, health verification via Loki, prompts mount check, VRAM report
---

# Deploy Workflow

Applies `VERIFY_TEST.COMPLETION_GATE` and `EVIDENCE_GATE` from CLAUDE.md.

1. Run smoke tests BEFORE declaring done — report pass/fail counts
2. Open PR with test results in description
3. Invoke the intent-review skill on the PR diff (design-intent conformance vs the
   touched services' DECISIONS blocks) — address critical/important findings BEFORE merge
4. After merge, verify deployment health via Loki (errors in last 10 min)
5. Check for volume-mount overrides on /prompts paths
6. Report VRAM/memory status post-deploy
7. List configs/gates touched (no silent workarounds around ansible gates)
