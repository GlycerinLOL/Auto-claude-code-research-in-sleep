---
name: research-refine
description: "Iteratively refine research problem decomposition and method via Claude + GPT-5.4 peer review loop. Use when user says \"refine my approach\", \"帮我细化方案\", \"decompose this problem\", \"打磨idea\", \"refine research plan\", \"细化研究方案\", or wants to turn a vague research idea into a concrete, top-venue-ready plan."
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, WebFetch, Agent, mcp__codex__codex, mcp__codex__codex-reply
---

# Research Refine: Iterative Problem Decomposition & Method Concretization

Refine and concretize: **$ARGUMENTS**

## Overview

Given a research PROBLEM and a vague APPROACH, iteratively refine the problem decomposition and method into a concrete, top-venue-ready plan through a Claude + GPT-5.4 peer review loop. This skill fills the gap between "I have a vague idea" and "let me implement it."

```
User input (PROBLEM + vague APPROACH)
  → Phase 1 (Claude): Scan local papers → Decompose problem → Concretize method → Write initial proposal
  → Phase 2 (Codex/GPT-5.4): 6-dimension peer review + scoring (1-10)
  → Phase 3 (Claude): Parse feedback → Revise proposal → Track changes
  → Phase 4 (Codex, same thread): Re-evaluate revised proposal
  → Repeat Phase 3-4 until OVERALL SCORE >= 7 or MAX_ROUNDS (5) reached
  → Phase 5: Save all iteration history to refine-logs/
```

## Constants

- **REVIEWER_MODEL = `gpt-5.4`** — Model used via Codex MCP for peer review. Must be an OpenAI model (e.g., `gpt-5.4`, `o3`, `gpt-4o`).
- **MAX_ROUNDS = 5** — Maximum number of review-revise rounds before forced termination.
- **SCORE_THRESHOLD = 7** — Minimum overall score to stop iterating. A plan rated 6/10 isn't worth implementing.
- **OUTPUT_DIR = `refine-logs/`** — Directory for per-round files and final report.
- **MAX_LOCAL_PAPERS = 15** — Maximum number of local papers to scan for grounding.

> 💡 Override via argument, e.g., `/research-refine "problem | approach" — max rounds: 3, threshold: 8`.

## Output Structure

```
refine-logs/
├── round-0-initial-proposal.md
├── round-1-review.md
├── round-1-refinement.md
├── round-2-review.md
├── round-2-refinement.md
├── ...
├── REFINEMENT_REPORT.md        (final summary + all raw responses)
└── score-history.md            (score evolution table)
```

## Workflow

### Phase 1: Problem Decomposition & Initial Proposal

Build a grounded, concrete initial proposal from the user's vague input.

#### Step 1.1: Scan Local Paper Library

Check `papers/` and `literature/` in the project directory for existing PDFs and notes. Read first 3 pages of relevant papers (up to MAX_LOCAL_PAPERS) to build a baseline understanding.

- Identify relevant methods, baselines, evaluation protocols
- Note recurring experimental setups and metrics in the area
- Extract key assumptions and limitations from related work
- Build a list of "grounding references" to cite in the proposal

Also search online if local papers are insufficient:
- Use WebSearch for recent arXiv preprints and top venue papers
- Focus on methodological details (not just abstracts)

#### Step 1.2: Decompose the Problem

Break the user's problem into concrete sub-problems:

1. **Core research question**: What exactly are we trying to answer?
2. **Sub-questions**: What smaller questions must be answered first?
3. **Assumptions**: What must be true for this approach to work?
4. **Key technical challenges**: What are the hardest parts?
5. **Evaluation criteria**: How will we know if the method works?

#### Step 1.3: Concretize the Method

Transform the vague approach into a specific method:

1. **Algorithm / pipeline**: Step-by-step description of the proposed method
2. **Key design choices**: Why this architecture / loss / training procedure?
3. **Baselines**: What are we comparing against? (minimum 2-3 strong baselines)
4. **Datasets**: Which datasets, what splits, what preprocessing?
5. **Metrics**: Primary and secondary evaluation metrics
6. **Ablations**: What components to ablate to prove each design choice matters?
7. **Expected results**: What do we expect to see, and why?

