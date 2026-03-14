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

### If SLURM is detected -> Delegate to `/slurm-job`

This project runs on a SLURM HPC cluster. Delegate monitoring:

```
/slurm-job monitor "$ARGUMENTS"
```

If the user also wants result collection:

```
/slurm-job collect "$ARGUMENTS"
```

The `/slurm-job` skill handles job status, log reading, result collection, comparison tables, and doc updates. It also enforces monitoring rate limits to avoid overloading the SLURM controller.

**Do NOT proceed with the generic workflow below if SLURM is detected.**

---

### If SSH Remote -> Generic Workflow

## Step 2: Check What's Running
```bash
ssh <server> "screen -ls"
```

## Step 3: Collect Output from Each Screen
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

## Step 5: Summarize Results

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
