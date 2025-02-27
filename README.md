# ESM++
[ESM++](https://github.com/Synthyra) is a faithful implementation of [ESMC](https://www.evolutionaryscale.ai/blog/esm-cambrian) ([license](https://www.evolutionaryscale.ai/policies/cambrian-open-license-agreement)) that allows for batching and standard Huggingface compatibility without requiring the ESM Python package.


## Use with 🤗 transformers
```python
from transformers import AutoModelForMaskedLM
model = AutoModelForMaskedLM.from_pretrained('Synthyra/ESMplusplus_small', trust_remote_code=True)
tokenizer = model.tokenizer

sequences = ['MPRTEIN', 'MSEQWENCE']
tokenized = tokenizer(sequences, padding=True, return_tensors='pt')

# tokenized['labels'] = tokenized['input_ids'].clone() # correctly mask input_ids and set unmasked instances of labels to -100 for MLM training

output = model(**tokenized) # get all hidden states with output_hidden_states=True
print(output.logits.shape) # language modeling logits, (batch_size, seq_len, vocab_size), (2, 11, 64)
print(output.last_hidden_state.shape) # last hidden state of the model, (batch_size, seq_len, hidden_size), (2, 11, 960)
print(output.loss) # language modeling loss if you passed labels
#print(output.hidden_states) # all hidden states if you passed output_hidden_states=True (in tuple)
```

ESM++ also supports sequence and token level classification tasks like ESM2. Simply pass the number of labels during initialization.

```python
from transformers import AutoModelForSequenceClassification, AutoModelForTokenClassification

model = AutoModelForSequenceClassification.from_pretrained('Synthyra/ESMplusplus_small', num_labels=2, trust_remote_code=True)
logits = model(**tokenized).logits
print(logits.shape) # (batch_size, num_labels), (2, 2)
```

ESM++ weights are fp32 by default. You can load them in fp16 or bf16 like this:
```python
import torch
model = AutoModelForMaskedLM.from_pretrained('Synthyra/ESMplusplus_small', trust_remote_code=True, torch_dtype=torch.float16) # or torch.bfloat16
```

## Embed entire datasets with no new code
To embed a list of protein sequences **fast**, just call embed_dataset. Sequences are sorted to reduce padding tokens, so the progress bar is usually much longer than the actual time.
```python
embeddings = model.embed_dataset(
    sequences=sequences, # list of protein strings
    batch_size=16, # embedding batch size
    max_len=2048, # truncate to max_len
    full_embeddings=True, # return residue-wise embeddings
    full_precision=False, # store as float32
    pooling_type='mean', # use mean pooling if protein-wise embeddings
    num_workers=0, # data loading num workers
    sql=False, # return dictionary of sequences and embeddings
)

_ = model.embed_dataset(
    sequences=sequences, # list of protein strings
    batch_size=16, # embedding batch size
    max_len=2048, # truncate to max_len
    full_embeddings=True, # return residue-wise embeddings
    full_precision=False, # store as float32
    pooling_type='mean', # use mean pooling if protein-wise embeddings
    num_workers=0, # data loading num workers
    sql=True, # store sequences in local SQL database
    sql_db_path='embeddings.db', # path to .db file of choice
)
```

### Comparison across floating-point precision and implementations
We measured the difference of the last hidden states of the fp32 weights vs. fp16 or bf16. We find that the fp16 is closer to the fp32 outputs, so we recommend loading in fp16.
Please note that the ESM package also loads ESMC in fp32 but casts to bf16 by default, which has its share of advantages and disadvantages in inference / training - so load whichever you like for half precision.

Average MSE FP32 vs. FP16: 0.00000003

Average MSE FP32 vs. BF16: 0.00000140

We also measured the difference between the outputs of ESM++ vs. ESMC (both in bfloat16) on 1000 random sequences to ensure compliance with the ESM package.

Average MSE of last hidden state: 7.74e-10

You can load the weights from the ESM package instead of transformers by replacing .from_pretrained(...) to .from_pretrained_esm('esmc_300m')

## Model probes
We employ linear probing techniques on various PLMs and standard datasets, similar our previous [paper](https://www.biorxiv.org/content/10.1101/2024.07.30.605924v1), to access the intrinsic correlation between pooled hidden states and valuable properties. ESMC (and thus ESM++) perform very well.

The plot below showcases performance normalized between the negative control (random vector embeddings) and the best performer. Classification task scores are averaged between MCC and F1 (or F1max for multilabel) and regression tasks are averaged between Spearman rho and R2.
![performance_heatmap](https://github.com/user-attachments/assets/9e9f1517-8a47-489e-8f9e-5ce92e249dee)


## Inference speeds
We look at various ESM models and their throughput on an H100. Adding efficient batching between ESMC and ESM++ significantly improves the throughput. ESM++ small is even faster than ESM2-35M with long sequences!
The most gains will be seen with PyTorch > 2.5 on linux machines.
![model_throughput (1)](https://github.com/user-attachments/assets/6047483a-f4b5-471a-9c34-6921aec714a5)


### Citation
If you use any of this implementation or work please cite it (as well as the ESMC preprint). Bibtex for both coming soon.

```
@misc {ESMPlusPlus,
	author       = { Hallee, L. and Bichara, D. and Gleghorn, J, P. },
	title        = { ESMPlusPlus },
	year         = 2024,
	url          = { https://huggingface.co/Synthyra/ESMplusplus_small },
	doi          = { 10.57967/hf/3725 },
	publisher    = { Hugging Face }
}
```