#### Step 1.4: Write Initial Proposal

Compile everything into a structured proposal document:

```markdown
# Research Proposal: [Title]

## Problem Statement
[Clear, specific problem definition]

## Research Questions
1. [Primary RQ]
2. [Sub-RQ 1]
3. [Sub-RQ 2]

## Proposed Method
### Overview
[1-paragraph summary]

### Detailed Method
[Step-by-step algorithm/pipeline description]

### Key Design Choices
[Justified decisions with references to local papers]

## Experimental Design
### Baselines
- [Baseline 1]: [why it's relevant]
- [Baseline 2]: [why it's relevant]
- [Baseline 3]: [why it's relevant]

### Datasets
- [Dataset 1]: [size, split, preprocessing]
- [Dataset 2]: [size, split, preprocessing]

### Evaluation Metrics
- Primary: [metric + justification]
- Secondary: [metrics]

### Ablation Studies
1. [Component 1]: remove/replace to test [hypothesis]
2. [Component 2]: remove/replace to test [hypothesis]

## Expected Results
[What we expect and why — be specific about magnitudes]

## Assumptions & Risks
- [Assumption 1]: [what happens if wrong]
- [Risk 1]: [mitigation strategy]

## Compute & Timeline Estimate
- Estimated GPU-hours: [X]
- Timeline: [X days/weeks]
```

Save this proposal to `refine-logs/round-0-initial-proposal.md`.

### Phase 2: External Peer Review (Round 1)

Send the full proposal to GPT-5.4 for a rigorous 6-dimension review:

```
mcp__codex__codex:
  model: REVIEWER_MODEL
  config: {"model_reasoning_effort": "xhigh"}
  prompt: |
    You are a senior ML reviewer for a top venue (NeurIPS/ICML/ICLR).
    Below is a research proposal. Please provide a rigorous peer review.

    === PROPOSAL ===
    [Paste the FULL proposal from Phase 1]
    === END PROPOSAL ===

    Score each dimension 1-10 and provide specific, actionable feedback:

    1. **Clarity** (1-10): Is the problem well-defined? Is the method description precise enough to reproduce? Are the research questions specific and answerable?

    2. **Novelty** (1-10): Does this propose something genuinely new, or is it incremental? How does it differentiate from the closest existing work? Would a reviewer say "so what?"

    3. **Feasibility** (1-10): Can this actually be implemented and run with reasonable resources? Are the compute estimates realistic? Are the datasets accessible?

    4. **Experimental Design** (1-10): Are the baselines strong and fair? Are the metrics appropriate? Are the ablations sufficient to isolate contributions? Is the evaluation rigorous?

    5. **Venue Fit** (1-10): Would this be competitive at a top venue? Is the contribution substantial enough? Does the framing match what reviewers expect?

    6. **Completeness** (1-10): Are there missing baselines, datasets, or analyses? Are there obvious experiments that should be included? Any gaps in the logical chain?

    **OVERALL SCORE** (1-10): Weighted average reflecting your holistic assessment.

    For each dimension scoring < 7, provide:
    - The specific weakness
    - A concrete, actionable suggestion to fix it
    - Priority: CRITICAL (must fix) / IMPORTANT (should fix) / MINOR (nice to fix)

    Finally, give a VERDICT: READY (score >= 7, plan is implementable) / REVISE (specific fixes needed) / RETHINK (fundamental issues with approach)
```

**CRITICAL: Save the threadId** from this call for all subsequent rounds.

**CRITICAL: Save the FULL raw response** verbatim — do NOT discard or summarize.

Save the review to `refine-logs/round-1-review.md` with the raw response in a `<details>` block.

### Phase 3: Parse Feedback & Revise Proposal

#### Step 3.1: Parse Review

Extract structured fields from the reviewer's response:

- **Per-dimension scores**: Clarity, Novelty, Feasibility, Exp Design, Venue Fit, Completeness
- **Overall score** (1-10)
- **Verdict**: READY / REVISE / RETHINK
- **Action items**: Ranked list with priority (CRITICAL / IMPORTANT / MINOR)

