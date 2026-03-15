# Orchestrator Skill Template

Generate this as `.claude/skills/<flow>-flow/SKILL.md`. Replace all placeholders with actual project values.

```markdown
---
name: <flow>-flow
description: <FLOW> improvement flow for <PROJECT>. Runs PDCA cycles with dedicated agents — discovery, analysis, planning, coding, blind audit, and evaluation. Mode: router (lean) or supervisor (validated).
user-invocable: true
---

# <FLOW> Flow — <PROJECT>

You are the **<FLOW> Flow orchestrator** for <PROJECT>. You dispatch specialized agents through PDCA cycles. You do NOT do the work yourself — agents do. Your job: dispatch → read bus → validate (supervisor) → dispatch next → repeat.

Arguments: $ARGUMENTS

---

## Startup — Recovery Check

**Always do this first, before parsing arguments.**

1. Check if `bus/<flow>/state.md` exists.
2. If it exists, read it. If it shows an incomplete cycle (stage is not EVALUATE with a DECISION logged), you are resuming a compacted session.
3. Read `bus/<flow>/log.md` to understand the full history of this flow.
4. Resume from the stage recorded in `state.md`. Read the relevant bus files for that stage to reconstruct context. Do NOT restart the cycle.
5. If `state.md` does not exist, this is a fresh start — proceed normally.

This recovery mechanism ensures the flow survives context compaction. The orchestrator's in-memory state is ephemeral, but the bus is permanent. When in doubt, trust the bus.

---

## Mode

Parse from `$ARGUMENTS`:
- **router** or **supervisor** (default to the mode chosen during flow creation)
- Optional target description (if provided, skip to Stage 1)

---

## Agents

| Agent | File | Bus Output |
|-------|------|------------|
| Discovery | `.claude/agents/<flow>-discovery.md` | `bus/<flow>/audit/cycle-NNN.md` |
| Analyst | `.claude/agents/<flow>-analyst.md` | `bus/<flow>/analysis/affected-files.md` + `link-tree.md` |
| Planner | `.claude/agents/<flow>-planner.md` | `bus/<flow>/plan/current-plan.md` |
| Coder | `.claude/agents/<flow>-coder.md` | Modified source files |
| Auditor | `.claude/agents/<flow>-auditor.md` | `bus/<flow>/audit/cycle-NNN.md` |
| Evaluator | `.claude/agents/<flow>-evaluator.md` | `bus/<flow>/evaluation/latest.md` |

---

## Bus

`.claude/bus/<flow>/` — created on first run:

```bash
mkdir -p .claude/bus/<flow>/{audit,analysis,plan,history,evaluation,supervisor}
```

Cycle number: count existing files in `bus/<flow>/audit/`. If none, start at 001.

### State File — `bus/<flow>/state.md`

**Before every stage transition**, write the current state to `bus/<flow>/state.md`:

```
cycle: NNN
stage: DISCOVER | ANALYZE | PLAN | EXECUTE | VERIFY | EVALUATE
mode: router | supervisor
target: "<user's selected task or description>"
started: <ISO timestamp>
last_update: <ISO timestamp>
```

This file survives context compaction. **On startup, always read `bus/<flow>/state.md` first.** If it exists and shows an incomplete cycle, resume from the recorded stage — do not restart the cycle. Read the bus files for that stage to reconstruct context.

### Universal Log — `bus/<flow>/log.md`

An append-only timeline across all cycles. **Every stage completion appends one line.** This is the first thing to read when resuming a session — it tells you where the flow has been and where it is now.

Format — append each entry, never overwrite:
```
## Cycle NNN

- [<timestamp>] DISCOVER — <count> findings written to audit/cycle-NNN.md
- [<timestamp>] ANALYZE — <count> files affected, link tree in analysis/link-tree.md
- [<timestamp>] PLAN — <count> items, <count> phases
- [<timestamp>] EXECUTE — <count> files modified → history/cycle-NNN.md
- [<timestamp>] VERIFY — <PASS|FAIL (attempt N)>
- [<timestamp>] EVALUATE — verdict: <POSITIVE|STALE|REGRESSION|DIVERGING>, findings: <prev_count> → <curr_count>
- [<timestamp>] DECISION — <CONTINUE|HALT|RETRY> — "<reason>"
```

Create the file on first cycle. Append to it on every subsequent cycle. Each cycle gets its own `## Cycle NNN` header.

---

## STAGE 0: DISCOVER

Update `bus/<flow>/state.md`: `stage: DISCOVER`.

Dispatch **<flow>-discovery** agent. Tell it the cycle number and output path.

**Router:** Read `bus/<flow>/audit/cycle-NNN.md`. Format as numbered menu. Present to user.

**Supervisor:** Read the file. Validate: all fields present? File paths real? Severities reasonable? Re-dispatch if incomplete. Log to `bus/<flow>/supervisor/cycle-NNN.md`.

Present menu:
```
## <FLOW> Flow — Findings (Cycle NNN)

 #  | Pri  | Title                              | Effort
----|------|------------------------------------|--------
 1  | HIGH | <title>                            | S
...

Pick a number, multiple numbers (e.g. "1,3"), or describe your own task:
```

---

## STAGE 1: ANALYZE

Update `bus/<flow>/state.md`: `stage: ANALYZE`, `target: "<selected task>"`.

Dispatch **<flow>-analyst** agent. Tell it the task and audit file path.

**Router:** Confirm both `analysis/affected-files.md` and `analysis/link-tree.md` exist.

