# ncsa-delta-guard

Safety guardrails for using [Claude Code](https://claude.com/claude-code) on the [NCSA Delta](https://docs.ncsa.illinois.edu/systems/delta/en/latest/) supercomputer cluster.

## What it does

This skill activates automatically when Claude Code detects you're working on Delta. It prevents common mistakes that can waste compute budget, destroy data, or disrupt other users on the shared cluster.

**Core behaviors:**

- **BLOCK** dangerous operations (e.g., `rm -rf` on unrecoverable paths, heavy computation on login nodes, `mpirun`) — explains the risk and asks for confirmation
- **CHECK** risky operations (e.g., job submission without verifying account balance, file deletion on shared paths) — validates before executing
- **SAFE** read-only operations (e.g., `squeue`, `accounts`, `module list`) — runs without extra confirmation
- **Never fabricates** — when unsure about any Delta-specific fact, searches the official documentation or tells you honestly

## Installation

Copy this skill into your Claude Code skills directory:

```bash
# Personal (all projects)
cp -r ncsa-delta-guard ~/.claude/skills/

# Or project-specific
cp -r ncsa-delta-guard .claude/skills/
```

The skill activates automatically when you mention Delta, Slurm, or related topics.

## File structure

```
ncsa-delta-guard/
├── SKILL.md                          # Main skill — safety rules, checklists, execution context
└── references/
    ├── partitions-and-billing.md     # Billing rules (Job Accounting page) + partition table (Running Jobs page)
    ├── job-templates.md              # Validated Slurm job script templates
    └── filesystem-safety.md          # Storage tiers, quotas, snapshots, data transfer
```

## Source attribution

All factual claims are sourced from official NCSA Delta documentation pages:

| Page | Content |
|------|---------|
| [Job Accounting](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/job_accounting.html) | SU definition, billing rules, `accounts`/`jobcharge` |
| [Running Jobs](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/running_jobs.html) | Partition table, charge factors, defaults, interactive examples |
| [Login](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/login.html) | Login node resource limits (cgroups) |
| [Data Management](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/data_mgmt.html) | Filesystem quotas, snapshots, data transfer |
| [Software](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/software.html) | Module system, conda/pip guidance, NGC containers |
| [Citizenship](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/citizenship.html) | Login node usage rules, auto-kill enforcement |
| [Architecture](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/architecture.html) | Node specs, storage bandwidth, hardware constraints |
| [Containers](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/containers.html) | Apptainer cache/tmp issues |
| [Programming Environment](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/prog_env.html) | No `mpirun`, MPS status |
| [Debugging](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/debug_perf.html) | Profiling constraints (`perf`, `nvperf`) |
| [FAQ](https://docs.ncsa.illinois.edu/systems/delta/en/latest/faq.html) | Common errors (`nvidia-smi`, `libstdc++`) |

Recommendations (partition selection, resource sizing) are explicitly labeled as non-official.

## Key safety principles

1. **Budget** — charges are based on reserved resources, not actual usage; always check balance with `accounts` before submitting
2. **Data** — `/work` and `/projects` have NO backups; HOME snapshots "are not backups and reside on the same hardware"
3. **Shared environment** — login nodes are shared (16 cores / 37-62 GB per user); production work gets auto-killed
4. **Honesty** — when uncertain, the skill instructs Claude to search official docs or tell the user, never fabricate

## License

This skill is provided as-is for safe usage of Claude Code on NCSA Delta. The official NCSA Delta documentation is the authoritative source for all cluster policies.
