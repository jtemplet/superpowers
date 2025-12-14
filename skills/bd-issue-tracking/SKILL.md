---
name: bd-issue-tracking
description: Use when work spans multiple sessions, has complex dependencies, or needs persistence across compaction - tracks issues with dependency graphs. TodoWrite for simple single-session tasks.
---

# bd Issue Tracking

## Overview

bd is a graph-based issue tracker for persistent memory across sessions. Use for multi-session work with complex dependencies; use TodoWrite for simple single-session tasks.

## Quick Start

**Every session start:**
1. `bd ready` - see available work
2. `bd list --status in_progress` - check active work
3. `bd show <issue-id>` - read notes for context
4. Report to user: "X items ready: [summary]"

**During work:**
- Update notes at milestones (70% token usage, blockers, major progress)
- Create issues for discovered work

**Critical:** Notes are your only memory after compaction. Write as if explaining to someone with zero conversation context.

## When to Use bd vs TodoWrite

### Use bd when:
- Multi-session work spanning multiple compaction cycles or days
- Complex dependencies with blockers or prerequisites
- Knowledge work (research, strategic documents, fuzzy boundaries)
- Side quests or exploratory work that might pause
- Need project memory to resume after weeks away

### Use TodoWrite when:
- Single-session tasks completing within current session
- Linear execution with no branching
- All information already in conversation
- Just need checklist to show progress

**Decision test:**
- "Will I need this context in 2 weeks?" Yes = bd
- "Could conversation history get compacted?" Yes = bd
- "Does this have blockers/dependencies?" Yes = bd
- "Will this be done in this session?" Yes = TodoWrite

**When in doubt: Use bd.** Better to have persistent memory you don't need than lose context you needed.

## Surviving Compaction

**Critical:** Compaction deletes conversation history but preserves beads.

**What survives:**
- All bead data (issues, notes, dependencies, status)
- Complete work history

**What doesn't survive:**
- Conversation history
- TodoWrite lists
- Recent discussion

**Notes format for post-compaction recovery:**
```
COMPLETED: Specific deliverables ("implemented JWT refresh endpoint + rate limiting")
IN PROGRESS: Current state + next step ("testing password reset, need user input on email template")
BLOCKERS: What's preventing progress
KEY DECISIONS: Important context or user guidance
NEXT: Immediate next action
```

**Notes quality check:**
- Could future-me resume with zero conversation history?
- Could another developer understand without asking?
- Are technical choices explained (not just stated)?
- Are user decisions captured?

**Good example:**
```
COMPLETED: JWT auth with RS256 (1hr access, 7d refresh tokens)
KEY DECISION: RS256 over HS256 per security review - enables key rotation
IN PROGRESS: Password reset flow - email service working, need rate limiting
BLOCKERS: Waiting on user: reset token expiry (15min vs 1hr trade-off)
NEXT: Implement rate limiting (5 attempts/15min) once expiry decided
```

## Progress Checkpointing

Update bd notes at these triggers:

**Critical triggers:**
- Token budget > 70% (proactively checkpoint)
- Token budget > 90% (automatically checkpoint)
- Major milestone reached
- Hit a blocker
- Task transition
- Before asking user for decision

**Checkpoint pattern:**
```bash
bd update <issue-id> --notes "COMPLETED: [what's done]
KEY DECISION: [important context]
IN PROGRESS: [current state]
BLOCKERS: [what's preventing progress]
NEXT: [immediate action]"
```

**Test:** "If compaction happened now, could future-me resume from these notes?"

## Session Start Protocol

bd auto-discovers database:
- Uses `.beads/*.db` in current project if exists
- Falls back to `~/.beads/default.db` otherwise

**Session start checklist:**
```
- [ ] Run bd ready to see available work
- [ ] Run bd list --status in_progress for active work
- [ ] If in_progress exists: bd show <issue-id> to read notes
- [ ] Report to user: "X items ready: [summary]"
- [ ] If nothing ready: bd blocked to check blockers
```

**Report pattern:** Establish shared context immediately without requiring user prompting.

## Core Operations

**Check ready work:**
```bash
bd ready
bd ready --priority 0           # Filter by priority
bd ready --assignee alice       # Filter by assignee
```

