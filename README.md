# Agentic PDCA — Self-Driving Engineering Sprints

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code) that generates multi-agent improvement flows for any codebase. It adapts the Deming cycle (Plan-Do-Check-Act) for AI agent orchestration — turning a single command into a self-driving engineering sprint where specialized agents audit, plan, implement, and verify changes while a human steers strategy.

---

## The Flow Architecture

### What Problem This Solves

A single AI agent trying to improve a codebase faces compounding problems: it runs out of context holding audit findings, analysis, plans, and code diffs simultaneously. It can't verify its own work objectively. It has no memory of what it already tried. And it has no mechanism to detect when it's going in circles.

This architecture solves all of that by splitting the work across isolated agents that communicate through files, not through the orchestrator's context window.

### The PDCA Cycle

Every flow runs the same fundamental loop — a Plan-Do-Check-Act cycle adapted for multi-agent orchestration:

```
  ┌──── PLAN ─────┐
  │  0. Discover   │  ← What needs doing?
  │  1. Analyze    │  ← What files are involved? What depends on what?
  │  2. Plan       │  ← What changes, in what order?
  ├──── DO ────────┤
  │  3. Execute    │  ← Make the changes
  ├──── CHECK ─────┤
  │  4. Verify     │  ← Does it build? Does it pass?
  ├──── ACT ───────┤
  │  5. Evaluate   │  ← Did it actually work? Are we making progress?
  └───────┬────────┘
          ↓
    Human steers → Next cycle
```

Each cycle changes the codebase. The next cycle's audit sees different things. Over iterations, the system converges toward zero findings — or halts if it detects it's not making progress.

### Stage-by-Stage Breakdown

**Stage 0 — Discover.** A discovery agent audits the entire codebase for actionable improvements. It categorizes findings by dimension (gaps, quality, security, infrastructure, completeness), assigns severity and effort estimates, and writes a prioritized list to the bus. The orchestrator formats this as a menu and the user picks what to work on. This is the only point where the user makes a strategic decision — everything else is automated. If the user already knows what they want, they pass it as an argument and this stage is skipped entirely.

**Stage 1 — Analyze.** An analyst agent reads the chosen target and the audit context, then produces two outputs: a list of every file that needs to change (with reasons), and a dependency graph showing which files depend on which. This determines the blast radius and the safe implementation order. The orchestrator needs both before it can plan.

**Stage 2 — Plan.** A planning agent reads the analysis and dependency graph, then produces an ordered checklist of changes — upstream layers first (types, schemas), downstream layers last (UI, pages). Each item has an exact file path, what to modify, and why. Ambiguous decisions are flagged for the user before coding begins.

**Stage 3 — Execute.** The orchestrator dispatches coding agents in phases following the dependency order from the plan. Independent layers can run in parallel. Dependent layers are sequential — upstream changes must complete before downstream work begins. No single coding agent touches the entire stack. Each gets a scoped assignment.

**Stage 4 — Verify.** The orchestrator runs the project's build, type-check, and lint commands. If anything fails, it dispatches a coding agent to fix the errors and re-runs verification. After three failed attempts, it halts and surfaces the errors to the user.

**Stage 5 — Evaluate.** This is where the architecture diverges from a naive loop. Instead of just asking "did it build?", the system evaluates whether the cycle made *real progress*. In supervisor mode, this involves a blind re-audit and cross-referencing (detailed below). In router mode, it logs what happened and asks the user what to do next.

---

## Core Design Decisions

### 1. Progressive Narrowing

Each stage reduces scope through a funnel from ambiguity to precision:

```
Stage 0: "What could we do?"        → entire codebase
Stage 1: "What files are involved?" → subset of files
Stage 2: "What changes, in order?"  → ordered checklist
Stage 3: "Write this exact code"    → single file at a time
```

This prevents any agent from being overwhelmed by context. The discovery agent sees the whole project but only produces a summary. The coding agent sees one file but has the full plan for context. No agent needs to hold everything simultaneously.

### 2. The File Bus

Agents don't talk to each other. They communicate through files on disk.

```
.claude/bus/<flow>/
    ├── audit/cycle-NNN.md        ← Discovery agent writes findings
    ├── analysis/
    │   ├── affected-files.md     ← Analyst writes file list
    │   └── link-tree.md          ← Analyst writes dependency graph
    ├── plan/current-plan.md      ← Planner writes ordered checklist
    ├── history/cycle-NNN.md      ← Orchestrator logs what was changed
    ├── evaluation/latest.md      ← Evaluator writes verdict
    └── supervisor/cycle-NNN.md   ← Orchestrator writes checkpoint log
```

**Why files instead of passing data through the orchestrator?**

