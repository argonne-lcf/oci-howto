
# Applications tests
## TorchTitan

Creating virtual environment
```bash
module load hpcx/v2.23/hpcx-mt-ompi
python3 -m venv $HOME/venvs/frameworks
git clone https://github.com/pytorch/torchtitan.git
cd torchtitan
pip3 install -r ./requirements.txt
```

Submitting job
```bash
#!/usr/bin/bash
#SBATCH --job-name=torchtitan
#SBATCH --nodes=2
#SBATCH --gpus-per-node=8
#SBATCH --cpus-per-task=96
#SBATCH --ntasks-per-node=8
#SBATCH --exclusive

set -ex
cd "$SLURM_SUBMIT_DIR"
source $HOME/venvs/frameworks/bin/activate
module load hpcx/v2.23/hpcx-mt-ompi
# use envs as local overwrites for convenience
# e.g.
# LOG_RANK=0,1 NGPU=4 ./run_train.sh

NGPU=${NGPU:-"8"}
export LOG_RANK=${LOG_RANK:-0}
NNODES=$(cat $SLURM_JOB_NODELIST | wc -l)

CONFIG_FILE=${CONFIG_FILE:-"./torchtitan/models/llama3/train_configs/debug_model.toml"}
TRAIN_FILE=${TRAIN_FILE:-"torchtitan.train"}

MASTER_ADDR=$(scontrol show hostnames "$SLURM_JOB_NODELIST" | head -n1)
MASTER_PORT=29500   # or any free port

TORCHFT_LIGHTHOUSE=${TORCHFT_LIGHTHOUSE:-"http://localhost:29510"}

PYTORCH_ALLOC_CONF="expandable_segments:True" \
TORCHFT_LIGHTHOUSE=${TORCHFT_LIGHTHOUSE} \
torchrun --nproc_per_node=${NGPU} --rdzv_backend c10d --rdzv_endpoint="${MASTER_ADDR}:${MASTER_PORT}" \
--local-ranks-filter ${LOG_RANK} --role rank --tee 3 \
-m ${TRAIN_FILE} --job.config_file ${CONFIG_FILE} "$@"
```