**Create issue (always quote titles, always include a description):**
```bash
bd create "Add password visibility toggle" \
  -d "Problem Statement: Users can't verify their password input on the login form, leading to failed logins.

Acceptance Criteria:
- Users can toggle password visibility on the login form.
- Default state keeps password masked.
- No passwords are logged or exposed in analytics." \
  -p 1 -t feature

bd create "Fix login 500 error" \
  -d "Problem Statement: Logging in with valid credentials sometimes redirects to a 500 error.

Acceptance Criteria:
- Valid logins consistently redirect to /dashboard.
- Errors are logged with correlation IDs." \
  -p 0 -t bug
```

**Update status:**
```bash
bd update issue-123 --status in_progress
bd update issue-123 --notes "COMPLETED: [summary]. IN PROGRESS: [current]. NEXT: [action]"
```

**Close work:**
```bash
bd close issue-123
bd close issue-123 --reason "Implemented in PR #42"
```

**Show details:**
```bash
bd show issue-123               # Read notes for context
```

**List issues:**
```bash
bd list --status open
bd list --status in_progress
bd list --priority 0
```

**Check project health:**
```bash
bd stats                        # Total, open, closed, blocked, ready
bd blocked                      # Find blocked work
```

## Dependencies

**Four dependency types:**

1. **blocks** - Hard blocker (A blocks B from starting)
2. **related** - Soft link (related but not blocking)
3. **parent-child** - Hierarchical (epic/subtask)
4. **discovered-from** - Provenance (B discovered while working on A)

**Usage:**
```bash
bd dep add task-id dependency-id       # task depends on dependency (dependency blocks task)
bd dep add subtask auth-epic --type parent-child
bd dep add found-bug main-task --type discovered-from
bd dep tree issue-123                  # Visualize dependencies
```

**Examples:**
```bash
# Example: Audit must complete before Conversion can start
bd dep add conversion-task audit-task  # conversion depends on audit
# Result: audit-task blocks conversion-task
# bd ready will show audit-task as ready first

# Example: Task B needs Task A done first
bd dep add B A                         # B depends on A (A must complete first)
```

## Verifying Dependencies

After adding dependencies, verify they're correct using `bd show`:

**Reading bd show output:**
```bash
bd show task-id
```

**Output interpretation:**
```
Depends on (2):                        # Tasks that block THIS task
  → dependency-1: Title [P0]           # → arrow: this task depends on these
  → dependency-2: Title [P1]           # Must complete before this task can start

Blocks (3):                            # Tasks that THIS task blocks
  ← blocked-task-1: Title [P0]        # ← arrow: this task blocks these
  ← blocked-task-2: Title [P1]        # These cannot start until this completes
  ← blocked-task-3: Title [P2]
```

**Arrow meanings:**
- `→` (right arrow): Dependencies - tasks this one depends on (incoming blockers)
- `←` (left arrow): Dependents - tasks this one blocks (outgoing blocks)

**Verification checklist:**
1. Run `bd show task-id` for the task you added dependencies to
2. Check "Depends on" section - does this task correctly depend on those tasks?
3. Check "Blocks" section - does this task correctly block those tasks?
4. Run `bd ready` - are the right tasks showing as ready?
5. If wrong, trace through the dependency chain with `bd show` on each task

**Example verification:**
```bash
# After: bd dep add conversion audit
bd show conversion
# Should show:
#   Depends on (1):
#     → audit: Audit landing page [P0]
# Meaning: conversion is blocked by audit

bd show audit
# Should show:
#   Blocks (1):
#     ← conversion: Convert to JAMStack [P0]
# Meaning: audit blocks conversion
```

## Planning and Ordering Work

**Encoding precedence (dependencies + priority):**

- If task B cannot start until task A is complete, always model that as a **blocks** dependency:
  ```bash
  bd dep add b-id a-id   # B depends on A (A blocks B, A must complete first)
  ```
- Use **priority** alongside dependencies:
  - Priority 0 = critical / do next once unblocked
  - Priority 1 = high
  - Priority 2 = normal
  - Priority 3 = low / nice-to-have

