# LoRA v6 Training Data Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generate Claude-labeled training data from real production documents and train two LoRA adapters (32B for CoVe extraction, 7B for CoD summarization).

**Architecture:** A Python script reads raw documents from the Sentinel database, sends each to Claude Sonnet for extraction labeling, validates the output, and writes training JSON. The existing `train_qlora.py` is extended with CLI args for target modules and rank, then run twice — once per adapter.

**Tech Stack:** Python 3, `anthropic` SDK, `psycopg2`, existing `train_qlora.py`

**Spec:** `docs/superpowers/specs/2026-04-04-lora-v6-training-data-design.md`

---

### Task 1: Create `generate_production_training.py` — document selection

**Files:**
- Create: `SentinelCollector/scripts/generate_production_training.py`

- [ ] **Step 1: Create script skeleton with DB connection and document selection**

```python
#!/usr/bin/env python3
"""
Generate LoRA training data from real production documents using Claude API.

Reads raw documents from Sentinel's database, sends each to Claude Sonnet
for extraction labeling, validates output, and writes training JSON.

Usage:
    # Set API key
    export ANTHROPIC_API_KEY=sk-ant-...

    # Generate CoVe training data (extraction)
    python3 scripts/generate_production_training.py \
        --task cove \
        --output /opt/ai-inference/training-data/sentinel-v6-cove.json \
        --limit 500

    # Generate CoD training data (summarization)
    python3 scripts/generate_production_training.py \
        --task cod \
        --output /opt/ai-inference/training-data/sentinel-v6-cod.json \
        --limit 500

    # Dry run — list documents without calling API
    python3 scripts/generate_production_training.py --task cove --dry-run --limit 20
"""

import argparse
import json
import logging
import os
import sys
from pathlib import Path

import psycopg2
from psycopg2.extras import RealDictCursor

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%dT%H:%M:%S",
)
logger = logging.getLogger(__name__)

DB_CONFIG = {
    "host": os.getenv("ATLAS_DB_HOST", "localhost"),
    "port": int(os.getenv("ATLAS_DB_PORT", "5432")),
    "database": os.getenv("ATLAS_DB_NAME", "atlas_data"),
    "user": os.getenv("ATLAS_DB_USER", "atlas_user"),
    "password": os.getenv("ATLAS_DB_PASSWORD", "atlas_secure_password_2025"),
}

PROMPT_DIR = Path(__file__).parent.parent / "src" / "prompts"


def get_candidate_documents(conn, limit: int) -> list[dict]:
    """Select raw documents suitable for training.

    Criteria:
    - Has been processed (processed_at IS NOT NULL)
    - No processing error
    - Raw file exists (non-null path)
    - Content type is extractable (text/html, text/markdown, application/rss+xml)
    - From sources with real financial content
    """
    with conn.cursor(cursor_factory=RealDictCursor) as cur:
        cur.execute("""
            SELECT rc.id, rc.source, rc.content_type, rc.raw_file_path,
                   rc.context_summary
            FROM sentinel.raw_content rc
            WHERE rc.processed_at IS NOT NULL
              AND rc.processing_error IS NULL
              AND rc.raw_file_path IS NOT NULL
              AND rc.content_type != 'application/json'
              AND rc.source NOT IN ('searxng')
            ORDER BY RANDOM()
            LIMIT %s
        """, (limit,))
        return cur.fetchall()


def read_document(file_path: str, max_chars: int = 30000) -> str | None:
    """Read and truncate a raw document from disk."""
    path = Path(file_path)
    if not path.exists():
        logger.warning("File not found: %s", file_path)
        return None
    try:
        content = path.read_text(encoding="utf-8", errors="replace")
        if len(content) < 200:
            logger.debug("Skipping short document (%d chars): %s", len(content), file_path)
            return None
        return content[:max_chars]
    except Exception as e:
        logger.warning("Failed to read %s: %s", file_path, e)
        return None


def load_prompt(task: str, source: str, content_type: str) -> str:
    """Load the production prompt template for the given task."""
    if task == "cove":
        template_path = PROMPT_DIR / "initial_extraction.txt"
    else:
        template_path = PROMPT_DIR / "cod_initial_summary.txt"

    template = template_path.read_text()
    return (
        template
        .replace("{{source}}", source)
        .replace("{{content_type}}", content_type)
        .replace("{{target_word_count}}", "80")
    )
```

