# Sentinel High Extraction Error Rate - 2026-01-22

## Alert
- **Title**: SentinelHighExtractionErrorRate
- **Severity**: Warning
- **Message**: Extraction error rate is above 10%. LLM extraction may be degraded.
- **Time**: 2026-01-20 08:59:25 UTC
- **Session**: 3598bc04

## Investigation

### Symptoms
- Extraction error rate spiking to 50-100% at multiple intervals
- Validation errors in logs: "conditional certainty requires a condition string"
- Pattern: LLM setting `certainty: "conditional"` without providing required `condition` field

### Root Cause
The extraction prompt (`SentinelCollector/src/prompts/initial_extraction.txt`) did not adequately explain the conditional certainty feature:

1. Schema showed `certainty` could be "conditional" but didn't explain when/how to use it
2. No examples demonstrated conditional certainty usage
3. Critical requirement that `condition` field MUST be provided when `certainty="conditional"` was not documented
4. Validation logic (ExtractionValidationResult.cs:61) enforces this rule but prompt didn't match

Result: LLM occasionally chose conditional certainty (likely for forward-looking statements) but failed to provide the required condition string, causing validation failures.

### Evidence
- **Loki logs**: Multiple "Extraction validation failed for rss (attempt 1): Result[0]: conditional certainty requires a condition string"
  - TraceId: fb4475d291a8b5e62d46ae950f44bdfe
  - TraceId: 6e9fb958d1148877ed10b3dbfb6f050f
- **Prometheus metrics**: `sum(rate(sentinel_extraction_error_total[5m])) / sum(rate(sentinel_extraction_success_total[5m]) + rate(sentinel_extraction_error_total[5m])) * 100` showed periods of 100% error rate
- **Ollama connectivity issues**: Some HTTP retries to ollama-gpu:11434 but not the primary cause

## Resolution

### Fix Applied
**Commit**: 71d8fc2 on branch `autofix/3598bc04`

Updated `SentinelCollector/src/prompts/initial_extraction.txt`:
1. Added CERTAINTY_LEVELS section explaining each certainty type with examples
2. Added explicit CRITICAL rule: "conditional certainty REQUIRES condition field"
3. Added `condition` field to schema documentation with "REQUIRED if certainty=conditional"
4. Added two new examples:
   - Conditional: "Analysts predict unemployment could reach 6% if the trade war intensifies"
   - Expected: "The Fed expects inflation to fall to 2.5% by year-end"
5. Updated all existing examples to consistently include `"condition":null` for non-conditional cases

### Deployment
**Status**: Committed but pending deployment to production

**Manual deployment steps**:
```bash
# Option 1: Direct file copy (requires appropriate permissions)
bash /tmp/deploy-sentinel-prompt.sh

# Option 2: Full ansible deployment (preferred per CLAUDE.md)
cd /home/james/ATLAS/deployment/ansible
ansible-playbook playbooks/deploy.yml --tags sentinel-collector -i inventory/hosts.yml
```

Note: FileBasedPromptProvider has file watching enabled, so it will automatically reload the prompt once the file is updated.

## Expected Outcome
- LLM will now understand when/how to use conditional certainty
- When choosing conditional certainty, LLM will provide the required condition string
- Validation errors should drop significantly
- Extraction error rate should return to baseline (<5%)

## Monitoring
Watch these metrics/logs for 24-48 hours after deployment:
- `sentinel_extraction_error_total` and `sentinel_extraction_success_total` rates
- Error rate percentage (should drop below 10%, ideally below 5%)
- Loki logs for "Extraction validation failed" messages (should decrease)
- Verify conditional extractions now include proper condition strings

## Follow-up Actions
- [ ] Deploy fix to production
- [ ] Monitor extraction metrics for 24-48 hours
- [ ] Verify alert resolves and doesn't re-trigger
- [ ] Consider adding integration test for conditional certainty validation
- [ ] Review other prompts for similar documentation gaps

## Related Issue: Ollama Performance Degradation

### Symptoms
Separate from the prompt issue, Ollama exhibits performance degradation over extended operation:
- Gradual slowdown in inference speed over time
- HTTP retries to ollama-gpu:11434 observed in logs
- Pattern suggests VRAM pressure with 32B model fully utilizing 32GB VRAM

