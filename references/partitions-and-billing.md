# Partitions and Billing Reference

Each section cites its official source page. All quotes are verbatim.

---

## Billing Rules

> Source: [Job Accounting](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/job_accounting.html)

**SU definition:** "the equivalent use of one compute core utilizing less than or equal to 2G of memory for one hour, or 1 GPU or fractional GPU using less than the corresponding amount of memory or cores for 1 hour."

**Charging principle:** "Charges are based on the resources that are reserved for your job and do not necessarily reflect how the resources are used."

**Charge calculation:** "Charges are based on either the number of cores or the fraction of the memory requested, whichever is larger."

**Discount mention:** "a weighting factor will discount the charge for the reduced-precision A40 nodes, as well as the novel AMD MI100 based node."

### SU Equivalents by Node Type

> Source: [Job Accounting](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/job_accounting.html)

| Node Type | Cores per SU | GPU per SU | Memory per SU |
|-----------|-------------|------------|---------------|
| CPU Node | 1 | N/A | 2 GB |
| Quad A100 (40 GB) | 16 | 1x A100 | 62.5 GB |
| Quad A40 (48 GB) | 16 | 1x A40 | 62.5 GB |
| 8-way A100 (40 GB) | 16 | 1x A100 | 250 GB |
| 8-way H200 (141 GB) | 12 | 1x H200 | 250 GB |
| 8-way MI100 (32 GB) | 16 | 1x MI100 | 250 GB |

### Quick Billing Checklist

1. Run `accounts` to see your balance before submitting *(source: Job Accounting)*
2. Know your job's resource request (cores, memory, GPUs)
3. Billing = max(core fraction, memory fraction) of the node *(source: Job Accounting — verbatim rule)*
4. Reserved resources are charged, not actual utilization *(source: Job Accounting — verbatim rule)*

### Budget Monitoring Commands

> Source: [Job Accounting](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/job_accounting.html)

```bash
accounts                              # View all accounts and balances
jobcharge <account-name> -d 10        # Charges for last 10 days
jobcharge <account-name> -d 10 --detail  # Per-job breakdown
jobcharge <account-name> -m 3 -y 2026    # Specific month
```

---

## Partition Table

> Source: [Running Jobs](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/running_jobs.html) — this is a separate official NCSA documentation page, not the Job Accounting page.

| Partition | Node Type | Max Nodes | Max Time | Charge Factor |
|-----------|-----------|-----------|----------|---------------|
| `cpu` | CPU | ALL | 48 hr | 1.0 |
| `cpu-interactive` | CPU | 4 | 1 hr | 2.0 |
| `cpu-preempt` | CPU | ALL | 48 hr | 0.5 |
| `gpuA100x4` | quad-A100 | ALL | 48 hr | 1.0 |
| `gpuA100x4-interactive` | quad-A100 | 4 | 1 hr | 2.0 |
| `gpuA100x4-preempt` | quad-A100 | ALL | 48 hr | 0.5 |
| `gpuA100x8` | octa-A100 | ALL | 48 hr | 1.5 |
| `gpuA100x8-interactive` | octa-A100 | 2 | 1 hr | 3.0 |
| `gpuH200x8` | octa-H200 | 1 | 48 hr | 3.0 |
| `gpuH200x8-interactive` | octa-H200 | 1 | 1 hr | 6.0 |
| `gpuA40x4` | quad-A40 | ALL | 48 hr | 0.5 |
| `gpuA40x4-interactive` | quad-A40 | 4 | 1 hr | 1.0 |
| `gpuA40x4-preempt` | quad-A40 | ALL | 48 hr | 0.25 |
| `gpuMI100x8` | octa-MI100 | ALL | 48 hr | 0.25 |
| `gpuMI100x8-interactive` | octa-MI100 | ALL | 1 hr | 0.5 |

> Source: [Running Jobs](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/running_jobs.html)

**Defaults:** "Default Memory per core: 1000 MB" and "Default Wall-clock time: 30 minutes"

---

## Partition Selection Guide

*Recommendation — not from any official NCSA page. Based on the partition table above.*

```
Need GPU?
├── No → cpu or cpu-preempt (lower charge factor per Running Jobs page; needs checkpoint)
└── Yes
    ├── Just debugging (< 1 hr)?
    │   └── gpuA40x4-interactive
    ├── A40 48GB sufficient?
    │   ├── Has checkpoint? → gpuA40x4-preempt (lowest charge factor per Running Jobs page)
    │   └── No checkpoint → gpuA40x4
    ├── Need A100 40GB / NVLink?
    │   └── gpuA100x4
    ├── Need 5-8 GPUs on one node?
    │   └── gpuA100x8
    ├── Need H200 141GB HBM3?
    │   └── gpuH200x8 (max 1 node per Running Jobs page)
    └── Need 2TB host memory (no GPU needed)?
        └── gpuMI100x8 — "good resource for large memory (2TB) jobs that do not need a GPU as the charging is set to promote use" *(Architecture page)*
```