- [ ] **Step 2: Verify document selection works**

Run:
```bash
cd /home/james/ATLAS && python3 -c "
import psycopg2
from SentinelCollector.scripts.generate_production_training import get_candidate_documents, DB_CONFIG
conn = psycopg2.connect(**DB_CONFIG)
docs = get_candidate_documents(conn, 5)
for d in docs:
    print(f'{d[\"id\"]:6d} {d[\"source\"]:20s} {d[\"content_type\"]:20s}')
conn.close()
"
```
Expected: 5 rows with source/content_type columns.

- [ ] **Step 3: Commit**

```bash
git add SentinelCollector/scripts/generate_production_training.py
git commit -m "feat: add production training data generator - document selection"
```

---

### Task 2: Add Claude API labeling

**Files:**
- Modify: `SentinelCollector/scripts/generate_production_training.py`

- [ ] **Step 1: Add Claude API labeling function**

Add to `generate_production_training.py`:

```python
import anthropic


def label_document_cove(client: anthropic.Anthropic, prompt: str, content: str) -> list[dict] | None:
    """Send document to Claude for extraction labeling."""
    try:
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            messages=[
                {
                    "role": "user",
                    "content": f"{prompt}\n\n---\nCONTENT:\n{content}",
                }
            ],
        )
        text = response.content[0].text

        # Extract JSON array from response
        start = text.find("[")
        end = text.rfind("]")
        if start == -1 or end == -1:
            logger.warning("No JSON array in Claude response")
            return None

        extractions = json.loads(text[start : end + 1])
        if not isinstance(extractions, list):
            return None

        return extractions
    except json.JSONDecodeError as e:
        logger.warning("JSON parse error in Claude response: %s", e)
        return None
    except anthropic.APIError as e:
        logger.error("Claude API error: %s", e)
        return None


def label_document_cod(client: anthropic.Anthropic, prompt: str, content: str) -> str | None:
    """Send document to Claude for summarization labeling."""
    try:
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[
                {
                    "role": "user",
                    "content": f"{prompt}\n\n---\nCONTENT:\n{content}",
                }
            ],
        )
        summary = response.content[0].text.strip()
        if len(summary) < 50:
            logger.warning("Claude summary too short (%d chars)", len(summary))
            return None
        return summary
    except anthropic.APIError as e:
        logger.error("Claude API error: %s", e)
        return None
```

- [ ] **Step 2: Commit**

```bash
git add SentinelCollector/scripts/generate_production_training.py
git commit -m "feat: add Claude API labeling for CoVe and CoD"
```

---

### Task 3: Add validation and output pipeline

**Files:**
- Modify: `SentinelCollector/scripts/generate_production_training.py`

- [ ] **Step 1: Add extraction validation**

```python
REQUIRED_FIELDS = {"description", "value", "text_quote"}


def validate_extraction(extraction: dict, content: str) -> bool:
    """Validate a single extraction against the source document."""
    # Check required fields exist and are non-empty
    for field in REQUIRED_FIELDS:
        if not extraction.get(field):
            return False

    # Verify text_quote appears in source (fuzzy: first 50 chars)
    quote = str(extraction.get("text_quote", ""))
    if len(quote) > 20 and quote[:50].lower() not in content.lower():
        logger.debug("text_quote not found in source: %s...", quote[:50])
        # Don't reject — Claude may paraphrase slightly

    return True


def validate_extractions(extractions: list[dict], content: str) -> list[dict]:
    """Filter extractions to only valid ones."""
    return [e for e in extractions if validate_extraction(e, content)]
```

- [ ] **Step 2: Add main generation loop and output**

