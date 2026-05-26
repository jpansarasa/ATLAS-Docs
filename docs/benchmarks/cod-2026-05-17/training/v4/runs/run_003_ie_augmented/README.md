---
base_model: Qwen/Qwen2.5-32B-Instruct
library_name: peft
model_name: run_003_ie_augmented
tags:
- base_model:adapter:Qwen/Qwen2.5-32B-Instruct
- lora
- sft
- transformers
- trl
licence: license
pipeline_tag: text-generation
---

# Model Card for run_003_ie_augmented

This model is a fine-tuned version of [Qwen/Qwen2.5-32B-Instruct](https://huggingface.co/Qwen/Qwen2.5-32B-Instruct).
It has been trained using [TRL](https://github.com/huggingface/trl).

## Quick start

```python
from transformers import pipeline

question = "If you had a time machine, but could only go to the past or the future once and never return, which would you choose and why?"
generator = pipeline("text-generation", model="None", device="cuda")
output = generator([{"role": "user", "content": question}], max_new_tokens=128, return_full_text=False)[0]
print(output["generated_text"])
```

## Training procedure

 


This model was trained with SFT.

### Framework versions

- PEFT 0.19.1
- TRL: 0.19.1
- Transformers: 4.55.4
- Pytorch: 2.12.0
- Datasets: 4.8.5
- Tokenizers: 0.21.4

## Citations



Cite TRL as:
    
```bibtex
@misc{vonwerra2022trl,
	title        = {{TRL: Transformer Reinforcement Learning}},
	author       = {Leandro von Werra and Younes Belkada and Lewis Tunstall and Edward Beeching and Tristan Thrush and Nathan Lambert and Shengyi Huang and Kashif Rasul and Quentin Gallou{\'e}dec},
	year         = 2020,
	journal      = {GitHub repository},
	publisher    = {GitHub},
	howpublished = {\url{https://github.com/huggingface/trl}}
}
```