Output
```
+ torchrun --nproc_per_node=8 --rdzv_backend c10d --rdzv_endpoint=GPU-4341:29500 --local-ranks-filter 0 --role rank --tee 3 -m torchtitan.train --job.config_file ./torchtitan/models/llama3/train_configs/debug_model.toml
W1125 19:44:46.787000 1122840 torch/distributed/run.py:803] 
W1125 19:44:46.787000 1122840 torch/distributed/run.py:803] *****************************************
W1125 19:44:46.787000 1122840 torch/distributed/run.py:803] Setting OMP_NUM_THREADS environment variable for each process to be 1 in default, to avoid your system being overloaded, please further tune the variable for optimal performance in your application as needed. 
W1125 19:44:46.787000 1122840 torch/distributed/run.py:803] *****************************************
[rank0]:[titan] 2025-11-25 19:44:53,858 - root - INFO - Starting job: Llama 3 debug training
[rank0]:[titan] 2025-11-25 19:44:55,284 - root - WARNING - ENV[TORCH_NCCL_ASYNC_ERROR_HANDLING] = 1 will be overridden to 3 based on job config
[rank0]:[titan] 2025-11-25 19:44:55,287 - root - INFO - Building 1-D device mesh with ['dp_shard'], [8]
[rank0]:[titan] 2025-11-25 19:44:55,290 - root - INFO - [GC] Initial GC collection took 0.00 seconds
[rank0]:NCCL version 2.27.5+cuda12.9
[rank0]:[titan] 2025-11-25 19:44:58,199 - root - INFO - Loading tokenizer from tokenizer.json
[rank0]:[titan] 2025-11-25 19:44:58,201 - root - INFO - Preparing c4_test dataset from tests/assets/c4_test
[rank0]:[titan] 2025-11-25 19:44:58,537 - root - INFO - Building llama3 debugmodel with TransformerModelArgs(_enforced='This field is used to enforce all fields have defaults.', dim=256, n_layers=6, n_heads=16, n_kv_heads=None, vocab_size=2048, multiple_of=256, ffn_dim_multiplier=None, norm_eps=1e-05, rope_theta=500000, rope_scaling_args=RoPEScalingArgs(scaling_factor=8.0, low_freq_factor=1.0, high_freq_factor=4.0, original_max_position_embeddings=8192), max_seq_len=2048, depth_init=True, use_flex_attn=False, attn_mask_type='causal', eos_id=0)
[rank0]:[titan] 2025-11-25 19:44:58,547 - root - INFO - CUDA capacity: NVIDIA H200 with 139.81GiB memory
[rank0]:[titan] 2025-11-25 19:44:58,595 - root - INFO - Model llama3 debugmodel size: 6,163,712 total parameters
[rank0]:[titan] 2025-11-25 19:44:58,595 - root - INFO - Applied selective activation checkpointing to the model
[rank0]:[titan] 2025-11-25 19:44:58,623 - root - INFO - Applied FSDP to the model
[rank0]:[titan] 2025-11-25 19:44:58,837 - root - INFO - Peak FLOPS used for computing MFU: 9.890e+14
[rank0]:[titan] 2025-11-25 19:44:58,837 - root - INFO - CUDA memory usage for model: 0.00GiB(0.00%)
[rank0]:[titan] 2025-11-25 19:44:58,838 - root - WARNING - model.safetensors.index.json not found at hf_assets_path: ./tests/assets/tokenizer/model.safetensors.index.json.                     Defaulting to saving a single safetensors file if checkpoint is saved in HF format
[rank0]:[titan] 2025-11-25 19:44:58,838 - root - INFO - Mixed precision training is handled by fully_shard
[rank0]:[titan] 2025-11-25 19:44:58,838 - root - INFO - Trainer is initialized with local batch size 8, global batch size 64, gradient accumulation steps 1, sequence length 2048, total steps 10 (warmup 2)
[rank0]:[titan] 2025-11-25 19:44:58,838 - root - INFO - Training starts at step 1
[rank0]:[titan] 2025-11-25 19:45:01,441 - root - INFO - step:  1  loss:  8.0277  grad_norm:  1.4604  memory:  1.21GiB(0.86%)  tps: 5,756  tflops: 0.41  mfu: 0.04%
[rank0]:[titan] 2025-11-25 19:45:01,441 - root - INFO - Synchronizing and adjusting timeout for all ProcessGroups to 0:01:40
[rank0]:[titan] 2025-11-25 19:45:01,496 - root - INFO - step:  2  loss:  7.7082  grad_norm:  1.5585  memory:  1.34GiB(0.96%)  tps: 301,092  tflops: 21.55  mfu: 2.18%
[rank0]:[titan] 2025-11-25 19:45:01,542 - root - INFO - step:  3  loss:  6.9532  grad_norm:  1.9775  memory:  1.34GiB(0.96%)  tps: 359,546  tflops: 25.74  mfu: 2.60%
[rank0]:[titan] 2025-11-25 19:45:01,589 - root - INFO - step:  4  loss:  6.0256  grad_norm:  2.4236  memory:  1.34GiB(0.96%)  tps: 347,712  tflops: 24.89  mfu: 2.52%
[rank0]:[titan] 2025-11-25 19:45:01,638 - root - INFO - step:  5  loss:  5.1762  grad_norm:  2.4350  memory:  1.34GiB(0.96%)  tps: 338,287  tflops: 24.22  mfu: 2.45%
[rank0]:[titan] 2025-11-25 19:45:01,690 - root - INFO - step:  6  loss:  4.6572  grad_norm:  2.3597  memory:  1.34GiB(0.96%)  tps: 317,373  tflops: 22.72  mfu: 2.30%
[rank0]:[titan] 2025-11-25 19:45:01,740 - root - INFO - step:  7  loss:  4.3305  grad_norm:  2.2470  memory:  1.34GiB(0.96%)  tps: 329,413  tflops: 23.58  mfu: 2.38%
[rank0]:[titan] 2025-11-25 19:45:01,785 - root - INFO - step:  8  loss:  4.1496  grad_norm:  2.1184  memory:  1.34GiB(0.96%)  tps: 364,703  tflops: 26.11  mfu: 2.64%
[rank0]:[titan] 2025-11-25 19:45:01,832 - root - INFO - step:  9  loss:  4.1254  grad_norm:  1.9130  memory:  1.34GiB(0.96%)  tps: 351,177  tflops: 25.14  mfu: 2.54%
[rank0]:[titan] 2025-11-25 19:45:01,883 - root - INFO - step: 10  loss:  3.9761  grad_norm:  1.9366  memory:  1.34GiB(0.96%)  tps: 324,245  tflops: 23.21  mfu: 2.35%
[rank0]:[titan] 2025-11-25 19:45:01,883 - root - INFO - Sleeping 2 seconds for other ranks to complete
[rank0]:[titan] 2025-11-25 19:45:03,885 - root - INFO - Training completed
[rank0]:[titan] 2025-11-25 19:45:04,083 - root - INFO - Process group destroyed
```

## Megatron-DeepSpeed
## DLIO
## 