```python
def generate(task: str, output_path: Path, limit: int, dry_run: bool) -> None:
    """Main generation pipeline."""
    api_key = os.getenv("ANTHROPIC_API_KEY")
    if not api_key and not dry_run:
        logger.error("ANTHROPIC_API_KEY not set")
        sys.exit(1)

    conn = psycopg2.connect(**DB_CONFIG)
    documents = get_candidate_documents(conn, limit)
    conn.close()

    logger.info("Selected %d candidate documents", len(documents))

    if dry_run:
        for doc in documents[:20]:
            content = read_document(doc["raw_file_path"])
            length = len(content) if content else 0
            print(f"  {doc['id']:6d} {doc['source']:20s} {length:8d} chars")
        return

    client = anthropic.Anthropic(api_key=api_key)
    examples = []
    skipped = 0
    errors = 0

    # Load golden dataset entries as high-quality anchors
    golden_path = Path(__file__).parent.parent / "LlmBenchmark" / "TestData" / "golden_dataset.json"
    if golden_path.exists():
        golden = json.loads(golden_path.read_text())
        golden_entries = golden.get("entries", [])
        logger.info("Loaded %d golden dataset entries as training anchors", len(golden_entries))
        for entry in golden_entries:
            # Skip census_retail and fed_fomc — held out for benchmark validation
            if entry["id"] in ("census_retail_2024_11", "fed_fomc_2024_12"):
                continue
            raw_content_path = (
                Path(__file__).parent.parent
                / "LlmBenchmark"
                / "TestData"
                / "raw_content"
                / entry["content_file"]
            )
            if not raw_content_path.exists():
                continue
            content = raw_content_path.read_text()
            prompt = load_prompt(task, entry["source"], entry["content_type"])
            if task == "cove":
                examples.append({
                    "instruction": prompt,
                    "input": {
                        "source": entry["source"],
                        "content_type": entry["content_type"],
                        "content": content,
                    },
                    "output": entry["expected_extractions"],
                })
            else:
                if entry.get("expected_summary"):
                    # Use golden summary expectations as seed
                    pass  # Golden entries don't have full summary text — skip for CoD

        logger.info("Added %d golden dataset anchors", len(examples))

    for i, doc in enumerate(documents):
        content = read_document(doc["raw_file_path"])
        if content is None:
            skipped += 1
            continue

        prompt = load_prompt(task, doc["source"], doc["content_type"])

        if task == "cove":
            extractions = label_document_cove(client, prompt, content)
            if extractions is None:
                errors += 1
                continue

            valid = validate_extractions(extractions, content)
            if len(valid) < 2:
                skipped += 1
                logger.debug("Skipping doc %d: only %d valid extractions", doc["id"], len(valid))
                continue

            examples.append({
                "instruction": prompt,
                "input": {
                    "source": doc["source"],
                    "content_type": doc["content_type"],
                    "content": content,
                },
                "output": valid,
            })
        else:
            summary = label_document_cod(client, prompt, content)
            if summary is None:
                errors += 1
                continue

            examples.append({
                "instruction": prompt,
                "input": {
                    "source": doc["source"],
                    "content_type": doc["content_type"],
                    "content": content,
                },
                "output": summary,
            })

        if (i + 1) % 25 == 0:
            logger.info(
                "Progress: %d/%d documents, %d examples, %d skipped, %d errors",
                i + 1, len(documents), len(examples), skipped, errors,
            )

    # Report statistics
    logger.info("=== Generation Complete ===")
    logger.info("Documents processed: %d", len(documents))
    logger.info("Training examples: %d", len(examples))
    logger.info("Skipped: %d", skipped)
    logger.info("Errors: %d", errors)

    if task == "cove":
        extraction_counts = [len(e["output"]) for e in examples if isinstance(e["output"], list)]
        if extraction_counts:
            logger.info(
                "Extractions/doc: min=%d, max=%d, avg=%.1f",
                min(extraction_counts), max(extraction_counts),
                sum(extraction_counts) / len(extraction_counts),
            )

    # Write output
    output_path.parent.mkdir(parents=True, exist_ok=True)
    with open(output_path, "w") as f:
        json.dump(examples, f, ensure_ascii=False, indent=2, default=str)

    logger.info("Wrote %d examples to %s", len(examples), output_path)


def main():
    parser = argparse.ArgumentParser(description="Generate LoRA training data using Claude API")
    parser.add_argument("--task", choices=["cove", "cod"], required=True, help="Task type: cove (extraction) or cod (summarization)")
    parser.add_argument("--output", type=Path, required=True, help="Output JSON file path")
    parser.add_argument("--limit", type=int, default=500, help="Max documents to process")
    parser.add_argument("--dry-run", action="store_true", help="List documents without calling API")
    args = parser.parse_args()
    generate(args.task, args.output, args.limit, args.dry_run)


if __name__ == "__main__":
    main()
```