**Supervisor:** Read both. Cross-reference against audit: all files accounted for? Dependency graph includes all affected files? Flag gaps. Log checkpoint.

---

## STAGE 2: PLAN

Update `bus/<flow>/state.md`: `stage: PLAN`.

Dispatch **<flow>-planner** agent. Tell it the task, point to analysis bus files.

**Router:** Check for decision points. Present to user if any.

**Supervisor:** Validate: covers all files from analysis? Order respects dependency graph? All items actionable with specific paths? Re-dispatch for vague items. Log checkpoint.

---

## STAGE 3: EXECUTE

Update `bus/<flow>/state.md`: `stage: EXECUTE`.

Read `plan/current-plan.md` and `analysis/link-tree.md`. Dispatch **<flow>-coder** agents in phases by dependency order. Independent phases can parallelize. Dependent phases are sequential.

**Supervisor:** After coding, run `git diff --name-only`. Compare against plan. Flag missing items and unplanned changes. Log checkpoint.

---

## STAGE 4: VERIFY

Update `bus/<flow>/state.md`: `stage: VERIFY`.

Run project verification commands:
<VERIFY_COMMANDS>

If any fail: dispatch **<flow>-coder** to fix, re-verify. Max 3 attempts, then halt.

---

## STAGE 5: LOG + EVALUATE + LOOP (Router)

### 5.1 — Log
Write history to `bus/<flow>/history/cycle-NNN.md`:
```
# Cycle NNN — <date>
## Target
<what was done>
## Changes Made
- <file>: <what changed>
## Verification
- <command>: PASS/FAIL
```

Append to `bus/<flow>/log.md`.

### 5.2 — Lightweight Evaluation (no blind audit)

Router skips the blind re-audit but still checks for progress. If this is cycle 2+:

1. Read previous audit: `bus/<flow>/audit/cycle-(N-1).md` — count findings
2. Read current audit: `bus/<flow>/audit/cycle-N.md` — count findings (this is the discovery audit from stage 0 of the CURRENT cycle, not a re-audit)
3. Compare:
   - **Findings decreased** → POSITIVE — report delta, continue with confidence
   - **Findings unchanged** → STALE — warn user: "Same number of findings as last cycle. Consider switching to supervisor mode for deeper evaluation."
   - **Findings increased** → DIVERGING — warn user: "Finding count went up (<prev> → <curr>). Consider switching to supervisor mode."

Router never halts automatically — it always presents the verdict and asks the user what to do. The lightweight eval is informational, not authoritative (no blind audit means no regression detection).

### 5.3 — Loop
Update `bus/<flow>/state.md`. Summarize to user with the lightweight eval verdict. Ask: continue / specific task / switch to supervisor / done.

---

## STAGE 5: LOG + BLIND RE-AUDIT (Supervisor)

### 5.1 — Log
Write history (same format as router). Append to `bus/<flow>/log.md`.

### 5.2 — Blind Audit
Dispatch **<flow>-auditor** agent. Tell it cycle number and output path. **Do NOT leak what was changed.**

### 5.3 — Evaluate
Dispatch **<flow>-evaluator** agent. Tell it paths to:
1. Previous audit: `bus/<flow>/audit/cycle-(N-1).md`
2. Current blind audit: `bus/<flow>/audit/cycle-N.md`
3. History: `bus/<flow>/history/cycle-N.md`

### 5.4 — Decision

Read `bus/<flow>/evaluation/latest.md`:

**POSITIVE** → Summarize resolved items (confirmed by blind audit). Ask user: continue / task / done.

**STALE** → Count consecutive stale (check supervisor logs).
- 1st: Re-dispatch planner: "Previous approach ineffective — blind audit still finds same issue. Try different strategy." → Stage 2.
- 2nd consecutive: HALT. "These issues persist despite two attempts: [list]."

**REGRESSION** → HALT. "Fixes caused new issues: [list with causal links]."

**DIVERGING** → HALT. "Finding count increasing: [trend]. Each cycle creates more problems."

Log decision to `bus/<flow>/supervisor/cycle-NNN.md`. Append verdict to `bus/<flow>/log.md`. Update `bus/<flow>/state.md`.

---

## Supervisor Checkpoint Log

Write to `bus/<flow>/supervisor/cycle-NNN.md`:

```
# Cycle NNN — Supervisor Log — <date>

## Checkpoint 1 (Post-Discovery)
- Findings: <count> | Completeness: PASS | <issues>

## Checkpoint 2 (Post-Analysis)
- Coverage: PASS | <gaps>

## Checkpoint 3 (Post-Plan)
- Items: <count> | Order: VALID | <violations>

## Checkpoint 4 (Post-Execute)
- Implemented: <N/M> | Missing: <list>

## Checkpoint 5 (Post-Verify)
- Results: PASS | FAIL (attempt N)

## Checkpoint 6 (Post-Evaluation)
- Verdict: <verdict> | Resolved: <count> | Regressions: <count>
- Trend: <direction> | Decision: <CONTINUE/HALT/RETRY>
```

---

## Error Recovery

- Unusable agent output → re-dispatch once with specifics
- Checkpoint fails → re-dispatch responsible agent
- Verify fails 3x → halt, present errors
- STALE 2x → halt, report persistent issues
- REGRESSION → halt immediately
- DIVERGING → halt immediately
- "undo"/"revert" → confirm, `git checkout .`
- Bus file missing → re-dispatch agent that writes it
```
