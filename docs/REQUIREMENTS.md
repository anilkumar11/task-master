# Task Master: Requirements (v0.2)

> Status: **Decisions made, ready to build phase 0.** See section 9 for what changed since v0.1.

## 1. Vision

This is a personal to-do list. I can add tasks, set their priority, give them detail, and hand any of them to a Claude agent. Each delegated task gets its own Claude session, which works on its own and stops only to ask me a critical question. I use it from the Claude app on my PC and on my phone, and it all stays in sync.

## 2. Users and surfaces

| Who | Surface | What they do |
|-----|---------|--------------|
| Me (owner, the only human user) | Claude app, desktop | Add and edit tasks, review agent work, answer questions |
| Me | Claude app, mobile | Capture tasks quickly, set priority, answer agent questions |
| Me | GitHub (web or mobile) | An acceptable fallback interface: tasks are GitHub Issues |
| **Front-desk Claude** | Any Claude Code session on this repo | Turns my plain-language requests into issues and starts agent sessions |
| **Task agent** | One Claude Code cloud session per delegated task | Does the work, asks questions, reports results and cost |
| **Scheduler** | A daily Claude routine | Sends due-date reminders, creates recurring tasks, picks up queued tasks |

Out of scope: multiple users and sharing.

## 3. Core concepts

### 3.1 Task fields (GitHub Issue)

| Field | Where it lives | Notes |
|-------|----------------|-------|
| `id` | Issue number | `#42` |
| `title` | Issue title | One line |
| `details` | Issue body | Context, Done-when, Workspace (from the issue template) |
| `priority` | Label `P0`–`P3` | Default `P2` |
| `status` | Open/closed plus a status label | See 3.2 |
| `assignee` | Label `assignee:claude` | No label means the task is mine |
| `type` | Label `type:code` / `type:research` / `type:writing` / `type:action` | Tells the agent which tools it needs |
| `due` | `Due: YYYY-MM-DD` line in the body | Read by the scheduler |
| `recurrence` | `Repeat: daily / weekly:Mon / monthly:1` line in the body | Read by the scheduler |
| `session` | `Session:` line in the body | Link to the agent's Claude session |
| `cost` | `Cost:` section in the body | Updated by the agent (see 4.7) |
| `history` | Issue comments | Progress, questions and answers, actions taken, results |

### 3.2 Task lifecycle

```
 [todo] ──delegate──► [queued] ─► [in-progress] ─► [review] ─► [done (closed)]
                                      │   ▲           │
                                      ▼   │ answer    │ "revise: …"
                                 [needs-input]        ▼
                                                [in-progress]
```

Status labels: `queued`, `in-progress`, `needs-input`, `review`. A closed issue means the task is done or was cancelled.

## 4. Functional requirements

Priority key: **M** = MVP, **S** = should, **C** = later.

### 4.1 Add tasks
- **FR-1 (M)** I add a task in plain language from the Claude app, e.g. "add: renew passport, P1, due Friday". Front-desk Claude creates the issue with the right labels and fields and confirms with its link.
- **FR-2 (M)** I can also add a task from GitHub using the issue template.
- **FR-3 (C)** Add by voice, or from an email or screenshot.

### 4.2 Prioritize
- **FR-4 (M)** Set and change priority (P0–P3) from the Claude app or GitHub.
- **FR-5 (M)** "Show my tasks" lists open tasks by priority, then due date, with `needs-input` tasks always first.
- **FR-6 (S)** "What should I focus on today?" returns a suggested top 3 with reasons. It is a suggestion only and changes nothing.

### 4.3 Details
- **FR-7 (M)** Edit markdown details on any task.
- **FR-8 (S)** Checklists inside details. The agent ticks items off as it completes them.

### 4.4 Delegate to Claude (one session per task)
- **FR-9 (M)** Any task type can be delegated: code, research, writing, or actions in my accounts (email, calendar, Drive).
- **FR-10 (M)** Delegating adds the `assignee:claude` and `queued` labels. Front-desk Claude then **starts a new Claude Code cloud session for that task** and writes the session link into the issue.
- **FR-11 (M)** If I label a task from GitHub instead of from Claude, the scheduler starts a session for it on its next run.
- **FR-12 (M)** The agent works **autonomously**, including outward-facing actions (merging, sending, publishing, booking). Every outward action is logged as a comment on the issue, saying what was done and where.
- **FR-13 (M)** When it finishes, the agent posts a result summary (what changed, links, actions taken) and moves the task to `review`. It never closes the issue itself.
- **FR-14 (M)** I review the task: closing the issue means done, and commenting "revise: …" sends it back to the agent.
- **FR-15 (S)** I can stop an agent ("stop #12"). Its session is interrupted and the task goes back to `todo`.

### 4.5 Agent asks critical questions
- **FR-16 (M)** The agent asks **only** when:
  1. there is **ambiguity that changes the result**: two reasonable readings give materially different outcomes;
  2. it is **missing access or information**: a credential, a file, or a decision only I can make;
  3. the work turns out **much larger** than the task describes.

  Anything else it decides itself, notes the choice in the issue, and keeps going.
- **FR-17 (M)** The agent asks in its own Claude session, so I get a notification in the Claude app on my phone or PC. It also mirrors the question to the issue and sets `needs-input`.
- **FR-18 (M)** I answer in the session from either device. The agent mirrors my answer to the issue, removes `needs-input`, and carries on.
- **FR-19 (M)** Each question is answerable on a phone: one question at a time, enough context to answer without scrolling, lettered options with a recommended one, e.g. `A) … B) … (recommended) C) other`.
- **FR-20 (M)** While it waits, the agent never guesses or times out into a default. The task stays `needs-input`.
- **FR-21 (S)** Questions that come up at the same moment are sent as one batch.

