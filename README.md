# CoLLiE

Full Parameter Fine-tuning of Large Language Models on Consumer Hardware.

## Beta release (0.0.1)

To install:

```bash
python setup.py install
```

## Speed Benchmark

### Throughput when training LLaMA with CoLLiE

| Model Size | Parallel |  Optimizer  |   Hardware   | Seq. Length | Batch Size | Sec. per Step |  TGS  |
|:----------:|:--------:|:-----------:|:------------:|:-----------:|:----------:|:-------------:|:-----:|
|     7B     | Pipeline |     SGD     | 8 * RTX 3090 |    1024     |    128     |     31.0      | 528.5 |
|     7B     |  ZeRO-3  | Inplace SGD | 1 * RTX 3090 |    1024     |     3      |      4.0      | 832.5 |
|    13B     | Pipeline |     SGD     | 8 * RTX 3090 |    1024     |    128     |     47.0      | 348.5 |
|    13B     |  ZeRO-3  | Inplace SGD | 2 * RTX 3090 |    1024     |     4      |     11.7      | 350.1 |
|    30B     |  ZeRO-3  | Inplace SGD | 4 * RTX 3090 |     960     |     2      |     34.1      | 56.4  |
|    65B     |  ZeRO-3  | Inplace SGD | 8 * RTX 3090 |     512     |     2      |     67.1      | 15.3  |


<font size=2>* TGS = **T**okens / **G**PU / **S**econd</font>

## How to use CoLLiE

Here's a simple example to run pipeline parallel:

```python
# Command: $torchrun --nproc_per_node=8 train.py
from collie.models.llama_colossalai import ModelArgs, get_7B_llama, load_state_dict
from collie.models.llama_tokenizer import HFLikeTokenizer, Tokenizer
from collie.trainer.colossalai_trainer import ColossalaiTrainer, TrainerArgs
from torch.utils.data import DataLoader

tokenizer = HFLikeTokenizer(tokenizer=Tokenizer(model_path='./tokenizer.model'))
model_args = ModelArgs()
model_args.pp_size = 8 # your pipeline parallel degree
model = get_7B_llama(model_args=model_args)
state_dict = load_state_dict(
    protocol="file", 
    source="raw", 
    file_folder="./llama/7B",  # where consolidated.00.pth in it
    model_args=model_args)
model.load_state_dict(state_dict)
train_sample = {
    "input_ids": tokenizer("CoLLiE is a python package for training large language models", return_tensors="pt")["input_ids"].long()
}, tokenizer("CoLLiE is a python package for training large language models", return_tensors="pt")["input_ids"].long()
eval_sample = {
    "input_ids": tokenizer("CoLLiE is a python package for", return_tensors="pt")["input_ids"].long()
}, tokenizer("CoLLiE is a python package for", return_tensors="pt")["input_ids"].long()

train_dataloader = DataLoader(
    [train_sample for _ in range(1000)],
    batch_size=1
)
eval_dataloader = DataLoader(
    [eval_sample for _ in range(1000)],
    batch_size=1
)
def compute_metrics(batch, generated_batch, epoch, step):
    print("\n")
    print(tokenizer.decode(generated_batch[0]["input_ids"].tolist()))
    print("\n")
    
trainer = ColossalaiTrainer(model=model,
                            train_dataloader=train_dataloader,
                            eval_dataloader=eval_dataloader,
                            tokenizer=tokenizer,
                            compute_metrics=compute_metrics)
trainer.train()
```

To run tensor parallel with tensor parallel and inplace:

```python
# Command: $torchrun --nproc_per_node 2 train.py ./example/config/tensor_args.yaml
import collie
from collie.models import llama
import torch
from torch.utils.data import Dataset, DataLoader
from datasets import load_dataset, load_from_disk
from transformers import HfArgumentParser
from collie.arguments import ModelArguments, DataArguments, CollieArguments
import os


def collate_fn(batch, tknz, max_len=512):
    text = [e['text'] for e in batch]
    tknz_batch = tknz(
        text,
        max_length=max_len,
        padding='max_length',
        truncation=True,
        return_tensors='pt'
    )
    eos_tensors = torch.tensor([tknz.eos_token_id] * tknz_batch['input_ids'].shape[0]).unsqueeze(1)
    tknz_batch['input_ids'] = torch.cat((tknz_batch['input_ids'][:, :max_len-1], eos_tensors), dim=1)
    tknz_batch['attention_mask'] = tknz_batch['attention_mask'][:, :max_len]
    return {
        'input_ids': tknz_batch['input_ids'],
        'attention_mask': tknz_batch['attention_mask'],
        'labels': tknz_batch['input_ids']
    }


def compute_metrics(all_pred, eval_dataset):
    print(len(all_pred), len(eval_dataset))
    return {'my_metric': len(all_pred[0]) - len(eval_dataset[0])}  # dummy metric


local_rank, world_size = llama.setup_model_parallel()
parser = HfArgumentParser((ModelArguments, DataArguments, CollieArguments))
if sys.argv[-1].endswith(".yaml"):
    model_args, data_args, collie_args = parser.parse_yaml_file(yaml_file=os.path.abspath(sys.argv[-1]))
else:
    model_args, data_args, collie_args = parser.parse_args_into_dataclasses()

model, tokenizer = llama.load_model(
    ckpt_dir=model_args.model_name_or_path,  # 7B, 13B, 30B, 65B
    tokenizer_path=os.path.join(model_args.llama_dir, 'tokenizer.model'),
    local_rank=local_rank,
    world_size=world_size,
    froze_embeddings=False,
    zero=False,
    tensor_parallel=True,
    pipeline_parallel=False,
    max_batch_size=collie_args.per_device_train_batch_size,
    max_seq_len=data_args.max_length,
)
tokenizer.pad_token_id = 0

dataset = load_from_disk(data_args.data_dir)['train'].select(range(1000))
train_dataloader = DataLoader(
    dataset,
    batch_size=collie_args.per_device_train_batch_size,
    collate_fn=lambda x: collate_fn(x, tokenizer, max_len=data_args.max_length)
)
eval_dataloader = DataLoader(
    dataset.select(range(4)),
    batch_size=collie_args.per_device_train_batch_size,
    collate_fn=lambda x: collate_fn(x, tokenizer, max_len=data_args.max_length)
)

trainer = collie.trainer.InplaceTensorTrainer(
    model=model,
    tokenizer=tokenizer,
    train_dataloader=train_dataloader,
    eval_dataloader=eval_dataloader,
    eval_dataset=dataset.select(range(4)),
    collie_args=collie_args,
    compute_metrics=compute_metrics,
)
trainer.train()
```

For more examples, check [CoLLiE Examples](https://github.com/OpenLMLab/collie/tree/main/examples).

## Features

1. Utilizes Colossal-AI's pipeline parallelism mode (requires installing [Colossal-AI](https://github.com/hpcaitech/ColossalAI)).
2. Replaces the original attention structure with flash-attention (requires installing [flash-attn](https://github.com/HazyResearch/flash-attention)).
3. Replaces the original dense layer with fused-dense (requires installing [fused_dense_lib](https://github.com/HazyResearch/flash-attention/tree/main/csrc/fused_dense_lib)).
4. Replaces the original rotary position encoding with rotary_emb (requires installing [rotary_emb](https://github.com/HazyResearch/flash-attention/tree/main/csrc/rotary)).
5. Employs fairscale's tensor parallelism mode (requires installing [fairscale](https://github.com/facebookresearch/fairscale)).
6. Utilizes deepspeed's ZeRO (requires installing [deepspeed](https://github.com/microsoft/DeepSpeed)).
7. Support Inplace SGD.

## Roadmap

+ [x] Utilizes Colossal-AI's pipeline parallelism.
+ [x] Utilizes FairScale's tensor parallelism.
+ [x] Utilizes Deepspeed's ZeRO.
+ [x] Implement Inplace SGD.
+ [x] Reimplement LlaMA with Colossal-AI APIs.
+ [x] Speed Benchmark.
+ [ ] Support Colossal-AI's tensor parallelism and ZeRO CPU offload.
+ [ ] Add more examples.
