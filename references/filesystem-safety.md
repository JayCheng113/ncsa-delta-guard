# Filesystem Safety Reference

> All data in this file is sourced from the official [Data Management](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/data_mgmt.html) page unless otherwise noted.

## Storage Tiers

| Filesystem | Path | Quota | Snapshots | Purged | Key Features (official) |
|------------|------|-------|-----------|--------|------------------------|
| HOME | `/u` | 100 GB, 750,000 files per user | Yes (30 days) | No | "Not intended as a source/destination for I/O during jobs." |
| PROJECTS | `/projects` | 500 GB (up to 1-25 TB by request; large requests may have a fee) | No | No | "Area for shared data for a project, common data sets, software, results." |
| WORK-HDD | `/work/hdd` | 1000 GB (up to 1-100 TB by request) | No | No | "Area for computation, largest allocations, where I/O from jobs should occur. (this is now your scratch volume)" |
| WORK-NVMe | `/work/nvme` | By request | No | No | "Area for computation, NVME is best for lots of small I/O from jobs." |
| /tmp | `/tmp` | 0.74 TB (CPU) / 1.5 TB (GPU), no quotas | No | After each job | "Locally attached disk for fast small file I/O." |

**Official warning:** "Delta does not provide storage resources for archiving data or other long-term storage."

## Critical Safety Rules

### /work and /projects — NO RECOVERY POSSIBLE

Files deleted from `/work/hdd`, `/work/nvme`, or `/projects` are **permanently gone**. There are no snapshots, no trash, no undelete.

Before any destructive operation on these paths:
1. List the target contents
2. Show the user what will be affected
3. Explicitly state: "This path has no backups. This action is irreversible."
4. Get explicit confirmation

### HOME — Snapshots exist but are NOT backups

Official warning: **"These snapshots are not backups and reside on the same hardware as the primary copy of the data."**

HOME (`/u/$USER`) has daily snapshots (30 days per official table):

```bash
# List available snapshots
ls ~/.snapshot/

# Recover a deleted file
cp ~/.snapshot/snapshot-daily-YYYY-MM-DD_HH_mm_ss_UTC/path/to/file ~/path/to/file
```

### /tmp — Ephemeral

Node-local `/tmp` is wiped after every job. Never store anything there that needs to persist.

## Directory Structure

```
/u/$USER                              # HOME — configs, scripts, small files
/projects/<project_code>/$USER        # Shared project data
/work/hdd/<project_code>/$USER        # Primary compute directory (HDD tier)
/work/nvme/<project_code>/$USER       # High-speed I/O (NVMe tier, by request)
```

Per [Data Management](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/data_mgmt.html) page: "`module reset` in a job script populates `$WORK` and `$SCRATCH` environment variables automatically." Manual equivalents from the same page:
```bash
WORK=/projects/<account>/$USER
SCRATCH=/scratch/<account>/$USER
```

**Important naming clarification:** `$WORK` points to `/projects/`, NOT to `/work/hdd` — despite the similar name. The WORK-HDD storage tier (`/work/hdd`) is described as "this is now your scratch volume" *(Data Management page)*, meaning `/scratch` and `/work/hdd` are the same underlying storage. When the official docs recommend `/scratch` as an install target, use `/work/hdd/<project>/$USER/` in practice.

## Quota Management

```bash
# Check quotas across all filesystems
quota
```

### Common Quota Issues

**HOME full (100 GB):** Usually caused by conda environments or pip cache.
- Check: `du -sh ~/.conda ~/.cache/pip ~/.local`
- Fix: Move conda envs to `/work/hdd` with `--prefix`
- Fix: Set `pip cache purge` or `pip install --no-cache-dir`

**Inode limit (750K files):** Conda environments can create hundreds of thousands of files.
- Check: `find ~/.conda -type f | wc -l`
- Fix: Move entire conda to `/work/hdd`

**Apptainer cache filling HOME** *(source: [Containers](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/containers.html))*:
- "If you encounter `$HOME` quota issues with Apptainer caching in `~/.apptainer`, the environment variable `APPTAINER_CACHEDIR` can be used to select a different location such as a directory under `/scratch`."
- `export APPTAINER_CACHEDIR=/work/hdd/<project>/$USER/.apptainer_cache`

## Performance Characteristics

> Source: [Architecture](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/architecture.html) page.

| Storage Tier | Aggregate Bandwidth | Best For |
|-------------|---------------------|----------|
| NVMe (`/work/nvme`) | ~800 GB/s | High-speed I/O, large checkpoints, data loading |
| HDD (`/work/hdd`) | ~60 GB/s | General compute, medium I/O |
| Taiga (`/projects`) | 150 GB/s | Shared project data, cross-project collaboration |
| Harbor (`/u`, `/sw`) | ~80 GB/s | Small files, configs |

## Data Transfer Methods

### Small to medium files — scp/rsync

```bash
# Upload
scp localfile.dat user@login.delta.ncsa.illinois.edu:/work/hdd/<project>/$USER/

# Upload directory with resume support
rsync -avz local_dir/ user@login.delta.ncsa.illinois.edu:/work/hdd/<project>/$USER/

# Download
scp user@login.delta.ncsa.illinois.edu:/work/hdd/<project>/$USER/result.dat ./
```

### Large files — Globus (recommended)

Use Globus web interface with endpoint:
- `NCSA Delta` or `ACCESS Delta`

Accessible paths: `/u/$USER`, `/work/hdd`, `/work/nvme`

### rsync --delete WARNING

`rsync --delete` removes files at the destination that don't exist at the source. On `/work` or `/projects`, this could permanently destroy data. Always do a dry run first:

```bash
rsync -avzn --delete source/ dest/   # -n = dry run, shows what WOULD happen
```

## Filesystem Constraints for Jobs

Declare which filesystems your job needs via `--constraint` *(source: [Running Jobs](https://docs.ncsa.illinois.edu/systems/delta/en/latest/user_guide/running_jobs.html) page)*:

| Filesystem | Constraint Label | Notes |
|-----------|-----------------|-------|
| /scratch (/work/hdd) | `scratch` | Most common; used in all official examples |
| /projects | `projects` or `taiga` | Same underlying FS |
| /work/hdd | `work` | Same storage as `scratch` — both labels valid per Running Jobs page |
| /ime | `ime` | Depends on /work/hdd |

```bash
#SBATCH --constraint="scratch"
#SBATCH --constraint="scratch&ime"
```