Without the bus, the orchestrator would need to receive the full audit output (thousands of tokens) into its context, then copy it into the analysis agent's prompt, receive the analysis, copy it into the planner's prompt, and so on. Each stage doubles the orchestrator's context. By stage 5, it's holding everything from every previous stage.

With the bus, the orchestrator says "write your output to `bus/audit/cycle-001.md`" and tells the next agent "read `bus/audit/cycle-001.md` for context." The orchestrator never holds the full content. It stays lean — dispatching, reading just enough to validate, and dispatching again.

This also gives you:
- **Persistence** — Results survive across sessions. Resume tomorrow where you left off.
- **Auditability** — Every intermediate result is inspectable on disk.
- **Decoupling** — Any agent can be replaced, retried, or run independently. They depend on files existing, not on who wrote them.

### 3. Agent Isolation

Each agent is a standalone file with a focused job and constrained tool permissions:

| Agent | Job | Can Do | Cannot Do |
|-------|-----|--------|-----------|
| Discovery | Audit the codebase | Read, Search | Write code |
| Analyst | Map affected files + deps | Read, Search | Write code |
| Planner | Design ordered changes | Read, Search | Write code, Run commands |
| Coder | Implement changes | Read, Search, Write, Edit | Spawn other agents |
| Auditor | Blind re-audit | Read, Search | Write code |
| Evaluator | Judge cycle progress | Read, Search | Write code, Run commands |

Discovery agents can't accidentally modify code. Coding agents can't spawn sub-agents that escape their scope. The planner can't run commands — it can only design the plan. This constraint-based isolation prevents agents from overstepping and makes the system predictable.

### 4. Dependency-Aware Execution

Every codebase has layers that depend on each other. A change to a shared type must land before the component that uses it. The flow respects this by:

1. The analyst traces import/export chains and builds a dependency graph
2. The planner groups changes into phases based on that graph
3. The orchestrator dispatches coding agents phase-by-phase — upstream first, downstream last
4. Independent phases can parallelize; dependent phases are sequential

```
Types ──→ Schema ──→ Actions ──→ Pages
                                   ↑
Components ────────────────────────┘
```

This mirrors how a real engineering team parallelizes work — frontend and backend can start simultaneously, but integration waits for both.

### 5. Human Steering, Not Human Labor

The user makes strategic decisions: *what* to work on, resolving ambiguity when the planner flags decision points, and deciding when to stop. They never pick files, write code, debug build errors, or manage the agent pipeline. The orchestrator handles all operational work.

The cycle presents a menu of findings. The user picks a number. Everything else is automated until the next decision point or the cycle completes.

---

## The Blind Audit Protocol

This is the most important design decision in the architecture. It's what prevents the system from spinning its wheels or silently regressing.

### The Problem With Self-Evaluation

If you tell an agent "we fixed the N+1 query in bookings.ts" and then ask it to audit the codebase, it will find the fix and move on without deeply verifying it. It has confirmation bias — it knows the intent, so it sees the intent as fulfilled. This is the same reason peer review exists in science: the person who did the work is the worst person to evaluate it.

### The Solution: Blind Re-Audit

After each cycle completes, the system runs a two-phase evaluation:

**Phase 1 — Blind Audit.** A fresh auditor agent examines the codebase as if seeing it for the first time. It is explicitly forbidden from reading the history log or evaluation files. It doesn't know what was changed, what was targeted, or what the previous audit found. It produces an honest, unbiased list of findings.

