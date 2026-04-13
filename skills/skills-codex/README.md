# `skills-codex`

Codex-native mirror of the base ARIS skill set.

## Scope

This package keeps the main `skills/` workflows available for OpenAI Codex CLI.

Recent core workflow follow-up skills mirrored here include:

- `slurm-job`
- `training-check`
- `result-to-claim`
- `ablation-planner`

These skills cover the experiment lifecycle on HPC clusters:

1. submit, monitor, and collect SLURM jobs
2. monitor training quality early
3. judge what claims the results actually support
4. design reviewer-facing ablations before paper writing

`run-experiment` and `monitor-experiment` auto-detect SLURM and delegate to `slurm-job` when available.

## Install

Copy this directory into your Codex skills path:

```bash
cp -a skills/skills-codex/* ~/.codex/skills/
```

Optional companion dependency for the `deepxiv` skill:

```bash
pip install deepxiv-sdk
```

If you also use reviewer overlay packages, install this base package first, then apply the overlay on top.