### Suspected Causes
1. **KV cache fragmentation** - Ollama's context management fragments over long sessions near VRAM limits
2. **Memory management** - Ollama is not aggressive about cleanup; models stay loaded, contexts persist longer than needed
3. **Zero headroom** - 32GB fully utilized leaves no room for KV cache growth during inference
4. **QLoRA adapter loading** - Ollama's LoRA support is relatively recent and may not be as optimized

### Current Workaround
Per CLAUDE.md: `GPU_OOM: restart_ollama_first ¬ downgrade_model`
Restarting Ollama clears fragmentation and restores performance.

### Alternatives Under Investigation

Context size reduction is **not viable** - RLM research indicates smaller context sizes degrade extraction accuracy.

**Current Configuration**:
- Model: `sentinel-extraction-v5` (QLoRA fine-tuned Qwen2.5:32b → GGUF)
- Context: 32768 tokens
- VRAM: 32GB fully utilized

#### Option 1: vLLM

**Pros**:
- PagedAttention eliminates KV cache fragmentation (treats GPU memory like virtual memory)
- Non-contiguous storage, dynamic allocation, memory sharing across requests
- 2-4x higher throughput vs Hugging Face pipelines
- Designed for production serving at scale
- Supports AWQ/GPTQ with Marlin kernels (2.6x faster than naive GPTQ)
- Supports KV cache quantization (FP8) for additional memory savings

**Cons**:
- GGUF support is experimental and slow (not recommended)
- Requires model conversion to AWQ or GPTQ format
- LoRA support less mature than exllamav2
- More complex deployment than Ollama

**Model Format**: AWQ or GPTQ-Int4 (pre-quantized models available: `Qwen/Qwen2.5-32B-Instruct-GPTQ-Int4`)

**Migration Complexity**: HIGH - Need to re-train LoRA on AWQ/GPTQ base or use base model without fine-tuning

#### Option 2: llama.cpp Server

**Pros**:
- Native GGUF support (no conversion needed)
- Explicit control over context size, batch settings, memory allocation
- Can cap `--ctx-size` to leave VRAM headroom
- Continuous batching support (`--cont-batching`)
- Direct control over KV cache data types (q8_0, q4_0, etc.)
- Lower memory overhead than Ollama

**Cons**:
- No Ollama conveniences (model management, API compatibility layer)
- Need to manage thread pool sizing in containers
- Less mature LoRA hot-swapping than exllamav2

**Model Format**: GGUF (current format - no conversion needed)

**Migration Complexity**: LOW - Same GGUF model, different server

#### Option 3: TabbyAPI + ExLlamaV2

**Pros**:
- Best memory efficiency with EXL2 format
- Excellent LoRA support with hot-swapping during inference
- OpenAI-compatible API
- Flash Attention 2.5.7+ with paged attention
- Dynamic batching, smart prompt caching, KV cache deduplication

**Cons**:
- Requires model conversion to EXL2 format
- Self-described as "hobby project" not meant for production
- Smaller community than vLLM

**Model Format**: EXL2 (requires conversion from GGUF or HF)

**Migration Complexity**: MEDIUM - Need to convert model, but LoRA workflow well-supported

### Quantization Comparison (Qwen2.5-32B)

| Format | Accuracy Loss | Speed (vLLM) | Notes |
|--------|---------------|--------------|-------|
| BF16 | baseline | baseline | Requires 64GB+ VRAM |
| GPTQ-Int8 | <2% | - | Near lossless |
| GPTQ-Int4 + Marlin | 3-6% | 712 tok/s | **Recommended for vLLM** |
| AWQ-Int4 | ~4% | 67 tok/s* | *Slow kernel in vLLM |
| GGUF Q4_K_M | ~5% | 81 tok/s | Best with llama.cpp |

*Qwen2.5 is notably quantization-tolerant, retaining 95-98% accuracy even at Q4_K_M.

### Recommendation

