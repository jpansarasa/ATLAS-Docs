# Sentinel Extraction Model: QLoRA Fine-Tuning Plan

## Overview

Fine-tune qwen2.5:32b-instruct using QLoRA to improve extraction accuracy and reduce the need for the 4-step Chain of Verification (CoVe) pipeline.

### Goals
1. Reduce extraction pipeline from 4 LLM calls to 1
2. Eliminate JSON parsing errors (empty strings, invalid types)
3. Improve precision on economic quantitative data
4. Reduce prompt size and inference latency

### Current Pain Points
| Issue | Impact | Fine-tuning Solution |
|-------|--------|---------------------|
| Empty strings for null values | Parse errors, code workarounds | Train on correct null handling |
| Extracts non-quantitative data | Noise in dataset | Train to skip metadata/dates |
| 4-step CoVe required | 4x latency, 4x compute | Single-call accurate extraction |
| Long prompts with examples | Slower inference | Internalized schema knowledge |
| Inconsistent field ordering | Minor | Consistent output format |

---

## Phase 1: Training Data Generation

### 1.0 Answer-First LLM Generation (Primary Approach)

**Key Insight**: Define the target extractions first, then use an LLM to creatively generate diverse input texts that contain those values. Validate outputs with simple text search.

```
Traditional: Documents → Annotation → Training Data (error-prone, slow)
Answer-First: Define Outputs → LLM Generates Inputs → Validate → Training Data
```

**Why LLM Generation > Programmatic Templates**:
| Aspect | Programmatic Templates | LLM Generation |
|--------|------------------------|----------------|
| Phrasing diversity | Limited to templates | Infinite variety |
| Language patterns | Repetitive, robotic | Natural, varied |
| Model learns | Specific phrases | True extraction task |
| Spin/bias coverage | Fixed sentiment words | Creative expressions |
| Scalability | Template-bound | Unlimited |

**Generation Flow**:
```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. Define Target Extraction                                         │
│    {value: 4.1, unit: percent, metric: "unemployment", period: Dec} │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          v
┌─────────────────────────────────────────────────────────────────────┐
│ 2. LLM Generation Prompt                                            │
│    "Write a sentence about unemployment being 4.1% in December.     │
│     Tone: negative/pessimistic. Be creative with phrasing.          │
│     Must include '4.1' and 'December' verbatim."                    │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          v
┌─────────────────────────────────────────────────────────────────────┐
│ 3. LLM Output                                                       │
│    "The labor market deteriorated further as unemployment crept up  │
│     to 4.1% in December, defying optimistic forecasts."             │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          v
┌─────────────────────────────────────────────────────────────────────┐
│ 4. Validation (simple text search)                                  │
│    "4.1" in text? ✓   "December" in text? ✓   → ACCEPT              │
│    If validation fails → retry with new generation                  │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          v
┌─────────────────────────────────────────────────────────────────────┐
│ 5. Training Pair                                                    │
│    {input: generated_text, output: target_extraction}               │
└─────────────────────────────────────────────────────────────────────┘
```

**Validation Rules**:
- All numeric values must appear verbatim in generated text
- Period references must be present (month, quarter, year)
- Retry up to 3 times if validation fails
- Log failures for analysis

### 1.1 Data Sources

**Primary: LLM-Generated Synthetic Data** (see `generate_training_data.py`)
- Define target extractions with required values
- LLM generates creative, natural text containing those values
- Simple text search validates required values are present
- Perfect ground truth by construction (values defined, then validated)

**Secondary: Verified Extractions** (supplement synthetic data)
```sql
-- High-confidence extractions with complete fields
SELECT
    rc.source,
    rc.content_type,
    rc.raw_file_path,
    eo.text_quote,
    eo.description,
    eo.value,
    eo.unit,
    eo.period,
    eo.source_entity,
    eo.is_comparison,
    eo.confidence,
    eo.certainty
FROM sentinel.extracted_observations eo
JOIN sentinel.raw_content rc ON eo.raw_content_id = rc.id
WHERE eo.confidence >= 0.9
  AND eo.value IS NOT NULL
  AND eo.description IS NOT NULL
  AND eo.description != ''
  AND eo.text_quote IS NOT NULL;
```

