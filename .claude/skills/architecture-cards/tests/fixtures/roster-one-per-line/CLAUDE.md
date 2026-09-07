# FIXTURE — not project instructions. Feeds enumerate-services.sh in test-audit.sh.
# Form A: one role per line (the shape the awk was written for).

## PROJECT_OVERVIEW
name: fixture

## SERVICES [monorepo]
collectors: FredCollector, AlphaVantageCollector, NasdaqCollector, FinnhubCollector, OfrCollector, SentinelCollector
processing: ThresholdEngine
alerting: AlertService
calendar: CalendarService
metadata: SecMaster
substrate: MacroSubstrate
shared: Events/, deployment/, docs/
mcp: FredCollector/mcp, ThresholdEngine/mcp, SecMaster/mcp

## DATA_FLOW
Collectors -> ThresholdEngine
