# Validated Job Script Templates

> Templates are based on examples from the official [Running Jobs](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/running_jobs.html) and [Software](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/software.html) pages, with safety best practices added (explicit account, time limit, memory, module reset, output file naming with job ID).

Replace `your-cpu-account` / `your-gpu-account` with the user's actual account name (found via `accounts` command).

> **Module versions** (e.g., `pytorch-conda/2.8`, `tensorflow-conda/2.18`) may change over time. Always verify with `module spider pytorch-conda` or `module spider tensorflow-conda` before using *(source: Software page)*.

## CPU Serial Job

```bash
#!/bin/bash
#SBATCH --job-name=cpu-serial
#SBATCH --partition=cpu
#SBATCH --account=your-cpu-account
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16g
#SBATCH --time=00:30:00
#SBATCH --constraint="scratch"
#SBATCH -o slurm-%j.out
#SBATCH -e slurm-%j.err

module reset
module load python
srun python3 myprog.py
```

## MPI Parallel Job (2 nodes)

```bash
#!/bin/bash
#SBATCH --job-name=mpi-parallel
#SBATCH --partition=cpu
#SBATCH --account=your-cpu-account
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=32
#SBATCH --cpus-per-task=2
#SBATCH --mem=16g
#SBATCH --time=01:00:00
#SBATCH --constraint="scratch"
#SBATCH -o slurm-%j.out
#SBATCH -e slurm-%j.err

module reset
# NOTE: Delta does NOT have mpirun — use srun instead
srun ./my_mpi_program
```

## OpenMP (Single Node, Multi-threaded)

```bash
#!/bin/bash
#SBATCH --job-name=openmp
#SBATCH --partition=cpu
#SBATCH --account=your-cpu-account
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=32
#SBATCH --mem=16g
#SBATCH --time=01:00:00
#SBATCH --constraint="scratch"
#SBATCH -o slurm-%j.out
#SBATCH -e slurm-%j.err

module reset
export OMP_NUM_THREADS=32
srun ./my_openmp_program
```

## Single GPU — Deep Learning Training (A40, best value)

```bash
#!/bin/bash
#SBATCH --job-name=train-1gpu
#SBATCH --partition=gpuA40x4
#SBATCH --account=your-gpu-account
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=16
#SBATCH --gpus-per-node=1
#SBATCH --gpu-bind=closest
#SBATCH --mem=50G
#SBATCH --time=04:00:00
#SBATCH --constraint="scratch"
#SBATCH --no-requeue
#SBATCH -o slurm-%j.out
#SBATCH -e slurm-%j.err

module reset
module load pytorch-conda/2.8

export OMP_NUM_THREADS=1
srun python3 train.py
```

## 4-GPU Exclusive Node (A100)

```bash
#!/bin/bash
#SBATCH --job-name=train-4gpu
#SBATCH --partition=gpuA100x4
#SBATCH --account=your-gpu-account
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=4
#SBATCH --cpus-per-task=16
#SBATCH --gpus-per-node=4
#SBATCH --gpu-bind=closest
#SBATCH --mem=208G
#SBATCH --exclusive              # reserves entire node — confirm budget with 'accounts' before submitting
#SBATCH --time=04:00:00
#SBATCH --constraint="scratch"
#SBATCH --no-requeue
#SBATCH -o slurm-%j.out
#SBATCH -e slurm-%j.err

module reset
module load pytorch-conda/2.8

export OMP_NUM_THREADS=1
srun python3 train_ddp.py
```

## Multi-Node Distributed PyTorch (2 nodes, 8 GPUs total)

```bash
#!/bin/bash
#SBATCH --job-name=dist-train
#SBATCH --partition=gpuA40x4
#SBATCH --account=your-gpu-account
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1
#SBATCH --gpus-per-node=4
#SBATCH --cpus-per-task=16       # torchrun spawns 4 procs internally; 16 CPUs shared across them
#SBATCH --gpu-bind=closest
#SBATCH --mem=200G
#SBATCH --time=02:00:00
#SBATCH --constraint="scratch"
#SBATCH --no-requeue              # prevent auto-resubmit on node failure (saves budget if no checkpoint)
#SBATCH -o ddp-%j.out
#SBATCH -e ddp-%j.err

nodes=( $( scontrol show hostnames $SLURM_JOB_NODELIST ) )
head_node=${nodes[0]}
head_node_ip=$(srun --nodes=1 --ntasks=1 -w "$head_node" hostname --ip-address)

module reset
module load pytorch-conda
export NCCL_DEBUG=INFO
export NCCL_SOCKET_IFNAME=hsn    # Slingshot network interface (from official Running Jobs multi-node example)
module load aws-ofi-nccl

srun torchrun --nnodes ${SLURM_NNODES} \
     --nproc_per_node ${SLURM_GPUS_PER_NODE} \
     --rdzv_id $RANDOM --rdzv_backend c10d \
     --rdzv_endpoint="$head_node_ip:29500" \
     train_distributed.py
```

## TensorFlow GPU Job

```bash
#!/bin/bash
#SBATCH --job-name=tf-train
#SBATCH --partition=gpuA100x4
#SBATCH --account=your-gpu-account
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=16
#SBATCH --gpus-per-node=1
#SBATCH --gpu-bind=closest
#SBATCH --mem=64G
#SBATCH --time=04:00:00
#SBATCH --constraint="scratch"
#SBATCH -o slurm-%j.out
#SBATCH -e slurm-%j.err

module reset
module load tensorflow-conda/2.18

srun python3 train_tf.py
```

## Interactive Sessions

> **WARNING:** Claude Code cannot interact with the subshell created by `--pty bash`. Present these commands for the **user to run manually**, not for Claude Code to execute directly.

> Both `--ntasks-per-node=1 --cpus-per-task=16` and `--tasks-per-node=16 --cpus-per-task=1` *(Running Jobs page)* allocate 16 cores per GPU. Templates below use the former for consistency with batch scripts.

```bash
# CPU interactive (4 cores, 16 GB, 30 min)
srun --account=your-cpu-account --partition=cpu-interactive \
  --nodes=1 --ntasks-per-node=1 \
  --cpus-per-task=4 --mem=16g --time=00:30:00 --pty bash

# GPU interactive — A40 (1 GPU, 16 cores, 20 GB, 30 min)
srun --account=your-gpu-account --partition=gpuA40x4-interactive \
  --nodes=1 --gpus-per-node=1 --ntasks-per-node=1 \
  --cpus-per-task=16 --gpu-bind=closest --mem=20g --time=00:30:00 --pty bash

# GPU interactive — A100 (1 GPU, 16 cores, 20 GB, 30 min)
srun --account=your-gpu-account --partition=gpuA100x4-interactive \
  --nodes=1 --gpus-per-node=1 --ntasks-per-node=1 \
  --cpus-per-task=16 --gpu-bind=closest --mem=20g --time=00:30:00 --pty bash
```

## Staggered Job Submission (Multiple Python Jobs)

Multiple Python jobs starting simultaneously can overload the filesystem. Use dependencies to stagger:

```bash
#!/bin/bash
DELAY=5  # minutes between jobs

JOBID=$(sbatch job.slurm | cut -d" " -f4)
echo "Submitted $JOBID"

for i in $(seq 1 4); do
  PREV=$JOBID
  JOBID=$(sbatch --dependency=after:${PREV}+${DELAY} job.slurm | cut -d" " -f4)
  echo "Submitted $JOBID (starts ${DELAY}min after $PREV)"
done
```
