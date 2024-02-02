# AQLM

Code for \textbf{Extreme Compression of Large Language Models via Additive Quantization} be here

## Installation

### Packages

Install packages from `requirements.txt`:
```bash
pip install -r requirements.txt
```

### Loading / caching datasets and tokenizer

The script will require downloading and caching locally the relevant tokenizer and the datasets. 
They will be saved in default Huggingface Datasets directory unless alternative location is provided by env variables.
See [relevant Datasets documentation section](https://huggingface.co/docs/datasets/main/en/cache#cache-directory)
## Models

This repository is currently designed to work with models of `LLaMA` and `Mixtral` families.

## Data

When quantizing models with AQLM, we recommend that you use a subset of the original data the model was trained on.

For Llama-2 models, the closest available dataset is [RedPajama](https://huggingface.co/datasets/togethercomputer/RedPajama-Data-1T-Sample) . To load subset of RedPajama provide "pajama" in --dataset argument.
This will process nsamples data and tokenize it using provided model tokenizer.

### WandB logging

One can optionally log the data to `Weights and Biases` service (wandb).
Run `pip install wandb` for W&B logging.
Specify `$WANDB_ENTITY`, `$WANDB_PROJECT`, `$WANDB_NAME` environment variables prior to running experiments. use `--wandb` argument to enable logging
# Launching

### GPU and RAM requirements
This code was developed and tested using a several A100 GPU with 80GB GPU RAM. 
You can use the `--offload activations` option to reduce VRAM usage.
For `Language Model Evaluation Harness` evaluation one needs to have enough memory to load whole model  + activation tensors 
on one or several devices.

### Model downloading
The code requires the LLaMA model to be downloaded in Huggingface format and saved locally. The scripts below assume that `$TRANSFORMERS_CACHE` variable points to the Huggingface Transformers cache folder.
To download and cache the models, run this in the same environment:

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
model_name = "meta-llama/Llama-2-7b-hf"  # or whatever else you wish to download
tokenizer = AutoTokenizer.from_pretrained(model_name, torch_dtype="auto")
model = AutoModelForCausalLM.from_pretrained(model_name, torch_dtype="auto")
```


### How to quantize a model with AQLM
This script compresses the model and then tests its performance in terms of perplexity using WikiText2, C4, and Penn Treebank datasets. 

The command to launch the script should look like this: 

```bash
export CUDA_VISIBLE_DEVICES=0   # or e.g. 0,1,2,3
export MODEL_PATH=<PATH_TO_MODEL_ON_HUB>
export DATASET_PATH=<INSERT DATASET NAME OR PATH TO CUSTOM DATA>
export SAVE_PATH=/path/to/save/quantized/model/
export WANDB_PROJECT=MY_AQ_EXPS
export WANDB_NAME=COOL_EXP_NAME

python main.py $MODEL_PATH $DATASET_PATH --nsamples=1024 \
 --num_codebooks=1 --nbits_per_codebook=16 --in_group_size=8 \
 --relative_mse_tolerance=0.01 --finetune_relative_mse_tolerance=0.001 \
 --finetune_batch_size=32 --local_batch_size=1 --offload_activations \
 --wandb --save $SAVE_PATH

```

Main CLI arguments:
- `CUDA_VISIBLE_DEVICES` - by default, the code will use all available GPUs. If you want to use specific GPUs (or one GPU), use this variable.
- `MODEL_PATH` - a path to either hugginface hub (e.g. meta-llama/Llama-2-7b-hf) or a local folder with transformers model and a tokenizer.
- `DATASET_PATH` - either a path to calibration data (see above) or a standard dataset `[c4, ptb, wikitext2]`
   - for llama-2 models, you can use `DATASET_PATH=./data/red_pajama_n=1024_4096_context_length.pth` for a slice of RedPajama (up to 1024 samples)
- `--nsamples` - the number of calibration data _sequences_. If this parameter is not set, take all calibration data avaialble.
- `--num_codebooks` - number of codebooks per layer
- `--nbits_per_codebook` - each codebook will contain 2 ** nbits_per_codebook vectors
- `--in_group_size` - how many weights are quantized together (aka "g" in the paper)
- `--finetune_batch_size` - (for fine-tuning only) the total number of sequences used for each optimization step
- `--local_batch_size` - when accumulating finetune_batch_size, process this many samples per GPU per forward pass (affects GPU RAM usage)
- `--relative_mse_tolerance`- (for initial calibration) - stop training when (current_epoch_mse / previous_epoch_mse) > (1 - relative_mse_tolerance)
- `--finetune_relative_mse_tolerance`- same, but for fine-tuning
- `--offload_activations` -- during calibration, move activations from GPU memory to RAM. This reduces VRAM usage while slowing calibration by ~10% (depending on your hardware). 
- `--save` -- path to save/load quantized model. (see also: `--load`)
- `--wandb` - if this parameter is set, the code will log results to wandb

There are additional hyperparameters aviailable. Run `python main.py --help` for more details on command line arguments, including compression parameters.

### Zero-shot benchmarks via LM Evaluation Harness

To perform zero-shot evaluation, we use [Language Model Evaluation Harness](https://github.com/EleutherAI/lm-evaluation-harness) framework with slight modifications. This repository contains a copy of LM Evaluation Harness repo from early 2023 in `lm-eval-harness` folder. 

Before running the code make sure that you have all the requirements and dependencies of `lm-eval-harness` installed. To install them run:
```
pip install -r lm-evaluation-harness/requirements.txt
```

The main script launching the evaluation procedure is `lmeval.py` .


```bash
export CUDA_VISIBLE_DEVICES=0,1,2,3  # optional: select GPUs
export QUANTZED_MODEL=<PATH_TO_SAVED_QUANTIZED_MODEL_FROM_MAIN.py>
export MODEL_PATH=<INSERT_PATH_TO_ORIINAL_MODEL_ON_HUB>
export DATASET=<INSERT DATASET NAME OR PATH TO CUSTOM DATA>
export WANDB_PROJECT=MY_AQ_LM_EVAL
export WANDB_NAME=COOL_EVAL_NAME

python lmeval.py \
    --model hf-causal \
    --model_args pretrained=$MODEL_PATH,dtype=float16,use_accelerate=True \
    --load $QUANTZED_MODEL \
    --tasks winogrande,piqa,hellaswag,arc_easy,arc_challenge \
    --batch_size 1
```

### Preparing models for inference

To convert a model into a _Hugging Face_ compatible format, use `convert_to_hf.py`

### Inference performance benchmarks

To run inference performace evaluation, first install the inference lib `cd inference_lib && pip install -e .`.

#### CPU
 - To perform layer-wise evaluation, run `matmul_benchmark_cpu.py`.
 - To perform end-to-end evaluation, run `benchmark_generate_cpu.py`, passing either an unquantized model, or a converted to Hugging Face format AQLM model.

#### GPU
 - To perform layer-wise evaluation, run `matmul_benchmark.py`.
 - To perform end-to-end evaluation, run `generate_benchmark.py`, passing either an unquantized model, or a converted to Hugging Face format AQLM model.