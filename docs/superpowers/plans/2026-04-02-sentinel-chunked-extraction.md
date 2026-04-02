# Sentinel Chunked Extraction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Handle large documents that exceed the 32K context window by applying chunked CoD with coalescing, enabling existing CoVe chunking, and fixing HTTP 400 error classification.

**Architecture:** Add a chunked path to `ChainOfDensity` that runs full 5-iteration CoD per chunk then coalesces results with another 5-iteration CoD pass. Enable the existing (dormant) CoVe chunking infrastructure. Classify HTTP 4xx as permanent failures. All changes are internal to existing classes — no new services or architectural changes.

**Tech Stack:** .NET 10 / C# 14, xUnit + NSubstitute + FluentAssertions

**Spec:** `docs/superpowers/specs/2026-04-02-sentinel-chunked-extraction-design.md`

---

### Task 1: Add CoD-Specific Chunking Configuration

**Files:**

- Modify: `SentinelCollector/src/Configuration/ExtractionOptions.cs:122-139`
- Modify: `SentinelCollector/src/appsettings.json:41-44`
- Test: `SentinelCollector/tests/SentinelCollector.UnitTests/Configuration/ExtractionOptionsTests.cs`

- [ ] **Step 1: Write test for new CoD config defaults**

```csharp
// In ExtractionOptionsTests.cs — add this test

[Fact]
public void CodChunkingDefaults_should_be_independent_from_cove_chunking()
{
    var options = new ExtractionOptions();

    options.CodChunkingThresholdTokens.Should().Be(8000);
    options.CodChunkSizeTokens.Should().Be(6000);
    options.CodChunkOverlapPercent.Should().Be(0);

    // CoVe defaults unchanged
    options.ChunkingThresholdTokens.Should().Be(24000);
    options.ChunkSizeTokens.Should().Be(1024);
    options.ChunkOverlapPercent.Should().Be(15);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `nerdctl compose exec -T sentinel-collector-dev dotnet test --filter 'Name~CodChunkingDefaults' src/tests/SentinelCollector.UnitTests`

Expected: FAIL — `ExtractionOptions` does not have `CodChunkingThresholdTokens` property.

- [ ] **Step 3: Add CoD chunking properties to ExtractionOptions**

Add these properties to `SentinelCollector/src/Configuration/ExtractionOptions.cs` after line 139 (after `ChunkOverlapPercent`):

```csharp
/// <summary>
/// Token threshold above which Chain of Density uses chunked processing.
/// Lower than CoVe threshold because CoD sends full content on every iteration (5x).
/// </summary>
public int CodChunkingThresholdTokens { get; set; } = 8000;

/// <summary>
/// Target chunk size in tokens for CoD chunking. Larger than CoVe chunks because
/// CoD compresses rather than extracts — more context per chunk means better summaries.
/// </summary>
public int CodChunkSizeTokens { get; set; } = 6000;

/// <summary>
/// Overlap percentage for CoD chunks. Zero by default — CoD is compressing,
/// so overlap creates redundancy that the coalescing pass must deduplicate.
/// </summary>
public int CodChunkOverlapPercent { get; set; } = 0;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `nerdctl compose exec -T sentinel-collector-dev dotnet test --filter 'Name~CodChunkingDefaults' src/tests/SentinelCollector.UnitTests`

Expected: PASS

- [ ] **Step 5: Update appsettings.json with CoD chunk config**

In `SentinelCollector/src/appsettings.json`, in the `"Extraction"` section after `"ChunkOverlapPercent": 15`, add:

```json
"CodChunkingThresholdTokens": 8000,
"CodChunkSizeTokens": 6000,
"CodChunkOverlapPercent": 0
```

- [ ] **Step 6: Commit**

```bash
git add SentinelCollector/src/Configuration/ExtractionOptions.cs \
       SentinelCollector/src/appsettings.json \
       SentinelCollector/tests/SentinelCollector.UnitTests/Configuration/ExtractionOptionsTests.cs
git commit -m "feat(sentinel): add CoD-specific chunking configuration"
```

---

### Task 2: Add Coalescing Prompt to Density Provider

**Files:**

- Modify: `SentinelCollector/src/Extraction/IDensityPromptProvider.cs`
- Modify: `SentinelCollector/src/Extraction/FileBasedDensityPromptProvider.cs`
- Create: `SentinelCollector/src/prompts/cod_coalescing.txt`
- Test: `SentinelCollector/tests/SentinelCollector.UnitTests/Extraction/FileBasedDensityPromptProviderTests.cs`