**Example mini-project:**
```bash
# Epic (overall goal)
bd create "Improve login UX" -t epic -p 1 -d "Problem Statement: Login failures are high...

Acceptance Criteria:
- Login-related support tickets decrease by 30%."

# Task 001: backend work
bd create "Harden login error handling" -t task -p 0 -d "Problem Statement: Valid logins sometimes result in 500 errors.

Acceptance Criteria:
- Valid logins consistently redirect to /dashboard.
- All login errors are logged with correlation IDs."

# Task 002: password visibility toggle (depends on 001)
bd create "Add password visibility toggle" -t task -p 0 -d "Problem Statement: Users can't verify their password input on the login form, leading to failed logins.

Acceptance Criteria:
- Users can toggle visibility of password input on login form.
- Default state is masked.
- No passwords are logged or exposed in analytics."

# Encode precedence and hierarchy
bd dep add login-002 login-001                 # 002 depends on 001 (001 blocks 002)
bd dep add login-001 login-epic --type parent-child
bd dep add login-002 login-epic --type parent-child
```

**Choosing what to work on next:**

When resuming a session:

1. Run:
   ```bash
   bd ready --json
   ```
2. Consider tasks where all **blocking dependencies are closed**; treat these as "ready-ready".
3. Within that set, order work by:
   - priority ascending (0 → 3)
   - then age (oldest first) or your judgment
4. Start with the top item. When reporting to the user, summarize as:
   - "Recommended order: 1) <id> (P0) – <short description>, 2) <id> (P1) – ..."

## Issue Lifecycle

**1. Discovery (proactive creation):**
```bash
# Found issue during work:
bd create "Found: auth doesn't handle profile permissions"
bd dep add new-issue current-task --type discovered-from
# Continue with original task
```

**2. Execution:**
```bash
bd update issue-123 --status in_progress
# Work on task, update notes at milestones
bd update issue-123 --notes "COMPLETED: [summary]..."
bd close issue-123 --reason "Implementation complete with tests"
```

**3. Planning (complex work):**
```bash
# Create parent epic
bd create "Implement user authentication" -t epic

# Create subtasks
bd create "Set up OAuth credentials" -t task
bd create "Implement authorization flow" -t task

# Link with dependencies
bd dep add auth-setup auth-epic --type parent-child
bd dep add auth-flow auth-setup    # auth-flow depends on auth-setup (auth-setup blocks auth-flow)
```

## Integration with TodoWrite

**Temporal layering pattern:**

**TodoWrite (short-term - this hour):**
- Tactical execution: "Review Section 3", "Expand Q&A"
- Marked completed as you go
- Present/future tense
- Ephemeral: disappears at session end

**Beads (long-term - this week/month):**
- Strategic objectives: "Continue strategic planning document"
- Key decisions and outcomes in notes
- Past tense in notes
- Persistent: survives compaction

**Handoff pattern:**
1. Session start: Read bead → Create TodoWrite items
2. During work: Mark TodoWrite items completed
3. Reach milestone: Update bead notes with outcomes
4. Session end: TodoWrite disappears, bead survives

**Don't duplicate:** TodoWrite tracks execution, Beads captures meaning and context.

## Common Patterns

**Pattern 1: Knowledge Work Session**
```bash
# Session start
bd ready
# Returns: bd-42 "Research analytics expansion proposal" (in_progress)

bd show bd-42
# Notes show: "IN PROGRESS: Drafting cost-benefit analysis"

# Create TodoWrite for immediate work, mark completed as you go

# At milestone:
bd update bd-42 --notes "COMPLETED: Cost-benefit drafted.
KEY DECISION: User confirmed $50k budget - ruled out enterprise options.
IN PROGRESS: Finalizing recommendations.
NEXT: Get user review before closing."
```

**Pattern 2: Side Quest**
1. Discover problem during main task
2. Create issue: `bd create "Found: inventory needs refactoring"`
3. Link: `bd dep add new-issue main-task --type discovered-from`
4. Assess: blocker or defer?
5. If blocker: `bd update main-task --status blocked`, work new issue
6. If deferrable: note in issue, continue main task

**Pattern 3: Multi-Session Resume**
1. `bd ready` - available work
2. `bd blocked` - what's stuck
3. `bd list --status closed --limit 10` - recent completions
4. `bd show issue-id` - read context
5. Update status and begin

## Issue Creation Guidelines

**Quick checklist:**
- Title: Clear, specific, action-oriented
- Description: Immutable Problem Statement + 1–3 lines of high-level Acceptance Criteria (write for future-you)
- Design: HOW to build (can change)
- Acceptance: Detailed success checklist (can also live in `acceptance-criteria` field)
- Priority: 0=critical, 1=high, 2=normal, 3=low
- Type: bug/feature/task/epic/chore

