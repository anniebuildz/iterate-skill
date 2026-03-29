# iterate

A Claude Code skill that replaces "change → eyeball it → looks fine" with **"change → auto-verify → independent scoring → fix based on feedback"**.

## What it does

`/iterate` sets up an autonomous Generator/Evaluator loop for iterative improvement. You point it at something — a UI, a calculation, a prompt — and it:

1. **Understands the target** — accepts screenshots, URLs, or text descriptions
2. **Defines scoring criteria** — 3-5 dimensions, each scored 1-10
3. **Iterates autonomously** — modifies code, verifies results, spawns an independent evaluator agent to score
4. **Stops when done** — all scores ≥ 8/10, or max rounds reached
5. **Produces a comparison** — HTML viewer for visual work, markdown report for everything else

The key insight: the Generator and Evaluator run in **separate contexts** to avoid self-evaluation bias. The agent that wrote the code never scores its own work.

## Install

```bash
# Claude Code
claude install-skill anniebuildz/iterate-skill
```

## Usage

```
/iterate              # then describe what to optimize
/iterate homepage     # optimize the homepage design
```

Input methods:
- **Screenshot path** — `~/Desktop/current-state.png`
- **URL** — `localhost:3000` (auto-captures via Playwright)
- **Text** — describe what needs improvement

## Scoring

For design/UI work, four default criteria:

| Dimension | What it measures |
|-----------|-----------------|
| **Design Quality** | Cohesion — does it feel like a unified whole, not assembled parts? |
| **Originality** | Evidence of intentional design decisions vs. template/AI defaults |
| **Craft** | Technical execution — type hierarchy, spacing, color harmony, contrast |
| **Functionality** | Usability independent of aesthetics — can users complete tasks? |

For non-visual work (logic, performance, prompts), criteria are generated to fit the target.

## How the loop works

```
┌─────────────┐
│  Generator   │ ← reads evaluator feedback
│  edits code  │
└──────┬──────┘
       ▼
┌─────────────┐
│   Verify     │ ← screenshot / test / output
└──────┬──────┘
       ▼
┌─────────────┐
│  Evaluator   │ ← independent agent, strict scoring
│  (separate)  │
└──────┬──────┘
       ▼
   All ≥ 8? ──yes──▶ Done
       │
      no
       │
       ▼
   Back to Generator
```

- Max 8 rounds by default (configurable)
- Each round makes small, targeted changes (2-3 issues)
- Original files are backed up before any modification
- Every iteration's screenshots and scores are saved

## Key rules

- **Independent evaluation** — the evaluator is always a separate agent
- **Small changes per round** — easier to attribute what worked
- **Stay in scope** — out-of-scope issues are noted but not fixed
- **Real data** — no placeholder content in mocks
- **Backups first** — originals are always preserved
