# OCI HPC Tutorial: Super-Ant Cluster (Llama 3.1 / Dolma 3)

These notes document how to:

- Create a user and propagate it across the cluster
- Log in and work with compute nodes
- Use shared storage (`/home`, `/fss`) effectively
- Cache Hugging Face datasets/models on `/fss`
- Run Slurm interactive jobs with GPUs
- Run Llama 3.1 pretraining and inference jobs (vLLM) on OCI HPC

> **Note**  
> Paths, hostnames, and user names (`super-ant-*`, `GPU-*`, `testnsn`, etc.) are specific to **our** OCI tenancy/instance. Adapt as needed for any new environment.

---

## Table of Contents

1. [User Management](#1-user-management)
   - [1.1 Create a local user](#11-create-a-local-user)
   - [1.2 Add user to controller node](#12-add-user-to-controller-node)
   - [1.3 Add user to compute nodes](#13-add-user-to-compute-nodes)
3. [Logging in as the new user](#2-logging-in-as-the-new-user)
4. [Working with Nodes and Shared Storage](#3-working-with-nodes-and-shared-storage)
   - [3.1 Listing compute nodes and using `clush`](#31-listing-compute-nodes-and-using-clush)
   - [3.2 Shared filesystems](#32-shared-filesystems)
5. [Shared Hugging Face Cache on `/fss`](#4-shared-hugging-face-cache-on-fss)
   - [4.1 Directory layout and permissions](#41-directory-layout-and-permissions)
   - [4.2 Download Dolma 3 Sample Mix to `/fss`](#42-download-dolma-3-sample-mix-to-fss)
   - [4.3 Download Llama 3.1 8B to `/fss` (license-aware)](#43-download-llama-31-8b-to-fss-license-aware)
6. [Interactive Slurm Sessions](#5-interactive-slurm-sessions)
7. [Pretraining Llama 3.1 8B with Slurm](#6-pretraining-llama-31-8b-with-slurm)
   - [6.1 Slurm job script (`llama31_pretrain.sbatch`)](#61-slurm-job-script-llama31_pretrainsbatch)
   - [6.2 Training script (`pretrain_llama31.py`)](#62-training-script-pretrain_llama31py)
8. [Inference with vLLM](#7-inference-with-vllm)
   - [7.1 SSH keys and chat template](#71-ssh-keys-and-chat-template)
   - [7.2 Single-node vLLM Slurm job](#72-single-node-vllm-slurm-job)
   - [7.3 OpenAI-compatible client script](#73-openai-compatible-client-script)
   - [7.4 Experimental multinode inference (Ray – currently broken)](#74-experimental-multinode-inference-ray--currently-broken)
9. [Posttraining](#8-posttraining)
10. [Extras and Useful Commands](#9-extras-and-useful-commands)

---

## 1. User Management
> ⚠️ **Use `cluster` tool instead**
> This section is the hacky way to add a user when the `cluster` tool is not working. Here I create a test user called `testnsn`. This test user will be used throughtout.

### 1.1 Create a local user

Create a Unix user on the login node and configure SSH access.

```bash
# 1. Create user
sudo useradd -m -s /bin/bash testnsn

# 2. Make .ssh directory for the new user
sudo mkdir -p /home/testnsn/.ssh

# 3. Reuse the same authorized_keys as the ubuntu user (or add your own)
sudo cp ~ubuntu/.ssh/authorized_keys /home/testnsn/.ssh/

# 4. Fix ownership and permissions
sudo chown -R testnsn:testnsn /home/testnsn/.ssh
sudo chmod 700 /home/testnsn/.ssh
sudo chmod 600 /home/testnsn/.ssh/authorized_keys
````

### 1.2 Add user to controller node
> ⚠️ **Use `id <username>` to get the user `uid` and `gid` **
> This section is the hacky way to add a user when the `cluster` tool is not working.
`add_user_to_controller.sh` ensures consistent UID/GID on the **controller** node.

```bash
# file: add_user_to_controller.sh
ssh ubuntu@super-ant-controller "sudo bash -s" <<'EOF'
set -e

USER_NAME="testnsn"
TARGET_UID=3001
TARGET_GID=3001

# Group: create or fix GID
if getent group "$USER_NAME" >/dev/null 2>&1; then
  existing_gid=$(getent group "$USER_NAME" | cut -d: -f3)
  if [ "$existing_gid" != "$TARGET_GID" ]; then
    # If some *other* group already uses 3001, stop – that’s a conflict
    other_group=$(getent group "$TARGET_GID" | cut -d: -f1 || true)
    if [ -n "$other_group" ] && [ "$other_group" != "$USER_NAME" ]; then
      echo "ERROR: GID $TARGET_GID already used by '$other_group' on $(hostname)" >&2
      exit 1
    fi
    echo "Changing GID of group '$USER_NAME' from $existing_gid to $TARGET_GID"
    groupmod -g "$TARGET_GID" "$USER_NAME"
  fi
else
  # No group named testnsn; make one with GID 3001 (if free)
  other_group=$(getent group "$TARGET_GID" | cut -d: -f1 || true)
  if [ -n "$other_group" ]; then
    echo "ERROR: GID $TARGET_GID already used by '$other_group' on $(hostname)" >&2
    exit 1
  fi
  echo "Creating group '$USER_NAME' with GID $TARGET_GID"
  groupadd -g "$TARGET_GID" "$USER_NAME"
fi

# User: create if missing (we don't silently change an existing UID here)
if id "$USER_NAME" >/dev/null 2>&1; then
  echo "User '$USER_NAME' already exists on $(hostname); leaving UID as-is."
else
  echo "Creating user '$USER_NAME' with UID $TARGET_UID / GID $TARGET_GID"
  useradd -u "$TARGET_UID" -g "$TARGET_GID" -s /bin/bash -M "$USER_NAME"
fi
EOF
```

Run this script from a host that can SSH to `super-ant-controller` as `ubuntu`.

### 1.3 Add user to compute nodes

`add_user_to_compute.sh` propagates the same UID/GID from the login node to **all compute nodes**.

```bash
# file: add_user_to_compute.sh
#!/usr/bin/env bash
set -euo pipefail

USER=testnsn

# Get UID/GID from the login node
uid=$(id -u "$USER")
gid=$(id -g "$USER")

echo "Using UID=$uid GID=$gid for $USER"

# Get list of compute nodes
nodes=$(mgmt nodes list --fields role=compute --format json --columns hostname \
        | jq -r '.[].hostname')

for node in $nodes; do
  echo "=== Configuring $USER on $node ==="

  ssh ubuntu@"$node" "sudo bash -s" <<EOF
set -e

USER_NAME="$USER"
TARGET_UID="$uid"
TARGET_GID="$gid"

echo "Node: \$(hostname)"

# --- Ensure group exists with correct GID ---
if getent group "\$USER_NAME" >/dev/null 2>&1; then
  existing_gid=\$(getent group "\$USER_NAME" | cut -d: -f3)
  if [ "\$existing_gid" != "\$TARGET_GID" ]; then
    # Is TARGET_GID already used by some *other* group?
    other_group=\$(getent group "\$TARGET_GID" | cut -d: -f1 || true)
    if [ -n "\$other_group" ] && [ "\$other_group" != "\$USER_NAME" ]; then
      echo "ERROR: On \$(hostname), GID \$TARGET_GID is already used by group '\$other_group'."
      echo "       Can't automatically change '\$USER_NAME' to that GID."
      exit 1
    fi

    echo "Changing GID of group '\$USER_NAME' from \$existing_gid to \$TARGET_GID"
    groupmod -g "\$TARGET_GID" "\$USER_NAME"
  else
    echo "Group '\$USER_NAME' already has correct GID \$TARGET_GID"
  fi
else
  # No group with this name; check if numeric GID is already taken
  other_group=\$(getent group "\$TARGET_GID" | cut -d: -f1 || true)
  if [ -n "\$other_group" ]; then
    echo "ERROR: On \$(hostname), GID \$TARGET_GID is already used by group '\$other_group'."
    echo "       Can't create group '\$USER_NAME' with that GID."
    exit 1
  fi

  echo "Creating group '\$USER_NAME' with GID \$TARGET_GID"
  groupadd -g "\$TARGET_GID" "\$USER_NAME"
fi

# --- Ensure user exists with correct UID (we don't auto-mutate existing UID) ---
if id "\$USER_NAME" >/dev/null 2>&1; then
  echo "User '\$USER_NAME' already exists on \$(hostname); skipping useradd."
else
  echo "Creating user '\$USER_NAME' with UID \$TARGET_UID and primary GID \$TARGET_GID"
  useradd -m -u "\$TARGET_UID" -g "\$TARGET_GID" -s /bin/bash "\$USER_NAME"
fi
EOF

done
```

---

## 2. Logging in as the new user

First, find the **public IP** of the login node:

```bash
ubuntu@super-ant-login:~$ curl -s ifconfig.me
192.9.140.156
```

Then log in as the new user with SSH, explicitly specifying key exchange and host key algorithms if needed:

```bash
ssh -o KexAlgorithms=curve25519-sha256 \
    -o HostKeyAlgorithms=ssh-ed25519 \
    testnsn@192.9.140.156
```

---

## 3. Working with Nodes and Shared Storage

### 3.1 Listing compute nodes and using `clush`

On the controller node (`super-ant-controller`), create a list of compute hostnames and broadcast a simple command to verify connectivity:

```bash
ubuntu@super-ant-controller:~$ mgmt nodes list --fields role=compute --format json --columns hostname \
  | jq -r '.[].hostname' > node_list.txt

ubuntu@super-ant-controller:~$ clush --hostfile "$HOME/node_list.txt" -- 'echo "hello from $(hostname)"'
GPU-3841: hello from GPU-3841
GPU-6550: hello from GPU-6550
```

### 3.2 Shared filesystems

From the login node, inspect mounted filesystems:

```bash
testnsn@super-ant-login:~$ df -h
Filesystem                                      Size  Used Avail Use% Mounted on
tmpfs                                            26G  1.7M   26G   1% /run
/dev/sda1                                       497G   23G  474G   5% /
// ...
fss-super-ant-controller:/config                8.0E  1.8G  8.0E   1% /config
fss-super-ant-controller.super-ant.local:/home  8.0E   18G  8.0E   1% /home
fss-super-ant-controller.super-ant.local:/fss   8.0E  152M  8.0E   1% /fss
```

Key points:

* `/` (`/dev/sda1`): **local disk** on the login node only.
* `/home` and `/fss`: OCI File Storage Service (FSS) shared across login + compute nodes.

Anything that should be persistent and visible to all nodes should live under:

* Per-user: `/home/<user>`
* Shared datasets/models/etc: `/fss`

---

## 4. Shared Hugging Face Cache on `/fss`

### 4.1 Directory layout and permissions

Centralize all HF datasets/models under `/fss`:

```bash
sudo mkdir -p /fss/datasets
sudo mkdir -p /fss/models

# Give ownership to a service/admin account, e.g. ubuntu
sudo chown -R ubuntu:ubuntu /fss/datasets
sudo chown -R ubuntu:ubuntu /fss/models

# Make parent dirs world-readable and searchable
sudo chmod -R a+rX /fss/datasets
sudo chmod -R a+rX /fss/models
```

This allows:

* Any user to **read** from `/fss/datasets` and `/fss/models`
* New files (with default `umask 022`) to be readable by others

### 4.2 Download Dolma 3 Sample Mix to `/fss`

Dolma 3 Sample Mix (150B) is licensed under **ODC-By**, intended for research/education. It’s reasonable to make a shared copy for your org (respecting attribution and responsible-use guidelines).

```bash
mkdir -p /fss/datasets/allenai/dolma3_mix-150B-1025

hf download allenai/dolma3_mix-150B-1025 \
  --repo-type dataset \
  --local-dir /fss/datasets/allenai/dolma3_mix-150B-1025 \
  --include "*"

sudo chown -R ubuntu:ubuntu /fss/datasets/allenai/dolma3_mix-150B-1025
sudo chmod -R a+rX /fss/datasets/allenai/dolma3_mix-150B-1025
```

Use it from `datasets`:

```python
from datasets import load_dataset

ds = load_dataset(
    "json",
    data_files="/fss/datasets/allenai/dolma3_mix-150B-1025/**/*.jsonl.zst",
    split="train",
    streaming=True,
)

print(ds)

for i, row in enumerate(ds):
    print(row)
    if i == 2:
        break
```

Hugging Face will reuse the local files instead of downloading again.

### 4.3 Download Llama 3.1 8B to `/fss` (license-aware)

Example download:

```bash
hf download meta-llama/Llama-3.1-8B \
  --repo-type model \
  --local-dir /fss/models/meta-llama/Llama-3.1-8B \
  --include "*"
```

Using the local path in `transformers`:

```python
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained(
    "/fss/models/meta-llama/Llama-3.1-8B",
)
```

> ⚠️ **License note**
> Llama 3.x uses a custom **Llama 3 Community License**. Users must accept the terms (Meta / HF), and there are restrictions on use and redistribution. Do **not** silently expose the weights to accounts that have not agreed to the license.

Safer approach:

```bash
# Create a restricted group
sudo groupadd llama3
sudo usermod -aG llama3 alice
sudo usermod -aG llama3 bob

sudo mkdir -p /fss/models/meta-llama/Llama-3.1-8B
sudo chown -R ubuntu:llama3 /fss/models/meta-llama/Llama-3.1-8B
sudo chmod -R 770 /fss/models/meta-llama/Llama-3.1-8B   # only group can read
```

Only add users to `llama3` after they have individually accepted the license.

---

## 5. Interactive Slurm Sessions

Request interactive shells with GPUs on the `compute` partition:

```bash
# Generic: request 2 nodes and 8 GPUs per node via gres
testnsn@super-ant-login:~$ srun -p compute --nodes=2 --gres=gpu:8 --time=02:00:00 --pty bash
srun: job 10706 queued and waiting for resources
srun: job 10706 has been allocated resources

# Alternative syntax: gpus-per-node
testnsn@super-ant-login:~$ srun -p compute --nodes=2 --gpus-per-node=8 --time=02:00:00 --pty bash
srun: job 10709 queued and waiting for resources
srun: job 10709 has been allocated resources
```

Inside the session:

```bash
testnsn@GPU-4341:~$ echo $SLURM_NODELIST
GPU-[4341,4287]

testnsn@GPU-4341:~$ echo $SLURM_JOB_NODELIST
GPU-[4341,4287]

testnsn@GPU-4341:~$ scontrol show hostnames "$SLURM_JOB_NODELIST"
GPU-4341
GPU-4287
```

> **Gotcha:**
> If you don’t explicitly request all GPUs, you won’t get them.

Example:

```bash
testnsn@super-ant-login:~$ srun -p compute --nodes=1 --gpus-per-node=1 --time=02:00:00 --pty bash
srun: job 10709 queued and waiting for resources
srun: job 10709 has been allocated resources

testnsn@GPU-4341:~$ nvidia-smi
# shows 1 visible GPU assigned to the job
```

---

## 6. Pretraining Llama 3.1 8B with Slurm

Directory layout example:

* `/fss/models/meta-llama/Llama-3.1-8B` – local Llama 3.1 8B model/config
* `/fss/datasets/allenai/dolma3_mix-150B-1025` – Dolma 3 dataset
* `/fss/llama3.1_pretrain` – training code & logs

### 6.1 Slurm job script `llama31_pretrain.sbatch`

```bash
# file: llama31_pretrain.sbatch
#!/bin/bash
#SBATCH --job-name=llama31-pretrain
#SBATCH --nodes=2

# One task per GPU
#SBATCH --ntasks-per-node=8
#SBATCH --gpus-per-task=1

# # 14 physical cores per GPU <-- FIXME not working
# #SBATCH --cpus-per-task=14
# #SBATCH --hint=nomultithread          # use 1 hw thread / core

#SBATCH --time=01:00:00
#SBATCH --partition=compute
#SBATCH --output=/fss/llama3.1_pretrain/logs/%x_%j.out
#SBATCH --error=/fss/llama3.1_pretrain/logs/%x_%j.err

mkdir -p /fss/llama3.1_pretrain/logs

# Slurm gives you:
#   SLURM_NNODES, SLURM_NTASKS, SLURM_PROCID, SLURM_LOCALID
export NNODES=$SLURM_NNODES
export GPUS_PER_NODE=8
export WORLD_SIZE=$SLURM_NTASKS  # = NNODES * GPUS_PER_NODE

# Master = first host in nodelist
export MASTER_ADDR=$(scontrol show hostnames "$SLURM_JOB_NODELIST" | head -n 1)
export MASTER_PORT=29500

echo "MASTER_ADDR=$MASTER_ADDR"
echo "MASTER_PORT=$MASTER_PORT"
echo "NNODES=$NNODES WORLD_SIZE=$WORLD_SIZE"

# Match to --cpus-per-task <-- FIXME hard coding for now (not sure if pinning is working)
#export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export OMP_NUM_THREADS=14

# --- NCCL options ---
export NCCL_DEBUG=WARN
#export NCCL_SOCKET_IFNAME=^lo,docker  # don't use lo (loopback) or docker interfaces

# Use RDMA NICs instead of eth0
#export NCCL_SOCKET_IFNAME=rdma0,rdma1,rdma2,rdma3,rdma4,rdma5,rdma6,rdma7
#export NCCL_IB_DISABLE=0

# Use only the management Ethernet NIC for now (to get things working)
export NCCL_SOCKET_IFNAME=eth0
export NCCL_IB_DISABLE=1

# Disable cuMem-based P2P / host allocations; fall back to older IPC paths
export NCCL_CUMEM_ENABLE=0
export NCCL_CUMEM_HOST_ENABLE=0
export NCCL_P2P_DISABLE=1

export NCCL_IB_SPLIT_DATA_ON_QPS=0
export NCCL_IGNORE_CPU_AFFINITY=1

# One process per GPU, pinned to nearby CPUs+GPU
srun \
  --ntasks-per-node=$GPUS_PER_NODE \
  --gpus-per-task=1 \
  --gpu-bind=closest \
  --cpu-bind=cores \
  python3 -u /fss/llama3.1_pretrain/pretrain_llama31.py
```

Submit with:

```bash
sbatch llama31_pretrain.sbatch
```

### 6.2 Training script `pretrain_llama31.py`

```python
# file: pretrain_llama31.py
#!/usr/bin/env python3
import os
from dataclasses import dataclass
from typing import Optional, Dict, Any, List

import torch
from datasets import load_dataset
from transformers import (
    AutoConfig,
    AutoModelForCausalLM,
    PreTrainedTokenizerFast,
    Trainer,
    TrainingArguments,
)
from transformers.trainer_utils import get_last_checkpoint


# ----------------- CONFIG AREA -----------------

# Local model + tokenizer
MODEL_PATH = "/fss/models/meta-llama/Llama-3.1-8B"

# Dolma 3 Sample Mix (150B) path
DOLMA_PATH = "/fss/datasets/allenai/dolma3_mix-150B-1025/**/*.jsonl.zst"

# Shared output dir for logs + checkpoints
OUTPUT_DIR = "/fss/llama3.1_pretrain/runs/run1"

# Sequence length & training schedule
MAX_SEQ_LEN = 4096
#MAX_STEPS = 100000
MAX_STEPS = 10

PER_DEVICE_BATCH_SIZE = 1
GRAD_ACCUM_STEPS = 16         # adjust for effective global batch size

SAVE_STEPS = 5
LOGGING_STEPS = 1
EVAL_STEPS = None  # set if you add a val split

USE_BF16 = True               # H200 supports bf16
# -------------------------------------------------


def setup_distributed_from_slurm():
    """
    If we're launched with `srun` on Slurm, map Slurm's env variables
    to what HF Trainer / Accelerate expect.
    """
    # Global rank of this process
    if "SLURM_PROCID" in os.environ and "RANK" not in os.environ:
        os.environ["RANK"] = os.environ["SLURM_PROCID"]

    # World size (total processes)
    if "SLURM_NTASKS" in os.environ and "WORLD_SIZE" not in os.environ:
        os.environ["WORLD_SIZE"] = os.environ["SLURM_NTASKS"]

    # Local rank on this node
    if "SLURM_LOCALID" in os.environ and "LOCAL_RANK" not in os.environ:
        os.environ["LOCAL_RANK"] = os.environ["SLURM_LOCALID"]


def load_tokenizer() -> PreTrainedTokenizerFast:
    print("Loading tokenizer from:", MODEL_PATH)
    tok_file = os.path.join(MODEL_PATH, "tokenizer.json")
    tokenizer = PreTrainedTokenizerFast(tokenizer_file=tok_file)

    # For training we usually use right padding
    tokenizer.padding_side = "right"

    if tokenizer.pad_token is None:
        tokenizer.pad_token = tokenizer.eos_token

    return tokenizer


def load_model_from_scratch(tokenizer: PreTrainedTokenizerFast):
    """
    Initialize a Llama 3.1 8B *from scratch* (random weights) using the
    local config at MODEL_PATH.
    """
    print("Loading config from:", MODEL_PATH)
    config = AutoConfig.from_pretrained(MODEL_PATH)
    config.vocab_size = len(tokenizer)

    if config.pad_token_id is None:
        config.pad_token_id = tokenizer.pad_token_id

    print("Initializing model from config (random weights).")
    model = AutoModelForCausalLM.from_config(config)
    return model


def load_model_from_pretrained(tokenizer: PreTrainedTokenizerFast):
    """
    Alternative: continue pretraining from the existing weights rather
    than from scratch.
    """
    print("Loading model weights from:", MODEL_PATH)
    model = AutoModelForCausalLM.from_pretrained(
        MODEL_PATH,
        torch_dtype=torch.bfloat16 if torch.cuda.is_available() else torch.float32,
        device_map=None,  # let DDP handle device placement
    )

    if model.config.pad_token_id is None:
        model.config.pad_token_id = tokenizer.pad_token_id

    return model


def build_streaming_dataset(tokenizer: PreTrainedTokenizerFast):
    """
    Build a streaming Dolma dataset and tokenize it on the fly.

    We keep it as a streaming (Iterable) dataset so we don't try to
    materialize 150B tokens on disk/RAM.
    """

    print("Loading Dolma streaming dataset from:", DOLMA_PATH)
    raw = load_dataset(
        "json",
        data_files=DOLMA_PATH,
        split="train",
        streaming=True,
    )

    # Try to detect a reasonable text field
    if getattr(raw, "features", None) is not None:
        # normal case: features are known
        column_names = list(raw.features.keys())
    else:
        # streaming JSON/CSV/etc: features may be None until you start reading
        print("raw.features is None, inferring column names from the first example...")
        try:
            first_example = next(iter(raw.take(1)))
        except StopIteration:
            raise RuntimeError(f"No examples found in dataset at {DOLMA_PATH!r}")
        column_names = list(first_example.keys())

    text_column = None
    for candidate in ["text", "content", "raw_content", "document"]:
        if candidate in column_names:
            text_column = candidate
            break
    if text_column is None:
        # fall back to first column, and hope it's text-like
        text_column = column_names[0]

    print(f"Using text column: {text_column!r}")

    def tokenize_batch(batch: Dict[str, List[Any]]) -> Dict[str, Any]:
        texts = batch[text_column]
        enc = tokenizer(
            texts,
            truncation=True,
            max_length=MAX_SEQ_LEN,
            padding="max_length",
            return_attention_mask=True,
            return_token_type_ids=False,
        )
        # For causal LM pretraining, labels are just the input_ids shifted inside the model
        # but Trainer expects 'labels'
        enc["labels"] = enc["input_ids"]
        return enc

    tokenized = raw.map(
        tokenize_batch,
        batched=True,
        remove_columns=column_names,
    )

    # Make sure we emit torch tensors
    tokenized = tokenized.with_format("torch")

    return tokenized


def main():
    # --- set up distributed env from Slurm if needed ---
    setup_distributed_from_slurm()

    # --- distributed env info ---
    local_rank = int(os.environ.get("LOCAL_RANK", 0))
    rank = int(os.environ.get("RANK", 0))
    world_size = int(os.environ.get("WORLD_SIZE", 1))

    if rank == 0:
        print(f"WORLD_SIZE={world_size}, RANK={rank}, LOCAL_RANK={local_rank}")
        print("Output dir:", OUTPUT_DIR)

    # --- tokenizer & model ---
    tokenizer = load_tokenizer()

    # Choose one:
    model = load_model_from_scratch(tokenizer)       # <- true "from scratch"
    # model = load_model_from_pretrained(tokenizer)  # <- continue pretraining

    # --- dataset ---
    train_dataset = build_streaming_dataset(tokenizer)

    # --- training args ---
    training_args = TrainingArguments(
        output_dir=OUTPUT_DIR,
        overwrite_output_dir=False,

        # Streaming data: use max_steps instead of epochs
        max_steps=MAX_STEPS,
        num_train_epochs=1,  # ignored for IterableDataset, but must be > 0

        per_device_train_batch_size=PER_DEVICE_BATCH_SIZE,
        gradient_accumulation_steps=GRAD_ACCUM_STEPS,
        learning_rate=2e-4,
        #warmup_steps=2000,
        warmup_steps=1,
        weight_decay=0.1,

        logging_steps=LOGGING_STEPS,
        save_strategy="steps",
        save_steps=SAVE_STEPS,
        save_total_limit=3,

        bf16=USE_BF16,
        fp16=False,
        gradient_checkpointing=True,
        dataloader_num_workers=4,
        dataloader_drop_last=True,

        report_to="none",
        remove_unused_columns=False,  # important for streaming + custom columns
        ddp_backend="nccl",
    )

    # --- auto-resume logic ---
    last_checkpoint = None
    if os.path.isdir(OUTPUT_DIR):
        last_checkpoint = get_last_checkpoint(OUTPUT_DIR)
        if rank == 0:
            if last_checkpoint is not None:
                print(f"Resuming from checkpoint: {last_checkpoint}")
            else:
                print(f"No checkpoint found in {OUTPUT_DIR}, starting from scratch.")
    else:
        if rank == 0:
            print(f"Creating output dir {OUTPUT_DIR}")
        os.makedirs(OUTPUT_DIR, exist_ok=True)

    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=train_dataset,
        tokenizer=tokenizer,
        data_collator=None,  # labels already in examples
    )

    trainer.train(resume_from_checkpoint=last_checkpoint)
    if rank == 0:
        trainer.save_model()
        tokenizer.save_pretrained(OUTPUT_DIR)
        print("Training complete. Final model saved to:", OUTPUT_DIR)


if __name__ == "__main__":
    main()
```

---

## 7. Inference with vLLM

Directory layout example:

* `/fss/llama3.1_infer` – inference scripts and logs
* `/fss/models/meta-llama/Llama-3.1-8B` – model weights
* Chat template copied from vLLM repo

### 7.1 SSH keys and chat template

Generate an SSH key for your user (if not already present) and authorize it:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
mkdir -p ~/.ssh
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
```

Download the Llama 3.1 chat template from the vLLM repo:

```bash
cd /fss/llama3.1_infer
git clone https://github.com/vllm-project/vllm.git vllm_repo   # or download just the file

cp vllm_repo/examples/tool_chat_template_llama3.1_json.jinja \
   /fss/llama3.1_infer/llama3.1_chat_template.jinja

rm -rf vllm_repo
```

### 7.2 Single-node vLLM Slurm job

Submit the single-node vLLM job:

```bash
testnsn@super-ant-login:/fss/llama3.1_infer$ sbatch llama31_vllm_single_node_mp.sbatch
Submitted batch job 11944
```

Example job script:

```bash
# file: llama31_vllm_single_node_mp.sbatch
#!/bin/bash
#SBATCH --job-name=llama31-vllm-mp
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --gpus-per-task=8           # use all 8 H200s on the node
#SBATCH --time=02:00:00
#SBATCH --partition=compute
#SBATCH --output=/fss/llama3.1_infer/logs/%x_%j.out
#SBATCH --error=/fss/llama3.1_infer/logs/%x_%j.err

set -euo pipefail

# --- 0. Prep -----------------------------------------------------------------
mkdir -p /fss/llama3.1_infer/logs

# Load modules / env (edit for your cluster)
# module load cuda/12.4
# module load python/3.11
# source ~/miniconda3/etc/profile.d/conda.sh
# conda activate vllm-env

MODEL_PATH="/fss/models/meta-llama/Llama-3.1-8B"
CHAT_TEMPLATE="/fss/llama3.1_infer/llama3.1_chat_template.jinja"
PORT="${PORT:-8000}"

# OpenAI-compatible auth for vLLM
API_KEY="${API_KEY:-llama-local-key}"        # client must use this
SERVED_MODEL_NAME="${SERVED_MODEL_NAME:-llama-3.1-8b}"

# Recommended TP on H100/H200 is often 2 for throughput vs overhead;
# change to 1, 2, 4, or 8 as you like.
TP_SIZE="${TP_SIZE:-2}"

HOST_SHORT="$(hostname -s)"

echo "======================================================="
echo " Single-node vLLM (mp backend)"
echo " Node:             ${HOST_SHORT}"
echo " GPUs per node:    8"
echo " Tensor parallel:  ${TP_SIZE}"
echo " Model path:       ${MODEL_PATH}"
echo " Listening on:     http://${HOST_SHORT}:${PORT}/v1"
echo " API key:          ${API_KEY}"
echo " Served model id:  ${SERVED_MODEL_NAME}"
echo "======================================================="

# --- 1. Start vLLM OpenAI-compatible server (multiprocessing backend) -------

srun vllm serve "${MODEL_PATH}" \
  --host 0.0.0.0 \
  --port "${PORT}" \
  --dtype auto \
  --tensor-parallel-size "${TP_SIZE}" \
  --distributed-executor-backend mp \
  --api-key "${API_KEY}" \
  --served-model-name "${SERVED_MODEL_NAME}" \
  --gpu-memory-utilization 0.9 \
  --chat-template "$CHAT_TEMPLATE"

# Job ends when the server stops.
```

### 7.3 OpenAI-compatible client script

Example Python client:

```python
# file: chat_with_llama31.py
#!/usr/bin/env python3
import os
from openai import OpenAI

# Defaults:
#   Node:            GPU-4341
#   Listening on:    http://GPU-4341:8000/v1
#   API key:         llama-local-key
#   Served model id: llama-3.1-8b

def default_base_url() -> str:
    host = os.environ.get("VLLM_HOST", "GPU-4341")
    port = os.environ.get("VLLM_PORT", "8000")
    return f"http://{host}:{port}/v1"

API_BASE = os.environ.get("OPENAI_API_BASE", default_base_url())
API_KEY = os.environ.get("OPENAI_API_KEY", "llama-local-key")
MODEL_NAME = os.environ.get("OPENAI_MODEL_NAME", "llama-3.1-8b")

client = OpenAI(
    base_url=API_BASE,
    api_key=API_KEY,
)

def main():
    print(f"Using base URL: {API_BASE}")
    print(f"Using model:    {MODEL_NAME}")

    messages = [
        {"role": "system", "content": "You are a helpful assistant running on an OCI HPC cluster."},
        {"role": "user", "content": "Explain where you are running and what hardware you might be using."},
    ]

    completion = client.chat.completions.create(
        model=MODEL_NAME,
        messages=messages,
        max_tokens=256,
        temperature=0.7,
    )

    print("\n=== RESPONSE ===")
    print(completion.choices[0].message.content)

if __name__ == "__main__":
    main()
```

Example run (after the Slurm job has started vLLM):

```bash
testnsn@super-ant-login:/fss/llama3.1_infer$ python3 chat_with_llama31.py
Using base URL: http://GPU-4341:8000/v1
Using model:    llama-3.1-8b

=== RESPONSE ===
... (model output) ...
```

### 7.4 Experimental multinode inference (Ray – currently broken)

Multinode vLLM with Ray was attempted but did **not** work due to [issues with **Ray Compiled DAG** and the current vLLM version](https://github.com/vllm-project/vllm/issues/29373).

The job script below is kept for reference / future debugging:

```bash
# file: llama31_vllm_ray.sbatch (EXPERIMENTAL / BROKEN)
#!/bin/bash
#SBATCH --job-name=llama31-vllm-ray
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1          # one Ray/vLLM driver per node
#SBATCH --gpus-per-task=8            # all 8 GPUs per node
#SBATCH --time=04:00:00
#SBATCH --partition=compute
#SBATCH --output=/fss/llama3.1_infer/logs/%x_%j.out
#SBATCH --error=/fss/llama3.1_infer/logs/%x_%j.err

set -xe

# ---- User config ----
MODEL_PATH="/fss/models/meta-llama/Llama-3.1-8B"
CHAT_TEMPLATE="/fss/llama3.1_infer/llama3.1_chat_template.jinja"
SERVED_MODEL_NAME="llama-3.1-8b"

API_PORT=8000
API_KEY="${VLLM_API_KEY:-llama-local-key}"

# Ray cluster ports
RAY_PORT=6379           # GCS / cluster port
RAY_DASHBOARD_PORT=8265 # dashboard / job API

echo "Job ID:        $SLURM_JOB_ID"
echo "Nodes:         $SLURM_NODELIST"

# ---- Determine head / worker nodes ----
nodes=$(scontrol show hostnames "$SLURM_JOB_NODELIST")
nodes_array=($nodes)

head_node=${nodes_array[0]}
echo "Head node:     $head_node"

# Get IP of head node (first IP in case multiple are returned)
head_ip=$(srun --nodes=1 --ntasks=1 -w "$head_node" hostname --ip-address | awk '{print $1}')
echo "Head node IP:  $head_ip"

ip_head="$head_ip:$RAY_PORT"
export ip_head

# Optional per-job temp dirs for Ray
head_tmp_dir="/tmp/ray_tmp_${SLURM_JOB_ID}_head"

export RAY_CGRAPH_get_timeout=60
export RAY_CGRAPH_submit_timeout=60

echo "Starting Ray HEAD on $head_node"
srun --nodes=1 --ntasks=1 -w "$head_node" \
  ray start --head \
    --node-ip-address="$head_ip" \
    --port="$RAY_PORT" \
    --dashboard-host=0.0.0.0 \
    --dashboard-port="$RAY_DASHBOARD_PORT" \
    --num-gpus=8 \
    --temp-dir="$head_tmp_dir" \
    --block &

# Give head a moment to come up
sleep 60

# Start Ray worker(s) on remaining node(s)
worker_num=$((SLURM_JOB_NUM_NODES - 1))
for ((i = 1; i <= worker_num; i++)); do
  node_i=${nodes_array[$i]}
  worker_tmp_dir="/tmp/ray_tmp_${SLURM_JOB_ID}_worker_$i"

  echo "Starting Ray WORKER $i on $node_i"
  srun --nodes=1 --ntasks=1 -w "$node_i" \
    ray start \
      --address="$ip_head" \
      --num-gpus=8 \
      --temp-dir="$worker_tmp_dir" \
      --block &

  sleep 5
done

echo "Waiting for Ray cluster to stabilize..."

# Tell the Ray client where the head is
export RAY_ADDRESS="$ip_head"

python3 - << 'EOF'
import os
import time
import ray

address = os.environ["RAY_ADDRESS"]
expected_nodes = int(os.environ["SLURM_JOB_NUM_NODES"])
gpus_per_node = 8
expected_gpus = expected_nodes * gpus_per_node

print(f"Expecting {expected_nodes} nodes and {expected_gpus} GPUs in the Ray cluster.")
while True:
    try:
        ray.init(address=address, ignore_reinit_error=True, namespace="default")
        nodes = [n for n in ray.nodes() if n.get("Alive", False)]
        num_nodes = len(nodes)
        resources = ray.cluster_resources()
        num_gpus = int(resources.get("GPU", 0))

        print(f"Ray cluster status: {num_nodes} alive nodes, {num_gpus} GPUs")

        if num_nodes >= expected_nodes and num_gpus >= expected_gpus:
            print("Ray cluster looks ready.")
            break
    except Exception as e:
        print(f"Ray not ready yet: {e}")

    time.sleep(5)
EOF

# ---- Start vLLM on the HEAD node, using Ray backend ----
echo "Starting vLLM OpenAI server on $head_node"

# Tell vLLM how to reach Ray and what IP to use
export VLLM_HOST_IP="$head_ip"
export RAY_ADDRESS="$head_ip:$RAY_PORT"

# Disable Ray Compiled DAG (problematic with current vLLM)
export VLLM_USE_RAY_COMPILED_DAG=0
export VLLM_USE_RAY_COMPILED_DAG_NCCL_CHANNEL=0

vllm serve "$MODEL_PATH" \
  --enforce-eager \
  --served-model-name "$SERVED_MODEL_NAME" \
  --host 0.0.0.0 \
  --port "$API_PORT" \
  --api-key "$API_KEY" \
  --tensor-parallel-size 8 \
  --pipeline-parallel-size 2 \
  --distributed-executor-backend ray \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.9 \
  --chat-template "$CHAT_TEMPLATE"

# vLLM stays in the foreground. When it exits or the job times out,
# Slurm will clean up the Ray head and worker processes (started with --block &).
```

---

## 8. Posttraining

A posttraining / finetuning job script was planned but not completed/tested in this environment.

> **TODO:** Add a tested posttraining SBATCH script and Python entrypoint.

---

## 9. Extras and Useful Commands

Install basic Python packages (e.g., in a virtualenv or conda env):

```bash
pip install datasets
pip install zstandard
pip install torch
pip install transformers
```

Inspect node hardware (example: CPU and memory on `GPU-4287`):

```bash
ubuntu@super-ant-controller:~$ scontrol show node GPU-4287 | egrep -i 'CPUTot|RealMemory'
   CPUAlloc=0 CPUEfctv=224 CPUTot=224 CPULoad=1.63
   RealMemory=3096048 AllocMem=0 FreeMem=3064470 Sockets=2 Boards=1
```

This confirms:

* 224 CPU cores per node
* ~3 TB of RAM per node

---

Happy hacking on the super-ant OCI HPC cluster 🐜🧠