- [ ] **Step 1: Write test for coalescing prompt**

First read the existing `FileBasedDensityPromptProviderTests.cs` to understand the test pattern, then add:

```csharp
[Fact]
public void GetCoalescingPrompt_should_substitute_placeholders()
{
    // Arrange — write the coalescing prompt file to temp dir
    var coalescingPrompt = @"# COD.COALESCE [{{source}}] [{{target_word_count}} words]
Synthesize these section summaries:
{{chunk_summaries}}";

    File.WriteAllText(Path.Combine(_promptDir, "cod_coalescing.txt"), coalescingPrompt);

    var provider = CreateProvider();

    // Act
    var result = provider.GetCoalescingPrompt("reuters", "Summary 1. Summary 2.", 80);

    // Assert
    result.Should().Contain("reuters");
    result.Should().Contain("Summary 1. Summary 2.");
    result.Should().Contain("80");
    result.Should().NotContain("{{source}}");
    result.Should().NotContain("{{chunk_summaries}}");
    result.Should().NotContain("{{target_word_count}}");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `nerdctl compose exec -T sentinel-collector-dev dotnet test --filter 'Name~GetCoalescingPrompt' src/tests/SentinelCollector.UnitTests`

Expected: FAIL — `GetCoalescingPrompt` method does not exist.

- [ ] **Step 3: Add method to IDensityPromptProvider**

Replace the full content of `SentinelCollector/src/Extraction/IDensityPromptProvider.cs`:

```csharp
namespace SentinelCollector.Extraction;

public interface IDensityPromptProvider
{
    string GetInitialSummaryPrompt(string source, string content, int targetWordCount);
    string GetDensificationPrompt(string originalContent, string previousSummary, int iteration, int targetWordCount);
    string GetEpistemicMarkersPrompt(string summary);
    string GetCoalescingPrompt(string source, string chunkSummaries, int targetWordCount);
}
```

- [ ] **Step 4: Implement in FileBasedDensityPromptProvider**

Add constant after line 16 in `FileBasedDensityPromptProvider.cs`:

```csharp
private const string CoalescingFile = "cod_coalescing.txt";
```

Update `ValidatePromptsExist` — change the `requiredFiles` array to include the new file:

```csharp
var requiredFiles = new[] { InitialSummaryFile, DensificationFile, EpistemicMarkersFile, CoalescingFile };
```

Update `LoadAllPrompts` — change the `files` array:

```csharp
var files = new[] { InitialSummaryFile, DensificationFile, EpistemicMarkersFile, CoalescingFile };
```

Update the `FileSystemWatcher` filter in the constructor (line 30) — it already watches `cod_*.txt` so no change needed.

Add the method implementation after `GetEpistemicMarkersPrompt`:

```csharp
public string GetCoalescingPrompt(string source, string chunkSummaries, int targetWordCount)
{
    var template = GetPrompt(CoalescingFile);
    return template
        .Replace("{{source}}", source)
        .Replace("{{chunk_summaries}}", chunkSummaries)
        .Replace("{{target_word_count}}", targetWordCount.ToString());
}
```

- [ ] **Step 5: Create the coalescing prompt file**

Create `SentinelCollector/src/prompts/cod_coalescing.txt`:

```
# COD.COALESCE [{{source}}] [{{target_word_count}} words]

## CONTEXT
Below are dense summaries of consecutive sections from a single document.
Each summary was independently compressed through iterative densification.
Your task: synthesize these into ONE coherent summary.

## RULES
1. MERGE overlapping_facts → single_mention (prefer most precise value)
2. PRESERVE chronological_order where present
3. PRESERVE epistemic_markers: might | could | expected | projected | estimated
4. PRESERVE attributions: who_said/reported_what
5. RESOLVE contradictions → prefer later_section (more recent info)
6. TARGET ~{{target_word_count}} words (4-5 sentences)

## PRIORITY [must_include]
1. HEADLINE_METRIC: primary_value + exact_number
2. ATTRIBUTION: source_entity
3. PERIOD: time_reference
4. SECONDARY: 1-2 supporting_metrics

## OUTPUT
summary_paragraph_only ∧ ¬preamble ∧ ¬bullets ∧ ¬explanation

