# Specialized Extraction Prompts Plan

## Problem Statement

Current extraction uses a single generic prompt for all sources, leading to:
- Mixed metric types tagged with same symbol (monthly vs YTD vs comparisons)
- Missing context about source structure and publication cadence
- Inconsistent data quality for known, structured reports

## Current Sources

### RSS Feeds (Structured - High Value for Specialization)

| Feed | Category | Data Points | Current Issues |
|------|----------|-------------|----------------|
| Challenger Gray - Job Cuts | employment | Monthly layoffs, YTD, industry breakdown | All tagged as CHALLENGER_JOB_CUTS |
| Hellenic Shipping - Dry Bulk | freight | Daily BDI index, route rates | Not yet extracting BDI |

### RSS Feeds (News - Keep Generic)

| Feed | Category | Notes |
|------|----------|-------|
| CNBC Business News | business | General news, use generic prompt |
| CNBC Economy | economy | General news, use generic prompt |
| CNBC Earnings | earnings | General news, use generic prompt |
| CNBC US Top News | markets | General news, use generic prompt |
| Investing.com News | markets | General news, use generic prompt |

### Search Queries (Exploratory - Keep Generic)

| Query | Schedule | Notes |
|-------|----------|-------|
| Latest market news economic data | MarketTimes | Exploratory |
| Financial market sentiment indicators | MarketTimes | Exploratory |
| Weekly economic outlook analysis | Weekly | Exploratory |
| Central bank policy announcement analysis | Weekly | Exploratory |
| What is happening in the market today? | Daily | Exploratory |

### Validation Queries (Pattern-Triggered)

| Trigger | Series | Notes |
|---------|--------|-------|
| sahm-rule | - | Seeks confirming/disconfirming evidence |
| yield-curve-inversion | - | Seeks confirming/disconfirming evidence |
| - | GDP | GDP-related validation |
| - | UNRATE | Unemployment validation |

## Proposed Architecture

### Directory Structure

```
prompts/
├── initial_extraction.txt           # Default generic prompt
├── verification_questions.txt       # Unchanged
├── verification_answers.txt         # Unchanged
├── final_verification.txt           # Unchanged
└── sources/
    ├── challenger.txt               # Challenger Gray & Christmas
    ├── baltic_dry.txt               # Hellenic Shipping BDI
    ├── truflation.txt               # Truflation daily (future)
    ├── adp_payroll.txt              # ADP employment (future)
    └── ism_manufacturing.txt        # ISM PMI (future)
```

### Prompt Selection Logic

```csharp
// In FileBasedPromptProvider.cs
public string GetInitialExtractionPrompt(string source, string contentType, string content)
{
    var promptFile = ResolvePromptFile(source);
    var template = GetPrompt(promptFile);
    return template
        .Replace("{{source}}", source)
        .Replace("{{content_type}}", contentType)
        .Replace("{{content}}", content);
}

private string ResolvePromptFile(string source)
{
    // Normalize source name and check for specialized prompt
    var normalized = NormalizeSourceName(source);
    var specializedPath = $"sources/{normalized}.txt";

    if (File.Exists(Path.Combine(_options.PromptDirectory, specializedPath)))
    {
        _logger.LogDebug("Using specialized prompt for source: {Source}", source);
        return specializedPath;
    }

    return InitialPromptFile; // Fallback to generic
}

private static string NormalizeSourceName(string source)
{
    // Map RSS feed names/URLs to prompt files
    return source.ToLowerInvariant() switch
    {
        var s when s.Contains("challenger") => "challenger",
        var s when s.Contains("hellenic") || s.Contains("dry-bulk") => "baltic_dry",
        var s when s.Contains("truflation") => "truflation",
        var s when s.Contains("adp") => "adp_payroll",
        var s when s.Contains("ism") || s.Contains("pmi") => "ism_manufacturing",
        _ => "generic"
    };
}
```

## Specialized Prompt Designs

### 1. Challenger Gray & Christmas (`challenger.txt`)

**Context:** Monthly layoff announcements report, published first Thursday of each month.

**Symbol Taxonomy:**

