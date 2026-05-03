
NOOB
Agentic SDLC Orchestration Platform

Product Specification
Version 1.7  ·  Phase 1
4 April 2026
Supersedes v1.6

# 1. Overview
NOOB is an async human-in-the-loop orchestration platform for AI-driven software development. It coordinates stateless AI agents — called Noobs — with human stakeholders across a structured, opinionated SDLC.

Three core principles:

> **System owns truth.** JSON artefacts, messages, feedback, and validation results. Strict, enforced, auditable.

> **Git supports it.** Storage, history, diffs. Not the domain model.

> **UI drives behaviour.** Opinionated and structured. No freeform chaos.

## 1.1 What Changed in v1.1
- JSON as source of truth for all artefacts
- Markdown is a rendered view only — never edited directly
- Model routing: Type A / B / C task classification
- Serial editing model: one writer per artefact at a time
- Feedback entity promoted to first-class versioned object
- Commit ↔ message coupling made mandatory
- Artefact validation layer added
- Stuck detection added
- Human direct editing deferred entirely

## 1.2 What Changed in v1.2
- Concurrency model revised: multiple Noobs per stage; one Noob per artefact
- Artefact ownership model formalised
- Noob lifecycle expanded: Working / Waiting / Parked / Dismissed
- NFA definition revised
- Message reply vs Send Feedback clarified
- Stage panel Noobs section added
- Hire a Noob dialog introduced

## 1.3 What Changed in v1.3
- Artefact naming convention: `{type}_{qualifier}` (e.g. `spec_auth`)
- Multiple artefact instances of same type per stage supported
- Hire a Noob dialog: type prefix + qualifier input
- Stage rename: Release → DevOps
- DevOps stage defined
- Build progress tracking clarified: tasks artefact is living list
- UAT feedback explicitly in scope; loops are first-class
- Telemetry layer added (Phase 1 data model)
- Analytics view defined (Phase 2 UI)
- Process Refiner role introduced
- Deployment tracking: git is source of truth

## 1.4 What Changed in v1.4
- **Oracle mode and @mention** introduced: `@<noobname>` invokes a Noob in oracle mode — fast, read-only, no container. Same Noob, two execution modes.
- **Working-state snapshot** defined: the continuity mechanism for Noobs. Structured object stored in DB, written at end of each worker run.
- **Three state layers** formalised: Permanent / Active / Ephemeral.
- **Chat model** made explicit: normal message = human discussion; `@<noobname>` = oracle query; Send Feedback = worker mutation. No auto-routing.
- **Worker agent execution model** defined: dockerised, thin runner, strict contract boundary.
- **Two-repo model** introduced: docs repo (artefacts, default workspace) + code repo (checked out on demand). Rationale: isolation, repo size, speed of agent creation.
- **Notifications** defined as a first-class entity distinct from messages: FYI only, no reply, no lifecycle.
- **Audit path** explicitly defined: artefact ↔ commit ↔ feedback ↔ message.
- **Cost strategy** formalised: deterministic → Haiku → Sonnet → RAG, in order.
- **Jira** explicitly out of scope and speculative. Not a phase target.

## 1.5 What Changed in v1.5
- **Test Prep stage added**: new QA-owned stage that runs in parallel with Design and Build. Reads `spec` and `tds`. Produces `testcases` artefact.
- **Review stage revised**: Review now executes tests against built code rather than writing them. Produces `test_results` artefact. Can only open once Build has reached deployable state AND Test Prep has produced a valid `testcases` artefact.
- **`testcases` artefact ownership moved**: from Review to Test Prep.
- **`test_results` artefact introduced**: owned by Review. Records execution results and QA sign-off.
- **Artefact precheck added**: before a worker container is started, the system validates that the assigned artefact is not currently owned by another active Noob. Dispatch is rejected before compute is spent.
- **Stage gate for Review formalised**: Review opens only when both Build (deployable state) and Test Prep (valid `testcases`) gate conditions are met.

## 1.6 What Changed in v1.6
- **Artefact lifecycle formalised**: artefacts now have explicit states — Active (owned) and Archived (unowned, read-only). Archive is a human action. Archived artefacts remain in the docs repo as reference material.
- **Artefact reassignment defined**: a Noob's assigned artefact can be changed by a human when the Noob is Waiting or Parked. Reassignment must be accompanied by new feedback scoping the updated task.
- **`#` artefact mention syntax introduced**: typing `#` in any message thread opens an autocomplete of project artefacts. Inserts a structured reference (e.g. `#spec_auth`) that renders as a tappable chip. Artefact refs are parsed and stored on the feedback entity.
- **Cross-artefact write guard options documented**: three candidate mechanisms identified (Section 3.7). Not yet decided.

## 1.7 What Changed in v1.7
- **Storage architecture formalised**: docs repo is one-per-project, auto-created by NOOB at project creation as a local bare git repo. Code repo is externally owned — connected at project setup by providing a remote URL and SSH deploy key.
- **Source files introduced**: a new category of human-uploaded input files (transcripts, meeting notes, legacy docs) distinct from agent-produced artefacts. Stored in the docs repo. Text formats only in Phase 1. Referenced in messages via `$` mention syntax.
- **`$` source file mention syntax**: typing `$` in any message thread opens an autocomplete of source files uploaded to the current stage and project. Inserts a structured reference (e.g. `$transcript_discovery`) that renders as a tappable chip. Available to Noobs in their execution context.
- **Project identity split**: every project has a human-readable display name (mutable) and a git slug (immutable after creation). Slug is auto-generated from the display name; user can override at creation time. Used in branch names and repo paths.
- **Branch-per-Noob in code repo**: each Noob that writes code gets its own branch (`noob/{project-slug}/{noob-name}`), created on first code commit. PRs and merges are handled by developers outside NOOB. Docs repo always commits to main.

# 2. System Architecture

## 2.1 Three Paths

**Write path** — controlled, validated, auditable
```
Human → Send Feedback → Worker Noob (container) → JSON artefact → Validation → Commit → Notify
```

**Read path** — fast, lightweight, read-only
```
Human → @<noobname> → Oracle mode → Docs repo + Noob snapshot → Answer
```

**Audit path** — the complete record
```
artefact ↔ commit ↔ feedback ↔ message
```

Every mutation traces back to a human decision. Every human decision traces forward to an artefact change.

## 2.2 Two Execution Modes
A Noob is a persona with two modes of execution, not two separate agents.

