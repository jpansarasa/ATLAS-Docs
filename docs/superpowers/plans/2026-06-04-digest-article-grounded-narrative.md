# Article-Grounded Digest Narrative — Implementation Plan

> **For agentic workers:** Implement task-by-task, TDD. Steps use checkbox (`- [ ]`) syntax. Spec: `docs/superpowers/specs/2026-06-04-digest-article-grounded-narrative-design.md`.

**Goal:** Generate the digest LLM narrative from the cleaned text of the day's top ~25 articles (theme-stratified, signal-ranked), anchored by their CoD-extracted values — instead of from atomic observation fragments.

**Architecture:** Three units inside SentinelCollector: a pure `DigestArticleSelector` (observations+themes → ranked articles), an `IDigestArticleTextProvider` (reads `raw_file_path` → trafilatura → truncated excerpt), and a modified `DigestNarrativeGenerator.BuildPrompt` that assembles {excerpt + values + Tier-1 aggregates}. Tier-1 content, the vLLM client (#609), and the Degraded/Unavailable handling are unchanged.

**Tech Stack:** .NET10/C#14, xUnit + FluentAssertions + NSubstitute, existing `IRawContentService` + `ITrafilaturaClient` (HTTP → trafilatura:3109), the dedicated-vLLM `IOllamaClient` from #609.

**Build/verify gate (every task ends green):** `SentinelCollector/.devcontainer/compile.sh` (with tests) → 0 errors, 0 warnings, all pass. Worktree-isolated. No push/deploy/raw-DB.

---

## File structure

- **Create** `SentinelCollector/src/Services/DigestArticleSelector.cs` — `IDigestArticleSelector` + impl. Pure ranking; no I/O.
- **Create** `SentinelCollector/src/Services/DigestArticleTextProvider.cs` — `IDigestArticleTextProvider` + impl. Reads raw file via `IRawContentService`, cleans via `ITrafilaturaClient`, truncates.
- **Modify** `SentinelCollector/src/Services/DigestNarrativeGenerator.cs` — inject the selector + text provider; rebuild `BuildPrompt` around article excerpts + values + aggregates; assemble within a token budget.
- **Modify** `SentinelCollector/src/prompts/digest/narrative.md` — new prompt template (host-mounted; also update the repo copy that ansible syncs).
- **Modify** `SentinelCollector/src/DependencyInjection.cs` — register `IDigestArticleSelector`, `IDigestArticleTextProvider`; add their ctor deps to the `DigestNarrativeGenerator` factory.
- **Modify** `SentinelCollector/src/Telemetry/SentinelMeter.cs` — `DigestNarrativeArticleCounter{outcome}`.
- **Tests:** `tests/SentinelCollector.UnitTests/Services/DigestArticleSelectorTests.cs`, `DigestArticleTextProviderTests.cs`, extend `DigestNarrativeGeneratorTests.cs`.

Confirm exact existing signatures before coding: `IRawContentService` (how to read a raw file by `raw_file_path` / RawContent), `ITrafilaturaClient` (HTML→text method + return shape), `DigestObservationRow` fields, and how `DigestNarrativeGenerator` currently receives `themes`/`stats`.

---

## Task 1: `DigestArticleSelector` (pure ranking)

**Files:** Create `DigestArticleSelector.cs`; Test `DigestArticleSelectorTests.cs`.

Define:
```csharp
public sealed record SelectedArticleValue(string Description, decimal? Value, string? Unit, string? SourceEntity, string? AtlasSectorCode);
public sealed record SelectedArticle(long RawContentId, string Theme, string SourceUrl, string? Publisher, IReadOnlyList<SelectedArticleValue> Values);
public interface IDigestArticleSelector
{
    IReadOnlyList<SelectedArticle> Select(
        IReadOnlyList<DigestObservationRow> observations,
        IReadOnlyDictionary<long, DigestTheme> themes,
        int maxArticles, int perThemeCap);
}
```
Ranking: group observations by `themes[o.Id]` (default `Other`); within a theme group by `RawContentId`; score each article = `Σ Confidence`; order articles desc; take `perThemeCap` per theme; flatten; dedup by `RawContentId` keeping the first (highest-scoring) occurrence; cap to `maxArticles`. `Publisher` = host of `SourceUrl` (reuse the digest's existing domain-extraction helper — extract it to a shared spot if private). `Values` = that article's obs (the ones already passed in, i.e. confidence ≥ floor) mapped to `SelectedArticleValue`, capped to a small number (e.g. 8) by confidence.

- [ ] **Step 1 — failing tests** (write all, then implement):
  - `should_take_top_articles_per_theme_by_summed_confidence`
  - `should_dedup_article_across_themes_keeping_highest_score`
  - `should_cap_total_at_max_articles`
  - `should_return_empty_for_empty_observations`
  - `should_carry_article_values_capped_and_confidence_ordered`
- [ ] **Step 2** — run, verify fail (type/method missing).
- [ ] **Step 3** — implement the selector.
- [ ] **Step 4** — `compile.sh`, tests pass.
- [ ] **Step 5** — commit `feat(sentinel): DigestArticleSelector (theme-stratified, signal-ranked)`.

## Task 2: `DigestArticleTextProvider` (read + clean + truncate)

**Files:** Create `DigestArticleTextProvider.cs`; Test `DigestArticleTextProviderTests.cs`.

```csharp
public interface IDigestArticleTextProvider
{
    Task<string> GetExcerptAsync(long rawContentId, int maxChars, CancellationToken ct = default);
}
```
Impl: load the `RawContent` (via `SentinelDbContext` or `IRawContentService`) to get `raw_file_path` + `content_type`; read the raw file (the `raw-data` dir is mounted `:ro`); clean via `ITrafilaturaClient` (HTML) — if `content_type` is already text/markdown, skip trafilatura; collapse whitespace; truncate to `maxChars` at a word boundary. On missing file / read error / trafilatura failure: `LogWarning` with `{rawContentId, raw_file_path}` and return `string.Empty` (caller drops the article). Never throw.

- [ ] **Step 1 — failing tests** (fake `IRawContentService`/`ITrafilaturaClient` + a temp file):
  - `should_return_cleaned_text_truncated_to_max_chars`
  - `should_truncate_at_word_boundary`
  - `should_return_empty_when_raw_file_missing` (no throw)
  - `should_return_empty_when_trafilatura_fails` (no throw)
  - `should_skip_trafilatura_for_text_content_type`
- [ ] **Step 2** — run, verify fail.
- [ ] **Step 3** — implement.
- [ ] **Step 4** — `compile.sh`, tests pass.
- [ ] **Step 5** — commit `feat(sentinel): DigestArticleTextProvider (raw file → trafilatura → truncated excerpt)`.

## Task 3: rebuild `DigestNarrativeGenerator.BuildPrompt` + token budget

**Files:** Modify `DigestNarrativeGenerator.cs`, `prompts/digest/narrative.md`; extend `DigestNarrativeGeneratorTests.cs`.

Inject `IDigestArticleSelector` + `IDigestArticleTextProvider` (ctor; update DI in Task 4). New `GenerateAsync` flow:
1. `selector.Select(observations, themes, maxArticles, perThemeCap)`.
2. Compute dynamic budget: `perArticleChars = clamp( floor(textBudgetTokens * CHARS_PER_TOKEN / max(1, selected.Count)), MIN_CHARS, MAX_CHARS )`. Constants on `DigestOptions`: `NarrativeMaxArticles=25`, `NarrativePerThemeCap=4`, `NarrativeTextBudgetTokens=24000`, `NarrativeMaxCharsPerArticle≈6000`, `NarrativeMinCharsPerArticle≈1500`. `CHARS_PER_TOKEN≈4`.
3. For each selected article: `GetExcerptAsync(id, perArticleChars)`; drop empties.
4. **Hard safety cap:** estimate assembled prompt tokens (`len/4`); if over `NarrativeSafetyTokens≈30000`, drop lowest-ranked articles until under. (Never rely on the server to reject — the #608 bug class.)
5. Build prompt from the template: per-article block `{publisher, theme, excerpt, values}` + the Tier-1 theme/sector roll-up summary (reuse `stats`). Keep the existing 400/Degraded + timeout/Unavailable catches and `DigestLlmStatus` handling unchanged. Emit `SentinelMeter.DigestNarrativeArticleCounter` with `outcome=selected|read|skipped`.

New template `narrative.md`: instruct the model to write a macro-financial narrative grounded ONLY in the provided article excerpts, using the extracted figures as the quantitative backbone, citing publishers; placeholders `{{articles}}`, `{{aggregate_rollup}}`, `{{period_start}}`, `{{period_end}}`.

- [ ] **Step 1 — failing tests** (fakes for selector/text-provider/IOllamaClient capturing the prompt):
  - `should_build_prompt_from_selected_article_excerpts_and_values`
  - `should_drop_articles_when_assembled_prompt_exceeds_safety_token_cap`
  - `should_widen_per_article_budget_when_few_articles`
  - `should_skip_narrative_and_return_degraded_or_tier1_when_no_articles_selected`
  - (keep existing 400→Degraded / timeout→Unavailable tests green; adapt construction to new ctor)
- [ ] **Step 2** — run, verify fail.
- [ ] **Step 3** — implement generator + template.
- [ ] **Step 4** — `compile.sh`, tests pass.
- [ ] **Step 5** — commit `feat(sentinel): article-grounded digest narrative prompt + token budget`.

## Task 4: DI wiring + metric

**Files:** Modify `DependencyInjection.cs`, `SentinelMeter.cs`.

Register `services.AddScoped<IDigestArticleSelector, DigestArticleSelector>();` and `IDigestArticleTextProvider` (with its `IRawContentService`/`ITrafilaturaClient`/`SentinelDbContext` deps). Update the existing `IDigestNarrativeGenerator` factory (the dedicated-vLLM one from #609 at the digest block) to resolve + pass the two new deps — keep the vLLM client wiring intact. Add `DigestNarrativeArticleCounter` (`sentinel_digest_narrative_articles_total`, tag `outcome`) to `SentinelMeter`.

- [ ] **Step 1** — make the change.
- [ ] **Step 2** — `compile.sh`, full suite passes (DI resolves; existing `DigestNarrativeBackendBindingTests` still green — narrative stays on vLLM).
- [ ] **Step 3** — commit `feat(sentinel): wire article-grounded narrative deps + article counter`.

## Task 5: prompt sync + final verify

**Files:** ensure `narrative.md` exists in the repo path ansible syncs (`SentinelCollector/src/prompts/digest/...`) so deploy ships it (host `/prompts` is overwritten from repo, force:true).

- [ ] **Step 1** — confirm template path matches `ExtractionOptions.PromptDirectory` + `digest/narrative.md` and the ansible prompt-sync source.
- [ ] **Step 2** — `compile.sh` full suite, 0/0, all pass. Record counts.
- [ ] **Step 3** — commit any prompt-path fix.

---

## Self-review (done)
- **Spec coverage:** selector (Task 1), text source/trafilatura/truncation (Task 2), prompt+budget+fallback (Task 3), DI/metric (Task 4), prompt sync (Task 5). Tier-1 unchanged (not touched). ✓
- **Placeholders:** none — interfaces, constants, and test names are concrete; implementation code is written TDD by the executor against these contracts.
- **Type consistency:** `SelectedArticle`/`SelectedArticleValue`/`IDigestArticleSelector`/`IDigestArticleTextProvider.GetExcerptAsync`/`DigestNarrativeArticleCounter` referenced consistently across tasks.
- **Pre-code confirmations flagged:** exact `IRawContentService` read API, `ITrafilaturaClient` method, and the digest domain-helper location must be checked before Task 1/2 (noted in File Structure).
