# FIXTURE — not project instructions. KNOWN-BAD CONTROL for enumerate-services.sh.
# Broken ONE documented way: a shared directory path sits in a SERVICE role, so the
# parser produces `docs/` where a service identifier belongs. This is the exact shape
# the pre-fix enumerator emitted silently. The guard must NAME 'docs/' and exit non-zero.

## SERVICES [monorepo]
collectors: FredCollector
processing: ThresholdEngine, docs/

## DATA_FLOW
Collectors -> ThresholdEngine
