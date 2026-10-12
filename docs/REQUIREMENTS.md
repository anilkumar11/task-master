# Task Master: Requirements (v0.4)

> Status: **MVP built.** The app, the agent rules and the daily scheduler are live. Section 10 lists what still needs checking.

## 1. Vision

This is a personal to-do list. I can add tasks, set their priority, give them detail, and hand any of them to a Claude agent. Each delegated task gets its own Claude session, which works on its own and stops only to ask me a critical question. I use it from the Claude app on my PC and on my phone, and it all stays in sync.

## 2. Pieces

| Piece | What it is | Where |
|-------|------------|-------|
| **Task Master app** | Private claude.ai artifact: the UI and the task database | https://claude.ai/artifact/9XBCgHprdE8jEVJRrBLaS3, source `app/index.html` |
| **Task agent** | One Claude Code cloud session per delegated task | Started from the app, or by the scheduler |
| **Agent rules** | How agents, the scheduler and chat sessions behave | `CLAUDE.md` |
| **Hourly pickup** | Claude routine, every hour: starts queued tasks, fills repeat gaps, writes exact costs | Routine "Task Master hourly pickup" |
| **Daily reminder** | Claude routine at 07:52 IST: push summary | Routine "Task Master daily reminder" |
| **Front desk** (optional) | Any Claude Code chat on this repo, for managing tasks by typing | Claude app → Code |

```
 Me ──► Task Master app (phone + PC)
          │  add / edit / prioritize / focus / answer / review
          │  private task database (owner-only access rule)
          │  "Delegate" ── Claude Code Remote connector ──► new Claude session per task
          ▼                                                    │
     tasks database ◄──── ArtifactData reads/writes ───────────┘ Task agent
          ▲                                                      (also asks me in-session)
          └──── Hourly pickup (start queued, repeats, exact cost) + daily 07:52 reminder
```

Out of scope: multiple users and sharing.

## 3. Task model

Stored as documents in the app's `tasks` collection. `CLAUDE.md` holds the exact schema.

| Field | Values |
|-------|--------|
| title, details | Text; details are markdown |
| priority | `P0` urgent · `P1` this week · `P2` normal (default) · `P3` someday |
| status | `todo` → `queued` → `in-progress` ⇄ `needs-input` → `review` → `done` |
| assignee | `me` or `claude` |
| type | code · research · writing · action (account actions) |
| due, repeat | Date; `daily` / `weekly` / `monthly` |
| session | Link to the agent's Claude session |
| question | The agent's open question, its options and the recommended one |
| cost | Run minutes, USD, and whether the figures are an estimate |
| history | Status changes, notes, questions and answers, actions, results (last 100 entries) |

## 4. Functional requirements

Key: ✅ built · ⏳ partly built · ☐ not built. **M** = must, **S** = should, **C** = could.

### 4.1 Add and edit
- ✅ **FR-1 (M)** Quick add from the app. `p0`–`p3`, `today` and `tomorrow` in the title set the priority or due date.
- ✅ **FR-2 (M)** Edit title, details, priority, due date, repeat and type. Changes save as you type.
- ✅ **FR-3 (S)** Add by typing in a Claude chat on this repo (front desk).
- ☐ **FR-4 (C)** Add by voice, or from an email or screenshot.

### 4.2 Prioritize and view
- ✅ **FR-5 (M)** Views: **Needs you**, **Open**, **With Claude**, **Done**. Tasks waiting on me sort first, then by priority, then by due date.
- ✅ **FR-6 (M)** Overdue and due-soon tasks are flagged in the list and in the header summary.
- ✅ **FR-7 (S)** "Focus today" asks Claude for a top 3 with reasons. It is a suggestion only and changes nothing.
- ✅ **FR-7a (M)** "Reprioritize" asks Claude to re-assess every open task's priority. The suggested changes appear with reasons; I untick any I disagree with, apply the rest, and can undo.
- ✅ **FR-7b (S)** The Open view is grouped by priority. Search, a one-tap complete circle, quick due dates (Today, Tomorrow, Next week), and undo for done and delete.

### 4.3 Delegate to Claude
- ✅ **FR-8 (M)** Any task type can be delegated. After an in-page confirmation, the app starts a new Claude Code session with the task's title and details and links it on the task.
- ✅ **FR-9 (M)** The agent acts autonomously, including outward actions, and logs each one in the task history.
- ✅ **FR-10 (M)** The agent finishes in `review`, never `done`. I accept the result, or send it back with a note that goes to the session.
- ✅ **FR-11 (S)** "Stop Claude" interrupts the session and hands the task back to me.
- ✅ **FR-12 (M)** Tasks queued without a session, such as repeats or failed starts, are started by the hourly pickup within an hour. Sessions run in auto mode.

