# Phase 4.7s.2 — LoRA eval REPORT (checkpoint `iter3-ckpt-250`)

Generated: 2026-05-26T14:22:35Z

## Validation split

- n = 355
- overall accuracy: 0.2507
- macro F1: 0.2209
- micro F1: 0.2507
- Foundry-agreement subset (n=107): 0.0093

### Per-class metrics

| label | precision | recall | F1 | support |
|---|---:|---:|---:|---:|
| consistent | 0.692 | 0.551 | 0.614 | 98 |
| related | 0.000 | 0.000 | 0.000 | 57 |
| unrelated | 0.889 | 0.050 | 0.095 | 159 |
| insufficient_evidence | 0.101 | 0.659 | 0.175 | 41 |

### Confusion matrix (validation)

| true \ pred | consistent | related | unrelated | insufficient_evidence |
|---|---:|---:|---:|---:|
| consistent | 54 | 0 | 0 | 44 |
| related | 2 | 0 | 0 | 55 |
| unrelated | 9 | 0 | 8 | 142 |
| insufficient_evidence | 13 | 0 | 1 | 27 |

## Held-out test split

- n = 1560
- overall accuracy: 0.2474
- macro F1: 0.2275
- micro F1: 0.2474
- Foundry-agreement subset (n=563): 0.0036

### Per-class metrics

| label | precision | recall | F1 | support |
|---|---:|---:|---:|---:|
| consistent | 0.774 | 0.718 | 0.745 | 419 |
| related | 0.000 | 0.000 | 0.000 | 198 |
| unrelated | 0.978 | 0.052 | 0.098 | 869 |
| insufficient_evidence | 0.036 | 0.541 | 0.067 | 74 |

### Confusion matrix (held-out test)

| true \ pred | consistent | related | unrelated | insufficient_evidence |
|---|---:|---:|---:|---:|
| consistent | 301 | 0 | 0 | 118 |
| related | 4 | 0 | 0 | 194 |
| unrelated | 51 | 0 | 45 | 773 |
| insufficient_evidence | 33 | 0 | 1 | 40 |

