---
name: run-experiment
description: Deploy and run ML experiments on local or remote GPU servers. Use when user says "run experiment", "deploy to server", "跑实验", or needs to launch training jobs.
---

# Run Experiment

Deploy and run ML experiment: $ARGUMENTS

## Step 1: Detect Environment

Determine execution environment by checking:

```bash
# SLURM cluster?
which squeue 2>/dev/null && echo "SLURM"

# Local GPU?
nvidia-smi 2>/dev/null && echo "LOCAL_GPU"
```

### If SLURM is detected -> Delegate to `/slurm-job`

This project runs on a SLURM HPC cluster. Delegate the entire workflow:

```
/slurm-job submit "$ARGUMENTS"
```

The `/slurm-job` skill handles config changes, pre-flight checks, permission checking (per CLAUDE.md), submission, and status reporting.

**Do NOT proceed with the generic workflow below if SLURM is detected.**

---

### If Local GPU or SSH Remote -> Generic Workflow

## Step 2: Pre-flight Check

Check GPU availability:

**Remote:**
```bash
ssh <server> nvidia-smi --query-gpu=index,memory.used,memory.total --format=csv,noheader
```

**Local:**
```bash
nvidia-smi --query-gpu=index,memory.used,memory.total --format=csv,noheader
```

Free GPU = memory.used < 500 MiB.

## Step 3: Sync Code (Remote Only)

```bash
rsync -avz --include='*.py' --exclude='*' <local_src>/ <server>:<remote_dst>/
```

## Step 4: Deploy

#### Remote (via SSH + screen)
```bash
ssh <server> "screen -dmS <exp_name> bash -c '\
  eval \"\$(<conda_path>/conda shell.bash hook)\" && \
  conda activate <env> && \
  CUDA_VISIBLE_DEVICES=<gpu_id> python <script> <args> 2>&1 | tee <log_file>'"
```

#### Local
```bash
CUDA_VISIBLE_DEVICES=<gpu_id> python <script> <args> 2>&1 | tee <log_file>
```

## Step 5: Verify Launch

**Remote:** `ssh <server> "screen -ls"`
**Local:** Check process is running.

## Step 6: Feishu Notification (if configured)

Check `~/.claude/feishu.json` and send notification if enabled.

## Key Rules

- ALWAYS check GPU availability first
- Each experiment gets its own screen session + GPU (remote) or background process (local)
- Use `tee` to save logs
- Report back: which GPU, which screen/process, what command, estimated time
