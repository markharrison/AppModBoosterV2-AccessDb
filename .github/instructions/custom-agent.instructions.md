---
description: "How to scaffold high-quality custom agent definition files for GitHub Copilot"
applyTo: "**/*.agent.md"
---

# Custom Agent Definition Guide

How to create well-structured, maintainable custom agent files for GitHub Copilot.
This guide covers three agent archetypes — Agent, Orchestrator, and Subagent — with
conventions for frontmatter, body structure, state management, and quality gates.

Official docs: <https://docs.github.com/en/copilot/how-tos/use-copilot-agents>

---

## Agent Archetypes

Every agent falls into one of three archetypes. Choose the right one before writing anything.

|                         |        Agent        |      Orchestrator      |           Subagent           |
| ----------------------- | :-----------------: | :--------------------: | :--------------------------: |
| **Purpose**             | Self-contained task | Coordinates a pipeline |  Executes one pipeline step  |
| **User-invocable**      |       ✅ Yes        |         ✅ Yes         |            ❌ No             |
| **Delegates work**      |         No          |   Yes (to subagents)   |              No              |
| **Writes state files**  |  ✅ Yes (progress)  |  Yes (pipeline state)  |     Yes (step output MD)     |
| **Writes deliverables** |      Optional       |           No           |           Optional           |
| **Owns approval gates** |         No          |          Yes           |              No              |
| **Maintains state**     |  Yes (checkpoints)  | Yes (via state folder) |              No              |
| **Has handoffs**        |      Optional       |           No           |              No              |
| **Location**            |  `.github/agents/`  |   `.github/agents/`    | `.github/agents/_subagents/` |

### When to use each

- **Agent**: A single agent that does one job end-to-end. No pipeline, no
  coordination. **REQUIRED:** Write a state file to `./output/state/00-{agent-name}.md`
  at startup (before Phase 1) with all phases unchecked. Update it after each phase.
- **Orchestrator**: A multi-step workflow where each step needs a specialist
  agent, human approval between steps, and the ability to resume, revert,
  or stop. **REQUIRED:** Write `00-{orchestrator-name}.md` to track pipeline state, parameters,
  and step checkboxes.
- **Subagent**: A specialist that handles one step in an orchestrator's pipeline. Never
  invoked directly by users. **REQUIRED:** Write a step state file (`{NN}-{step}.md`)
  at startup with all phases unchecked. Update it progressively after each phase.

---

## Frontmatter

Every `.agent.md` file starts with YAML frontmatter between `---` markers.
Use spaces (no tabs). Keep keys simple and consistent.

### Required Fields — ALL Archetypes

```yaml
---
name: "{Display Name}"
description: "{1-2 sentences: what it does and what it does NOT do}"
model: ["{model-name}"]
tools: ["{tool-ids}"]
---
```

| Field         | Rules                                                                                              |
| ------------- | -------------------------------------------------------------------------------------------------- |
| `name`        | Title Case, 1-3 words. Stable — renames confuse users and docs.                                    |
| `description` | 50-150 characters. State scope AND boundaries.                                                     |
| `model`       | Array format: `["Claude Opus 4.6"]`. See Model Selection below.                                    |
| `tools`       | Explicit list of tool IDs. See Tool Requirements below. Omit = all tools enabled. `[]` = no tools. |

### Archetype-Specific Fields

**Agent:**

```yaml
user-invocable: true # project convention — see note below
agents: [] # or ["*"] if it needs to call other agents
```

**Orchestrator:**

```yaml
user-invocable: true
agents: ["*"] # must be able to invoke subagents
argument-hint: "Describe what you want to build end-to-end"
```

**Subagent:**

```yaml
user-invocable: false
agents: []
```

### Tool Requirements by Archetype

**All agents write state files and therefore require file editing tools.**

Minimum required tools:

**Agent:**

```yaml
tools: ["read", "edit"] # edit required for state file + optional deliverables
```

**Orchestrator:**

```yaml
tools: ["read", "edit", "agent"] # edit for pipeline state, agent for subagent invocation
```

**Subagent:**

```yaml
tools: ["read", "edit"] # edit required for step state file + optional deliverables
```

Add additional tools based on the agent's specific needs:

- `"search"` — if the agent needs to search code/files
- `"execute"` — if the agent runs terminal commands
- `"web"` — if the agent fetches web content
- `"azure-mcp/*"` — if the agent uses Azure MCP tools

**Common mistake**: Omitting `"edit"` from the tools list, which prevents the agent from writing required state files.

See **Tool ID Format** section below for detailed tool documentation.

### `agents` Field Values

The `agents` field controls which other agents this agent can invoke:

| Value                                  | Meaning                                    |
| -------------------------------------- | ------------------------------------------ |
| `[]`                                   | Cannot invoke any agents (default, safest) |
| `["*"]`                                | Can invoke any agent in the repo           |
| `["Research Agent", "Analysis Agent"]` | Can invoke only the named agents           |

Prefer explicit agent names over `["*"]` when the agent only needs
access to a known set — principle of least privilege applies.
Use `["*"]` only for orchestrators that must invoke all pipeline subagents.

### Model Selection

Match model capability to task complexity. Use concrete model names in frontmatter:

| Task Type                  | Recommended Model   | Rationale                                     |
| -------------------------- | ------------------- | --------------------------------------------- |
| Planning / reasoning       | `Claude Opus 4.6`   | Accuracy matters — bad plans cost more to fix |
| Implementation / execution | `Claude Sonnet 4.5` | Speed for code generation and tool use        |
| Lightweight / utility      | `Claude Haiku 4.5`  | Balance of speed and quality                  |

**Current recommendations as of Feb 2026. Review as new models become available.**

Guidelines:

- Use the most capable model the task _needs_ — not the most expensive available
- Planning tasks (orchestrators, architecture) benefit from strongest reasoning
- Implementation tasks (code generation, analysis) balance speed and quality
- Lightweight tasks (formatting, simple validation) can use faster models
- Document model choices in the agent's frontmatter comment if non-obvious
- Update this table when significantly better models are released

### Other Fields (Situational)

| Field         | When to Use                                                              |
| ------------- | ------------------------------------------------------------------------ |
| `target`      | Set to `'vscode'` or `'github-copilot'` if agent is environment-specific |
| `metadata`    | Name-value pairs for annotation (GitHub.com only)                        |
| `mcp-servers` | Configure MCP servers for org/enterprise-level agents only               |

---

## Agent Body Skeleton

The markdown content below frontmatter defines the agent's behavior. Structure varies
by archetype. Use this matrix to determine which sections to include:

| Body Section                                     |    Agent    | Orchestrator |  Subagent   |
| ------------------------------------------------ | :---------: | :----------: | :---------: |
| `# {Agent Name}` + role line                     |     ✅      |      ✅      |     ✅      |
| `## Context` / `## MANDATORY: Read Skills First` |     ✅      |      ✅      |     ✅      |
| `## DO / DON'T`                                  |     ✅      |      ✅      |     ✅      |
| `## Workflow` (phased internal work)             |     ✅      |      ❌      |     ✅      |
| `## Pipeline Definition`                         |     ❌      |      ✅      |     ❌      |
| `## State Management`                            |  Optional   |      ✅      |     ❌      |
| `## Gate Model`                                  |     ❌      |      ✅      |     ❌      |
| `## Subagent Delegation`                         |     ❌      |      ✅      |     ❌      |
| `## Prerequisites Check`                         |     ❌      |      ❌      |     ✅      |
| `## Output Files`                                | RECOMMENDED |      ✅      |     ✅      |
| `## Validation Checklist`                        |     ✅      |      ✅      |     ✅      |
| `## Error Handling`                              | RECOMMENDED | RECOMMENDED  | RECOMMENDED |

### Role Line

Every agent opens with a heading and a one-line role statement:

**Agent:**

```markdown
# {Agent Name}

{Brief role description — what this agent does.}
```

**Orchestrator:**

```markdown
# {Agent Name}

Master orchestrator for the {N}-step {domain} workflow.
```

**Subagent:**

```markdown
# {Agent Name}

**Step {N}** of the {N}-step workflow: `step-1 → step-2 → ... → [this-step] → ... → step-N`
```

The workflow position line uses `[brackets]` to highlight the current step.

---

## Context / Skills-First Pattern

### `ALL` — Read Before Working

Every agent should declare what it needs to read before doing any work.

**Agent** — use `## Context`:

```markdown
## Context

Before starting, read:

1. `path/to/relevant-config.md` — configuration and conventions
2. `path/to/reference-docs.md` — domain reference material
```

**Orchestrator and Subagent** — use `## MANDATORY: Read Skills First`:

```markdown
## MANDATORY: Read Skills First

**Before doing ANY work**, read these skills:

1. **Read** `.github/skills/{skill-name}/SKILL.md` — {what it provides}
2. **Read** `.github/skills/{other-skill}/SKILL.md` — {what it provides}
3. **Read** `.github/skills/{artifacts-skill}/templates/{NN}-{artifact}.template.md`

These skills are your single source of truth for conventions, defaults,
and template structures. Always read them at runtime — do not copy their
values into the agent body.
```

The numbered list format and closing statement are mandatory for orchestrated pipelines.
Skills contain shared knowledge (naming conventions, defaults, template structures) that
prevent drift between agents.

---

## DO / DON'T Conventions — `ALL`

Every agent MUST include a `## DO / DON'T` section with `### DO` and `### DON'T`
sub-headings using emoji bullets.

```markdown
## DO / DON'T

### DO

- ✅ {Positive instruction — what the agent should do}
- ✅ {Another positive instruction}

### DON'T

- ❌ {Negative instruction — what the agent must NOT do}
- ❌ {Another negative instruction}
```

### Must-Include Items

**All agents:**

- ✅ Scope boundaries: what this agent does
- ✅ Include `"edit"` in the `tools` frontmatter field — required for writing state files
- ❌ Anti-scope: what this agent does NOT do (explicit boundary with other agents)
- ❌ Omit `"edit"` from tools — all agents write state files and need file editing capability

**Agent (additional):**

- ✅ **STATE FILE REQUIRED:** Create `./output/state/00-{agent-name}.md` immediately on startup — BEFORE Phase 1 begins — with all phases unchecked.
- ✅ Update the state file after each phase completes — check off that phase with an inline summary. Progressive updates, not end-only.
- ✅ State file must include a `## Phases` checkbox list that mirrors the `## Workflow` phases exactly.
- ❌ Write the state file only at the end — create it first, update it progressively
- ❌ Skip the state file — all agents write state files
- ❌ Leave the state file untouched after startup — update it as each phase completes

**Subagents (additional):**

- ✅ Create the state file immediately on startup — before Phase 1 begins — with all phases unchecked
- ✅ Update the state file after each phase — check it off with an inline summary
- ✅ Save state files to `./output/state/`
- ✅ Save deliverables (code, templates) to `./output/` subfolders (e.g., `./output/src/`, `./output/infra/`)
- ✅ Match template headings exactly (if templates are defined)
- ❌ Do work outside this agent's pipeline step
- ❌ Write the state file only at the end — create it first, update it progressively
- ❌ Modify `00-{orchestrator-name}.md` — that file is the orchestrator's exclusive domain; subagents write only their own `{NN}-{step}.md`

**Orchestrator (additional):**

- ✅ Delegate all work to subagents
- ✅ Apply the configured gate type per step (`Approval` or `Auto`)
- ❌ Do the work directly — always delegate
- ❌ Skip gates or auto-approve

---

## Prerequisites Check — `SUBAGENT`

Subagents validate that the previous step's state files exist before starting work.

```markdown
## Prerequisites Check

Before starting, validate these state files exist in `./output/state/`:

- `{NN}-{previous-step}.md` — {what it contains}

If missing, STOP and report to the orchestrator.
```

The STOP pattern is critical — a subagent must never proceed with missing inputs.
The orchestrator handles the error (re-run previous step or ask user).

---

## Workflow — `AGENT` / `SUBAGENT`

Structure internal work as numbered phases. Even simple agents benefit from clear phases.

```markdown
## Workflow

### Phase 1: {Name}

{Description of what happens in this phase.}

### Phase 2: {Name}

{Description of what happens in this phase.}

### Phase 3: {Name}

{Description of what happens in this phase.}
```

Use `## Workflow` as the heading (not `## Core Workflow` or other variants — standardize).
Each phase should be a discrete unit of work with clear inputs and outputs.

### Checkpoint Rule

**Agents and subagents MUST create their state file immediately on startup** — before
beginning any phase work. Write the file to `./output/state/` with all phases
listed as unchecked (`- [ ]`). This gives users (and the orchestrator) live
visibility that the agent is running, even before it produces any output.

After completing each phase, update the state file — check off that phase and
append a brief inline summary. The checkpoint list must mirror the phases defined
in `## Workflow`.

**State file lifecycle:**

1. **On start**: create the file with all phases unchecked — do this before Phase 1 begins
2. **After each phase**: update the file — check off the completed phase with an inline summary
3. **Never**: write the state file only at the end — that defeats live visibility

---

## Pipeline Definition — `ORCHESTRATOR`

The orchestrator defines the full pipeline as a step table:

```markdown
## Pipeline Definition

| Step | Subagent     | State File       | Deliverables (optional)  | Gate     | On Fail (optional)            |
| ---- | ------------ | ---------------- | ------------------------ | -------- | ----------------------------- |
| 1    | {Agent Name} | `{NN}-{step}.md` | —                        | Approval | Stop                          |
| 2    | {Agent Name} | `{NN}-{step}.md` | `./output/src/{files}`   | Auto     | Retry once                    |
| 3    | {Agent Name} | `{NN}-{step}.md` | —                        | Approval | Back to Step 2 ({Agent Name}) |
| N    | {Agent Name} | `{NN}-{step}.md` | `./output/infra/{files}` | —        | Stop                          |
```

Rules:

- Number state files with zero-padded step numbers: `01-`, `02-`, etc.
- All state files go to `./output/state/`
- Deliverables go to `./output/` subfolders (e.g., `./output/src/`, `./output/infra/`)
  — never in `./output/state/`
- Gate values:
  - `Approval`: run validation, then require human decision
  - `Auto`: run validation, then proceed automatically if validation passes
  - `—`: no gate (typically final step)
- `On Fail` values (optional):
  - `Retry once`: rerun current subagent one time
  - `Back to Step N ({Agent Name})`: send failure details to an earlier step
  - `Stop`: halt and ask user
- A step can produce multiple state files — list them comma-separated
  in the State File column and use a shared prefix:
  `02-analysis.md, 02-analysis-appendix.md`

### Multi-File State Steps

When a step produces multiple state files, use a naming convention
so the orchestrator can match them by prefix:

```text
{NN}-{step}.md            ← primary state file (always required)
{NN}-{step}-{suffix}.md   ← supplementary files (matched by {NN}- prefix)
```

Example pipeline row:

```markdown
| 2 | Analysis Agent | `02-analysis.md, 02-analysis-constraints.md` | — | Auto | Retry once |
```

### Common Coder/Tester Loop

Use this pattern when tests should bounce failures back to code generation:

```markdown
| Step | Subagent     | State File   | Deliverables       | Gate | On Fail                      |
| ---- | ------------ | ------------ | ------------------ | ---- | ---------------------------- |
| 2    | Coder Agent  | `02-code.md` | `./output/src/*`   | Auto | Retry once                   |
| 3    | Tester Agent | `03-test.md` | `./output/tests/*` | Auto | Back to Step 2 (Coder Agent) |
```

When Step 3 fails validation, orchestrator writes failure details, invokes Step 2,
then retries Step 3 after Step 2 succeeds.

If the same failure happens again after one bounce, stop and ask the user.

The orchestrator validates a step as complete when **all** listed state files
exist in `./output/state/`. The primary file is mandatory; supplementary files
are listed explicitly so the gate knows exactly what to check.

---

## State Files vs Deliverables

Agents produce two distinct types of output. Keep them strictly separated.

| Type             | What it is                                                      | Where it goes                                                     |
| ---------------- | --------------------------------------------------------------- | ----------------------------------------------------------------- |
| **State files**  | MD docs tracking progress — decisions, plans, checkpoints, logs | `./output/state/`                                                 |
| **Deliverables** | Actual work product — code, templates, configs, scripts         | `./output/` subfolders (e.g., `./output/src/`, `./output/infra/`) |

### Why separate?

- Both live under `./output/` in separate subfolders, preserving clean reset semantics:
  - Delete `./output/state/` alone to reset pipeline position while keeping deliverables.
  - Delete `./output/` entirely for a full reset of both state and deliverables.
- State tracks _what was decided and what has run_ — it is ephemeral and safe to delete independently.
- Deliverables are the actual work product — they survive a state-only reset.

> **Everything goes under `./output/`.** State → `./output/state/`; deliverables → their own
> subfolders (e.g., `./output/src/`, `./output/tests/`). New deliverable types get a new subfolder
> under `./output/` — never a new top-level folder.

### Agent Checkpoints

**Agents MUST write a state file at startup (before Phase 1 begins)** with all phases
listed as unchecked. After each phase completes, update the file — check off that phase
and append a brief summary inline. This gives users live visibility into progress and
enables resuming if work is interrupted.

State file location: `./output/state/00-{agent-name}.md`

File structure:

```text
# {Agent Name} Progress

> Generated by {agent-name} | {YYYY-MM-DD HH:MM:SS}

## Phases

- [x] Phase 1: {Name} — {summary of what was done}
- [x] Phase 2: {Name} — {summary of what was done}
- [ ] Phase 3: {Name}
```

Create the file **before beginning Phase 1**, with all phases unchecked. After
completing each phase, update the file — check off that phase and append a
brief summary inline. Do not wait until the end — create it first for immediate
visibility into agent progress.

An agent state file is a single MD file (not step-numbered) named after the agent:

```text
./output/state/{agent-name}-progress.md
```

The file uses a checkbox list matching the agent's workflow phases:

```text
# {Agent Name} Progress

> Generated by {agent-name} | {YYYY-MM-DD HH:MM:SS}

## Phases

- [x] Phase 1: {Name} — {summary of what was done}
- [x] Phase 2: {Name} — {summary of what was done}
- [ ] Phase 3: {Name}
```

Create the file **before beginning Phase 1**, with all phases unchecked. After
completing each phase, update the file — check off that phase and append a
brief summary inline. Do not wait until the end — create it first for immediate
visibility into agent progress.

### Orchestrator Pipeline State

Orchestrators MUST write a pipeline state file to track workflow-level context:

```text
./output/state/00-{orchestrator-name}.md
```

This file is distinct from step state files (`{NN}-{step}.md`) written by
subagents. It records parameters extracted from the user prompt, gate decisions,
and pipeline status. See **State Management → Pipeline State File** for the
full template and update rules.

The `00-` prefix ensures it sorts before all step files and is clearly
identifiable as the orchestrator's own state — not a step output.

### Rules

- **Never** put generated code, templates, or scripts in `./output/state/`
- **Never** put pipeline state or tracking files outside `./output/state/`
- **Always** include a `## Phases` checkbox list in every state file — this is
  the universal progress indicator across all agent archetypes
- State files reference deliverable locations so reverts know what to clean up
- If a step produces deliverables, the state file should list them under a
  `## Deliverables` heading so the orchestrator can track and revert them

---

## State Management — `ORCHESTRATOR` / `AGENT` (optional)

The orchestrator maintains pipeline state through the `./output/state/` folder
using two mechanisms: a **pipeline state file** (`00-{orchestrator-name}.md`) written by
the orchestrator itself, and **step state files** (`{NN}-{step}.md`) written
by subagents. Together these form the complete pipeline state. Agents can
optionally use a single state file for progress checkpoints.

### How State Works

```markdown
## State Management

Pipeline state is determined by scanning `./output/state/`:

1. On invocation, read `00-{orchestrator-name}.md` — restore parameters and step checkboxes
2. Find the first unchecked step — that is where the pipeline resumes
3. Cross-reference step state files to confirm they match checked steps
4. Resume from the first unchecked step (invoke its subagent or present its gate)

### Reset

To reset state only, delete `./output/state/`.
To reset everything (state + deliverables), delete `./output/`.

### Revert

When a user requests revert to a previous step:

1. Accept either the **step number** (`2`) or **step name** (`Analysis Agent`)
2. Resolve the user input to a step number using the pipeline table
3. Delete all state files in `./output/state/` with step prefixes >= resolved step number
4. Delete any deliverables produced by those steps (if tracked in the state files)
5. Resume by invoking the subagent for the resolved step
```

### Pipeline State File

The orchestrator creates `00-{orchestrator-name}.md` in `./output/state/` when the pipeline
starts and updates it after each gate decision. This file preserves context that
would otherwise be lost between conversation sessions:

| Section        | Purpose                                               |
| -------------- | ----------------------------------------------------- |
| **Pipeline**   | Orchestrator name, start date, status                 |
| **Parameters** | Values extracted from the user's original prompt      |
| **Steps**      | Checkbox list — checked off as each step completes    |
| **Retry Log**  | Retry and bounce-back attempts with outcomes (if any) |

Structure:

> **Timestamp rule:** All `{datetime}` placeholders must be the actual system date and time in
> `YYYY-MM-DD HH:MM:SS` format. Never use `00:00:00` or other placeholder times. If the exact time is
> not known, run `Get-Date -Format "yyyy-MM-dd HH:mm:ss"` (PowerShell) or `date "+%Y-%m-%d %H:%M:%S"` (bash) before writing the file.

```text
# Pipeline State

> Generated by {orchestrator-name} | {YYYY-MM-DD HH:MM:SS}

## Pipeline

- **Orchestrator**: {name}
- **Started**: {YYYY-MM-DD HH:MM:SS}
- **Status**: {see Status values below}

## Parameters

- **{key}**: {value extracted from user prompt}

## Steps

- [x] Step 1: {Subagent Name} — {Decision} ({YYYY-MM-DD HH:MM:SS}, {notes})
- [x] Step 2: {Subagent Name} — {Decision} ({YYYY-MM-DD HH:MM:SS}, {notes})
- [ ] Step 3: {Subagent Name} — 🔄 Running since {YYYY-MM-DD HH:MM:SS}
- [ ] Step 4: {Subagent Name}

## Retry Log

- Step 2, Attempt 1: {reason} → {resolved|failed}
```

### How checkboxes work

- All steps start unchecked (`- [ ]`) when the pipeline is created
- **Before invoking a subagent**, annotate that step with `— 🔄 Running since {datetime}` so
  live progress is visible in the state file
- The orchestrator checks off a step (`- [x]`) after its gate is decided; the `🔄 Running`
  annotation is replaced by the gate decision at that point
- The decision and metadata are appended inline: `— Approved (2026-02-22 10:30:00, user confirmed)`
- On **revert**, uncheck the reverted step and all steps after it, removing
  their inline decision text
- On **resume**, if a step shows `🔄 Running`, treat it as not started and re-invoke its subagent
- On **resume**, the first unchecked step (without a `🔄 Running` annotation) tells the
  orchestrator where to pick up — no scanning required
- `**Status**` is updated after every gate action:
  - `running - Step {N}: {Subagent Name}` while a subagent is actively running (set before invoking)
  - `paused - Approval gate: Step {N} ({Subagent Name})` while awaiting user decision at an Approval gate
  - `paused - stopped at Step {N}: {Subagent Name}` when the user chose Stop
  - `completed` when all steps are checked
  - `failed - Step {N}: {reason}` on unresolvable error

### Decision values for inline annotation

| Decision / Annotation         | When used                                                           |
| ----------------------------- | ------------------------------------------------------------------- |
| `🔄 Running since {datetime}` | Transient — written before subagent starts; replaced when gate runs |
| `Approved`                    | User approved at an `Approval` gate                                 |
| `Auto`                        | `Auto` gate passed validation automatically                         |
| `Revised`                     | User requested revision (step re-ran, then passed)                  |
| `Skipped`                     | Step was skipped (if pipeline supports it)                          |

The orchestrator checks off each step and updates `## Status` after each
gate decision. `## Parameters` are written once at pipeline start.
`## Retry Log` is appended only when retries or bounces occur — omit the
section entirely if no retries have happened.

Without `00-{orchestrator-name}.md`, the orchestrator can still determine position by
scanning step files — but it **loses** user parameters and gate decisions on
resume, forcing re-extraction and re-approval of already-decided gates.

### Resume Behavior

When the orchestrator is invoked and `./output/state/` already contains files:

1. Read `00-{orchestrator-name}.md` first — restore parameters and step checkboxes
2. Scan step-numbered files to confirm they match checked steps
3. Resume based on combined state:

- **Empty or missing**: Create `00-{orchestrator-name}.md` with all steps unchecked, start Step 1
- **Some checked, some unchecked**: Resume from the first unchecked step
- **All steps checked**: Present final summary, offer to regenerate or finish

The checkbox list in `00-{orchestrator-name}.md` prevents re-prompting the user for
decisions they already made in a previous session.

---

## Gate Model — `ORCHESTRATOR`

Gates run between steps based on each row's `Gate` value in the pipeline table.

- `Approval` gate: validation + human decision
- `Auto` gate: validation only; continue automatically on success
- `—` gate: no gate; used for terminal/final steps

### Phase 1: Automatic Validation (No Human Input)

The orchestrator checks programmatically:

1. **State file exists** — expected MD file(s) written to `./output/state/`
2. **Content validation** — required sections or headings are present (if defined)
3. **Deliverables exist** — if the step produces code/files, verify they were written
4. **Lint passes** — run any configured validation commands (optional)

If automatic validation fails → apply the step's `On Fail` policy.

### Validation Failure Routing

Keep this simple:

1. Write `./output/state/{NN}-{step}-failure.md` with the failing checks and key errors
2. Persist retry/bounce count in that same failure file (for example: `Retry-Attempt: 1`)
   so resume behavior is deterministic
3. Apply `On Fail` from the pipeline row
4. Allow only one automatic retry/bounce for the same failing step
5. If it still fails, stop and ask the user

This prevents endless loops without extra machinery.

### Phase 2: Human Decision (Only for `Approval` Gates)

Present a summary and four options:

```markdown
✅ Step {N} Complete: {Subagent Name}

State file: ./output/state/{NN}-{step}.md
Deliverables: {list any code/files produced, or "none"}

Summary: {brief description of what was produced}

Choose:
✅ Approve → proceed to Step {N+1}
✏️ Revise → re-run Step {N} with your feedback
⏪ Revert → specify step number or step name
🛑 Stop → save progress, halt pipeline

Step map:

1. {Subagent A}
2. {Subagent B}
3. {Subagent C}
```

For `Auto` gates, do not prompt the user. After Phase 1 passes, proceed directly
to the next step and log that the auto gate passed.

### Gate Actions

| Action         | Orchestrator Behavior                               |
| -------------- | --------------------------------------------------- |
| **✅ Approve** | Invoke next subagent                                |
| **✏️ Revise**  | Re-invoke subagent with user feedback appended.     |
|                | Subagent overwrites state file and deliverables.    |
|                | Gate runs again after completion.                   |
| **⏪ Revert**  | User specifies step number or step name. Resolve to |
|                | step number, delete downstream files, re-invoke it. |
| **🛑 Stop**    | Do nothing. State is preserved in                   |
|                | `./output/state/`. Next invocation resumes here.    |

For `Auto` gates, the orchestrator skips the action menu and proceeds
immediately after successful validation.

---

## Subagent Delegation — `ORCHESTRATOR`

The orchestrator invokes subagents using the `runSubagent` tool
(referenced in chat as `#runSubagent`). Use a consistent wrapper prompt:

```markdown
## Subagent Delegation

For each step, invoke the subagent using the `runSubagent` tool with this
prompt pattern:

This step must be performed as the agent "{Agent Name}" defined in
".github/agents/\_subagents/{filename}.agent.md".

IMPORTANT:

- Read and apply the entire .agent.md spec (tools, constraints, standards).
- Write state files to `./output/state/`.
- Write deliverables (code, templates) to the appropriate `./output/` subfolder.
- Return a clear summary: actions taken, files produced, issues found.
```

> **Note:** The canonical invocation mechanism is the `runSubagent` tool.
> Do not use handoffs for pipeline steps — handoffs transfer control and
> prevent the orchestrator from managing gates, validation, and state.

### Tool Availability

The orchestrator's `tools` list acts as a ceiling for all subagents. If a subagent
needs `execute/runInTerminal`, the orchestrator must also include it.

Plan the orchestrator's tool list to be the union of all subagent requirements.

### Result Handling

After a subagent returns:

1. Verify expected state files exist in `./output/state/`
2. Verify any deliverables were written to the correct `./output/` subfolders
3. Summarize results concisely for the gate presentation (don't dump raw output)
4. Run automatic validation (Phase 1 of gate)
5. Present gate to user (Phase 2) or auto-proceed for `Auto` gates
6. Update `00-{orchestrator-name}.md` — record gate decision, advance current step

---

## Output Files — `ALL` (recommended for Agents, required for Orchestrators and Subagents)

Document what the agent produces, separating state files from deliverables:

```markdown
## Output Files

### State Files

| File             | Location          | Purpose                       |
| ---------------- | ----------------- | ----------------------------- |
| `{NN}-{step}.md` | `./output/state/` | {What this state file tracks} |

### Deliverables (if any)

| File / Folder | Location                | Purpose                              |
| ------------- | ----------------------- | ------------------------------------ |
| `*.bicep`     | `./output/infra/bicep/` | {Generated infrastructure templates} |
```

### State File Structure (Subagent)

Every subagent state file includes a checkbox list of its workflow phases.
The file is created at the **start** of the subagent run with all phases
unchecked, then updated progressively after each phase completes.

**Initial state — written before Phase 1 begins:**

```text
# {Step Name}

> Generated by {agent-name} | {YYYY-MM-DD HH:MM:SS}

## Phases

- [ ] Phase 1: {Name}
- [ ] Phase 2: {Name}
- [ ] Phase 3: {Name}
```

**Mid-run — after Phase 2 completes:**

```text
## Phases

- [x] Phase 1: {Name} — {summary}
- [x] Phase 2: {Name} — {summary}
- [ ] Phase 3: {Name}
```

**Final state — after all phases complete:**

```text
# {Step Name}

> Generated by {agent-name} | {YYYY-MM-DD HH:MM:SS}

## Phases

- [x] Phase 1: {Name} — {summary}
- [x] Phase 2: {Name} — {summary}
- [x] Phase 3: {Name} — {summary}

## {Content sections per template...}

## Deliverables

- `./output/src/index.html`
```

The `## Phases` section is always first after the attribution line.
Create the initial file before Phase 1, then check off each phase as it
completes with an inline summary. If the step produces deliverables, list
them under `## Deliverables` so the orchestrator can track and revert them.

Subagents MUST document their state files — the orchestrator depends on knowing
what files to check for state management and gate validation.

Orchestrators MUST document their state file:

```markdown
### State Files

| File                        | Location          | Purpose                                     |
| --------------------------- | ----------------- | ------------------------------------------- |
| `00-{orchestrator-name}.md` | `./output/state/` | Pipeline parameters, step checklist, status |
```

Deliverables are optional per step. Not every step produces code or templates.

Include an attribution line in every output file:

```markdown
> Generated by {agent-name} | {YYYY-MM-DD HH:MM:SS}
```

Always include the full date and time. Run `Get-Date -Format "yyyy-MM-dd HH:mm:ss"` (PowerShell)
or `date "+%Y-%m-%d %H:%M:%S"` (bash) to get the current timestamp before writing.

---

## Validation Checklist — `ALL`

Every agent ends with a `## Validation Checklist` using checkbox items:

```markdown
## Validation Checklist

- [ ] {Check 1 — artifact-specific validation}
- [ ] {Check 2 — scope/boundary compliance}
- [ ] {Check 3 — quality standard met}
```

Typical items by archetype:

**Agent:**

- [ ] **REQUIRED:** State file created at start of run (before Phase 1 begins) with all phases unchecked
- [ ] State file saved to `./output/state/00-{agent-name}.md`
- [ ] State file includes `## Phases` checkbox list matching workflow phases exactly
- [ ] State file updated after each phase with inline summary (not end-only)
- [ ] Output matches expected format
- [ ] Scope boundaries respected (agent did not exceed its role)

**Orchestrator:**

- [ ] All gate options presented at every step
- [ ] State management handles resume, revert, and stop
- [ ] No direct work done — all delegated to subagents

**Subagent:**

- [ ] Skills read before any work started
- [ ] Prerequisites validated (previous step artifacts exist)
- [ ] State file created at start of run (before Phase 1) with all phases unchecked
- [ ] State file updated after each phase — phase checked off with inline summary
- [ ] State files saved to `./output/state/`
- [ ] State file includes `## Phases` checkbox list matching workflow phases
- [ ] Deliverables saved to `./output/` subfolders (not `./output/state/`)
- [ ] Template headings match exactly (if templates defined)
- [ ] Attribution header present on output files

---

## Handoffs — `AGENT` Only

Handoffs add UI buttons after a chat response, letting users jump to another agent.
They are a convenience for **agents only** — orchestrators and subagents
must NOT use handoffs because handoffs transfer control and prevent state tracking,
automatic validation, and gate management.

> [!WARNING]
> **Do NOT use handoffs in orchestrated pipelines.** Handoffs transfer control to the
> target agent — the orchestrator cannot inspect results, run validation, manage gates,
> or handle revert/resume. Use `#runSubagent` instead.

### When Handoffs Are Useful

An agent finishes its work and offers navigation buttons to related agents:

- A "Code Reviewer" finishes → `▶ Fix Issues` button jumps to a "Code Fixer" agent
- A "Test Writer" finishes → `▶ Run Tests` button jumps to a "Test Runner" agent

No state, no gates — just one-click navigation with a pre-filled prompt.

### Frontmatter Structure

```yaml
handoffs:
  - label: "▶ Fix Issues"
    agent: "Code Fixer"
    prompt: "Fix the issues identified in the review above."
    send: false
```

### Properties

| Property | Type    | Required | Description                                                 |
| -------- | ------- | :------: | ----------------------------------------------------------- |
| `label`  | string  |    ✅    | Button text shown in the chat UI                            |
| `agent`  | string  |    ✅    | Target agent name (must match `name` in target frontmatter) |
| `prompt` | string  |    ❌    | Pre-filled prompt for the target agent                      |
| `send`   | boolean |    ❌    | `true`: auto-submit. `false`: user reviews first (default)  |

### Guidelines

- Limit to 2-3 handoffs — only the most logical next steps
- Use `send: false` so users review before submitting
- Only reference agents that exist in the repo
- Use action-oriented labels: `▶ Fix Issues`, not `Next`

---

## Tool Configuration — `ALL`

### Tool ID Format

```yaml
tools:
  - execute/runInTerminal
  - read/readFile
  - search/textSearch
  - azure-mcp/keyvault
```

### Standard Tool Aliases

Can use aliases for brevity — they are supported as shorthand (case-insensitive):

| Alias     | Covers                                                           |
| --------- | ---------------------------------------------------------------- |
| `execute` | Shell execution (`runInTerminal`, `getTerminalOutput`, etc.)     |
| `read`    | File reading (`readFile`, `readNotebookCellOutput`, etc.)        |
| `edit`    | File editing (`createFile`, `editFiles`, etc.)                   |
| `search`  | Code search (`textSearch`, `fileSearch`, `searchSubagent`, etc.) |
| `agent`   | Subagent invocation (`runSubagent`)                              |
| `web`     | Web access (`fetch`, `githubRepo`)                               |
| `todo`    | Task management (`TodoWrite`) — VS Code only                     |

### MCP Server Tools

```yaml
tools:
  - "azure-mcp/*" # all tools from an MCP server
  - "azure-mcp/keyvault" # specific tool
```

### Best Practices

- **Principle of least privilege**: only enable tools the agent actually needs
- **Subagents**: use minimal tool sets — they do focused work
- **Orchestrator**: needs the union of all subagent tools plus `agent/runSubagent`
- **Empty list** (`tools: []`): explicitly disables all tools (for read-only advisory agents)
- Unrecognized tool names are silently ignored (allows environment-specific tools)

---

## Variables and Dynamic Parameters — `RECOMMENDED`

Agents can extract values from user input and pass them through the pipeline.

### Extraction Strategy

1. **From user prompt**: Look for explicit mentions (names, paths, options)
2. **From workspace**: Current file, folder structure, config files
3. **Ask user**: If critical information is missing, ask — don't guess

### Passing to Subagents

Pass variables through the `#runSubagent` prompt as concrete values, not placeholders:

```text
Write the state file to `./output/state/02-analysis.md`.
Write generated components to `./src/components/`.
The target framework is React with TypeScript.
```

Subagents receive resolved context — they work with concrete paths and values.

### Common Variables

| Variable         | Source              | Example                            |
| ---------------- | ------------------- | ---------------------------------- |
| State path       | Convention          | `./output/state/`                  |
| Deliverable path | Agent definition    | `./output/src/`, `./output/infra/` |
| Step number      | Pipeline definition | `02`                               |
| User preferences | Phase 1 questions   | `React`, `TypeScript`              |

---

## Error Handling — `RECOMMENDED`

Document how the agent handles failures. Use a table mapping errors to responses:

```markdown
## Error Handling

| Error                     | Response                           |
| ------------------------- | ---------------------------------- |
| Missing prerequisite file | STOP — report to orchestrator      |
| Tool unavailable          | Fall back to alternative approach  |
| Validation failure        | Report specific error, suggest fix |
| Auth failure              | Guide user through authentication  |
```

For orchestrators, error handling is built into the gate model — automatic validation
catches failures before presenting to the user.

---

## Common Mistakes

### Frontmatter Errors

- ❌ Missing `description` field
- ❌ Description not wrapped in quotes
- ❌ Model arrays with space-separated values instead of commas:
  `["Model A" "Model B"]` → `["Model A", "Model B"]`
- ❌ Invalid YAML syntax (tabs instead of spaces, broken indentation)
- ❌ Subagent with `user-invocable: true` (must be `false`)

### Body Structure

- ❌ No `## DO / DON'T` section — every agent needs scope boundaries
- ❌ Flat, unstructured prompt without phases or sections
- ❌ Workflow section named inconsistently (`Core Workflow`, `Main Workflow`) —
  standardize on `## Workflow`
- ❌ Missing `## Validation Checklist` — no way to verify quality
- ❌ Subagent without `## Prerequisites Check` — will proceed with missing inputs
- ❌ **Agent without a state file** — all agents write state files to `./output/state/`
- ❌ **Orchestrator without `00-{orchestrator-name}.md`** — loses parameters and gate decisions on resume
- ❌ **Subagent without step state file** — orchestrator cannot track progress or revert

### Orchestrator-Specific

- ❌ Doing work directly instead of delegating to subagents
- ❌ Skipping gates or auto-approving without presenting options
- ❌ No state management — can't resume after stopping
- ❌ Not deleting downstream files on revert
- ❌ No `00-{orchestrator-name}.md` — loses user parameters and gate decisions on resume

### Agent-Specific

- ❌ **Defining `## Workflow` phases without writing a state file** — if your workflow
  has discrete phases, state file is REQUIRED. This is the most common Agent mistake.
  Either write the state file or remove/restructure the workflow to be atomic.
- ❌ **Silently omitting state file without documenting opt-out in `## Context`** —
  opt-out justification MUST be explicit and placed in Context, not DO/DON'T
- ❌ **State file created but never updated** — checkpoint must be created at startup
  (before Phase 1) and updated progressively after each phase, not at the end
- ❌ **State file saved to wrong location** — agents write to `./output/state/00-{agent-name}.md`,
  not arbitrary paths or root `./output/`

### Tool Configuration

- ❌ **Omitting `"edit"` from tools list** — all agents write state files and need file editing capability
- ❌ Granting all tools when only a few are needed
- ❌ Subagent with `agents: ["*"]` (should be `[]`)
- ❌ Orchestrator missing tools that subagents require
- ❌ Orchestrator missing `"agent"` tool for subagent invocation

### Content

- ❌ Vague descriptions without scope boundaries ("Helps with Azure stuff")
- ❌ Hardcoded attribution headers instead of using templates
- ❌ No skills reference — agent drifts from shared conventions
- ❌ Conflicting instructions between agent body and referenced skill files

---

## Examples

### Agent

```yaml
---
name: "Code Reviewer"
description: "Reviews code for quality, security, and maintainability. Does NOT modify code."
model: ["Claude Sonnet 4.5"]
tools: ["read", "search", "edit"]
user-invocable: true
agents: []
---
```

````markdown
# Code Reviewer

Reviews code for quality, security, and best practices.

## Context

Before starting, read:

1. `.github/instructions/code-review.instructions.md` — review standards

## DO / DON'T

### DO

- ✅ Check for security vulnerabilities (OWASP Top 10)
- ✅ Verify error handling and edge cases
- ✅ Assess code readability and maintainability
- ✅ Provide specific, actionable feedback with line references

### DON'T

- ❌ Modify any code — review only
- ❌ Suggest rewrites without explaining the problem first
- ❌ Ignore test coverage gaps

## Workflow

### Phase 1: Understand

Read the code under review. Identify the purpose, patterns, and dependencies.

### Phase 2: Analyze

Check against quality standards: security, error handling, readability, performance.

### Phase 3: Report

Present findings grouped by severity (Critical / High / Medium / Low).
Update `./output/state/00-code-reviewer.md` — check off all phases.

## Output Files

### State Files (optional)

| File                  | Location          | Purpose             |
| --------------------- | ----------------- | ------------------- |
| `00-code-reviewer.md` | `./output/state/` | Progress checkpoint |

State file structure:

```text
# Code Reviewer Progress

> Generated by code-reviewer | 2026-02-21 14:32:00

## Phases

- [x] Phase 1: Understand — identified 3 modules, 12 dependencies
- [x] Phase 2: Analyze — found 2 critical, 5 medium issues
- [x] Phase 3: Report — findings presented by severity
```
````

## Validation Checklist

- [ ] All findings include severity and specific file/line references
- [ ] Security issues flagged separately
- [ ] No code modifications made

````

### Orchestrator

```yaml
---
name: "Pipeline Conductor"
description: "Orchestrates the 3-step analysis workflow. Delegates to subagents, manages gates and state. Does NOT do analysis work directly."
model: ["Claude Opus 4.6"]
tools: ["read", "search", "edit", "execute", "agent"]
user-invocable: true
agents: ["*"]
argument-hint: "Describe what you want to analyze"
---
````

```markdown
# Pipeline Conductor

Master orchestrator for the 3-step analysis workflow.

## MANDATORY: Read Skills First

**Before doing ANY work**, read:

1. **Read** `.github/skills/{domain}/SKILL.md` — shared conventions

## DO / DON'T

### DO

- ✅ Delegate ALL work to subagents
- ✅ Present gate options at EVERY step boundary
- ✅ Check `./output/state/` on startup to detect resume point
- ✅ Create and maintain `00-{orchestrator-name}.md` with parameters and step checkboxes
- ✅ Delete downstream files on revert

### DON'T

- ❌ Do analysis, research, or writing directly
- ❌ Skip gates or auto-approve
- ❌ Proceed past a gate without user confirmation

## Pipeline Definition

| Step | Subagent       | State File       | Gate     | On Fail                         |
| ---- | -------------- | ---------------- | -------- | ------------------------------- |
| 1    | Research Agent | `01-research.md` | Approval | Stop                            |
| 2    | Analysis Agent | `02-analysis.md` | Auto     | Retry once                      |
| 3    | Report Agent   | `03-report.md`   | —        | Back to Step 2 (Analysis Agent) |

All state files → `./output/state/`
Orchestrator state: `00-{orchestrator-name}.md`

## State Management

Pipeline state is tracked via `00-{orchestrator-name}.md` and step files in `./output/state/`:

1. On invocation, read `00-{orchestrator-name}.md` — restore parameters and step checkboxes
2. Find the first unchecked step — that is where the pipeline resumes
3. Verify step state files match checked steps

- **Reset**: delete `./output/state/` → state-only reset; delete `./output/` → full reset
- **Revert**: accept step number or step name, resolve to a step, delete files with
  prefixes >= that step, update `00-{orchestrator-name}.md`, then resume from that step

## Gate Model

After each subagent completes:

**Phase 1 — Automatic validation:**

1. Expected state file(s) exist in `./output/state/`
2. Required sections present (if defined)

If validation fails → report error, re-run subagent.

On validation failure, follow the pipeline row's `On Fail` policy.
Write failure details to `./output/state/{NN}-{step}-failure.md` before rerouting.

**Phase 2 — Apply gate type:**

- If Gate = `Approval`: present options below
- If Gate = `Auto`: proceed immediately to next step

✅ Step {N} Complete: {Subagent Name}
State file: ./output/state/{NN}-{step}.md

Choose:
✅ Approve → proceed to Step {N+1}
✏️ Revise → re-run with your feedback
⏪ Revert → specify step number or step name
🛑 Stop → halt, resume later

Step map:

1. Research Agent
2. Analysis Agent
3. Report Agent

## Subagent Delegation

Invoke subagents via `#runSubagent` with this pattern:

This step must be performed as the agent "{Name}" defined in
".github/agents/\_subagents/{file}.agent.md".

IMPORTANT:

- Read and apply the entire .agent.md spec.
- Write state files to `./output/state/`.
- Write deliverables to the appropriate `./output/` subfolder.
- Return a summary: actions taken, files produced, issues found.

## Validation Checklist

- [ ] All steps delegated to subagents (no direct work)
- [ ] `00-{orchestrator-name}.md` created at start, updated after each gate
- [ ] Gate behavior matches each pipeline row (`Approval` or `Auto`)
- [ ] State management handles resume, revert, stop
- [ ] Downstream files deleted on revert

## Output Files

### State Files

| File                        | Location          | Purpose                                     |
| --------------------------- | ----------------- | ------------------------------------------- |
| `00-{orchestrator-name}.md` | `./output/state/` | Pipeline parameters, step checklist, status |
```

### Subagent

```yaml
---
name: "Research Agent"
description: "Gathers and synthesizes research for Step 1. Invoked by orchestrator only."
model: ["Claude Opus 4.6"]
tools: ["read", "search", "web", "edit"]
user-invocable: false
agents: []
---
```

````markdown
# Research Agent

**Step 1** of the 3-step workflow: `[research] → analysis → report`

## MANDATORY: Read Skills First

**Before doing ANY work**, read:

1. **Read** `.github/skills/{domain}/SKILL.md` — conventions and templates
2. **Read** `.github/skills/{artifacts}/templates/01-research.template.md`

These skills are your single source of truth for conventions, defaults,
and template structures. Always read them at runtime — do not copy their
values into the agent body.

## DO / DON'T

### DO

- ✅ Search multiple sources before synthesizing findings
- ✅ Save state file to `./output/state/01-research.md`
- ✅ Match template headings exactly
- ✅ Include source references for all claims

### DON'T

- ❌ Perform analysis — that is the Analysis Agent's job
- ❌ Write state files to any location other than `./output/state/`
- ❌ Skip source validation

## Prerequisites Check

This is Step 1 — no prerequisites required.

## Workflow

Before beginning Phase 1, create `./output/state/01-research.md` with the
attribution header and all phases listed as unchecked (`- [ ]`). This makes
the subagent's progress visible immediately.

### Phase 1: Discovery

Search for relevant information across configured sources.
After completing this phase, update the state file — check off Phase 1 with a brief summary.

### Phase 2: Synthesis

Organize findings into the template structure.
After completing this phase, update the state file — check off Phase 2 with a brief summary.

### Phase 3: Finalize

Write final content sections to the state file.
Check off Phase 3 with a brief summary.

## Output Files

### State Files

| File             | Location          | Purpose           |
| ---------------- | ----------------- | ----------------- |
| `01-research.md` | `./output/state/` | Research findings |

State file structure:

```text
# Research

> Generated by research-agent | 2026-02-21 14:32:00

## Phases

- [ ] Phase 1: Discovery          ← initial state, created before Phase 1 starts
- [ ] Phase 2: Synthesis
- [ ] Phase 3: Finalize
```
````

```text
                                  ← after all phases complete:
## Phases

- [x] Phase 1: Discovery — searched 4 sources, found 12 references
- [x] Phase 2: Synthesis — organized into 3 themes
- [x] Phase 3: Finalize — findings written to state file

## Findings

{research content...}
```

## Error Handling

| Error                     | Response                           |
| ------------------------- | ---------------------------------- |
| Missing prerequisite file | STOP — report to orchestrator      |
| Tool unavailable          | Fall back to alternative approach  |
| Validation failure        | Report specific error, suggest fix |

## Validation Checklist

- [ ] Skills read before starting
- [ ] All findings include source references
- [ ] State file saved to `./output/state/01-research.md`
- [ ] Template headings match exactly
- [ ] Attribution header present

`````

---

## Quick Self-Check

Run through this checklist before committing an agent file.

### All Agents

- [ ] `name` and `description` present in frontmatter
- [ ] `description` states what the agent does AND does not do
- [ ] `model` specified as an array
- [ ] `tools` contains only valid tool IDs/patterns
- [ ] `## DO / DON'T` section present with ✅/❌ bullets
- [ ] `## Validation Checklist` present
- [ ] Scope boundaries clearly defined
- [ ] No hardcoded secrets, subscription IDs, or tenant IDs

### Orchestrator (additional)

- [ ] `user-invocable: true` and `agents: ['*']`
- [ ] `## Pipeline Definition` with step table
- [ ] `## State Management` with resume/revert/reset behavior
- [ ] `00-{orchestrator-name}.md` convention defined with parameters and step checkboxes
- [ ] `## Gate Model` supports both `Approval` and `Auto` gate behavior
- [ ] Validation failure routing policy defined for each step
- [ ] `## Subagent Delegation` with invocation prompt pattern
- [ ] No handoffs defined (use `#runSubagent` for pipeline steps)
- [ ] Tool list is the union of all subagent tool requirements

### Subagent (additional)

- [ ] `user-invocable: false` and `agents: []`
- [ ] Located in `.github/agents/_subagents/`
- [ ] `## MANDATORY: Read Skills First` present
- [ ] `## Prerequisites Check` present (or noted as Step 1)
- [ ] `## Output Files` table present
- [ ] `## Workflow` uses `### Phase N:` headings
- [ ] Workflow instructs: create state file before Phase 1 (all phases unchecked)
- [ ] Workflow instructs: update state file after each phase (check off + summary)
- [ ] No handoffs defined

### Agent (additional)

- [ ] `user-invocable: true`
- [ ] `## Context` section present
- [ ] `## Workflow` with phased structure
- [ ] Self-contained — no dependency on pipeline state files
- [ ] State file created at start of run (before Phase 1 begins) with all phases unchecked
- [ ] State file saved to `./output/state/00-{agent-name}.md`
- [ ] State file updated after each phase with inline summary
- [ ] State file includes `## Phases` checkbox list matching workflow phases

---

## Writing Style

- Use `#` headings (`##`, `###`) — not underline-style headings
- Keep markdown lines ≤ 120 characters
- Use tables for comparisons, decision matrices, and checklists
- Prefer relative links for repo content
- If including fenced code blocks inside a fenced template, use quadruple fences
  (` ```` `) for the outer fence to avoid accidental termination

## File Organization

| Type              | Location                     | Naming                   |
| ----------------- | ---------------------------- | ------------------------ |
| Agents            | `.github/agents/`            | `{name}.agent.md`        |
| Orchestrators     | `.github/agents/`            | `{name}.agent.md`        |
| Subagents         | `.github/agents/_subagents/` | `{name}.agent.md`        |
| Pipeline state    | `./output/state/`            | `00-{orchestrator-name}.md`       |
| Step state        | `./output/state/`            | `{NN}-{step}.md`         |
| Skills            | `.github/skills/{name}/`     | `SKILL.md`               |
| Instructions      | `.github/instructions/`      | `{name}.instructions.md` |

File naming: lowercase with hyphens. Allowed characters: `.`, `-`, `_`, `a-z`, `0-9`.

## Additional Resources

- [Creating Custom Agents](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-custom-agents)
- [Custom Agents in VS Code](https://code.visualstudio.com/docs/copilot/customization/custom-agents)
- [MCP Integration](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/extend-coding-agent-with-mcp)
`````