**Description template (copy into `-d` when creating issues):**
```text
Problem Statement: [1–2 sentences explaining the underlying problem and context]

Acceptance Criteria:
- [outcome-focused bullet 1]
- [outcome-focused bullet 2]
```

**Example:**
```bash
bd create "Add password visibility toggle" \
  -d "Problem Statement: Users can't verify their password input on the login form, leading to failed logins.

Acceptance Criteria:
- Users can toggle password visibility on the login form.
- Default state keeps password masked.
- No passwords are logged or exposed in analytics." \
  -p 1 -t feature
```

**Acceptance criteria test:**
"If I changed implementation, would these criteria still apply?"
- Yes = Good criteria (outcome-focused)
- No = Move to design field (implementation-focused)

**Example:**
- Good: "User tokens persist across sessions and refresh automatically"
- Wrong: "Use JWT tokens with 1-hour expiry" (that's design)

## Field Usage

| Field | Purpose | When to Set | Update Frequency |
|-------|---------|-------------|------------------|
| **description** | Immutable Problem Statement + brief Acceptance Criteria | At creation | Never |
| **design** | Initial approach, architecture | During planning | Rarely |
| **acceptance-criteria** | Deliverables checklist | When design clear | Mark completed |
| **notes** | Session handoff (COMPLETED/IN_PROGRESS/NEXT) | During work | At milestones |
| **status** | Workflow state (open/in_progress/closed) | As work progresses | When changing phases |
| **priority** | Urgency (0=highest, 3=lowest) | At creation | If priorities shift |

**Priority display formats:**
- Command line: `0`, `1`, `2`, `3` (when setting with `-p` or `--priority`)
- Human output: `P0`, `P1`, `P2`, `P3` (in `bd list`, `bd ready` output)
- bd show output: `[P0]`, `[P1]`, `[P2]`, `[P3]` (in brackets)
- JSON output: `"priority": 0` (numeric value)

**Priority meanings:**
- **P0 (priority 0)**: Critical - security issues, data loss, broken builds, production outages
- **P1 (priority 1)**: High - major features, important bugs, blocking issues
- **P2 (priority 2)**: Normal - standard work items, nice-to-have features (default)
- **P3 (priority 3)**: Low - polish, optimization, backlog ideas, future considerations

## Advanced Features

**JSON output (all commands):**
```bash
bd ready --json
bd show issue-123 --json
bd stats --json
```

### Practical JSON + jq Examples

**List all P0 tasks with titles:**
```bash
bd list --status open --priority 0 --json | jq -r '.[] | "\(.id) | P\(.priority) | \(.title)"'
```

**Count tasks by status:**
```bash
bd list --status open --json | jq -r 'length'
```

**Get task IDs for scripting:**
```bash
bd ready --json | jq -r '.[].id'
```

**Find tasks with dependencies:**
```bash
bd list --status open --json | jq -r '.[] | select(.dependency_count > 0) | "\(.id): \(.dependency_count) deps"'
```

**Extract specific fields:**
```bash
bd show task-id --json | jq -r '.description'
bd show task-id --json | jq -r '.notes'
```

**Filter by multiple criteria:**
```bash
bd list --json | jq -r '.[] | select(.priority == 0 and .status == "open") | .id'
```

**Bulk operations (with confirmation):**
```bash
# List all P2 tasks, then close them
bd list --priority 2 --json | jq -r '.[].id' | while read id; do
  echo "Closing $id"
  bd close "$id" --reason "Bulk cleanup"
done
```

**Bulk operations:**
```bash
bd close issue-1 issue-2 issue-3 --reason "Completed in sprint 5"
```

**Built-in help:**
```bash
bd quickstart                   # Comprehensive guide
bd create --help                # Command-specific help
```

## Troubleshooting

**Issues seem lost:** Use `bd list --status closed` (closed issues remain in database permanently)

**Dependencies wrong:** `bd show issue-id` shows full tree. Dependencies are directional: `bd dep add task-id dependency-id` means task depends on dependency (dependency blocks task, dependency must complete first).

**Database selection:** bd auto-discovers `.beads/*.db` in project or `~/.beads/default.db`. Use `--db /path/to/db` for explicit selection.

## Golden Rule

Write notes as if explaining to future-you with zero conversation context. After compaction, notes are your only memory.