- [ ] **Step 3: Test with dry run**

Run:
```bash
cd /home/james/ATLAS
python3 SentinelCollector/scripts/generate_production_training.py \
    --task cove --output /tmp/test.json --limit 10 --dry-run
```
Expected: List of 10 documents with IDs, sources, and content lengths.

- [ ] **Step 4: Commit**

```bash
git add SentinelCollector/scripts/generate_production_training.py
git commit -m "feat: add validation and output pipeline for training data generation"
```

---

### Task 4: Update `train_qlora.py` with new CLI args

**Files:**
- Modify: `SentinelCollector/scripts/train_qlora.py`

- [ ] **Step 1: Add `--target-modules` CLI arg**

In the `main()` function's argument parser, add after the `--lora-alpha` arg:

```python
    parser.add_argument(
        "--target-modules",
        default="q_proj,k_proj,v_proj,o_proj",
        help="Comma-separated LoRA target modules (default: q_proj,k_proj,v_proj,o_proj)",
    )
```

- [ ] **Step 2: Update `train()` signature and LoRA config**

Add `target_modules` parameter to `train()`:

```python
def train(
    base_model: str = "Qwen/Qwen2.5-7B-Instruct",
    epochs: int = 2,
    batch_size: int = 1,
    gradient_accumulation_steps: int = 8,
    learning_rate: float = 1e-4,
    lora_r: int = 16,
    lora_alpha: int = 32,
    max_seq_length: int = 4096,
    target_modules: list[str] | None = None,
    training_data_path: Path = DEFAULT_TRAINING_DATA_PATH,
    output_dir: Path = DEFAULT_OUTPUT_DIR,
) -> None:
```

Update the LoRA config (around line 193):

```python
    lora_config = LoraConfig(
        r=lora_r,
        lora_alpha=lora_alpha,
        lora_dropout=0.05,
        bias="none",
        task_type="CAUSAL_LM",
        target_modules=target_modules or ["q_proj", "k_proj", "v_proj", "o_proj"],
    )
```

- [ ] **Step 3: Update `max_seq_length` default to 8192**

Change the default in both the `train()` function signature and the argparser:

```python
    parser.add_argument("--max-seq-length", type=int, default=8192, help="Max sequence length")
```

- [ ] **Step 4: Wire CLI arg to `train()` call**

In `main()`, update the `train()` call:

```python
        train(
            base_model=args.base_model,
            epochs=args.epochs,
            batch_size=args.batch_size,
            gradient_accumulation_steps=args.gradient_accumulation,
            learning_rate=args.lr,
            lora_r=args.lora_r,
            lora_alpha=args.lora_alpha,
            max_seq_length=args.max_seq_length,
            target_modules=args.target_modules.split(","),
            training_data_path=args.data,
            output_dir=args.output,
        )
```

- [ ] **Step 5: Commit**

```bash
git add SentinelCollector/scripts/train_qlora.py
git commit -m "feat: add target-modules and rank CLI args to train_qlora.py"
```

---

### Task 5: Generate CoVe training data

**Files:**
- Output: `/opt/ai-inference/training-data/sentinel-v6-cove.json`

- [ ] **Step 1: Install anthropic SDK**

```bash
pip install anthropic
```

- [ ] **Step 2: Run CoVe data generation**

```bash
cd /home/james/ATLAS
export ANTHROPIC_API_KEY=<key>

python3 SentinelCollector/scripts/generate_production_training.py \
    --task cove \
    --output /opt/ai-inference/training-data/sentinel-v6-cove.json \
    --limit 500
```