| Symbol | Description | Expected Range | Example |
|--------|-------------|----------------|---------|
| CHALLENGER_JOB_CUTS | Monthly headline count | 20K-200K | "57,727 job cuts announced" |
| CHALLENGER_JOB_CUTS_YTD | Year-to-date cumulative | 200K-1.5M | "721,677 cuts through December" |
| CHALLENGER_JOB_CUTS_YOY_PCT | Year-over-year % change | -50% to +200% | "up 13.3% from prior year" |
| CHALLENGER_JOB_CUTS_TECH | Tech sector monthly | 5K-50K | "Technology: 15,234 cuts" |
| CHALLENGER_JOB_CUTS_RETAIL | Retail sector monthly | 5K-30K | "Retail: 8,456 cuts" |
| CHALLENGER_JOB_CUTS_GOVT | Government sector monthly | 1K-100K | "Government: 62,530 cuts" |

**Prompt:**
```
Extract data from the Challenger Gray & Christmas monthly layoff report.

CONTEXT:
- Challenger publishes monthly on the first Thursday
- Reports ANNOUNCED layoffs, not actual terminations
- Data covers US-based employers

SYMBOL MAPPING - Use the correct symbol for each metric type:

| Metric | Symbol | Period Format |
|--------|--------|---------------|
| Monthly headline total | CHALLENGER_JOB_CUTS | YYYY-MM |
| Year-to-date cumulative | CHALLENGER_JOB_CUTS_YTD | "up to YYYY-MM" |
| YoY percentage change | CHALLENGER_JOB_CUTS_YOY_PCT | YYYY-MM |
| Tech sector cuts | CHALLENGER_JOB_CUTS_TECH | YYYY-MM |
| Retail sector cuts | CHALLENGER_JOB_CUTS_RETAIL | YYYY-MM |
| Government sector cuts | CHALLENGER_JOB_CUTS_GOVT | YYYY-MM |

EXTRACTION RULES:
1. Monthly headline is the PRIMARY metric - always extract if present
2. For YoY percentage, extract as decimal (13.3% = 13.3, not 0.133)
3. For sector breakdowns, only extract if explicitly stated
4. Set is_comparison=true for YoY percentages and prior year references
5. Period should be the reporting month (YYYY-MM), not publication date

Source: {{source}}
Content: {{content}}
```

### 2. Baltic Dry Index (`baltic_dry.txt`)

**Context:** Daily shipping rate index from Baltic Exchange, reported via Hellenic Shipping News.

**Symbol Taxonomy:**

| Symbol | Description | Expected Range | Example |
|--------|-------------|----------------|---------|
| BALTIC_DRY_INDEX | Daily BDI composite | 300-5000 | "BDI at 1,547 points" |
| BALTIC_DRY_INDEX_CHG | Daily point change | -200 to +200 | "up 23 points" |
| BALTIC_DRY_INDEX_CHG_PCT | Daily % change | -10% to +10% | "rose 1.5%" |
| BALTIC_CAPESIZE | Capesize route avg | 5K-50K $/day | "$15,234/day" |
| BALTIC_PANAMAX | Panamax route avg | 5K-30K $/day | "$12,456/day" |
| BALTIC_SUPRAMAX | Supramax route avg | 5K-25K $/day | "$10,789/day" |

**Prompt:**
```
Extract data from Baltic Dry Index shipping market reports.

CONTEXT:
- Baltic Exchange publishes daily (London business days)
- BDI is a composite of Capesize, Panamax, and Supramax rates
- Higher BDI = stronger global trade demand

SYMBOL MAPPING:

| Metric | Symbol | Unit |
|--------|--------|------|
| BDI composite index | BALTIC_DRY_INDEX | points |
| Daily point change | BALTIC_DRY_INDEX_CHG | points |
| Daily % change | BALTIC_DRY_INDEX_CHG_PCT | percent |
| Capesize daily rate | BALTIC_CAPESIZE | USD/day |
| Panamax daily rate | BALTIC_PANAMAX | USD/day |
| Supramax daily rate | BALTIC_SUPRAMAX | USD/day |

EXTRACTION RULES:
1. BDI level is the PRIMARY metric - always extract if present
2. Period should be the trading date (YYYY-MM-DD)
3. For rate changes, set is_comparison=true
4. Ship rates are typically quoted in $/day

Source: {{source}}
Content: {{content}}
```

### 3. Truflation (`truflation.txt`) - Future

**Context:** Daily inflation index, alternative to CPI with more frequent updates.