**Tertiary: Manual Curation**
- Review and correct extraction errors from production
- Add edge cases discovered during operation

### 1.2 Training Data Schema

```json
{
  "instruction": "Extract quantitative economic data from the following content.",
  "input": {
    "source": "BLS",
    "content_type": "text/html",
    "content": "Total nonfarm payroll employment rose by 256,000 in December..."
  },
  "output": [
    {
      "description": "nonfarm payrolls",
      "text_quote": "Total nonfarm payroll employment rose by 256,000 in December",
      "value": 256000,
      "unit": "count",
      "period": "2024-12",
      "source_entity": "BLS",
      "is_comparison": false,
      "confidence": 0.95,
      "certainty": "definite"
    }
  ]
}
```

### 1.3 Data Collection Pipeline

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ Current CoVe    │────>│ Quality Filter   │────>│ Training        │
│ Extractions     │     │ (confidence>0.9) │     │ Dataset         │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                                                          │
                                                          v
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ Manual Review   │────>│ Correction &     │────>│ Curated         │
│ Interface       │     │ Annotation       │     │ Examples        │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                                                          │
                                                          v
                                                 ┌─────────────────┐
                                                 │ Negative        │
                                                 │ Examples        │
                                                 │ (no data)       │
                                                 └─────────────────┘
```

### 1.4 Target Dataset Size

| Category | Base Templates | Variations Each | Total Examples |
|----------|----------------|-----------------|----------------|
| Employment metrics | 20 | 15 | 300 |
| Inflation/CPI | 15 | 15 | 225 |
| GDP/growth | 15 | 15 | 225 |
| Trade/deficit | 10 | 15 | 150 |
| Consumer confidence | 10 | 15 | 150 |
| Fed/monetary policy | 15 | 15 | 225 |
| Negation patterns | 20 | 10 | 200 |
| Spin/bias variations | 30 | 20 | 600 |
| Multi-value sentences | 20 | 10 | 200 |
| Comparison (actual vs expected) | 15 | 10 | 150 |
| Negative examples (no data) | 25 | 8 | 200 |
| **Total** | **195** | | **2,625** |

**Key**: Each base template defines a target extraction. Variations generate different text phrasings that all produce the same extraction.

### 1.5 Spin/Bias Augmentation Strategy

**Problem**: LLMs trained on human text absorb editorial bias. Same number with different framing may extract differently.

**Solution**: For each training example, generate 3 versions with identical expected outputs:
1. **Neutral** - factual, unbiased language (original)
2. **Positive spin** - optimistic framing ("surged", "beat expectations", "robust")
3. **Negative spin** - pessimistic framing ("disappointing", "fell short", "stubbornly elevated")

**Augmentation Templates**:

```python
SPIN_TEMPLATES = {
    "neutral": [
        "{metric} was {value}{unit} in {period}.",
        "{metric} {changed} to {value}{unit} in {period}.",
    ],
    "positive": [
        "{metric} {surged_to} {value}{unit} in {period}, {beating_expectations}.",
        "{metric} improved to {value}{unit} in {period}, signaling {strength}.",
        "A {robust} {value}{unit} {metric} was recorded in {period}.",
    ],
    "negative": [
        "{metric} {remained_at} a {disappointing} {value}{unit} in {period}.",
        "{metric} {fell_to} {value}{unit} in {period}, {missing_expectations}.",
        "A mere {value}{unit} {metric} was recorded in {period}, {weakness}.",
    ],
}