**Phase 2 — Cross-Reference.** A separate evaluation agent receives three inputs:
1. The previous audit (what was wrong before)
2. The blind audit (what's wrong now, from an unbiased perspective)
3. The history log (what the cycle claims to have fixed)

It then performs five analyses:

- **Resolution check** — The history says X was fixed. Does the blind audit confirm X is gone? If the blind auditor still flags it, the fix was ineffective.
- **Persistence check** — Which findings appear in both audits? These were either not targeted or the fix didn't work.
- **Regression check** — Are there *new* findings in the blind audit that trace back to files changed in the history? A fix broke something else.
- **Novel findings** — New findings unrelated to recent changes. Pre-existing issues the previous auditor missed.
- **Trend analysis** — Across all cycles, is the finding count decreasing (converging), flat (stagnating), or increasing (diverging)?

### Verdicts

The evaluation produces one of four verdicts:

| Verdict | What It Means | What Happens |
|---------|--------------|--------------|
| **POSITIVE** | Blind audit confirms fixes worked, no regressions | Continue with confidence |
| **STALE** | Same findings persist despite claimed fixes | Retry with different approach, halt after 2nd consecutive stale |
| **REGRESSION** | Fixes caused new problems traceable to changed files | Halt immediately, report causal links |
| **DIVERGING** | Finding count increasing across cycles | Halt immediately, report trend data |

The blind audit is the circuit breaker. Without it, the orchestrator would loop forever — repeatedly "fixing" the same issues or creating cascading regressions with no mechanism to detect either failure mode.

---

## Flow Types

Not all work is the same. Fixing a bug requires different evaluation criteria than adding a feature or closing a security vulnerability. The flow type determines what agents look for and what "success" means.

### Bugfix

The goal is to find incorrect behavior and correct it at the root cause — not mask the symptom.

- **Discovery** focuses on: logic errors, crash paths, data corruption, broken workflows, error handling gaps, race conditions
- **Evaluation** checks: Is the root cause gone from the code, or was only the symptom addressed? Did the fix handle edge cases? Did it break related code paths that shared the same logic?
- **POSITIVE** = root cause removed, no related breakage
- **STALE** = symptom gone but root cause persists (e.g., a retry masking a race condition)
- **REGRESSION** = fix broke a different code path

### Feature

The goal is not just to make something compile — it's to reach a state where a new capability is fully integrated across all layers with clean, production-ready code.

- **Discovery** focuses on: stubs, TODO/FIXME comments, skeleton pages, mock/simulated functionality, incomplete wiring, forms without validation, missing loading/error states
- **Evaluation** checks: Does the feature exist end-to-end? All layers wired? Any dangling references, placeholder code, or junk code (commented-out blocks, unused imports)? Is it production-grade — error states, loading states, strong typing?
- **POSITIVE** = fully integrated, clean, production-ready
- **STALE** = partially wired — the component exists but isn't connected to data, or the route exists but has no validation
- **REGRESSION** = feature code broke existing functionality

### Security

The goal is to eliminate vulnerabilities without breaking legitimate functionality. A surface-level patch doesn't count.

- **Discovery** focuses on: unprotected routes, missing auth checks, input sanitization gaps, injection vectors, exposed secrets, missing CSRF/CORS, rate limiting, secrets in client bundles
- **Evaluation** checks: Is the attack surface actually closed, or was it a surface-level patch (client-side check without server-side)? Did the fix break legitimate functionality? Did it create new attack surfaces?
- **POSITIVE** = vulnerability eliminated, no new attack surfaces, no broken functionality
- **STALE** = vulnerability persists or was only surface-patched (e.g., added middleware check but the action itself is still unprotected)
- **REGRESSION** = security fix broke legitimate functionality (users can't log in, valid inputs rejected)

---

## Two Operating Modes

The generated orchestrator skills accept a mode argument at runtime that controls how much validation happens between stages.

### Router — Lightweight Dispatch

The orchestrator dispatches agents, confirms bus files exist, and presents decisions to the user. No validation between stages. Minimal token cost. Good for fast iteration when you trust agents to produce complete outputs and just need coordination.

```
/feature-flow router
```

### Supervisor — Validated Dispatch

Same dispatch pattern, plus the orchestrator reads and validates every bus file between stages:

- **Post-Discovery:** Are findings complete? All required fields present? File paths real? Severities reasonable?
- **Post-Analysis:** Do analyzed files match the audit findings? Any files missing from the analysis that the audit mentioned?
- **Post-Plan:** Does the plan cover all files from the analysis? Does the order respect the dependency graph? Are all items actionable with specific file paths (not "update relevant files")?
- **Post-Execute:** Did the actual code changes match the plan? Any plan items not implemented? Any files modified that weren't in the plan?
- **Post-Verify:** Build gates pass?
- **Post-Evaluation:** Cycle making progress or looping?

The supervisor catches information loss at every handoff. Agents silently drop details at every boundary — the analyst misses a file the auditor mentioned, the planner drops two files the analyst identified, the coder skips a plan item. Without the supervisor, these losses compound silently. With it, they're caught and corrected before they propagate.

Supervisor mode also runs the full blind audit protocol (described above) — router mode skips it and just logs + loops.

```
/feature-flow supervisor
```

---

## The Meta-Pattern

Zoom out and the architecture is a self-driving engineering sprint. The orchestrator plays the role of a tech lead who:

- Runs a code review (discovery)
- Presents findings to the product owner (user picks from menu)
- Delegates to specialists (analyst, planner, coder agents)
- Runs CI (verify)
- Holds a retro (blind audit + evaluation)
- Starts the next sprint

The reusable pattern: **autonomous discovery → human gate → automated execution → verification → evaluation → loop.**

Each cycle makes the codebase different. The next audit reflects those changes. Over iterations, findings decrease and severity drops. The system converges — or the blind audit detects stagnation and halts before wasting effort.

---

## How the Skill Creates Flows

The `flow-creator` skill is what brings this architecture to life for a specific project. It's a generator — you run it once per project, and it produces everything described above, tailored to your codebase.

### What It Does

1. **Analyzes the project** — Launches an exploration agent that maps the dependency layers (types → schema → logic → API → UI → pages), discovers build/verification commands, traces the directory structure, and identifies coding conventions.

2. **Creates agent files** — For each requested flow (bugfix, feature, security), it writes 6 agent files to `.claude/agents/`, each with:
   - The project's actual layer names and directory paths (not generic placeholders)
   - The project's tech stack and conventions baked in
   - Flow-type-specific audit dimensions and evaluation criteria
   - Appropriate tool permissions in YAML frontmatter

3. **Creates orchestrator skills** — For each flow, it writes an orchestrator skill to `.claude/skills/<flow>-flow/SKILL.md` that references its 6 agents by exact file name, knows its bus namespace, contains the full PDCA stage logic, and includes the mode switch (router/supervisor).

4. **Creates bus directories** — Sets up `.claude/bus/<flow>/` with subdirectories for each stage's output.

### What Gets Generated

```
.claude/
├── agents/
│   ├── bugfix-discovery.md       ← Knows your stack, audits for logic errors
│   ├── bugfix-analyst.md         ← Knows your layers, maps affected files
│   ├── bugfix-planner.md         ← Knows your deps, designs ordered plan
│   ├── bugfix-coder.md           ← Knows your conventions, implements changes
│   ├── bugfix-auditor.md         ← Blind re-audit, forbidden from reading history
│   ├── bugfix-evaluator.md       ← Cross-references blind audit vs history
│   ├── feature-discovery.md
│   ├── feature-analyst.md
│   ├── feature-planner.md
│   ├── feature-coder.md
│   ├── feature-auditor.md
│   ├── feature-evaluator.md
│   ├── security-discovery.md
│   ├── security-analyst.md
│   ├── security-planner.md
│   ├── security-coder.md
│   ├── security-auditor.md
│   └── security-evaluator.md
├── skills/
│   ├── bugfix-flow/SKILL.md      ← Orchestrator: dispatches bugfix-* agents
│   ├── feature-flow/SKILL.md     ← Orchestrator: dispatches feature-* agents
│   └── security-flow/SKILL.md    ← Orchestrator: dispatches security-* agents
└── bus/
    ├── bugfix/{audit,analysis,plan,history,evaluation,supervisor}/
    ├── feature/{audit,analysis,plan,history,evaluation,supervisor}/
    └── security/{audit,analysis,plan,history,evaluation,supervisor}/
```

### Why Generate Instead of Use Directly?

A generic flow that works on "any project" would need to rediscover the project's architecture at the start of every cycle — wasting tokens and context on the same exploration each time. By generating project-specific agents once, every cycle starts with agents that already know the stack, paths, layers, and conventions. The exploration cost is paid once during generation, then amortized across all future cycles.

The generated agents are also more effective because their audit dimensions and evaluation criteria are scoped to the project. A security discovery agent for a Next.js app knows to look for unprotected server actions and exposed API keys in client bundles. A generic agent would check a broad, unfocused list.

---

## Installation

### Global (available in all projects)

```bash
mkdir -p ~/.claude/skills/flow-creator/references
cp skills/flow-creator/SKILL.md ~/.claude/skills/flow-creator/SKILL.md
cp skills/flow-creator/references/*.md ~/.claude/skills/flow-creator/references/
```

### Per-project

```bash
mkdir -p .claude/skills/flow-creator/references
cp skills/flow-creator/SKILL.md .claude/skills/flow-creator/SKILL.md
cp skills/flow-creator/references/*.md .claude/skills/flow-creator/references/
```

## Usage

```bash
# Generate flows for your project
/flow-creator all                    # Create bugfix, feature, and security flows
/flow-creator bugfix                 # Create only the bugfix flow
/flow-creator feature,security       # Create specific flows

# Use the generated flows
/bugfix-flow router                  # Lightweight bugfix cycle
/feature-flow supervisor             # Thorough feature cycle with blind re-audits
/security-flow supervisor            # Security hardening with regression detection
/feature-flow router add payments    # Skip audit, implement a specific feature
```

## Project Structure

```
flow-creator-skill/
├── skills/
│   └── flow-creator/
│       ├── SKILL.md                    # The generator skill
│       └── references/
│           ├── agent-templates.md      # Agent file structures
│           └── orchestrator-template.md # Orchestrator skill structure
├── README.md
├── LICENSE
└── .gitignore
```

## License

Apache 2.0 — see [LICENSE](LICENSE).