---
SECTION SUMMARIES:
{{chunk_summaries}}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `nerdctl compose exec -T sentinel-collector-dev dotnet test --filter 'Name~GetCoalescingPrompt' src/tests/SentinelCollector.UnitTests`

Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add SentinelCollector/src/Extraction/IDensityPromptProvider.cs \
       SentinelCollector/src/Extraction/FileBasedDensityPromptProvider.cs \
       SentinelCollector/src/prompts/cod_coalescing.txt \
       SentinelCollector/tests/SentinelCollector.UnitTests/Extraction/FileBasedDensityPromptProviderTests.cs
git commit -m "feat(sentinel): add coalescing prompt for chunked CoD"
```

---

### Task 3: Add CoD Chunked Counter Metric

**Files:**

- Modify: `SentinelCollector/src/Telemetry/SentinelMeter.cs`

- [ ] **Step 1: Add counter to SentinelMeter**

Add after `ChunkExtractionCounter` (line 145) in `SentinelCollector/src/Telemetry/SentinelMeter.cs`:

```csharp
public static readonly Counter<long> CodChunkedCounter = Meter.CreateCounter<long>(
    "sentinel_cod_chunked_total",
    description: "Total number of documents processed via chunked Chain of Density");
```

- [ ] **Step 2: Verify compilation**

Run: `SentinelCollector/.devcontainer/compile.sh --no-test`

Expected: Build succeeded. 0 Warning(s). 0 Error(s).

- [ ] **Step 3: Commit**

```bash
git add SentinelCollector/src/Telemetry/SentinelMeter.cs
git commit -m "feat(sentinel): add CoD chunked processing counter metric"
```

---

### Task 4: Implement Chunked Chain of Density

**Files:**

- Modify: `SentinelCollector/src/Extraction/ChainOfDensity.cs`

- [ ] **Step 1: Write test for chunked CoD path**

Create `SentinelCollector/tests/SentinelCollector.UnitTests/Extraction/ChainOfDensityTests.cs`:

```csharp
using FluentAssertions;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using NSubstitute;
using SentinelCollector.Configuration;
using SentinelCollector.Extraction;
using SentinelCollector.Services;
using Xunit;

namespace SentinelCollector.UnitTests.Extraction;

public class ChainOfDensityTests
{
    private readonly IOllamaClient _ollamaClient;
    private readonly IDensityPromptProvider _promptProvider;
    private readonly ILogger _logger;

    public ChainOfDensityTests()
    {
        _ollamaClient = Substitute.For<IOllamaClient>();
        _promptProvider = Substitute.For<IDensityPromptProvider>();
        _logger = Substitute.For<ILogger>();
    }

    [Fact]
    public async Task DensifyAsync_SmallContent_UsesStandardPath()
    {
        // Arrange — content under threshold (8000 tokens = 32000 chars)
        var content = new string('x', 20000); // ~5000 tokens, under 8000 threshold
        var options = Options.Create(new ExtractionOptions
        {
            CodChunkingThresholdTokens = 8000,
            MinTimeoutSeconds = 5,
            MaxTimeoutSeconds = 600
        });

        _promptProvider.GetInitialSummaryPrompt(Arg.Any<string>(), Arg.Any<string>(), Arg.Any<int>())
            .Returns("initial prompt");
        _promptProvider.GetDensificationPrompt(Arg.Any<string>(), Arg.Any<string>(), Arg.Any<int>(), Arg.Any<int>())
            .Returns("densification prompt");
        _promptProvider.GetEpistemicMarkersPrompt(Arg.Any<string>())
            .Returns("markers prompt");

        _ollamaClient.GenerateAsync(Arg.Any<string>(), Arg.Any<TimeSpan?>(), Arg.Any<TimeSpan?>(), Arg.Any<CancellationToken>())
            .Returns("A dense summary of the content.", "{\"UncertaintyMarkers\":[],\"Conditions\":[],\"Attributions\":[]}");

        var cod = new ChainOfDensity(_ollamaClient, _promptProvider, options, _logger);

        // Act
        var result = await cod.DensifyAsync("test-source", content);

        // Assert — should NOT call coalescing prompt (standard path)
        _promptProvider.DidNotReceive().GetCoalescingPrompt(
            Arg.Any<string>(), Arg.Any<string>(), Arg.Any<int>());

        // 5 iterations + 1 epistemic markers = 6 calls
        await _ollamaClient.Received(6).GenerateAsync(
            Arg.Any<string>(), Arg.Any<TimeSpan?>(), Arg.Any<TimeSpan?>(), Arg.Any<CancellationToken>());

        result.FinalSummary.Should().NotBeEmpty();
    }