SENTIMENT_WORDS = {
    "surged_to": ["surged to", "jumped to", "soared to", "climbed to"],
    "remained_at": ["remained stubbornly at", "stayed elevated at", "held at"],
    "fell_to": ["slipped to", "declined to", "dropped to", "fell to"],
    "disappointing": ["disappointing", "lackluster", "weak", "underwhelming"],
    "robust": ["robust", "solid", "strong", "healthy"],
    "beating_expectations": ["beating expectations", "exceeding forecasts", "surprising to the upside"],
    "missing_expectations": ["missing expectations", "falling short of forecasts", "disappointing analysts"],
    "strength": ["economic strength", "labor market resilience", "solid fundamentals"],
    "weakness": ["raising concerns", "signaling weakness", "dampening optimism"],
}
```

**Augmentation Script**:

```python
def augment_with_spin(example: dict) -> list[dict]:
    """Generate neutral, positive, and negative spin versions of a training example."""

    original_content = example["input"]["content"]
    expected_output = example["output"]

    augmented = []

    for spin in ["neutral", "positive", "negative"]:
        # Rewrite content with spin while preserving numbers
        spun_content = apply_spin(original_content, spin)

        augmented.append({
            "instruction": example["instruction"],
            "input": {
                **example["input"],
                "content": spun_content,
                "spin_variant": spin,  # Track for analysis
            },
            # OUTPUT STAYS THE SAME - this is the key!
            "output": expected_output
        })

    return augmented
```

**Key Principle**: The expected output is **identical** across all spin variants. This teaches the model that:
- "unemployment plunged to 4.1%" → value: 4.1
- "unemployment was 4.1%" → value: 4.1
- "unemployment remained elevated at 4.1%" → value: 4.1

### 1.6 Negation Test Cases

Special attention to negation patterns that break extraction:

| Pattern | Input | Expected Output |
|---------|-------|-----------------|
| Simple negation | "did not fall below 4.1%" | value: 4.1 |
| Double negation | "was not above 4.1%" | value: 4.1 |
| Negative change | "declining by 0.3%" | value: -0.3 |
| Threshold | "did not exceed 220,000" | value: 220000 |
| Conditional | "if unemployment does not rise above 4.5%" | value: 4.5, certainty: conditional |

Generate explicit training examples for each negation pattern to prevent the model from outputting phrases like `"value": "did not fall below 4.1%"`.

### 1.7 Data Generation Script

Create `SentinelCollector/scripts/generate_training_data.py` - see actual implementation.

**Usage**:
```bash
# Generate full training dataset
python3 SentinelCollector/scripts/generate_training_data.py --output training_data.json

# Generate with specific categories
python3 SentinelCollector/scripts/generate_training_data.py --categories employment,inflation

# Preview without writing
python3 SentinelCollector/scripts/generate_training_data.py --preview --count 5
```

### 1.7.1 Legacy: Database Export Script

For supplementing synthetic data with real verified extractions, use `SentinelCollector/scripts/export_training_data.py`:

```python
#!/usr/bin/env python3
"""Export verified extractions for QLoRA training."""

import json
import psycopg2
from pathlib import Path

QUERY = """
SELECT
    rc.id as content_id,
    rc.source,
    rc.content_type,
    rc.raw_file_path,
    json_agg(json_build_object(
        'description', eo.description,
        'text_quote', eo.text_quote,
        'value', eo.value,
        'unit', eo.unit,
        'period', eo.period,
        'source_entity', eo.source_entity,
        'is_comparison', eo.is_comparison,
        'confidence', eo.confidence,
        'certainty', eo.certainty
    )) as extractions
FROM sentinel.raw_content rc
JOIN sentinel.extracted_observations eo ON eo.raw_content_id = rc.id
WHERE eo.confidence >= 0.9
  AND eo.value IS NOT NULL
  AND eo.description IS NOT NULL
  AND eo.description != ''
GROUP BY rc.id, rc.source, rc.content_type, rc.raw_file_path
HAVING COUNT(*) >= 1;
"""