**Symbol Taxonomy:**

| Symbol | Description | Expected Range |
|--------|-------------|----------------|
| TRUFLATION_RATE | Daily inflation rate | 0-15% |
| TRUFLATION_FOOD | Food category | 0-20% |
| TRUFLATION_HOUSING | Housing category | 0-15% |
| TRUFLATION_TRANSPORT | Transportation | -10% to +30% |

### 4. ADP Payroll (`adp_payroll.txt`) - Future

**Context:** Monthly private payroll employment change, released 2 days before BLS.

**Symbol Taxonomy:**

| Symbol | Description | Expected Range |
|--------|-------------|----------------|
| ADP_EMPLOYMENT_CHG | Monthly change | -500K to +500K |
| ADP_EMPLOYMENT_CHG_GOODS | Goods-producing | -100K to +100K |
| ADP_EMPLOYMENT_CHG_SERVICES | Service-providing | -400K to +400K |

### 5. ISM Manufacturing (`ism_manufacturing.txt`) - Future

**Context:** Monthly PMI survey, 50 = expansion/contraction boundary.

**Symbol Taxonomy:**

| Symbol | Description | Expected Range |
|--------|-------------|----------------|
| ISM_PMI | Headline PMI | 30-70 |
| ISM_NEW_ORDERS | New orders index | 30-70 |
| ISM_EMPLOYMENT | Employment index | 30-70 |
| ISM_PRICES | Prices paid index | 30-90 |

## ThresholdEngine Pattern Updates

After implementing specialized prompts, update patterns to use new symbols:

### Current → Updated

```json
// challenger-layoff-surge.json
{
  "requiredSeries": ["CHALLENGER_JOB_CUTS"],  // Monthly only now
  "expression": "var cuts = ctx.GetLatest(\"CHALLENGER_JOB_CUTS\"); return cuts.HasValue && cuts.Value > 100000m;"
}

// New pattern: challenger-acceleration.json
{
  "requiredSeries": ["CHALLENGER_JOB_CUTS_YOY_PCT"],
  "expression": "var yoy = ctx.GetLatest(\"CHALLENGER_JOB_CUTS_YOY_PCT\"); return yoy.HasValue && yoy.Value > 50m;"
}
```

## Implementation Plan

### Phase 1: Architecture (This PR)
- [ ] Update `FileBasedPromptProvider` with source resolution logic
- [ ] Add `sources/` directory structure
- [ ] Create `challenger.txt` specialized prompt
- [ ] Create `baltic_dry.txt` specialized prompt
- [ ] Update prompt validation to include sources directory
- [ ] Add unit tests for prompt selection

### Phase 2: Deployment & Validation
- [ ] Deploy updated SentinelCollector
- [ ] Monitor extractions for correct symbol assignment
- [ ] Verify ThresholdEngine receives properly tagged data
- [ ] Tune prompts based on extraction quality

### Phase 3: Additional Sources (Future PRs)
- [ ] Add Truflation prompt when RSS/API available
- [ ] Add ADP Payroll prompt
- [ ] Add ISM Manufacturing prompt
- [ ] Add any new validated data sources

### Phase 4: Pattern Enrichment
- [ ] Create new patterns using enriched symbols (YTD, YoY, sector)
- [ ] Add cross-source correlation patterns
- [ ] Update dashboards to show new metrics

## Testing Strategy

### Unit Tests
```csharp
[Fact]
public void ResolvePromptFile_ChallengerSource_ReturnsSpecializedPrompt()
{
    var provider = new FileBasedPromptProvider(...);
    var result = provider.ResolvePromptFile("challenger-rss");
    Assert.Equal("sources/challenger.txt", result);
}

[Fact]
public void ResolvePromptFile_UnknownSource_ReturnsFallback()
{
    var provider = new FileBasedPromptProvider(...);
    var result = provider.ResolvePromptFile("random-news-site");
    Assert.Equal("initial_extraction.txt", result);
}
```

### Integration Tests
- Extract sample Challenger content with specialized prompt
- Verify correct symbols assigned to each metric type
- Verify ThresholdEngine pattern evaluation with new data

## Rollback Plan

If specialized prompts cause issues:
1. Delete `sources/` directory from container
2. Service falls back to generic prompt automatically
3. No code changes required for rollback
