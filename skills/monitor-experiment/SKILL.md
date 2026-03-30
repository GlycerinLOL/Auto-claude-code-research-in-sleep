---
name: monitor-experiment
description: Monitor running experiments, check progress, collect results. Use when user says "check results", "is it done", "monitor", or wants experiment output.
---

# Monitor Experiment Results

Monitor: $ARGUMENTS

## Step 1: Detect Environment

```bash
which squeue 2>/dev/null && echo "SLURM"
```

### If SLURM is detected -> Use CronCreate for background monitoring

This project runs on a SLURM HPC cluster. Set up automatic background monitoring using CronCreate:

**Step A: One-time status check first**

```bash
squeue -u "$USER" --format="%.18i %.9P %.50j %.8u %.8T %.10M %.6D %R"
tail -30 slurm-logs/<JOBID>.out 2>/dev/null
```

Report current status to user.

**Step B: Set up CronCreate for ongoing monitoring**

```
CronCreate(
  cron: "*/5 * * * *",
  prompt: "Check SLURM job <JOBID> (<exp_name>):
1. Run: squeue -j <JOBID> --noheader 2>/dev/null
2. If running → read last 30 lines of slurm-logs/<JOBID>.out, report progress (step/total, loss, ETA). Only notify user on anomalies (NaN, OOM, divergence).
3. If done → sacct -j <JOBID> --format=JobID,State,ExitCode,Elapsed --noheader
   - COMPLETED → collect results via /slurm-job collect <JOBID>, then CronDelete this job
   - FAILED → read slurm-logs/<JOBID>.err, report error, then CronDelete this job"
)
```

**Step C: If the user also wants immediate result collection** (job already finished):

```
/slurm-job collect "$ARGUMENTS"
```

**Do NOT use `/loop` — always use CronCreate for background monitoring.**
**Do NOT proceed with the generic workflow below if SLURM is detected.**

---

### If SSH Remote -> Generic Workflow

## Step 2: Check What's Running
```bash
ssh <server> "screen -ls"
```

**Vast.ai instance** (read `ssh_host`, `ssh_port` from `vast-instances.json`):
```bash
ssh -p <PORT> root@<HOST> "screen -ls"
```

Also check vast.ai instance status:
```bash
vastai show instances
```

### Step 2: Collect Output from Each Screen
For each screen session, capture the last N lines:
```bash
ssh <server> "screen -S <name> -X hardcopy /tmp/screen_<name>.txt && tail -50 /tmp/screen_<name>.txt"
```

## Step 4: Check for JSON Result Files
```bash
ssh <server> "ls -lt <results_dir>/*.json 2>/dev/null | head -20"
```

If JSON results exist, fetch and parse them:
```bash
ssh <server> "cat <results_dir>/<latest>.json"
```

### Step 3.5: Pull W&B Metrics (when `wandb: true` in CLAUDE.md)

**Skip this step entirely if `wandb` is not set or is `false` in CLAUDE.md.**

Pull training curves and metrics from Weights & Biases via Python API:

```bash
# List recent runs in the project
ssh <server> "python3 -c \"
import wandb
api = wandb.Api()
runs = api.runs('<entity>/<project>', per_page=10)
for r in runs:
    print(f'{r.id}  {r.state}  {r.name}  {r.summary.get(\"eval/loss\", \"N/A\")}')
\""

# Pull specific metrics from a run (last 50 steps)
ssh <server> "python3 -c \"
import wandb, json
api = wandb.Api()
run = api.run('<entity>/<project>/<run_id>')
history = list(run.scan_history(keys=['train/loss', 'eval/loss', 'eval/ppl', 'train/lr'], page_size=50))
print(json.dumps(history[-10:], indent=2))
\""

# Pull run summary (final metrics)
ssh <server> "python3 -c \"
import wandb, json
api = wandb.Api()
run = api.run('<entity>/<project>/<run_id>')
print(json.dumps(dict(run.summary), indent=2, default=str))
\""
```

**What to extract:**
- **Training loss curve** — is it converging? diverging? plateauing?
- **Eval metrics** — loss, PPL, accuracy at latest checkpoint
- **Learning rate** — is the schedule behaving as expected?
- **GPU memory** — any OOM risk?
- **Run status** — running / finished / crashed?

**W&B dashboard link** (include in summary for user):
```
https://wandb.ai/<entity>/<project>/runs/<run_id>
```

> This gives the auto-review-loop richer signal than just screen output — training dynamics, loss curves, and metric trends over time.

### Step 4: Summarize Results

Present results in a comparison table:
```
| Experiment | Metric | Delta vs Baseline | Status |
|-----------|--------|-------------------|--------|
| Baseline  | X.XX   |                   | done   |
| Method A  | X.XX   | +Y.Y              | done   |
```

## Step 6: Interpret
- Compare against known baselines
- Flag unexpected results (negative delta, NaN, divergence)
- Suggest next steps based on findings

## Step 7: Feishu Notification (if configured)

Check `~/.claude/feishu.json` and send notification if enabled.

## Key Rules
- Always show raw numbers before interpretation
- Compare against the correct baseline (same config)
- Note if experiments are still running (check progress bars, iteration counts)
- If results look wrong, check training logs for errors before concluding
- **Vast.ai cost awareness**: When monitoring vast.ai instances, report the running cost (hours * $/hr from `vast-instances.json`). If all experiments on an instance are done, remind the user to run `/vast-gpu destroy <instance_id>` to stop billing