def export_training_data(output_path: str):
    conn = psycopg2.connect(
        host="localhost",
        port=5432,
        database="atlas_data",
        user="atlas_user",
        password="atlas_secure_password_2025"
    )

    training_data = []

    with conn.cursor() as cur:
        cur.execute(QUERY)
        for row in cur.fetchall():
            content_id, source, content_type, raw_file_path, extractions = row

            # Read raw content
            content = Path(raw_file_path).read_text() if raw_file_path else ""

            training_data.append({
                "instruction": "Extract quantitative economic data from the following content. Return a JSON array of extractions with fields: description, text_quote, value, unit, period, source_entity, is_comparison, confidence, certainty. Return an empty array [] if no quantitative data is found.",
                "input": {
                    "source": source,
                    "content_type": content_type,
                    "content": content[:8000]  # Truncate for context window
                },
                "output": extractions
            })

    with open(output_path, 'w') as f:
        json.dump(training_data, f, indent=2)

    print(f"Exported {len(training_data)} training examples")

if __name__ == "__main__":
    export_training_data("/opt/ai-inference/training-data/sentinel-extraction-v1.json")
```

### 1.8 Compact Prompt Format

**Discovery**: A/B testing showed that compact symbolic notation performs identically to verbose natural language prompts while using **57% fewer tokens**.

**Benefits for Training**:
| Metric | Verbose Prompt | Compact Prompt |
|--------|---------------|----------------|
| Size | 9,296 chars | 3,960 chars |
| Token reduction | - | 57% |
| Extraction accuracy | 100% | 100% |
| Spin/negation handling | ✓ | ✓ |

**Compact Syntax Elements**:
```
SYMBOLS:
  → : implies/produces
  ∧ : AND
  ¬ : NOT
  | : OR
  ✗ : anti-pattern (do NOT do this)
  ✓ : correct pattern

STRUCTURE:
  RULE_NAME: condition → outcome
  examples: "input" → value: output
  ✗ bad_pattern # comment explaining why
```

**Training Data Format**: Use compact instruction format to teach the model efficient rule parsing:

```json
{
  "instruction": "# EXTRACTION.RULES\n## RULES\nDESCRIPTION: required ∧ ¬empty\nVALUE: numeric_only ¬ phrases\n  ✗ value: \"did not fall below 4.1%\"\n  ✓ value: 4.1",
  "input": {
    "source": "BLS",
    "content": "The unemployment rate was 4.1 percent in December."
  },
  "output": [{"description": "unemployment rate", "value": 4.1, ...}]
}
```

**Rationale**:
1. Smaller prompts → model learns task semantics, not verbosity
2. Symbolic rules → clearer decision boundaries
3. Token efficiency → more context window for actual content
4. Matches production prompt format → no train/inference mismatch

---

## Phase 2: QLoRA Training

### 2.1 Hardware Requirements

| Resource | Requirement | Mercury Server |
|----------|-------------|----------------|
| GPU VRAM | 24-32 GB | 32 GB (RTX 4090) |
| System RAM | 64 GB | 128 GB |
| Storage | 100 GB | Available |

### 2.2 Training Configuration

```yaml
# qlora_config.yaml
model:
  base: "qwen2.5:32b-instruct"
  quantization: "4bit"  # QLoRA

lora:
  r: 64              # LoRA rank
  alpha: 128         # LoRA alpha
  dropout: 0.05
  target_modules:
    - q_proj
    - k_proj
    - v_proj
    - o_proj
    - gate_proj
    - up_proj
    - down_proj

training:
  epochs: 3
  batch_size: 1
  gradient_accumulation: 8
  learning_rate: 2e-4
  warmup_ratio: 0.03
  max_seq_length: 8192

data:
  train_split: 0.9
  eval_split: 0.1

output:
  model_name: "sentinel-extraction-v1"
  save_path: "/opt/ai-inference/models/sentinel-extraction-v1"
```

### 2.3 Training Script

```python
#!/usr/bin/env python3
"""QLoRA fine-tuning for Sentinel extraction model."""

from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from datasets import load_dataset
import torch