| Mode | Trigger | Container | Artefact access | Writes |
|------|---------|-----------|-----------------|--------|
| Worker | Send Feedback | Yes (Docker) | Docs repo (+ code repo if needed) | Yes — validated, committed |
| Oracle | @mention | No | Docs repo + own snapshot | Never |

Same name. Same context. Different execution path.

## 2.3 Two-Repo Model
Agents operate against two repositories with different access patterns.

**Docs repo**
- Contains all NOOB artefacts (spec, tasks, prd, tds, testcases, release-notes) and source files
- Small, fast to clone
- Default workspace for all Noob runs
- Always available to oracle mode
- **Created automatically by NOOB** at project creation as a local bare git repo (`/data/repos/{project-id}/docs.git`). No manual setup required.
- Always commits to `main`. No branching — artefact ownership prevents conflicts.

**Code repo**
- Contains the codebase being built
- Potentially large; slow to clone
- Checked out only when the worker Noob explicitly needs to read or write code
- Not included in oracle mode context by default
- **Externally owned** — the team's existing repository. Connected at project setup by providing a remote URL. NOOB generates an SSH deploy keypair per project at creation; the human adds the public key to the repo's deploy key settings once.
- For greenfield projects: human creates an empty repo on their preferred platform, then connects it.
- Each Noob that writes code commits to its own branch: `noob/{project-slug}/{noob-name}`. Branch is created on first code commit. PRs and merges handled by developers outside NOOB.