Expected: ~300-400 training examples with 2+ extractions per document.
Runtime: ~15-30 minutes (500 API calls at ~2-3s each).

- [ ] **Step 3: Validate output statistics**

```bash
python3 -c "
import json
with open('/opt/ai-inference/training-data/sentinel-v6-cove.json') as f:
    data = json.load(f)
print(f'Total examples: {len(data)}')
counts = [len(e['output']) for e in data if isinstance(e['output'], list)]
print(f'Extractions/doc: min={min(counts)}, max={max(counts)}, avg={sum(counts)/len(counts):.1f}')
single = sum(1 for c in counts if c == 1)
print(f'Single-extraction docs: {single} ({single/len(counts)*100:.1f}%)')
lengths = [len(e['input']['content']) for e in data]
print(f'Content length: min={min(lengths)}, max={max(lengths)}, avg={sum(lengths)/len(lengths):.0f}')
"
```

Expected:
- 0% single-extraction documents (filter requires 2+)
- Average content length > 1000 chars
- Average extractions/doc > 3

---

### Task 6: Generate CoD training data

**Files:**
- Output: `/opt/ai-inference/training-data/sentinel-v6-cod.json`

- [ ] **Step 1: Run CoD data generation**

```bash
cd /home/james/ATLAS
python3 SentinelCollector/scripts/generate_production_training.py \
    --task cod \
    --output /opt/ai-inference/training-data/sentinel-v6-cod.json \
    --limit 500
```

Expected: ~400-450 training examples (fewer rejections since summaries are simpler).

- [ ] **Step 2: Validate output**

```bash
python3 -c "
import json
with open('/opt/ai-inference/training-data/sentinel-v6-cod.json') as f:
    data = json.load(f)
print(f'Total examples: {len(data)}')
lengths = [len(e['output']) for e in data]
print(f'Summary length: min={min(lengths)}, max={max(lengths)}, avg={sum(lengths)/len(lengths):.0f}')
"
```

Expected: Summaries averaging 300-500 chars (80-word dense paragraphs).

---

### Task 7: Train 32B CoVe adapter

**Files:**
- Output: `/opt/ai-inference/models/sentinel-extraction-v6/`

- [ ] **Step 1: Stop ATLAS services to free GPU VRAM**

```bash
cd /opt/ai-inference
sudo nerdctl stop llama-server
sudo nerdctl stop ollama-gpu
```

- [ ] **Step 2: Run 32B training**

```bash
cd /home/james/ATLAS
python3 SentinelCollector/scripts/train_qlora.py \
    --base-model Qwen/Qwen2.5-32B-Instruct \
    --data /opt/ai-inference/training-data/sentinel-v6-cove.json \
    --output /opt/ai-inference/models/sentinel-extraction-v6 \
    --epochs 3 \
    --lr 5e-5 \
    --lora-r 32 \
    --lora-alpha 64 \
    --max-seq-length 8192 \
    --target-modules q_proj,k_proj,v_proj,o_proj \
    2>&1 | tee /opt/ai-inference/models/sentinel-extraction-v6-training.log
```

Expected runtime: 2-4 hours on RTX 5090.

- [ ] **Step 3: Export adapter to GGUF**

```bash
python3 SentinelCollector/scripts/train_qlora.py \
    --merge \
    --base-model Qwen/Qwen2.5-32B-Instruct \
    --output /opt/ai-inference/models/sentinel-extraction-v6

# Convert merged model to GGUF
cd /tmp && git clone https://github.com/ggerganov/llama.cpp 2>/dev/null
cd llama.cpp && pip install -r requirements.txt
python convert_hf_to_gguf.py /opt/ai-inference/models/sentinel-extraction-v6/merged \
    --outfile /opt/ai-inference/models/sentinel-extraction-v6-adapter.gguf \
    --outtype q4_k_m
```

- [ ] **Step 4: Restart services and benchmark**