### 4.6 Reminders and recurring tasks
- **FR-22 (M)** **Due-date reminders.** Each morning the scheduler finds open tasks that are due today, overdue, or due tomorrow, and sends me one summary notification.
- **FR-23 (M)** **Recurring tasks.** When a task with a `Repeat:` line is closed, the scheduler creates the next one with the same title, details, labels and assignee and the next due date. A recurring task assigned to Claude is queued automatically.
- **FR-24 (S)** Snooze: "snooze #12 to Monday" changes the due date.

### 4.7 Cost tracking (per task)
- **FR-25 (M)** Each delegated task shows its cost in a `Cost:` section of the issue body. The agent updates it at every status change and at the end.
- **FR-26 (M)** The cost record holds: the session link, active run time, and cost in USD (or tokens) where the platform reports it. Where it doesn't, the record shows run time and is marked `estimate`.
- **FR-27 (S)** "Show cost" sums cost across tasks by week and by type.
- **FR-28 (C)** Cost limits. Not enforced at the start, by my choice, but the fields exist so limits can be added later.

### 4.8 Views (via front-desk Claude, or GitHub label filters)
- **FR-29 (M)** **Needs my input**: label `needs-input`. The most important view.
- **FR-30 (M)** **My open tasks**: open issues without `assignee:claude`, sorted by priority.
- **FR-31 (M)** **Agent activity**: `assignee:claude` with `queued`, `in-progress` or `review`.
- **FR-32 (C)** Dashboard: a private claude.ai Artifact showing these views (phase 3).

## 5. Non-functional requirements

- **NFR-1 Cross-device:** Every MVP action works from the Claude app on phone and PC with nothing to install.
- **NFR-2 Single source of truth:** GitHub Issues in this repo hold all tasks. Sessions are disposable; anything important goes into the issue.
- **NFR-3 Privacy:** This repo stays **private**.
- **NFR-4 Secrets:** Secrets never go in issues. The agent uses connectors and environment secrets.
- **NFR-5 Auditability:** Because the agent acts alone (FR-12), every outward action is logged on its issue.
- **NFR-6 Low maintenance:** No servers. The system uses only GitHub, Claude Code cloud sessions, and Claude routines.

## 6. Architecture

```
 Me (Claude app: phone / PC)
   │  "add …", "delegate #12", answers
   ▼
 Front-desk Claude session ──creates/edits──► GitHub Issues (task-master)
   │                                            ▲          ▲
   └──starts one session per task──► Task agent ┘          │
                                     (asks me in-session,   │
                                      mirrors to issue)     │
 Daily scheduler routine ── reminders / recurring / pick up queued
```

- **The rules live in the repo.** `CLAUDE.md` defines the front-desk and task-agent behaviour, the question format, and the cost reporting. Every session reads it automatically.
- **Connectors:** delegated `type:action` tasks need Gmail, Calendar and Drive. *Phase 0 must check that sessions started for a task actually have access to these connectors.*

## 7. Example flows

**Capture and delegate on mobile**
> Me: "Add task: find 3 home insurance quotes, P1, due Oct 20, delegate."
> Front desk: creates #12 (`P1`, `type:research`, `assignee:claude`, `queued`), starts a session, and replies "Started: <session link>".

**Critical question**
> Agent (notification on my phone): "❓ #12: Which coverage should I compare? A) Building only B) Building + contents (recommended, matches current policy) C) Other."
> I reply "B". The agent mirrors the Q&A to #12 and continues.

**Result**
> Agent: comments with a comparison table, sets `review`, and records `Cost: 18 min · $0.42 · <session link>`. I close #12.

**Recurring**
> #20 "Pay rent", `Repeat: monthly:1`. I close it on the 2nd and the scheduler creates #31 due on the 1st of next month.

## 8. Phased roadmap

| Phase | Scope | Exit criteria |
|-------|-------|---------------|
| **0. Setup** | Labels, issue template, `CLAUDE.md` rules. Check that sessions started per task get the connectors | I add a labelled task from my phone through Claude |
| **1. MVP** | FR-1, 2, 4, 5, 7, 9–14, 16–20, 22, 23, 25, 26, 29–31. Scheduler routine | One task of each type delegated end to end, with one question answered on my phone |
| **2. Polish** | Focus-today, stop/snooze, batched questions, cost rollup, checklists | Used every day for 2 weeks |
| **3. Dashboard** | claude.ai Artifact showing the views | Triage everything from one screen |
| **4. Extras** | Voice, email capture, cost limits | Nice to have |

## 9. Decisions log

| # | Question | Decision |
|---|----------|----------|
| 1 | Task types to delegate | **All**: code, research, writing, account actions |
| 2 | Agent runtime | **One Claude session per task** |
| 3 | GitHub UI acceptable for phase 1 | **Yes** |
| 4 | Notifications | **Claude app and GitHub notifications are enough** |
| 5 | Cost | **No limits at the start**, but **cost is tracked and shown per task** |
| 6 | Autonomy | **The agent acts on its own**, including outward actions. It asks only for ambiguity, missing information or scope growth, and logs every action |
| 7 | Recurring tasks and reminders | **Yes, in the MVP** |

## 10. Known risks

- **Cost visibility:** a session may not be able to read its own exact cost. FR-26 falls back to run time marked `estimate`.
- **Connector access in task sessions:** must be checked in phase 0. If it fails, `type:action` tasks run from the front-desk session instead.
- **Full autonomy:** a misread task can lead to a real outward action. Mitigated by the critical-question rules (FR-16), the action log (NFR-5), and the option to add an "always ask before" list in `CLAUDE.md` later.