**Short-term**: Switch to **llama.cpp server** directly
- Zero model conversion (same GGUF)
- Explicit memory control to prevent fragmentation
- Lower complexity than alternatives

**Long-term**: Evaluate **vLLM with GPTQ-Int4 + Marlin**
- Better for scale if concurrent extraction load increases
- Would require re-validating extraction accuracy with base model (no fine-tuning)
- Or: train LoRA on GPTQ base model

### Diagnostic Commands

```bash
# Check current VRAM state
nvidia-smi

# Check Ollama memory over time (run periodically during degradation)
nvidia-smi --query-gpu=memory.used,memory.free --format=csv -l 60

# Manually unload model to test if idle caching is the culprit
curl -X POST http://ollama-gpu:11434/api/generate \
  -d '{"model": "sentinel-extraction-v5", "keep_alive": 0}'

# Or via CLI
sudo nerdctl exec ollama-gpu ollama stop sentinel-extraction-v5
```

**Key diagnostic**: If manual unload fixes degradation, Ollama's idle model caching is the culprit → switch to llama.cpp server with explicit lifecycle control.

### Action Items

- [x] Research alternative inference servers (vLLM, llama.cpp, TabbyAPI)
- [x] Evaluate model format requirements (GGUF vs AWQ/GPTQ vs EXL2)
- [x] Prototype llama.cpp server with current GGUF model
- [ ] Deploy and test llama-server in staging
- [ ] Benchmark llama.cpp vs Ollama for memory stability over 24h
- [ ] If llama.cpp insufficient: evaluate GPTQ-Int4 conversion + vLLM
- [ ] Consider automated Ollama restart on degradation detection (interim)

### Prototype Implementation (2026-01-22)

A complete llama.cpp server prototype has been implemented:

**Files Created/Modified:**

1. `SentinelCollector/src/Services/LlamaServerClient.cs` - New client implementing `IOllamaClient`
   - Maps llama.cpp `/completion` endpoint to existing interface
   - Supports `json_schema` parameter for structured output
   - Model lifecycle is server-managed (no keep_alive needed)

2. `SentinelCollector/src/Configuration/ExtractionOptions.cs` - Added backend selection
   - New `InferenceBackend` enum: `Ollama` | `LlamaServer`
   - New `LlamaServerEndpoint` configuration option

3. `SentinelCollector/src/DependencyInjection.cs` - Backend-aware DI registration
   - Registers `OllamaClient` or `LlamaServerClient` based on config

4. `deployment/ansible/playbooks/deploy.yml` - Ansible tasks for llama-server
   - Uses `ghcr.io/ggml-org/llama.cpp:server-cuda` official image
   - Symlinks to Ollama's base model blob (no duplication)
   - Applies LoRA adapter via `--lora` flag
   - Exposes on port 8081 (alongside Ollama on 11434)

5. `deployment/ansible/group_vars/all.yml` - Added `llama_server: 8081` port

**Deployment:**

```bash
# Deploy llama-server (does not replace Ollama, runs alongside)
cd /home/james/ATLAS/deployment/ansible
ansible-playbook playbooks/deploy.yml --tags llama-server -e deploy_llama_server=true

# Switch SentinelCollector to use llama-server (edit appsettings.json):
# "Extraction": {
#   "Backend": "LlamaServer",
#   "LlamaServerEndpoint": "http://llama-server:8080"
# }
```

**Key Differences from Ollama:**

| Aspect | Ollama | llama-server |
|--------|--------|--------------|
| Lifecycle | Idle caching, auto-unload | Server-managed, always loaded |
| Memory | May fragment over time | Clean state on restart |
| API | `/api/generate` | `/completion` |
| Model format | Modelfile + adapter | GGUF + `--lora` flag |

## Lessons Learned
1. **Prompt-schema alignment**: When validation logic enforces constraints, those constraints must be explicitly documented in the prompt with examples
2. **LLM schema inference limitations**: Even with JSON schema, LLMs benefit from explicit rules and examples rather than inferring constraints
3. **Observability value**: The extraction validation retry loop with detailed error messages made root cause identification straightforward
4. **File watching**: The FileBasedPromptProvider's hot-reload capability allows prompt fixes without service restarts
