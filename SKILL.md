---
name: ai-company-with-herdr
description: "Run a hierarchical AI agent company over Herdr: a Tech Lead session brainstorms and hands tasks to up to 3 parallel Senior Engineer panes that report through an inbox file and are closed as soon as their task finishes. Use when the user wants tasks delegated to named Herdr agents (tech lead, senior engineer, employees, reviewer), asks to hand off a task, or mentions the AI company workflow. Requires HERDR_ENV=1 and the herdr CLI."
---

# AI Company with Herdr

A company-style agent hierarchy on top of Herdr named-agent panes. One session brainstorms, other sessions execute.

## Roles

- **CEO**: the human. Talks only to the Tech Lead, makes decisions that matter.
- **Tech Lead**: the session the user is talking to. Brainstorms, writes handoffs, checks on staff. Never executes tasks itself.
- **Senior Engineer** (`senior-eng-1..3`, up to 3 in parallel): owns one task end to end, spawns employees, hires the reviewer.
- **Employees**: the senior's subagents or pane processes for parallel work.
- **Reviewer**: reviews every deliverable before DONE.

## Folder layout

All coordination artifacts live in one folder (default `~/Developer/AI-Company`), never in `/tmp`, so old files stay readable and agents can learn why decisions were made:

```
~/Developer/AI-Company/
  handoffs/   one markdown task brief per task
  inbox/      one markdown status file per task (STATUS/DONE lines)
  reports/    research and analysis deliverables
  decisions/  decision logs: what was decided and why
```

## The run

### 1. Brainstorm (Tech Lead + user only)

Discuss scope, challenge the task, agree on the goal. Write `~/Developer/AI-Company/handoffs/<slug>.md` containing: goal, scope, deliverable, acceptance checks, and the protocol block (below).

### 2. Hand off

```bash
# addressable name for QUESTION escalations
herdr agent rename "$HERDR_PANE_ID" tech-lead

# pick a free slot among senior-eng-1..3
herdr agent list
# split a sibling pane with the task cwd; note <pane-id> for cleanup in step 4
herdr pane split --current --direction right --cwd <task-cwd> --no-focus
herdr agent start senior-eng-<n> --kind grok --pane <pane-id>
# no --wait, keeps the Tech Lead free
herdr agent prompt senior-eng-<n> "Execute handoff: ~/Developer/AI-Company/handoffs/<slug>.md"
```

### 3. Check in (pull, not push)

The senior reports only through the inbox (protocol). The Tech Lead pulls before every new handoff and whenever the CEO asks:

```bash
herdr agent get senior-eng-<n>
herdr agent read senior-eng-<n> --source recent --lines 60
cat ~/Developer/AI-Company/inbox/<slug>.md
```

To wait in the background, watch the inbox file for a `DONE:` line instead of polling the pane.

### 4. Done and cleanup

A task is done when its inbox has a DONE line in the protocol format, including review evidence. Then the Tech Lead frees the slot:

```bash
herdr pane close <pane-id>
```

## Model routing

- **Reviewer**: always the strongest available model (e.g. `kimi-k3` or `glm-5-3`); a weak reviewer defeats the point of review.
- **Workers/employees**: prefer the strongest, fall back on rate limits (`kimi-k3` → `glm-5-3` → `glm-5-3-flash`); trivial mechanical work (navigation, lookups) goes straight to flash-tier.

## Handoff protocol block (paste into every handoff)

```markdown
## Protocol
- Append STATUS/DONE lines to ~/Developer/AI-Company/inbox/<slug>.md. Never prompt tech-lead for routine updates.
- QUESTION may prompt tech-lead via: herdr agent prompt tech-lead "QUESTION: ..."
- Close an employee's pane as soon as its part of the task finishes.
- Reviewer: strongest available model; record the reviewer model in DONE.
- All deliverables use absolute paths.
```

## Rules that keep it stable

- Cross-task context lives in the AI-Company folder, not in panes. Read old handoffs, inbox files, and decision logs before similar new tasks.