**Rationale:** Isolation (artefact mutations never touch the codebase directly), repo size (don't clone a large codebase on every agent run), speed of agent creation (oracle and most worker runs only need the docs repo).

## 2.4 Source Files
Source files are human-uploaded input materials — meeting transcripts, stakeholder notes, legacy documentation, wireframes — that Noobs read but never write.

| Property | Value |
|----------|-------|
| Storage | Docs repo, `/inputs/{stage}/` or `/inputs/project/` |
| Format | Text only in Phase 1 (`.txt`, `.md`). Binary/PDF Phase 2. |
| Size limit | 5 MB per file |
| Ownership | None — not owned by any Noob |
| Lifecycle | Append-only. Duplicate filename rejected; user must rename before upload. |
| Scope | Stage-level (attached to one stage) or project-level (available to all stages) |
| Agent access | Read-only. Text content injected into Noob context when referenced. |
| Mention syntax | `$filename` in any message or brief. Autocomplete opens on `$`. Renders as tappable chip. |

Source files are distinct from artefacts: they have no schema, no version history, no ownership, and no validation. They are reference material, not outputs.

## 2.4 Editing Model

> Humans do not edit artefacts directly. They express intent via messages. The agent reads the feedback, applies changes to the JSON artefact, the system validates, generates a diff, and presents it. The message thread is the *why*. The diff is the *what*.

### Artefact format
All artefacts stored as JSON. Markdown rendered on demand — never committed, never edited directly.

**Why JSON:** Schema validation is trivial. Referential integrity is a dictionary lookup. Progress metrics are reliable field counts. Diffs are structural. Agents have a clean, unambiguous contract.

### Audit trail

| Layer | Entity | Records |
|-------|--------|---------|
| Intent | Message thread | What humans discussed and decided |
| Decision | Feedback (versioned) | The synthesised instruction sent to the agent |
| Outcome | Commit diff (JSON + rendered) | What actually changed in the artefact |

### Commit ↔ message coupling
Every agent commit must reference a messageId. A commit without a message reference is rejected. This enforces the audit trail — every change has a traceable reason.


# 3. Agent Architecture

## 3.1 Worker Noobs
Dockerised, stateless agents that mutate artefacts.

**Execution contract:**
```
input (brief + feedback) → run agent → write files to docs repo → validation → commit → notify
```

**Key constraint:** The container proposes. The system accepts or rejects. Agents are untrusted producers. All output is treated as untrusted input until validated.

The complexity lives in the contracts and validation layer — not inside the container. The container itself is a thin runner: receive input, run model, write files, exit.

**No business logic inside the container.** Stage awareness, artefact ownership, concurrency rules, validation — all enforced outside.

### Worker run outputs
At the end of every worker run, the Noob writes a working-state snapshot (see Section 3.3).

## 3.2 Oracle Mode
A lightweight, read-only query mode invoked by `@<noobname>` in any message thread.

**Properties:**
- No container — runs as a direct API call
- Read-only — never writes to artefacts or message state
- Fast — designed for conversational response times
- Scoped — sees all docs repo artefacts (globally shared) + the named Noob's own working-state snapshot and message history

**What @barry can answer:**
- Questions about any artefact in the docs repo (spec, tasks, prd, tds, etc.)
- Questions about his own current task, status, open questions, and history
- He cannot answer questions that require reading the code repo unless that context is included in his snapshot

**Oracle mode does not mutate anything.** It cannot trigger a worker run, commit a file, or advance a stage. It is purely informational.

## 3.3 Working-State Snapshot
The mechanism that gives a stateless Noob apparent continuity between runs.

At the end of every worker run, the Noob writes a structured snapshot to the DB:

| Field | Description |
|-------|-------------|
| noobId | Identity of the Noob |
| currentTask | What they were working on |
| openQuestions | Unresolved ambiguities or blockers |
| status | `waiting` \| `active` \| `blocked` |
| nextLikelyActions | What they expect to do next |
| updatedAt | Timestamp |

This snapshot is:
- Read by oracle mode to answer `@barry` queries
- Read by the worker runner at the start of the next run to restore context
- Not a knowledge base — it is runtime context only

> Working state is not knowledge. It is the agent's short-term memory between runs.

## 3.4 State Retention Model

| Layer | Contents | Lifetime | Storage |
|-------|----------|----------|---------|
| Permanent | Artefacts, messages, feedback, decisions, diffs | Forever | DB + git |
| Active | Working-state snapshot per Noob | Until Noob is dismissed | DB |
| Ephemeral | Oracle chat sessions | Session only | Not persisted |

## 3.5 Model Routing

| Type | Name | Examples | Model |
|------|------|----------|-------|
| A | Deep reasoning | Interpreting threads, resolving ambiguity, generating artefact content | Sonnet |
| B | Structured edits | Update JSON fields, add/remove items, change statuses | Haiku |
| C | Deterministic | Validation, diffing, rendering, referential checks, telemetry | No model (code) |

### Cost strategy
Deterministic first (cheapest) → Haiku for bounded queries → Sonnet for deep reasoning → RAG only as fallback for fuzzy retrieval. Each step is only invoked when the previous is insufficient.

> Avoid unnecessary embedding and retrieval cost. Most operations should never touch an LLM.

### Oracle cost profile
Oracle queries are typically Type B or C — structured reads against known data. RAG is only invoked for genuinely fuzzy questions that can't be answered by direct artefact lookup.

## 3.6 Validation Contract
All agent output is treated as untrusted input.

Enforced outside the container:
- Schema validation (required fields, correct types)
- Referential integrity (spec refs, task refs)
- File boundaries (agent may only write to its assigned artefact)
- Stage ownership (agent may not write to artefacts it doesn't own)
- Identity rules (commit must reference a valid Noob and messageId)

**Validation results:** Pass / Soft Fail / Hard Fail. Hard fail rejects the commit and returns the agent to Working state with an error message.


## 3.7 Cross-artefact Write Guard (Undecided)

The system must prevent a Noob from writing to artefacts it does not own. Commit validation (Section 3.6) is the definitive backstop — a commit touching files outside the Noob's assigned artefact is always rejected. The open question is whether an earlier, cheaper guard should fire before the Noob does substantive work.

Three candidate mechanisms are identified. Not yet decided.

### Option A — `#` structural mentions (zero model cost)
If feedback to a Noob contains `#artefact` references not matching the Noob's owned artefact, the system detects this structurally at dispatch time and warns the human before the container starts. No model involved. Covers all cases where humans use `#` syntax correctly. Does not catch free-text cross-artefact instructions.

### Option B — Pre-dispatch lightweight intent check
Before container start, a Haiku call reads the feedback and asks: "does this feedback instruct work on artefacts other than `{owned_artefact}`?" On a positive signal, dispatch is paused and the human is warned. Fires on every dispatch. Adds latency and cost per dispatch. Catches unstructured free-text cases that Option A misses.

### Option C — Container-side intent declaration + test commit
The agent's first act on starting is to declare which artefact(s) it intends to write to. The system validates ownership deterministically. If validation passes, the agent proceeds. If it fails, the agent posts back immediately — no substantive work is done. Requires a minimal LLM reasoning step (declaring intent) but no additional infrastructure. Eliminates wasted full runs against non-owned artefacts.

### Current position
Option A (`#` mentions) is implemented regardless — it is a UX feature with a guard side-effect, not pure infrastructure. Options B and C address the residual free-text case. The tradeoff is latency and cost (B) vs. container-side complexity (C). Commit validation backstops all three options.

# 4. Chat Model

## 4.1 Three Message Types
There is no auto-routing. Humans explicitly choose the type of interaction.

| Interaction | How | What happens |
|------------|-----|--------------|
| Human discussion | Normal message in thread | Visible to other humans immediately. Noob sees nothing. |
| Oracle query | `@<noobname>` in thread | Routes to oracle mode. Fast read-only response. No mutation. |
| Worker instruction | Send Feedback button | Packages replies into Feedback. Dispatches to worker Noob. Mutation may follow. |

## 4.2 @mention Behaviour
`@<noobname>` can be used in any message thread. The named Noob responds in oracle mode.

The oracle response is inline in the thread — it appears as a message from that Noob but is clearly labelled as an oracle (read-only) response, not a worker action.

Multiple Noobs can be @mentioned in a single thread. Each responds independently in oracle mode.

## 4.3 Artefact Mentions (`#`)
Typing `#` in any message thread opens an autocomplete showing artefacts available in the current project. Selecting one inserts a structured reference (e.g. `#spec_auth`) into the message body. Artefact mentions:
- Render as tappable chips in the sent message — clicking opens the artefact in the table
- Are parsed from message and feedback bodies and stored as `artefactRefs`
- Give the system structured intent signal for cross-artefact write detection (Section 3.7, Option A)
- Are visible to the Noob in its execution context

Free-text artefact references (without `#`) are not parsed. Using `#` syntax is encouraged but not enforced.

## 4.4 Source File Mentions (`$`)
Typing `$` in any message thread opens an autocomplete showing source files uploaded to the current stage and project. Selecting one inserts a structured reference (e.g. `$transcript_discovery`) into the message body. Source file mentions:
- Render as tappable chips in the sent message — clicking opens the file
- Are available to the Noob in its execution context; the runner injects the file content (text) when building the agent's context
- Stage-level files appear first in the autocomplete; project-level files follow

The three compose-time mention sigils:

| Sigil | Targets | What it does |
|-------|---------|--------------|
| `@name` | Noobs | Oracle query — fast read-only response |
| `#name` | Artefacts | Structured reference; parsed as `artefactRefs`; guards intent |
| `$name` | Source files | Injects file content into Noob context |

## 4.4 Send Feedback
Send Feedback is the only action that instructs a worker. It is a deliberate, explicit handoff.

> Send Feedback is an editorial act, not a forward button. The stakeholder synthesises the discussion into a single coherent instruction.

Send Feedback is only enabled when at least one human reply exists in the current cycle.


# 5. Core Concepts

## 5.1 Projects
A Project is initiated by a human recruiting a PM agent. That single act creates the project, opens the Discovery stage, hires the first Noob, and auto-creates the docs repo.

- Projects contain stages, team members, artefact files, and source files
- Project status is a generated summary of all active stage statuses
- Projects are never deleted — stages become Dormant when work completes

### Project identity
Every project has two names:

| Field | Description | Mutable |
|-------|-------------|---------|
| `name` | Human-readable display name. e.g. "Payments Redesign Q3" | Yes — can be changed at any time |
| `slug` | Git-safe identifier. e.g. `payments-redesign-q3` | No — set at creation, never changes |

The slug is auto-generated from the display name at creation time (lowercased, spaces and special characters replaced with hyphens). The user can override it before confirming. After creation it is locked. Slugs must be unique across all projects. The slug is used in: docs repo directory paths, code repo branch names (`noob/{slug}/{noob-name}`), and any path that git or the filesystem touches.

## 5.2 Stages
Stages are hardcoded and opinionated. Never marked complete — become Dormant when no agents are working and no messages are open. Can reopen at any time.

### Stage order
Discovery → Clarification → Design → Build → Review → DevOps

Test Prep runs in parallel with Design and Build:
```
Discovery → Clarification ──→ Design ─────────────┐
                          └──→ Test Prep (parallel)─┤→ Build → Review → DevOps
```

Review gate: opens only when Build has reached deployable state AND Test Prep has produced a valid `testcases` artefact.

Not strictly enforced — stages can be active simultaneously and can reopen. Loops are expected (Section 5.7).

### Stage states
- **Active** — at least one Noob is currently working
- **Awaiting Feedback** — messages in stakeholder discussion phase
- **Dormant** — no Noobs working, no open messages
- **Stalled** — Active but no commits in N hours

### Stage roles and artefacts

| Stage | Role | Reads | Produces | Progress |
|-------|------|-------|----------|----------|
| Discovery | Product Owner | Brief | `prd` | Sections complete |
| Clarification | Business Analyst | `prd` | `spec` | Requirements resolved |
| Design | Architect | `spec` | `tds` | Spec items covered |
| Test Prep | QA | `spec`, `tds` | `testcases` | Cases written |
| Build | Developer | `spec`, `tds` | `tasks` + code | Tasks defined / implemented |
| Review | QA | `testcases` + code | `test_results` | Cases passing / sign-off |
| DevOps | DevOps | `tasks`, `test_results` | `release-notes` | — |

### Sub-stage progress

| Stage | Sub-stage 1 | Sub-stage 2 |
|-------|-------------|-------------|
| Build | Spec → Tasks | Tasks → Code |
| Test Prep | Spec → Cases | — |
| Review | Cases → Run | Run → Pass |

Progress computed from JSON field counts (Type C — no model).

## 5.3 Artefact Naming
Each stage has a fixed type prefix. Human provides optional qualifier at hire time.

```
{type}_{qualifier}   →   spec_auth, tasks_payments, testcases_regression
```

No qualifier = just the type prefix (`spec`, `tasks`, etc.). Conflict detection checks the fully-qualified name.

### Artefact types by stage

| Stage | Type prefix |
|-------|-------------|
| Discovery | `prd` |
| Clarification | `spec` |
| Design | `tds` |
| Test Prep | `testcases` |
| Build | `tasks` |
| Review | `test_results` |
| DevOps | `release-notes` |

## 5.4 Artefact Lifecycle

### States

| State | Description |
|-------|-------------|
| Active | Owned by exactly one Noob. The Noob may write to it. |
| Archived | Unowned. Read-only. Preserved in docs repo as reference. Not selectable for new assignments. |

### Archive
Archiving is a human action, available on any artefact in the Stage Detail panel. It unassigns the artefact from its Noob and marks it read-only. The Noob is not dismissed — they remain in their current state and must be reassigned or dismissed separately.

Typical trigger: a unified artefact (e.g. `spec`) needs splitting. The human archives the original, creates new qualified artefacts (`spec_auth`, `spec_payments`), and assigns Noobs to each. The archived artefact remains accessible in the docs repo as source material for the new Noobs' briefs.

### Artefact creation
A new artefact is created when a Noob is hired and assigned it via the Hire dialog. The artefact file does not exist until the Noob's first commit.

An artefact can also be pre-created (empty stub) as a human action in the Stage Detail panel, independent of hiring — useful for declaring structure before the Noob is ready to be hired.

### Reassignment
A Noob's assigned artefact can be changed by a human action. Conditions:
- The Noob must be in Waiting or Parked state (not Working)
- The new artefact must not be currently owned by another active Noob
- Reassignment must be accompanied by new feedback scoping the Noob's updated task
- The Noob's working-state snapshot is preserved; the new feedback replaces the prior execution context

Typical trigger: after an artefact split, the original Noob is reassigned from the archived artefact to the new qualified artefact, with feedback clarifying the tighter scope.

## 5.5 Build Stage
The `tasks` artefact is a living list of work items, not a static document. Agent generates it from the spec, updates statuses as PRs are merged.

Progress = `tasks[].status == 'implemented'` / `tasks.length`. No prose parsing, no manual entry.

The completed tasks artefact is the input for DevOps release notes generation.

## 5.6 DevOps Stage
Handles everything after Build completion.

**Agent responsibilities:**
1. Monitor Build tasks artefact for completion signal
2. Trigger UAT branch deploy; monitor CI/CD pipeline
3. Surface pipeline state (read from git — branch state, tags, pass/fail)
4. Generate release notes from tasks artefact
5. Wait for human UAT sign-off

**Human gates:**
- UAT sign-off — confirms UAT passed; required before production
- Mark as released to production — records prod deploy; closes stage

**Production deploy:** Off-system in Phase 1. NOOB provides a "Mark as released" action but does not trigger the deploy.

**Deployment tracking:** Git is the source of truth (branch merges, tags, pipeline results). NOOB surfaces this — does not maintain a parallel deployment record.

## 5.7 Noobs

### States
- **Working** — actively executing; visible in Agent Activity panel
- **Waiting on humans** — has posted a message; visible in Message list
- **Parked** — NFA'd; idle; visible only in stage Noobs view
- **Dismissed** — removed entirely; no longer visible

### Lifecycle
Hire → Work → Post message → (Resume via Send Feedback | NFA to Parked) → (New task | Dismiss)

Dismiss is only available when Parked.

### Hire a Noob Dialog
- **Noob name** — auto-assigned; user can override. Must be unique within the project.
- **Brief** — required
- **Artefact** — type prefix shown (e.g. `spec_`) + free-text qualifier input. Conflict detection on fully-qualified name. Currently claimed artefacts listed for reference.

### Code branch
Noobs that write code (Build, DevOps) are assigned a branch in the code repo on first commit:
```
noob/{project-slug}/{noob-name-lowercase}
```
e.g. `noob/payments-redesign/karen`

The branch name is stored on the Noob record (`codeBranch`) and set to `null` until first code commit. PRs and merges are handled by developers outside NOOB. The branch is not deleted on Noob dismissal — branch lifecycle is the developer's responsibility.

## 5.8 Loops
Loops are expected and first-class. Not failures — how the system self-corrects.

**Common loops:**
- UAT finds a bug → DevOps notifies Build → new task in Build's artefact
- Spec is ambiguous during Build → message back to Clarification → spec updated
- Test cases fail → QA (Review) loops with Build on failing implementation
- Test Prep finds spec ambiguity → loops back to Clarification before cases are written

Loop count per stage is recorded in the telemetry layer.

## 5.9 Messages

### States
- **Open** — with humans; in message list; accepting replies
- **With agent** — feedback sent; Noob Working; not in active list
- **Parked** — NFA'd; visible only in stage Noobs view
- **Dismissed** — Noob dismissed; archived

### Gate actions
- **Send Feedback** — synthesises human replies into Feedback record; dispatches to worker Noob
- **NFA (No Further Action)** — parks thread; no feedback sent; Noob moves to Parked

NFA is not "proceed without changes." It is "this conversation is over."

## 5.10 Notifications
Notifications are a distinct first-class entity, separate from messages.

| | Messages | Notifications |
|---|---------|--------------|
| Require action | Yes | No |
| Reply | Yes | No |
| Lifecycle | Open → Parked/With agent | Fire and forget |
| Appear in message list | Yes | No — separate surface |

Notifications are generated by: SOT artefact changes, stuck detection flags, UAT loop-back events, pipeline status changes.

They reduce noise by keeping FYI information out of the actionable message list.

## 5.11 Feedback Entity

| Field | Description |
|-------|-------------|
| id | Unique identifier |
| messageId | The message thread this closes |
| authorUserId | Who composed and sent the feedback |
| stage | Which stage this belongs to |
| body | The synthesised instruction |
| artefactRefs | Artefact names parsed from `#` mentions in the body |
| version | Incremental per message thread (v1, v2...) |
| createdAt | Timestamp |


# 6. Artefact Validation

## 6.1 When Validation Runs
- On every agent commit registration
- Type C — pure code, no model
- Result gates the commit: pass, soft fail, or hard fail

## 6.2 Validation Levels

| Level | Behaviour |
|-------|-----------|
| Pass | Commit accepted. Progress metrics update. |
| Soft Fail | Commit accepted but flagged. Warning shown. Metrics not updated. |
| Hard Fail | Commit rejected. Message remains open. Agent must fix and resubmit. |

## 6.3 Phase 1 Validation Scope
Spec and Tasks artefacts only. PRD, TDS, Test Cases deferred to Phase 2.

### spec validation
- Required fields: id, title, requirements[]
- Each requirement: id, title, status (open | resolved)
- No duplicate requirement IDs

### tasks validation
- Required fields: id, title, tasks[]
- Each task: id, title, specRef, status (defined | in-progress | implemented)
- Referential integrity: every specRef exists in spec requirements
- No duplicate task IDs

## 6.4 Cross-stage Impact
Upstream artefact change → system flags potentially stale downstream artefacts. Never auto-overwrites. Human decides.


# 7. Telemetry Layer

## 7.1 Principles

Telemetry is recorded for all agent actions in Phase 1. The Analytics UI is Phase 2. **The data model must be right from the start — retrofitting provenance is hard.**

Metrics are grouped into three capture difficulty tiers:
- **Easy** — derivable from timestamps and existing status fields. No extra instrumentation. Captured Phase 1.
- **Medium** — requires light additional data capture or heuristic classification. Captured Phase 1 data model; surfaced Phase 2.
- **Hard** — requires semantic analysis, LLM-as-judge, or agent-internal instrumentation. Phase 3.

Token consumption is **Hard**, not Easy, because Noobs run via the Claude CLI (not direct API calls), so token counts are not trivially accessible. This requires either CLI output parsing or a wrapper that intercepts usage metadata.

---

## 7.2 Per-run record (Easy)
Recorded on every agent invocation. Type C — no model, pure system capture.

| Field | Type | Description |
|-------|------|-------------|
| runId | uuid | Unique identifier for this invocation |
| noobId | string | Which noob ran |
| taskId | string | Which task was attempted |
| stage | string | Stage at time of run |
| project | string | Project at time of run |
| attemptNumber | integer | How many times this task has been attempted (1-indexed) |
| triggerReason | enum | `initial` \| `human_feedback` \| `system_retry` \| `loop_return` |
| startedAt | ISO8601 | When the run began |
| endedAt | ISO8601 | When the run completed or failed |
| durationMs | integer | Derived from start/end |
| outcome | enum | `success` \| `soft_fail` \| `hard_fail` \| `abandoned` |
| firstPassAccepted | boolean | True if outcome was `success` with attemptNumber = 1 and no feedback cycles |
| idleMs | integer | Gap between task assigned and first output posted — detects poor briefs |

---

## 7.3 Failure record (Easy / Medium)
Appended to the run record on any non-success outcome.

| Field | Type | Difficulty | Description |
|-------|------|-----------|-------------|
| retryReason | string | Easy | Agent-supplied text describing why it failed or what it needs |
| formatViolation | boolean | Easy | True if outcome was caused by schema/format rejection |
| validationResult | enum | Easy | `pass` \| `soft_fail` \| `hard_fail` — from the commit validation gate |
| failureCategory | enum | Medium | Classified reason: `ambiguous_brief` \| `missing_context` \| `schema_violation` \| `hallucinated_dependency` \| `contradicted_artefact` \| `scope_error` \| `other` |
| selfCorrected | boolean | Hard | True if the noob identified and fixed its own mistake before submitting — requires agent-internal observability |

`failureCategory` in Phase 1 is recorded as `null` with provision for backfill via a judge pass in Phase 3. The field must exist in the schema from the start.

---

## 7.4 Feedback cycle record (Easy / Medium)
Captures the cost and shape of each human feedback cycle, recorded against the task/run.

| Field | Type | Difficulty | Description |
|-------|------|-----------|-------------|
| feedbackCycleCount | integer | Easy | Total human reply rounds before the output was accepted |
| humanFirstResponseMs | integer | Easy | Time from message appearing to first human reply |
| humanDispatchMs | integer | Easy | Time from message opened to "Dispatch to Agent" pressed — measures human review time |
| humanBlockingMs | integer | Easy | Total time the noob was in `waiting` state across all cycles for this task |
| noobBlockingMs | integer | Easy | Total time humans were waiting on the noob across this task (`working` state duration) |
| clarificationRatio | float | Medium | Fraction of noob messages that were clarification requests rather than output submissions — high values indicate poor brief quality |
| briefWordCount | integer | Easy | Word count of the original brief — proxy for brief quality |
| briefHasGoal | boolean | Medium | Heuristic: does the brief contain a clear goal statement |
| briefHasConstraints | boolean | Medium | Heuristic: does the brief mention constraints or boundaries |

`humanDispatchMs` requires a UI timestamp captured when the user presses "Dispatch to Agent". This must be instrumented in the frontend.

---

## 7.5 Output quality record (Medium / Hard)
Assessed after each completed run.

| Field | Type | Difficulty | Description |
|-------|------|-----------|-------------|
| revisionMagnitude | enum | Medium | `none` \| `minor` \| `moderate` \| `major` — derived by diffing consecutive outputs; classified by % changed |
| scopeCreepDetected | boolean | Hard | True if output contains content not present in brief or referenced artefacts — requires semantic analysis |
| instructionAdherenceScore | float | Hard | 0–1 score; semantic comparison of output against agent instructions and global standards |
| specFidelityScore | float | Hard | 0–1 score; coverage of spec requirements in the output artefact — schema-assisted for structured artefacts |

`revisionMagnitude` thresholds (Phase 2 tunable): minor < 15% changed lines, moderate 15–40%, major > 40%.

---

## 7.6 Resource record (Hard)
Captured where available. Not required for Phase 1 or 2 functionality.

| Field | Type | Difficulty | Description |
|-------|------|-----------|-------------|
| cliTokensIn | integer | Hard | Prompt tokens consumed — requires CLI output parsing or wrapper interception |
| cliTokensOut | integer | Hard | Completion tokens — same constraint |
| cliCostUsd | float | Hard | Derived from token counts and model pricing — only meaningful once token capture is solved |

**Note:** Because Noobs run via the Claude CLI rather than direct API calls, token metadata is not returned in a structured form. Capturing this requires either parsing CLI stdout/stderr for usage lines, or wrapping the CLI invocation in a proxy that intercepts and logs the data. This is a non-trivial infrastructure concern; defer to Phase 3.

---

## 7.7 Derived / aggregate metrics (Phase 2)
Computed from the per-run records above. Not stored raw — calculated on read.

| Metric | Derivation |
|--------|-----------|
| First-pass acceptance rate | `count(firstPassAccepted=true) / count(runs)` per noob or stage |
| Average feedback cycles | `mean(feedbackCycleCount)` per noob / stage / project |
| Task retry rate | `count(attemptNumber > 1) / count(tasks)` per stage |
| Human blocking ratio | `mean(humanBlockingMs / (humanBlockingMs + noobBlockingMs))` |
| Repeat failure rate | `count(tasks with same failureCategory on ≥2 attempts) / count(retried tasks)` |
| Brief quality correlation | Pearson r between `briefWordCount + briefHasGoal + briefHasConstraints` composite and `firstPassAccepted` |
| Skills impact delta | Diff of first-pass rate and avg cycles between runs with vs without each skill tag |

---

## 7.8 Repeat failure detection (Medium)
When `failureCategory` is populated, the system checks whether the same noob has failed for the same reason on a previous task. If so, a `repeatedFailureFlag` is set and surfaced as an insight in Analytics.

This is the highest-value Medium metric: it converts a raw retry rate into a prescription (e.g. "60% of Karen's Build failures are `schema_violation` — consider injecting the csharp-conventions skill").

---

## 7.9 Human interaction metrics

Measures the humans in the loop — their responsiveness, engagement quality, and contribution to outcomes. Captured against the human's userId and role, enabling cross-human and cross-role comparisons.

### 7.9.1 Per-thread human record (Easy / Medium)

| Field | Type | Difficulty | Description |
|-------|------|-----------|-------------|
| humanResponseMs | integer | Easy | Time from noob message posted to first human reply in this thread |
| dispatchMs | integer | Easy | Time from thread opened to "Dispatch to Agent" pressed — measures active review time. Requires `openedAt` timestamp captured in the UI. |
| nfaRate | float | Easy | Fraction of threads this human parks (NFA) vs engages with — high rate may indicate overload or disengagement |
| feedbackLength | integer | Easy | Word count of human replies — proxy for depth of engagement |
| messageVolume | integer | Easy | Number of active threads this human is carrying at any point in time |
| participationCoverage | float | Medium | Of all threads assigned to this human's stage/role, what fraction did they actually respond to — identifies humans who are nominally assigned but not engaging |
| bottleneckScore | float | Medium | This human's share of total `humanBlockingMs` across all threads they're involved in — surfaces who is most often the longest wait |
| feedbackConsistency | boolean | Medium | False if multiple humans in the same thread gave contradictory instructions (detected by comparing directive statements across replies in a thread) |
| reviewThoroughness | float | Medium | Correlation of `feedbackLength` with `revisionMagnitude` improvement — long feedback that doesn't improve outcomes scores low |

### 7.9.2 Derived human metrics (Phase 2)

| Metric | Derivation | Why it matters |
|--------|-----------|----------------|
| Avg response time | `mean(humanResponseMs)` per human | Identifies chronically slow responders |
| Dispatch vs NFA ratio | `count(dispatched) / count(nfa + dispatched)` | Signals engagement vs avoidance |
| Human bottleneck share | `sum(humanBlockingMs) / total pipeline humanBlockingMs` | Shows whose calendar is constraining delivery |
| Decision quality score | Cross-reference this human's approvals against downstream rework rate in later stages | Accountability for sign-offs. A BA whose specs consistently cause Build loops is visible here |
| Feedback effectiveness | `mean(revisionMagnitude improvement)` after this human's dispatches | Did the feedback move the needle? |

**Note on decision quality:** This metric is Hard because it requires semantic linking between a human's approval in stage N and rework events in stage N+1 or beyond. The Phase 1 data model must capture `approvedBy` and `approvedAt` on stage transitions to make this tractable in Phase 3.

---

## 7.10 Process metrics

Measures the pipeline itself — throughput, handoff quality, gate effectiveness, and trend over time. These are cross-cutting and project-scoped rather than tied to a single noob or human.

### 7.10.1 Per-stage process record (Easy / Medium)

| Field | Type | Difficulty | Description |
|-------|------|-----------|-------------|
| stageCycleMs | integer | Easy | Wall-clock time from stage activation to completion |
| gatePassRate | float | Easy | Fraction of commits passing the gate first time for this stage |
| loopCount | integer | Easy | How many times this stage sent work back to the previous stage |
| stallCount | integer | Easy | How many times this stage hit the stall threshold during this project |
| stallTotalMs | integer | Easy | Total time spent in stalled state across all stall events |
| velocityVsBaseline | float | Medium | `stageCycleMs / historicalMedianMs` — ratio > 1 means slower than baseline |
| handoffQualityScore | float | Medium | Correlation between this stage's output validation result and the retry rate of the next stage — does a soft-fail here predict loops downstream? |

### 7.10.2 Per-pipeline process record (Easy / Medium)

| Field | Type | Difficulty | Description |
|-------|------|-----------|-------------|
| totalPipelineDurationMs | integer | Easy | Discovery activation to final stage completion |
| bottleneckStage | string | Medium | Stage with highest `velocityVsBaseline` ratio, consistently across runs |
| processHealthTrend | enum | Medium | `improving` \| `stable` \| `degrading` — derived from rolling window of gate pass rates and retry rates |
| reworkFlowMatrix | object | Hard | For each stage pair (N, M), the count of issues found in stage M that originated in stage N — requires semantic cross-artefact linking |
| defectEscapeRate | float | Hard | Fraction of issues found in Review or DevOps that should have been caught by an earlier gate — requires semantic analysis |

### 7.10.3 Derived process metrics (Phase 2)

| Metric | Derivation | Why it matters |
|--------|-----------|----------------|
| Pipeline throughput | Features reaching Review completion per week | Top-line velocity |
| Gate effectiveness | For each gate: does passing predict downstream success? (`correlation(gatePass, nextStageFirstPassRate)`) | Identifies gates that are too loose or too strict |
| Bottleneck stage | `argmax(velocityVsBaseline)` across stages, per project | Where to focus process improvement |
| Cross-project stage comparison | `mean(stageCycleMs)` per stage across all projects | Surfaces which stage type is universally slow vs project-specific |
| Process health trend | Rolling 4-week window of `gatePassRate` and `loopCount` — slope positive/flat/negative | Leading indicator before problems become crises |

**Handoff quality** is the highest-value Medium metric here. If a soft-fail spec reliably predicts 30%+ more Build loops, that's a measured justification for tightening the Clarification gate — turning a subjective rule into an evidence-based threshold.

### 7.10.4 Phase 1 data model requirement

`approvedBy`, `approvedAt`, and `stageTransitionReason` must be recorded on every stage state change in Phase 1. These fields are the foundation for gate effectiveness, decision quality (section 7.9.2), and rework flow analysis. Retrofitting provenance onto stage transitions is hard.

---

## 7.11 Phase commitments

| Phase | Captures |
|-------|---------|
| Phase 1 | All Easy fields across noob (7.2–7.4), human (7.9), and process (7.10) records. Schema includes all Medium and Hard fields as nullable. `approvedBy`, `approvedAt`, `stageTransitionReason` captured on all stage changes. |
| Phase 2 | Medium fields active. All derived metrics in 7.7, 7.9.2, 7.10.3 live. Repeat failure detection (7.8) live. Analytics UI surfaces human and process views alongside noob view. |
| Phase 3 | Hard fields: judge-LLM for failure taxonomy, semantic cross-artefact linking for rework flow and defect escape, CLI token wrapper, instruction adherence and spec fidelity scores. |

`attemptCount` and `lastRetryReason` are also stored on individual task records in the tasks artefact (as before), making task-level retry data available without joining the telemetry store.


# 8. Stuck Detection

## 8.1 Rules

| Flag | Condition | Action |
|------|-----------|--------|
| message.isStale | Open with no activity for N hours (default: 48h) | Sort boost. Badge on message. |
| stage.isStalled | Active but no commits in N hours (default: 24h) | Badge on Stage view. In status summary. |

Phase 1: visibility only. No enforcement.


# 9. Source of Truth Notifications

| File | Notifies |
|------|----------|
| prd | Product Owner, BA, QA |
| spec | Developer, QA, Test Prep QA |
| tds | Developer, Test Prep QA |
| testcases | Developer, QA |
| test_results | DevOps |

SOT changes registered via Agent API. Dashboard display is a Notification (Section 5.9), not a message.

Phase 1: detection implemented. Notification display deferred to Phase 2.


# 10. Team and Role Model

## 10.1 Roles
Per-project. A user may hold multiple roles. Role membership drives stage stakeholder assignment automatically.

| Role | Colour | Stage |
|------|--------|-------|
| PM | #1A1A1A | All |
| Product Owner | #A23B72 | Discovery |
| Business Analyst | #2E86AB | Clarification |
| Architect | #F18F01 | Design |
| QA | #E84855 | Test Prep, Review |
| Developer | #3BB273 | Build |
| DevOps | #7B2D8B | DevOps |

## 10.2 Process Refiner (Phase 2)
Cross-cutting human (or hybrid human+agent) role. Read access to Analytics view across all projects. Identifies systemic patterns. Tunes Noob configurations and stage gates. The system surfaces signal; the Refiner provides interpretation.

## 10.3 Observers
Per-project. Read-only unless explicitly invited into a thread.


# 11. Views

## 11.1 Message View
Primary working view. Open messages across projects and stages.
- Filters: Assigned to me / Invited / All
- Sort: Staleness (default) / Project / Stage
- Oracle responses visible inline in threads, labelled as read-only

## 11.2 Stage View
Project and stage overview.
- Stage strip: one tile per stage — name, state, active Noob, message count, stall flag
- Stage drill-down: status summary, progress bars (Build/Review), artefact list, Noobs section, Hire button, Settings button

## 11.3 Analytics View (Phase 2)
Cross-cutting diagnostic view. Read-only. Powers the Process Refiner role. Scoped to all projects or a single project via the top project selector.

**All-projects view:** health scores, stall alerts, cross-project retry heatmap by stage, feedback cycle comparison, surfaced insights.

**Single-project view:** stage velocity (actual vs baseline), noob performance table (tasks completed, avg cycles, days active, first-pass rate, artefacts), artefact quality list, project-specific retry and feedback cycle breakdowns.

**Surfaced insights** are generated from the derived metrics in sections 7.7, 7.9.2, and 7.10.3, and the repeat failure detection in section 7.8. Insight severity levels: `alert` (stall / hard-fail / decision quality anomaly), `warn` (elevated retry rate / repeated failure / human bottleneck / degrading trend), `info` (positive signal / ahead of baseline).

The Analytics view covers three lenses, all scoped to the selected project or all projects:
- **Noob lens** — per-noob performance, retry rates, feedback cycles, artefact quality, stage velocity
- **Human lens** — response times, bottleneck scores, NFA rates, feedback effectiveness, participation coverage
- **Process lens** — pipeline throughput, gate pass rates, stage velocity vs baseline, handoff quality, health trend

Full metric definitions and capture difficulty tiers in section 7.

## 11.4 Diff View
Accessible from any message with a registered commit hash.
- JSON diff (structural)
- Rendered Markdown diff (human-readable)
- Side-by-side or toggled


# 12. Agent API

| Endpoint | Purpose |
|----------|---------|
| POST /api/agent/message | Post a message. Optionally tag roles. |
| POST /api/agent/commit | Register commit. Triggers validation. Returns pass/soft fail/hard fail. |
| GET /api/agent/task/{id} | Read task state: brief, phase, open messages, latest feedback. |
| GET /api/agent/feedback/{messageId} | Read latest feedback for a thread. |
| POST /api/agent/sot-change | Notify SOT file change. Triggers cross-stage invalidation. |
| POST /api/agent/telemetry | Record action telemetry. |
| GET /api/agent/snapshot/{noobId} | Read working-state snapshot (used by worker runner at start of run). |
| PUT /api/agent/snapshot/{noobId} | Write working-state snapshot (written by worker runner at end of run). |

### Commit registration

| Field | Required | Description |
|-------|----------|-------------|
| messageId | Yes | Mandatory coupling — commit rejected without it |
| commitHash | Yes | For diff generation |
| artefactFiles | Yes | Artefact paths changed in this commit |
| sotFiles | No | SOT files changed — triggers downstream notifications |


# 13. Artefact Schemas (Phase 1)

## 13.1 spec
```json
{
  "id": "string",
  "title": "string",
  "qualifier": "string",
  "version": "number",
  "updatedAt": "ISO8601",
  "requirements": [
    {
      "id": "string",
      "title": "string",
      "description": "string",
      "status": "open | resolved",
      "resolvedAt": "ISO8601 | null"
    }
  ]
}
```

## 13.2 tasks
```json
{
  "id": "string",
  "title": "string",
  "qualifier": "string",
  "specRef": "string",
  "version": "number",
  "updatedAt": "ISO8601",
  "tasks": [
    {
      "id": "string",
      "title": "string",
      "description": "string",
      "specRef": "string",
      "status": "defined | in-progress | implemented",
      "commitHash": "string | null",
      "attemptCount": "number",
      "lastRetryReason": "string | null"
    }
  ]
}
```

## 13.3 working-state snapshot
```json
{
  "noobId": "string",
  "currentTask": "string",
  "openQuestions": ["string"],
  "status": "waiting | active | blocked",
  "nextLikelyActions": ["string"],
  "updatedAt": "ISO8601"
}
```


# 14. Phase 1 Scope

## 14.1 In Scope
- C# minimal API + SQLite backend
- React/Vite frontend — single container, single port
- Username-only auth — role model fully implemented
- Full Noob lifecycle (all four states)
- Per-artefact concurrency enforcement
- Artefact naming: `{type}_{qualifier}` enforced at hire time
- Two-repo model: docs repo default; code repo on-demand checkout
- Worker agent execution: dockerised thin runner, validation outside container
- Working-state snapshot: written at end of worker run, read at start of next run
- Oracle mode: `@<noobname>` routes to read-only response; sees docs repo + named Noob's snapshot
- Chat model: normal message / @mention / Send Feedback — explicit, no auto-routing
- Message view and Stage view
- Collaborative human replies
- Send Feedback gate action
- NFA gate action
- Stage Noobs section
- Feedback entity (versioned, first-class)
- Agent API including snapshot and telemetry endpoints
- Commit ↔ message coupling enforcement
- Artefact validation: spec and tasks
- JSON diff + rendered Markdown diff view
- Stuck detection: stale message + stalled stage flags (background job)
- Telemetry recording for all agent actions (data captured; UI Phase 2)
- UAT feedback loops
- Test Prep stage: QA-owned, parallel with Design and Build, produces `testcases`
- Review stage revised: produces `test_results`, requires Build + Test Prep gate
- Artefact lifecycle: archive, reassignment, pre-creation
- `#` artefact mention syntax: autocomplete, chip rendering, artefactRefs on feedback
- `$` source file mention syntax: autocomplete, chip rendering, content injection into Noob context
- Cross-artefact write guard: Option A (`#` mentions) implemented; Options B/C undecided
- DevOps stage: UAT deploy surface, pipeline state, release notes, "Mark as released" action
- Notifications as distinct entity (data model; display surface Phase 2)
- Role colour coding
- Dark mode toggle
- Manual refresh — no real-time push
- Project identity: display name + git slug (auto-generated, immutable after creation)
- Docs repo: auto-created per project as local bare git repo at project creation
- Code repo: externally connected via URL + SSH deploy key; per-Noob branches (`noob/{slug}/{noob-name}`)
- Source files: human-uploaded per stage or project; text formats only; 5 MB limit; duplicate filename rejected; stored in docs repo

## 14.2 Explicitly Deferred to Phase 2
- Real authentication and login
- SOT notification display
- Generated stage / project status summaries
- Sub-stage progress bars (artefact parsing infrastructure)
- Real-time push (SignalR)
- PM agent integration
- Stuck detection escalation
- Validation of PRD, TDS, Test Case artefacts
- Human direct artefact editing
- Per-project stuck detection thresholds
- Analytics view (burndowns, retry heatmaps, loop frequency)
- Process Refiner role and cross-project metrics
- Production deploy automation
- Bug tracking (live production bugs)
- Notification display surface

## 14.3 Explicitly Out of Scope
- Jira integration (speculative; not a phase target)

## 14.4 Open Questions
- PM agent brief intake: text message, file upload, or git repo path?
- Generated summaries trigger: on new message, new commit, or both?
- UAT loop-back mechanism: new message in Build, or direct task append to tasks artefact? Who creates it — DevOps agent or human?
- Oracle response threading: does the oracle response appear as a Noob message in the thread, or as a distinct UI element?
- Telemetry push vs derived: does the agent push telemetry via API, or does the system derive it from commit/message timestamps?

---

> Humans decide. Agents mutate. System records.

> NOOB is opinionated. Fixed stages, fixed artefact schemas, fixed role-to-stage mapping. This is not a general-purpose agent framework — it is a c-SDLC implementation. The constraint is the feature.

> Agents are the only writers. Humans are the only deciders. The system is the record.