Update `refine-logs/score-history.md`:

```markdown
# Score Evolution

| Round | Clarity | Novelty | Feasibility | Exp Design | Venue Fit | Completeness | Overall | Verdict |
|-------|---------|---------|-------------|------------|-----------|--------------|---------|---------|
| 1     | X       | X       | X           | X          | X         | X            | X       | REVISE  |
```

**STOP CONDITION**: If overall score >= SCORE_THRESHOLD AND verdict is READY → skip to Phase 5.

#### Step 3.2: Revise Proposal (with Pushback)

For each action item (CRITICAL first, then IMPORTANT, then MINOR):

1. **Assess the feedback**: Does this criticism hold up against the local papers you've read?
   - If **valid**: Implement the suggested fix in the proposal
   - If **debatable**: Revise the proposal but add a "Rebuttal" note explaining your reasoning, citing specific local papers as evidence. The reviewer may not have access to all relevant work.
   - If **wrong**: Do NOT blindly comply. Add a "Pushback" note with evidence from local papers. This makes the loop more robust than blindly accepting all feedback.

2. **Track all changes**: For each revision, note:
   - What was changed
   - Why (reviewer feedback + your reasoning)
   - What evidence supports the change (or pushback)

3. **Rewrite the full proposal**: Produce a complete updated proposal (not a diff — GPT needs the full text in context).

Save to `refine-logs/round-N-refinement.md`:

```markdown
# Round N Refinement

## Changes Made

### 1. [Section changed]
- **Reviewer said**: [quote]
- **Action**: [what was changed]
- **Reasoning**: [why, with evidence]

### 2. [Section changed]
- **Reviewer said**: [quote]
- **Action**: [pushback — kept original with justification]
- **Evidence**: [citation from local papers]

...

## Revised Proposal

[Full updated proposal text]
```

### Phase 4: Re-evaluation (Round 2+)

Send the revised proposal back to GPT-5.4 using the **same thread** for continuity:

```
mcp__codex__codex-reply:
  threadId: [saved from Phase 2]
  model: REVIEWER_MODEL
  config: {"model_reasoning_effort": "xhigh"}
  prompt: |
    [Round N re-evaluation]

    I have revised the proposal based on your feedback. Here are the key changes:

    1. [Change 1]: [brief description + reasoning]
    2. [Change 2]: [brief description + reasoning]
    3. [Pushback on point X]: [your evidence for disagreeing]

    === REVISED PROPOSAL ===
    [Paste the FULL revised proposal — you cannot read files]
    === END REVISED PROPOSAL ===

    Please re-score all 6 dimensions and overall. For each dimension:
    - Has the score improved, stayed the same, or decreased?
    - If the author pushed back on your feedback, do you accept their reasoning?
    - Are there NEW issues introduced by the revisions?

    Same format: 6 dimension scores, overall score, verdict, remaining action items with priorities.
```

Save review to `refine-logs/round-N-review.md`.

**Then return to Phase 3** (parse → revise → re-evaluate) until:
- **Overall score >= SCORE_THRESHOLD** AND verdict is READY → proceed to Phase 5
- **MAX_ROUNDS reached** → proceed to Phase 5 with current state

### Phase 5: Final Report & Logs

#### Step 5.1: Write REFINEMENT_REPORT.md

Save to `refine-logs/REFINEMENT_REPORT.md`:

```markdown
# Refinement Report

**Problem**: [user's problem]
**Initial Approach**: [user's vague approach]
**Date**: [today]
**Rounds**: N / MAX_ROUNDS
**Final Score**: X / 10
**Final Verdict**: [READY / REVISE / RETHINK]

## Score Evolution

| Round | Clarity | Novelty | Feasibility | Exp Design | Venue Fit | Completeness | Overall | Verdict |
|-------|---------|---------|-------------|------------|-----------|--------------|---------|---------|
| 1     | ...     | ...     | ...         | ...        | ...       | ...          | ...     | ...     |
| 2     | ...     | ...     | ...         | ...        | ...       | ...          | ...     | ...     |
| ...   | ...     | ...     | ...         | ...        | ...       | ...          | ...     | ...     |

## Final Proposal

[Full text of the final refined proposal]

## Key Refinements Made

1. [Most impactful change + reasoning]
2. [Second most impactful change + reasoning]
3. ...

## Pushback Log

| Round | Reviewer Said | Author Response | Outcome |
|-------|---------------|-----------------|---------|
| 1     | [criticism]   | [pushback + evidence] | [accepted/rejected by reviewer] |
| ...   | ...           | ...             | ...     |

## Remaining Weaknesses

[Issues not fully resolved — honest assessment]

## Raw Reviewer Responses

<details>
<summary>Round 1 Review</summary>

[Full verbatim response from GPT-5.4]

</details>

<details>
<summary>Round 2 Review</summary>

[Full verbatim response from GPT-5.4]

</details>

...

## Next Steps

- If READY: proceed to implementation → `/run-experiment` → `/auto-review-loop`
- If REVISE: address remaining weaknesses manually, then re-run `/research-refine`
- If RETHINK: reconsider the fundamental approach, possibly re-run `/idea-creator`
```

#### Step 5.2: Finalize score-history.md

Ensure `refine-logs/score-history.md` contains the complete score evolution table.

#### Step 5.3: Present Summary to User

```
📋 Refinement complete after N rounds.

Final score: X/10 (Verdict: READY/REVISE/RETHINK)

Score evolution:
  Round 1: X/10 → Round 2: Y/10 → ... → Round N: Z/10

Key improvements:
- [biggest change 1]
- [biggest change 2]

Remaining concerns:
- [if any]

Full report: refine-logs/REFINEMENT_REPORT.md
Final proposal: refine-logs/round-N-refinement.md
```

## Key Rules

- **ALWAYS use `config: {"model_reasoning_effort": "xhigh"}`** for all Codex calls — maximum reasoning depth for plan-level review.
- **Save threadId from Phase 2**, use `mcp__codex__codex-reply` for all subsequent rounds (Phase 4). Do NOT create a new thread per round.
- **Full proposal re-sent every round** — GPT cannot read files. Always paste the complete revised proposal in the prompt, not just a diff.
- **Pushback is encouraged.** Do NOT blindly accept all reviewer feedback. Use local paper evidence to disagree when the reviewer is wrong. This makes the loop more robust than one-directional compliance.
- **Per-round files, not a single log.** Save each review and refinement as a separate file in `refine-logs/` for easy diffing and tracking.
- **Threshold 7, not 6.** A plan rated 6/10 isn't worth implementing — the bar must be higher than for a finished paper review because the plan still needs to survive implementation and full review.
- **Do NOT fabricate results.** The proposal should describe *expected* results and *planned* experiments, not claim results that don't exist yet.
- **Be specific about compute.** Vague "we'll train a model" is not a plan. Specify GPU-hours, dataset sizes, training iterations.
- **Baselines are non-negotiable.** Every proposal must include at least 2-3 strong, relevant baselines. "We compare against random" is not acceptable.
- **Document everything.** Every round's raw review, every change made, every pushback. The refinement log should be a complete record of the plan's evolution.

## Composing with Other Skills

This skill sits between idea discovery and implementation in the research pipeline:

```
/idea-creator "direction"       → ranked ideas with pilot results
/research-refine "PROBLEM: ... | APPROACH: ..."  ← you are here
/run-experiment                 → deploy full-scale experiments
/auto-review-loop               → iterate on results until submission-ready
```

**Typical flow:**
1. `/idea-creator` produces a top idea with a positive pilot signal
2. `/research-refine` takes that idea and turns the vague plan into a concrete, reviewable proposal
3. Implementation begins based on the refined proposal
4. `/auto-review-loop` iterates on the actual paper + results

**Can also be used standalone:** If you already have a problem and approach (e.g., from reading papers or from a collaborator's suggestion), invoke directly without running `/idea-creator` first.