def train():
    # 4-bit quantization config
    bnb_config = BitsAndBytesConfig(
        load_in_4bit=True,
        bnb_4bit_quant_type="nf4",
        bnb_4bit_compute_dtype=torch.bfloat16,
        bnb_4bit_use_double_quant=True,
    )

    # Load base model
    model = AutoModelForCausalLM.from_pretrained(
        "Qwen/Qwen2.5-32B-Instruct",
        quantization_config=bnb_config,
        device_map="auto",
        trust_remote_code=True,
    )

    # LoRA config
    lora_config = LoraConfig(
        r=64,
        lora_alpha=128,
        target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                       "gate_proj", "up_proj", "down_proj"],
        lora_dropout=0.05,
        bias="none",
        task_type="CAUSAL_LM",
    )

    model = prepare_model_for_kbit_training(model)
    model = get_peft_model(model, lora_config)

    # Load training data
    dataset = load_dataset("json", data_files={
        "train": "/opt/ai-inference/training-data/sentinel-extraction-v1.json"
    })

    # Training...
    # (Full training loop implementation)

if __name__ == "__main__":
    train()
```

### 2.4 Ollama Integration

After training, convert to GGUF and create Ollama Modelfile:

```dockerfile
# Modelfile.sentinel-extraction
FROM /opt/ai-inference/models/sentinel-extraction-v1.gguf

PARAMETER temperature 0.1
PARAMETER top_p 0.9
PARAMETER num_ctx 8192

SYSTEM """You are an economic data extraction system. Extract quantitative data points from content and return a JSON array. Each extraction must have: description, text_quote, value, unit, period, source_entity, is_comparison, confidence, certainty. Return [] if no quantitative data is found."""
```

```bash
# Create Ollama model
ollama create sentinel-extraction -f Modelfile.sentinel-extraction
```

---

## Phase 3: Evaluation

### 3.1 Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| JSON Parse Success | >99% | No parsing errors |
| Field Completeness | >98% | Required fields present |
| Value Accuracy | >95% | Extracted values match source |
| Precision | >90% | Relevant extractions / total |
| Recall | >85% | Found / actual data points |
| Latency | <30s | Single extraction call |

### 3.2 Evaluation Dataset

Hold out 10% of curated data for evaluation:
- 80 positive examples
- 20 negative examples
- Manual accuracy verification

### 3.3 A/B Testing

Run both pipelines in parallel:
1. Current CoVe (4-step)
2. Fine-tuned single-call

Compare:
- Extraction quality
- Latency
- Resource usage

---

## Phase 4: Deployment

### 4.1 Rollout Strategy

1. **Shadow mode**: Run fine-tuned model alongside CoVe, log differences
2. **Canary**: 10% of extractions use fine-tuned model
3. **Gradual rollout**: Increase to 50%, then 100%
4. **Fallback**: Keep CoVe available for edge cases

### 4.2 Configuration

```json
// appsettings.json
{
  "Extraction": {
    "Model": "sentinel-extraction:latest",
    "FallbackModel": "qwen2.5:32b-instruct",
    "UseFallbackOnError": true,
    "EnableCoVe": false
  }
}
```

### 4.3 Monitoring

- Track parse error rate
- Monitor extraction quality scores
- Alert on accuracy degradation
- Log model version with each extraction

---

## Timeline

| Phase | Duration | Prerequisites |
|-------|----------|---------------|
| 1. Data Collection | 1-2 weeks | 500+ verified extractions |
| 2. Training | 2-3 days | Training data ready |
| 3. Evaluation | 1 week | Trained model |
| 4. Deployment | 1-2 weeks | Evaluation passed |

**Current Status**: Phase 1 in progress - collecting verified extractions via CoVe pipeline.

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Overfitting to current sources | Include diverse source types in training |
| Loss of generalization | Keep base model as fallback |
| Training data quality | Manual review of training examples |
| VRAM constraints | QLoRA reduces memory requirements |
| Model drift over time | Periodic retraining with new data |

---

## Success Criteria

- [ ] 1,000+ training examples collected
- [ ] QLoRA training completes successfully
- [ ] Evaluation metrics meet targets
- [ ] Shadow mode shows comparable quality
- [ ] Production deployment with <1% error rate
- [ ] 4x latency improvement (4 calls → 1)