    [Fact]
    public async Task DensifyAsync_LargeContent_UsesChunkedPath()
    {
        // Arrange — content over threshold (8000 tokens = 32000 chars)
        // 48000 chars = ~12000 tokens. With 6000-token chunks = 2 chunks.
        var content = new string('x', 48000);
        var options = Options.Create(new ExtractionOptions
        {
            CodChunkingThresholdTokens = 8000,
            CodChunkSizeTokens = 6000,
            CodChunkOverlapPercent = 0,
            MinTimeoutSeconds = 5,
            MaxTimeoutSeconds = 600
        });

        _promptProvider.GetInitialSummaryPrompt(Arg.Any<string>(), Arg.Any<string>(), Arg.Any<int>())
            .Returns("initial prompt");
        _promptProvider.GetDensificationPrompt(Arg.Any<string>(), Arg.Any<string>(), Arg.Any<int>(), Arg.Any<int>())
            .Returns("densification prompt");
        _promptProvider.GetCoalescingPrompt(Arg.Any<string>(), Arg.Any<string>(), Arg.Any<int>())
            .Returns("coalescing prompt");
        _promptProvider.GetEpistemicMarkersPrompt(Arg.Any<string>())
            .Returns("markers prompt");

        _ollamaClient.GenerateAsync(Arg.Any<string>(), Arg.Any<TimeSpan?>(), Arg.Any<TimeSpan?>(), Arg.Any<CancellationToken>())
            .Returns("A dense summary.", "{\"UncertaintyMarkers\":[],\"Conditions\":[],\"Attributions\":[]}");

        var cod = new ChainOfDensity(_ollamaClient, _promptProvider, options, _logger);

        // Act
        var result = await cod.DensifyAsync("test-source", content);

        // Assert — should call coalescing prompt (chunked path)
        _promptProvider.Received(1).GetCoalescingPrompt(
            "test-source", Arg.Any<string>(), 80);

        result.FinalSummary.Should().NotBeEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `nerdctl compose exec -T sentinel-collector-dev dotnet test --filter 'FullyQualifiedName~ChainOfDensityTests' src/tests/SentinelCollector.UnitTests`

Expected: `DensifyAsync_SmallContent_UsesStandardPath` may pass (existing code path). `DensifyAsync_LargeContent_UsesChunkedPath` will FAIL — no chunked path exists.

- [ ] **Step 3: Implement chunked CoD in ChainOfDensity.cs**

Replace the `DensifyAsync` method body in `ChainOfDensity.cs` (lines 30-114) with:

```csharp
public async Task<DensitySummary> DensifyAsync(
    string source,
    string content,
    CancellationToken cancellationToken = default)
{
    var options = extractionOptions.Value;
    var estimatedTokens = DocumentChunker.EstimateTokens(content);

    if (estimatedTokens > options.CodChunkingThresholdTokens)
    {
        return await DensifyChunkedAsync(source, content, cancellationToken);
    }

    return await DensifySingleAsync(source, content, cancellationToken);
}
```

Add the single-pass method (extract the existing logic):

```csharp
private async Task<DensitySummary> DensifySingleAsync(
    string source,
    string content,
    CancellationToken cancellationToken)
{
    using var activity = SentinelActivitySource.Source.StartActivity("ChainOfDensity");
    activity?.SetTag("source", source);
    activity?.SetTag("iterations", Iterations);
    activity?.SetTag("chunked", false);

    var options = extractionOptions.Value;
    var fullTimeout = options.CalculateTimeout(content.Length);
    activity?.SetTag("timeout_seconds", fullTimeout.TotalSeconds);

    var currentSummary = await RunDensificationIterationsAsync(
        source, content, fullTimeout, cancellationToken);

    var markers = await ExtractEpistemicMarkersAsync(source, currentSummary.Summary, cancellationToken);

    activity?.SetTag("final_word_count", CountWords(currentSummary.Summary));
    activity?.SetTag("uncertainty_markers", markers.Markers.UncertaintyMarkers.Count);
    activity?.SetTag("markers_extraction_failed", markers.Failed);

    return new DensitySummary(
        FinalSummary: currentSummary.Summary,
        Iterations: currentSummary.Iterations,
        UncertaintyMarkers: markers.Markers.UncertaintyMarkers,
        Conditions: markers.Markers.Conditions,
        Attributions: markers.Markers.Attributions,
        MarkersExtractionFailed: markers.Failed
    );
}
```

Add the chunked path:

```csharp
private async Task<DensitySummary> DensifyChunkedAsync(
    string source,
    string content,
    CancellationToken cancellationToken)
{
    using var activity = SentinelActivitySource.Source.StartActivity("ChainOfDensity.Chunked");
    activity?.SetTag("source", source);

    var options = extractionOptions.Value;
    var chunks = DocumentChunker.Chunk(content, options.CodChunkSizeTokens, options.CodChunkOverlapPercent);

    activity?.SetTag("chunk_count", chunks.Count);
    SentinelMeter.CodChunkedCounter.Add(1,
        new KeyValuePair<string, object?>("chunks", chunks.Count));

    logger.LogInformation(
        "Chunked CoD for {Source}: {TokenCount} estimated tokens, {ChunkCount} chunks",
        source, DocumentChunker.EstimateTokens(content), chunks.Count);

    // Phase 1: Full 5-iteration CoD per chunk
    var allIterations = new List<string>();
    var chunkSummaries = new List<string>();

    for (var i = 0; i < chunks.Count; i++)
    {
        var chunkTimeout = options.CalculateTimeout(chunks[i].Length);
        var chunkResult = await RunDensificationIterationsAsync(
            $"{source}[chunk:{i + 1}/{chunks.Count}]",
            chunks[i],
            chunkTimeout,
            cancellationToken);

        chunkSummaries.Add(chunkResult.Summary);
        allIterations.AddRange(chunkResult.Iterations);
    }

    // Phase 2: Coalesce chunk summaries with another 5-iteration CoD pass
    var coalescedContent = string.Join("\n\n", chunkSummaries);
    var coalescedTimeout = options.CalculateTimeout(coalescedContent.Length);

    logger.LogInformation(
        "Coalescing {ChunkCount} chunk summaries for {Source} ({CoalescedTokens} tokens)",
        chunks.Count, source, DocumentChunker.EstimateTokens(coalescedContent));

    var coalescedResult = await RunCoalescingIterationsAsync(
        source, coalescedContent, coalescedTimeout, cancellationToken);

    allIterations.AddRange(coalescedResult.Iterations);

    // Epistemic markers from final summary
    var markers = await ExtractEpistemicMarkersAsync(source, coalescedResult.Summary, cancellationToken);

    activity?.SetTag("final_word_count", CountWords(coalescedResult.Summary));
    activity?.SetTag("markers_extraction_failed", markers.Failed);

    return new DensitySummary(
        FinalSummary: coalescedResult.Summary,
        Iterations: allIterations,
        UncertaintyMarkers: markers.Markers.UncertaintyMarkers,
        Conditions: markers.Markers.Conditions,
        Attributions: markers.Markers.Attributions,
        MarkersExtractionFailed: markers.Failed
    );
}
```

Add the shared iteration runner (extracts duplicate logic from both paths):

```csharp
private async Task<(string Summary, List<string> Iterations)> RunDensificationIterationsAsync(
    string source,
    string content,
    TimeSpan timeout,
    CancellationToken cancellationToken)
{
    var summaries = new List<string>();
    var currentSummary = string.Empty;

    for (var iteration = 1; iteration <= Iterations; iteration++)
    {
        using var iterActivity = SentinelActivitySource.Source.StartActivity($"CoD_Iteration_{iteration}");

        var prompt = iteration == 1
            ? promptProvider.GetInitialSummaryPrompt(source, content, TargetWordCount)
            : promptProvider.GetDensificationPrompt(content, currentSummary, iteration, TargetWordCount);

        try
        {
            var response = await ollamaClient.GenerateAsync(prompt, timeout: timeout, cancellationToken: cancellationToken);
            currentSummary = ExtractSummary(response);
            summaries.Add(currentSummary);
        }
        catch (TimeoutException ex)
        {
            iterActivity?.SetStatus(System.Diagnostics.ActivityStatusCode.Error, $"Timeout at iteration {iteration}");
            iterActivity?.AddException(ex);
            logger.LogWarning(
                ex,
                "CoD iteration {Iteration}/{Total} timed out for {Source} after {TimeoutSeconds}s",
                iteration, Iterations, source, timeout.TotalSeconds);
            throw;
        }

        iterActivity?.SetTag("word_count", CountWords(currentSummary));

        logger.LogDebug(
            "CoD iteration {Iteration}/{Total} for {Source}: {WordCount} words",
            iteration, Iterations, source, CountWords(currentSummary));
    }

    return (currentSummary, summaries);
}

private async Task<(string Summary, List<string> Iterations)> RunCoalescingIterationsAsync(
    string source,
    string coalescedContent,
    TimeSpan timeout,
    CancellationToken cancellationToken)
{
    var summaries = new List<string>();
    var currentSummary = string.Empty;

    for (var iteration = 1; iteration <= Iterations; iteration++)
    {
        using var iterActivity = SentinelActivitySource.Source.StartActivity($"CoD_Coalesce_{iteration}");

        var prompt = iteration == 1
            ? promptProvider.GetCoalescingPrompt(source, coalescedContent, TargetWordCount)
            : promptProvider.GetDensificationPrompt(coalescedContent, currentSummary, iteration, TargetWordCount);

        try
        {
            var response = await ollamaClient.GenerateAsync(prompt, timeout: timeout, cancellationToken: cancellationToken);
            currentSummary = ExtractSummary(response);
            summaries.Add(currentSummary);
        }
        catch (TimeoutException ex)
        {
            iterActivity?.SetStatus(System.Diagnostics.ActivityStatusCode.Error, $"Timeout at coalescing iteration {iteration}");
            iterActivity?.AddException(ex);
            logger.LogWarning(
                ex,
                "CoD coalescing iteration {Iteration}/{Total} timed out for {Source} after {TimeoutSeconds}s",
                iteration, Iterations, source, timeout.TotalSeconds);
            throw;
        }

        iterActivity?.SetTag("word_count", CountWords(currentSummary));
    }

    return (currentSummary, summaries);
}
```

Add the epistemic markers helper (extracted from old code):

```csharp
private async Task<(EpistemicMarkers Markers, bool Failed)> ExtractEpistemicMarkersAsync(
    string source,
    string summary,
    CancellationToken cancellationToken)
{
    var options = extractionOptions.Value;
    var minTimeout = TimeSpan.FromSeconds(options.MinTimeoutSeconds);
    var markersPrompt = promptProvider.GetEpistemicMarkersPrompt(summary);

    try
    {
        var markersResponse = await ollamaClient.GenerateAsync(markersPrompt, timeout: minTimeout, cancellationToken: cancellationToken);
        var (markers, failed) = ParseEpistemicMarkers(markersResponse);
        return (markers, failed);
    }
    catch (TimeoutException ex)
    {
        logger.LogWarning(ex, "Epistemic markers extraction timed out for {Source}, returning partial result", source);
        return (EpistemicMarkers.Empty, true);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `nerdctl compose exec -T sentinel-collector-dev dotnet test --filter 'FullyQualifiedName~ChainOfDensityTests' src/tests/SentinelCollector.UnitTests`

Expected: PASS (both tests)

- [ ] **Step 5: Run all existing tests to check for regressions**

Run: `nerdctl compose exec -T sentinel-collector-dev dotnet test src/tests/SentinelCollector.UnitTests`

Expected: All tests pass. The refactored `DensifySingleAsync` preserves identical behavior to the original.

- [ ] **Step 6: Commit**

```bash
git add SentinelCollector/src/Extraction/ChainOfDensity.cs \
       SentinelCollector/tests/SentinelCollector.UnitTests/Extraction/ChainOfDensityTests.cs
git commit -m "feat(sentinel): implement chunked Chain of Density with coalescing"
```

---

### Task 5: Enable CoVe Chunking

**Files:**

- Modify: `SentinelCollector/src/appsettings.json:41`

- [ ] **Step 1: Flip EnableChunking to true**

In `SentinelCollector/src/appsettings.json`, change:

```json
"EnableChunking": true,
```

- [ ] **Step 2: Verify compilation**

Run: `SentinelCollector/.devcontainer/compile.sh --no-test`

Expected: Build succeeded.

- [ ] **Step 3: Commit**

```bash
git add SentinelCollector/src/appsettings.json
git commit -m "feat(sentinel): enable CoVe chunking for large documents"
```

---

### Task 6: Fix HTTP 400 Error Classification

**Files:**

- Modify: `SentinelCollector/src/Workers/ExtractionProcessor.cs:297-298`
- Test: `SentinelCollector/tests/SentinelCollector.UnitTests/Workers/ExtractionProcessorTests.cs`

- [ ] **Step 1: Write test for HTTP 400 as permanent failure**

Add to `ExtractionProcessorTests.cs`:

```csharp
[Fact]
public async Task ProcessBatchAsync_WhenHttp400_MarksPermanentErrorImmediately()
{
    // Arrange — HTTP 400 is a client error, retrying won't help
    var rawContent = CreateRawContent("test-400");

    _rawContentRepo.GetUnprocessedAsync(Arg.Any<int>(), Arg.Any<CancellationToken>())
        .Returns(new List<RawContent> { rawContent });

    _extractionService.ExtractAsync(rawContent, Arg.Any<CancellationToken>())
        .ThrowsAsync(new HttpRequestException("Bad Request", null, System.Net.HttpStatusCode.BadRequest));

    var processor = CreateProcessor();

    // Act
    await processor.TestProcessBatchAsync(CancellationToken.None);

    // Assert — 400 should be permanent, not retried
    await _rawContentRepo.Received(1).UpdateAsync(
        Arg.Is<RawContent>(r => r.ProcessingError != null && r.RetryCount == 0),
        Arg.Any<CancellationToken>());
}

[Fact]
public async Task ProcessBatchAsync_WhenHttp500_RequeuesItem()
{
    // Arrange — HTTP 500 is a server error, should retry
    var rawContent = CreateRawContent("test-500");

    _rawContentRepo.GetUnprocessedAsync(Arg.Any<int>(), Arg.Any<CancellationToken>())
        .Returns(new List<RawContent> { rawContent });

    _extractionService.ExtractAsync(rawContent, Arg.Any<CancellationToken>())
        .ThrowsAsync(new HttpRequestException("Internal Server Error", null, System.Net.HttpStatusCode.InternalServerError));

    var processor = CreateProcessor();

    // Act
    await processor.TestProcessBatchAsync(CancellationToken.None);

    // Assert — 500 should be transient, retried
    await _rawContentRepo.Received(1).UpdateAsync(
        Arg.Is<RawContent>(r => r.ProcessingError == null && r.RetryCount == 1),
        Arg.Any<CancellationToken>());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `nerdctl compose exec -T sentinel-collector-dev dotnet test --filter 'Name~Http400|Name~Http500' src/tests/SentinelCollector.UnitTests`

Expected: `WhenHttp400_MarksPermanentErrorImmediately` FAILS (currently treated as transient). `WhenHttp500_RequeuesItem` may pass (already transient).

- [ ] **Step 3: Fix the isTransient classification**

In `SentinelCollector/src/Workers/ExtractionProcessor.cs`, replace lines 297-298:

```csharp
var isTransient = ex is HttpRequestException or TimeoutException
    || (ex is TaskCanceledException tce && tce.InnerException is TimeoutException);
```

With:

```csharp
var isTransient = ex is TimeoutException
    || (ex is TaskCanceledException tce && tce.InnerException is TimeoutException)
    || (ex is HttpRequestException httpEx && httpEx.StatusCode is null or >= System.Net.HttpStatusCode.InternalServerError);
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `nerdctl compose exec -T sentinel-collector-dev dotnet test --filter 'FullyQualifiedName~ExtractionProcessorTests' src/tests/SentinelCollector.UnitTests`

Expected: ALL ExtractionProcessor tests pass, including existing transient/permanent tests.

- [ ] **Step 5: Commit**

```bash
git add SentinelCollector/src/Workers/ExtractionProcessor.cs \
       SentinelCollector/tests/SentinelCollector.UnitTests/Workers/ExtractionProcessorTests.cs
git commit -m "fix(sentinel): classify HTTP 4xx as permanent failure, not transient"
```

---

### Task 7: Full Build Verification

**Files:** None (verification only)

- [ ] **Step 1: Run full compilation with tests**

Run: `SentinelCollector/.devcontainer/compile.sh`

Expected: Build succeeded. 0 Warning(s). 0 Error(s). All tests pass.

- [ ] **Step 2: Build the container image**

Run: `SentinelCollector/.devcontainer/build.sh`

Expected: Image builds successfully.