### 4.4 Critical questions
- ✅ **FR-13 (M)** The agent asks only when:
  - there is ambiguity that changes the result;
  - it is missing access or information;
  - the work is much larger than described.

  For everything else it decides, notes why, and continues.
- ✅ **FR-14 (M)** A question sets `needs-input` and appears in **Needs you** as option buttons, with the recommended option marked. The agent also asks in its session, which notifies me in the Claude app.
- ✅ **FR-15 (M)** I answer from the app (sent to the session) or in the session itself. The agent never guesses while waiting.

### 4.5 Reminders and repeats
- ✅ **FR-16 (M)** The daily scheduler sends one push summary of overdue, due-today and due-tomorrow tasks.
- ✅ **FR-17 (M)** Marking a repeating task done creates the next copy. A copy of a Claude task is queued again for Claude.
- ✅ **FR-18 (S)** Snooze with quick due-date chips.

### 4.6 Cost
- ✅ **FR-19 (M)** Each delegated task shows run time and cost.
- ⏳ **FR-20 (M)** The agent writes an estimate, and the scheduler replaces it with the exact `total_cost_usd` from the session's record. The record shows the exact cost, but the scheduler hasn't been seen filling it in yet.
- ☐ **FR-21 (S)** Cost totals by week and by type.
- ☐ **FR-22 (C)** Cost limits. Deliberately off for now.

## 5. Non-functional requirements

- **NFR-1 Cross-device:** works in the Claude app on phone and PC with nothing to install. Phone layout: the task list, with the task detail as a full-screen sheet.
- **NFR-2 Single source of truth:** the app's database. Sessions are disposable, so anything important goes into the task.
- **NFR-3 Privacy:** the database access rule makes tasks readable and writable only by me, even if the page is shared. Secrets never go in tasks.
- **NFR-4 Auditability:** every outward agent action is logged in the task's history.
- **NFR-5 Low maintenance:** no servers. The system uses a claude.ai artifact, Claude Code cloud sessions and a Claude routine.

## 6. Example flows

**Delegate:** add "Find 3 home insurance quotes p1" → open it → add details → **Delegate to Claude** → **Start session**. The task moves to **With Claude** and gets a session link.

**Question:** the agent asks "Which coverage? A) Building only B) Building + contents (recommended) C) Other". The task appears in **Needs you** and the Claude app notifies me. I tap B, and the agent carries on.

**Review:** the agent posts a comparison and sets **Review**. I press **Accept and mark done**, or **Send back** with a note.

**Repeat:** "Pay rent, monthly" marked done → next month's copy appears.

## 7. Decisions log

| # | Question | Decision |
|---|----------|----------|
| 1 | Task types to delegate | All: code, research, writing, account actions |
| 2 | Agent runtime | One Claude session per task |
| 3 | GitHub UI acceptable | Initially yes; replaced by decision 8 |
| 4 | Notifications | Claude app notifications are enough |
| 5 | Cost | No limits at the start, but tracked and shown per task |
| 6 | Autonomy | The agent acts on its own. It asks only for ambiguity, missing information or scope growth, and logs every action |
| 7 | Recurring tasks and reminders | In the MVP |
| 8 | Interface | A real app UI. Built as a private claude.ai artifact, which is also the task store, because the page can't reach GitHub Issues |

## 8. Roadmap

| Phase | Scope | State |
|-------|-------|-------|
| 0. Setup | Rules, scheduler, connector test | Done |
| 1. MVP | App, delegation, questions, review, reminders, repeats, cost | Built; needs one real end-to-end delegation |
| 2. Polish | Snooze, cost totals, search in Done, batched questions | Next |
| 3. Extras | Voice capture, email-to-task, cost limits | Later |

## 9. Verified

- 2026-10-12: a task session can use Gmail ✅, Calendar ✅ and Drive ✅. For Drive it must use `search_files`, because `list_recent_files` isn't supported.
- 2026-10-12: the session record exposes the exact cost (`total_cost_usd`: $0.77 for the test task).
- 2026-10-12: the task database is hidden from view-level visitors; the owner can read and write it.

## 10. Open risks and checks

- **Agent writes to the app:** a delegated session needs the ArtifactData tool to update its task. Check this with the first task delegated from the app. If it fails, the agent still works and asks in its session, but the app won't show its progress.
- **Scheduler access:** the routine runs without connectors. A test run checks whether it can read tasks with ArtifactData.
- **Full autonomy:** a misread task can lead to a real outward action. Mitigations are the critical-question rules, the action log, and the option to add an "always ask before" list (e.g. payments, permanent deletion) to `CLAUDE.md`.
- **Public repo:** the code and rules are public, but tasks are not. Make the repo private if the rules themselves should stay private.