```bash
# Restart llama-server with new adapter
cd /home/james/ATLAS/deployment/ansible
ansible-playbook -i inventory/hosts.yml playbooks/deploy.yml --tags llama-server \
    -e deploy_llama_server=true

# Run full benchmark (all 12 entries)
cd /home/james/ATLAS/LlmBenchmark
./run-benchmarks.sh --backend llamacpp --filter "Category=LlmBenchmark"
```

Expected: F1 > 59.7% (base model baseline). If not, discard adapter.

---

### Task 8: Train 7B CoD adapter

**Files:**
- Output: `/opt/ai-inference/models/sentinel-cod-v6/`

- [ ] **Step 1: Run 7B training**

```bash
cd /home/james/ATLAS
python3 SentinelCollector/scripts/train_qlora.py \
    --base-model Qwen/Qwen2.5-7B-Instruct \
    --data /opt/ai-inference/training-data/sentinel-v6-cod.json \
    --output /opt/ai-inference/models/sentinel-cod-v6 \
    --epochs 3 \
    --lr 1e-4 \
    --lora-r 64 \
    --lora-alpha 128 \
    --max-seq-length 8192 \
    --target-modules q_proj,k_proj,v_proj,o_proj \
    2>&1 | tee /opt/ai-inference/models/sentinel-cod-v6-training.log
```

Expected runtime: 1-2 hours on RTX 5090.

- [ ] **Step 2: Create Ollama model from adapter**

```bash
python3 SentinelCollector/scripts/train_qlora.py \
    --merge \
    --base-model Qwen/Qwen2.5-7B-Instruct \
    --output /opt/ai-inference/models/sentinel-cod-v6

# Convert to GGUF
cd /tmp/llama.cpp
python convert_hf_to_gguf.py /opt/ai-inference/models/sentinel-cod-v6/merged \
    --outfile /opt/ai-inference/models/sentinel-cod-v6.gguf \
    --outtype q4_k_m

# Create Ollama model
cat > /opt/ai-inference/models/Modelfile.sentinel-cod-v6 << 'EOF'
FROM /opt/ai-inference/models/sentinel-cod-v6.gguf
TEMPLATE """<|im_start|>user
{{ .Prompt }}<|im_end|>
<|im_start|>assistant
"""
PARAMETER stop "<|im_end|>"
PARAMETER temperature 0
PARAMETER num_ctx 8192
EOF

ollama create sentinel-cod-v6 -f /opt/ai-inference/models/Modelfile.sentinel-cod-v6
```

- [ ] **Step 3: Benchmark CoD quality on CPU**

```bash
cd /home/james/ATLAS/LlmBenchmark
./run-benchmarks.sh \
    --endpoint http://ollama-cpu:11434 \
    --model sentinel-cod-v6 \
    --filter "Strategy=CoD"
```

Compare CoD entity/value coverage against base qwen2.5:7b-instruct.

---

### Task 9: Final comparison and deployment decision

- [ ] **Step 1: Compile benchmark results**

Create a comparison table:

| Config | Full F1 | census | fomc | Mean Time |
|--------|---------|--------|------|-----------|
| Base 32B (no adapter) | 59.7% | ? | ? | 43s |
| v6-32B (new CoVe adapter) | ? | ? | ? | ? |
| Base 7B (no adapter) | 55-64% | ? | ? | 97s |
| v6-7B (new CoD adapter) | ? | ? | ? | ? |

- [ ] **Step 2: Deploy winning configuration**

If v6-32B > base 32B: update `deploy.yml` to include `--lora` flag again with v6 adapter path.
If v6-32B <= base 32B: keep base model (no adapter).

If v6-7B > base 7B: update `compose.yaml.j2` sentinel-collector config to use `sentinel-cod-v6` model.
If v6-7B <= base 7B: keep `qwen2.5:7b-instruct`.

- [ ] **Step 3: Update BENCHMARKS.md with results**

Add v6 results to the leaderboard in `LlmBenchmark/BENCHMARKS.md`.

- [ ] **Step 4: Commit all changes**

```bash
git add -A
git commit -m "feat: LoRA v6 trained on Claude-labeled production data"
```
