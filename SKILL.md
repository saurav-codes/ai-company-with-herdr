---
name: ai-company-with-herdr
description: "Run a hierarchical AI agent company over Herdr: a Tech Lead session brainstorms with the user and hands off tasks to up to 3 parallel Senior Engineer agents that each execute with their own workers and a reviewer, communicating through handoff files and an inbox instead of typing into the user's terminal, and each agent's pane is closed as soon as its task finishes. Use when the user wants tasks delegated to named Herdr agents (tech lead, senior engineer, employees, reviewer), asks to hand off a task, or mentions the AI company workflow. Requires HERDR_ENV=1 and the herdr CLI."
---

# AI Company with Herdr

A company-style agent hierarchy on top of Herdr named-agent panes. One session brainstorms, other sessions execute. The user's input box stays sacred.

## Roles

- **CEO**: the human. Talks only to the Tech Lead, makes decisions that matter.
- **Tech Lead**: the session the user is talking to. Brainstorms, writes handoffs, checks on staff. Never executes tasks itself.
- **Senior Engineer** (`senior-eng-1..3`): Herdr agents that own execution end to end. Up to 3 run in parallel, one task each. The Tech Lead closes the pane as soon as the task is DONE, freeing the slot.
- **Reviewer**: reviews every deliverable before DONE. Always runs on the strongest model available.
- **Employees**: the Senior Engineer's subagents for parallel work. If an employee gets its own herdr pane, the senior closes it as soon as the employee's work finishes.

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

Discuss scope, challenge the task, agree on the goal. Write the handoff file:

`~/Developer/AI-Company/handoffs/<slug>.md` containing: goal, scope, deliverable with **absolute output path**, acceptance checks, and the protocol block (see below).

### 2. Hand off (fire and forget for the Tech Lead)

First make your session addressable, employees send `QUESTION:` to this name:

```bash
herdr agent rename "$HERDR_PANE_ID" tech-lead
```

```bash
# pick a free slot among senior-eng-1..3 (free = not listed or idle)
herdr agent list
# split a sibling pane with the right cwd for the task; note the pane id for cleanup
herdr pane split --current --direction right --cwd <task-cwd> --no-focus
# start the senior in that pane
herdr agent start senior-eng-<n> --kind grok --pane <pane-id>
# hand off without --wait so the Tech Lead session stays free
herdr agent prompt senior-eng-<n> "Execute handoff: ~/Developer/AI-Company/handoffs/<slug>.md"
```

The Tech Lead is now free to accept the next task.

### 3. Check in (pull, not push into the user's terminal)

- The Senior Engineer appends `STATUS:` and `DONE:` lines to `~/Developer/AI-Company/inbox/<slug>.md`. It never uses `herdr agent prompt tech-lead` for routine updates.
- `QUESTION:` is the only prompt allowed into the Tech Lead pane, when a real decision is needed. The Tech Lead answers or escalates to the CEO.
- The Tech Lead pulls status before every new handoff and whenever the CEO asks:

```bash
herdr agent list
herdr agent get senior-eng-<n>
herdr agent read senior-eng-<n> --source recent --lines 60
cat ~/Developer/AI-Company/inbox/<slug>.md
```

- To wait for completion in the background, watch the inbox file for a `DONE:` line instead of polling the pane.

### 4. Review and done

The Senior Engineer hires a reviewer for the final diff/deliverable before appending `DONE: <summary + verification evidence + reviewer model>`. A task without review evidence is not done.

Once the Tech Lead sees the `DONE:` line, it closes the senior's pane immediately: `herdr pane close <pane-id>`. The slot name (`senior-eng-<n>`) is then free for the next handoff. No pane outlives its task.

## Model routing

- **Reviewer**: always the strongest available model (e.g. `kimi-k3` or `glm-5-3`). Never flash-tier or deepseek; a weak reviewer defeats the point of review.
- **Workers**: prefer the strongest, fall back down the chain on rate limits (e.g. `kimi-k3` → `glm-5-3` → `glm-5-3-flash`). Trivial mechanical tasks go straight to flash-tier.
- **Cheap in-session subagents** (file navigation, lookups): flash-tier.

## Handoff protocol block (paste into every handoff)

```markdown
## Protocol
- Append STATUS/DONE lines to ~/Developer/AI-Company/inbox/<slug>.md. Never prompt tech-lead for routine updates.
- QUESTION may prompt tech-lead via: herdr agent prompt tech-lead "QUESTION: ..."
- Close an employee's pane as soon as its part of the task finishes.
- Reviewer runs on the strongest model available. Record the reviewer model in DONE.
- All deliverables use absolute paths.
```

## Rules that keep it stable

- Up to 3 senior engineers (`senior-eng-1..3`) run tasks in parallel; assign each handoff to a free slot.
- Panes are ephemeral: the Tech Lead closes a senior's pane on DONE and the senior closes an employee's pane on finish. Cross-task context lives in the AI-Company folder, not in panes.
- Old handoffs, inbox files, and decision logs are the company memory: read them before similar new tasks.
