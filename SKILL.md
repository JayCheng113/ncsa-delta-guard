---
name: ncsa-delta-guard
description: >
  Safety guardrails for using Claude Code on the NCSA Delta supercomputer cluster.
  Activate this skill whenever the user is working on Delta (login.delta.ncsa.illinois.edu),
  mentions Slurm (sbatch, srun, salloc, squeue, scancel), references Delta partitions
  (gpuA100x4, gpuA40x4, gpuH200x8, cpu, etc.), talks about SU/Service Units, or runs
  any shell command on a shared HPC cluster. Also activate when the user asks about
  job submission, GPU allocation, module loading, file transfers, conda environments,
  or storage quotas on Delta. This skill exists because Delta is a large shared
  supercomputer — mistakes can waste thousands of dollars in compute budget, delete
  irreplaceable data, or disrupt other users' work.
---

# NCSA Delta Safety Guard

Safety-first guardrails for Claude Code on the NCSA Delta supercomputer. All official quotes cite their source page from [docs.ncsa.illinois.edu/systems/delta](https://docs.ncsa.illinois.edu/systems/delta/en/latest/).

## Execution Context

All Bash commands from Claude Code execute on the **login node** unless the user is inside an interactive job.

- **Login node limits** *(Login page)*: 16 effective CPU cores per user; memory "37G for High and 62G for Max"; no swap; no GPUs
- "Do not run production applications on the login nodes" — automatic safeguards will kill offending processes *(Citizenship page)*
- Write job scripts and submit via `sbatch` for compute-heavy tasks
- Do NOT run `srun --pty bash` from Claude Code — it creates a subshell Claude Code cannot interact with
- "Interactive jobs are already a child process of srun, therefore, one cannot srun applications from within them" *(Running Jobs page)* — use `salloc` if nested `srun` is needed
- For `tmux`, connect to a specific login node (`dt-login01` through `dt-login04`) — round-robin may land on a different node

## Core Safety Rules

**Before executing ANY command, ask:** Will this waste budget? Will this destroy data? Will this harm the shared environment? If possibly yes — STOP and warn the user.

**When unsure about ANY Delta-specific fact — a command, a path, a policy, a limit — do NOT guess or fabricate.** Instead:
1. Search the official documentation at `https://docs.ncsa.illinois.edu/systems/delta/en/latest/` using WebFetch
2. If you cannot access the docs, tell the user honestly: "I'm not certain about this — please verify at [docs URL] or run `man <command>` / `scontrol show ...` on the cluster"
3. Never present unverified information as fact on a shared supercomputer — a wrong answer can destroy data or waste thousands of SU

## BLOCK — Refuse without explicit user confirmation

| Action | Risk | What to do instead |
|--------|------|---------------------|
| `rm -rf` on `/work`, `/projects`, `/u` | Permanent data loss — no backups on /work, /projects | `ls` first, confirm specific targets |
| `chmod -R 777` on shared paths | Exposes data to all cluster users | Use `750` or `700` |
| Heavy computation / Jupyter on login node | Auto-killed *(Citizenship + Software pages)* | Submit via `sbatch` or use Open OnDemand |
| Many thousands of small files in one dir | "Do not stress file systems with known-harmful access patterns" *(Citizenship page)* | Use tar/zip |
| `mpirun` | "There is no mpirun available on the system" *(Programming Environment page)* | Use `srun` |
| Apptainer build filling /tmp | "FATAL: No space left on device" *(Containers page)* | Clean `/tmp/build-temp*`; set `APPTAINER_CACHEDIR` |
| `sbatch` without `--account` or `--time` | Wrong allocation or default 30-min timeout | Verify with `accounts`; set explicit `--time` |
| `conda`/`pip install` in `$HOME` (`/u`) | HOME quota: 100 GB / 750K files | Use `--prefix /work/hdd/<project>/$USER/` |
| `*-preempt` partition without checkpoint | Job killed = zero progress saved | Use non-preempt, or implement checkpointing |
| `--exclusive --mem=0` on GPU nodes | Reserves entire node; very expensive | Share the node unless truly needed |

## CHECK — Validate before running

| Before this... | Do this first... |
|----------------|------------------|
| Submitting any job | `accounts` to verify balance and correct account name (GPU account for GPU partition, CPU for CPU) |
| Running `sbatch` | Review script: `--account`, `--time`, `--partition`, `--mem`, resource sanity |
| `rm` or `mv` on shared paths | `ls` the target; confirm with user |
| Multi-node / array jobs | Estimate total SU cost and warn user |

## SAFE — No extra confirmation needed

`squeue -u $USER`, `sinfo`, `scontrol show job`, `sacct`, `accounts`, `jobcharge`, `quota`, `module list/avail/spider`, `ls`, `cat`, `head`, `tail`, `wc`, `cd`, `pwd`, `echo`, `which`, `hostname`, file editing, `git`

## Job Script Checklist

When writing or reviewing a Slurm script, verify:

- [ ] `--account=` matches resource type (GPU account for GPU partition)
- [ ] `--partition=` is valid; see `references/partitions-and-billing.md` *(Running Jobs page)*
- [ ] `--time=` is set to estimate + 20% buffer (not 48-hr max)
- [ ] `--mem=` is set — default is only 1000 MB/core; available RAM is 5-10% less than physical *(Architecture page)*
- [ ] `module reset` at top of script body
- [ ] GPU jobs: `--gpus-per-node=`, `--gpu-bind=closest`
- [ ] Output files use `%j` for job ID
- [ ] Does NOT use `mpirun`
- [ ] If profiling: `--constraint=perf` (CPU) or `--constraint=perf,nvperf` (GPU) *(Debugging page)*

For templates, see `references/job-templates.md`.

## Filesystem Safety

> Full details in `references/filesystem-safety.md`.

**Key facts** *(Data Management page)*:
- `/work` and `/projects` have **NO backups or snapshots** — deleted files are gone permanently
- HOME (`/u`) snapshots exist but "are not backups and reside on the same hardware"
- HOME is "not intended as a source/destination for I/O during jobs"
- `/scratch` and `/work/hdd` are the same storage ("this is now your scratch volume")
- "Delta does not provide storage resources for archiving data or other long-term storage"

**Before any destructive operation on `/work` or `/projects`:** explicitly warn the user that files cannot be recovered.

## Environment Management

> Source: [Software](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/software.html)

- Install envs to `/work/hdd` or `/projects`, not `$HOME` — "target a filesystem with more quota space (not `$HOME`)"
- In job scripts, do NOT write `conda activate` — "the job inherits your environment"
- Pre-submit: `conda deactivate && module purge && module reset && conda activate <env>`
- "Unload cuda when building CPU-only packages" (official warning)
- pip `--user` can cause incompatibilities — use `PYTHONUSERBASE` to override
- For GPU work: "Delta staff recommend using an NGC container when possible"
- Verify module versions with `module spider` before using

## Billing Awareness

> Full rules and partition table in `references/partitions-and-billing.md`.

**Official rule** *(Job Accounting page)*: "Charges are based on the resources that are reserved for your job and do not necessarily reflect how the resources are used." Billing uses the larger of core fraction or memory fraction.

- Run `accounts` before submitting expensive jobs
- Interactive sessions reserve resources the entire time, even if idle; interactive partitions have higher charge factors *(Running Jobs page)*
- A40 and MI100 partitions have discounted charge weights *(Job Accounting page)*

## When Things Go Wrong

| Symptom | Cause | Action |
|---------|-------|--------|
| `(QOSGrpBillingMinutes)` | Insufficient balance for requested resources *(Job Accounting)* | `accounts`; PI submits supplement via XRAS |
| `(MaxGRESPerAccount)` | Exceeded GPU/core limit per user/project *(Running Jobs)* | Wait or reduce request |
| `nvidia-smi` — no GPU | On login/CPU node *(FAQ)* | Submit to GPU partition |
| `ImportError: libstdc++.so.6` | Software not built on Delta *(FAQ)* | `export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$LIBRARY_PATH` |
| Apptainer "No space left" | /tmp full *(Containers)* | `rm -rf /tmp/build-temp*`; set `APPTAINER_CACHEDIR` |
| HOME quota full | conda/pip/Apptainer cache in /u | Move to `/work/hdd` with `--prefix`; set `APPTAINER_CACHEDIR` |
| Job killed at `--time` | Wall clock limit | Increase `--time` or checkpoint |

## Reference Files

- `references/partitions-and-billing.md` — Billing rules, SU equivalents, partition table, partition selection guide
- `references/job-templates.md` — Validated Slurm job script templates (CPU, GPU, multi-GPU, distributed, interactive)
- `references/filesystem-safety.md` — Storage tiers, quotas, snapshots, data transfer, constraint labels
