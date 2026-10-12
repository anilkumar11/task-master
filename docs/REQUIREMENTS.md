# Task Master: Requirements (Draft v0.1)

> Status: **Draft for review.** Section 9 lists the open questions. Answer those before any code is written.

## 1. Vision

This is a personal to-do list. I can add tasks, set their priority, give them detail, and hand some of them to a Claude agent that does the work. While it works, the agent can stop and ask me a question when it needs a decision only I can make. I use it from the Claude app on my PC and on my phone, and it all stays in sync.

## 2. Users and surfaces

| Who | Surface | What they do |
|-----|---------|--------------|
| Me (owner, the only human user) | Claude app on desktop (Windows/Mac/web) | Add and edit tasks, review agent work, answer questions |
| Me | Claude app on mobile (iOS/Android) | Capture tasks quickly, set priority, answer agent questions, approve results |
| Claude agent | Cloud (Claude Code on the web) | Pick up delegated tasks, do the work, ask questions, report results |

Out of scope for v1: multiple users, teams, sharing tasks with other people.

## 3. Core concepts

### 3.1 Task fields

| Field | Required | Notes |
|-------|----------|-------|
| `id` | auto | Stable, short (e.g. `#42`) |
| `title` | yes | One line |
| `details` | no | Free-form markdown: context, links, acceptance criteria, sub-steps |
| `priority` | yes (default P2) | `P0` urgent/today · `P1` this week · `P2` normal · `P3` someday |
| `status` | auto | See 3.2 |
| `assignee` | yes (default `me`) | `me` or `claude` |
| `due` | no | Date |
| `tags` | no | e.g. `home`, `work`, `code` |
| `repo` / `workspace` | only when delegated | Where the agent may work (a GitHub repo, a doc, or "research only") |
| `history` | auto | Status changes, agent questions and answers, results |

### 3.2 Task lifecycle

```
            ┌──────────── (I do it) ───────────────┐
            │                                      ▼
 [todo] ──delegate──► [queued] ─► [in-progress] ─► [review] ─► [done]
            ▲                         │   ▲           │
            │                         ▼   │ answer    │ reject / revise
            │                    [needs-input]        │
            └─────────────────── [cancelled] ◄────────┘
```

- `needs-input`: the agent is blocked on a question from me. This status has to be impossible to miss on every surface.
- `review`: the agent believes the task is finished. I accept it (→ `done`) or send it back with comments (→ `in-progress`).

## 4. Functional requirements

Priority key: **M** = must for MVP, **S** = should, **C** = could, later.

### 4.1 Add tasks
- **FR-1 (M)** Add a task with just a title in one step, from desktop or mobile.
- **FR-2 (S)** Add from plain language, e.g. "remind me to renew passport, high priority, due Friday" becomes title, priority and due date.
- **FR-3 (C)** Add by voice on mobile, or from an email or screenshot.

### 4.2 Prioritize
- **FR-4 (M)** Set and change a task's priority (P0–P3).
- **FR-5 (M)** The default view sorts open tasks by priority, then due date.
- **FR-6 (S)** Ask Claude "what should I focus on today?" and get a suggested top 3 with reasons. It never reorders anything without my say-so.

### 4.3 Details
- **FR-7 (M)** Add and edit markdown details on any task.
- **FR-8 (S)** Add checklists or sub-tasks inside details.
- **FR-9 (C)** Attach files or links.

### 4.4 Delegate to Claude
- **FR-10 (M)** Mark any task as delegated to Claude. Before it starts, the agent gets the title, the details and the allowed workspace.
- **FR-11 (M)** Before accepting, the agent checks the task is clear enough to start. If it isn't, it asks first (see 4.5) rather than guessing.
- **FR-12 (M)** The agent posts progress updates and a final result summary to the task's history. The summary lists what changed and links to PRs, files or docs.
- **FR-13 (M)** The agent never marks a task `done` itself. It moves the task to `review`.
- **FR-14 (S)** I can stop or cancel a running agent task.
- **FR-15 (S)** Limits on concurrency and cost, e.g. at most 2 agent tasks running at once and a maximum run time per task.

### 4.5 Agent asks critical questions
- **FR-16 (M)** The agent can post a question, set the task to `needs-input`, and pause.
- **FR-17 (M)** I get notified on mobile and desktop and can answer from either. The agent picks up again with my answer.
- **FR-18 (M)** **What counts as "critical"** (the agent must ask):
  1. **Ambiguity that changes the result**: two reasonable readings would give materially different outcomes.
  2. **Irreversible or outward-facing actions**: sending email or messages, deleting data, publishing, merging to main, spending money.
  3. **Missing access or information**: credentials, a file, a decision only I can make.
  4. **Scope or cost growth**: the work turns out much bigger than the task described.
- **FR-19 (M)** **What is NOT critical** (the agent decides, writes the choice down, and keeps going): naming, formatting, picking between equivalent libraries or approaches, anything easy to undo.
- **FR-20 (M)** Each question is **self-contained and easy to answer on a phone**: one question per message, enough context to answer without scrolling, and lettered options where possible ("A) … B) … C) other"), with the agent's recommended option marked.
- **FR-21 (S)** Several questions found at the same moment are sent as one batch, not one after another.
- **FR-22 (S)** If I don't answer within N hours, the task stays parked. The agent never times out into a guess.

### 4.6 Views
- **FR-23 (M)** "My open tasks", sorted by priority.
- **FR-24 (M)** "Needs my input": every agent question waiting on me. This is the most important view on mobile.
- **FR-25 (S)** "Agent activity": what is queued, running and awaiting review.
- **FR-26 (C)** A done/archive view with search.

## 5. Non-functional requirements

- **NFR-1 Cross-device:** Every MVP action works from the Claude app on phone and PC with no local install.
- **NFR-2 Sync:** A change on one device shows on the other within about a minute.
- **NFR-3 Single source of truth:** Tasks live in exactly one store, which also serves as the audit log.
- **NFR-4 Security:**
  - The agent works only inside the workspace/repo named on the task.
  - Secrets never go in task details. The agent uses environment secrets, never the task text.
  - Merging, sending, publishing and deleting always need my explicit approval (see FR-18.2).
- **NFR-5 Privacy:** Tasks are private to me (private repo or private store).
- **NFR-6 Durability:** Tasks survive app restarts and lost sessions. Nothing important lives only in one chat session.
- **NFR-7 Low maintenance:** No server for me to run in v1. Use managed pieces: GitHub, Claude Code on the web, claude.ai.

## 6. Architecture options

| | A. GitHub Issues + Claude Code (recommended) | B. claude.ai Artifact app with a shared database | C. Custom web app + Claude API |
|---|---|---|---|
| Task store | Issues in `task-master` (labels hold priority and status) | The Artifact's built-in database | Own database (e.g. Supabase) |
| Add/edit on mobile | Ask Claude in the Claude app, or use the GitHub mobile app | Open the Artifact page in the Claude app | Own PWA |
| Delegation | Label `assignee:claude` or comment `@claude`; a Claude Code cloud session or GitHub Action picks it up | Needs a bridge to start an agent | Build my own agent loop |
| Agent questions | Issue comment + `needs-input` label; I reply in a comment | Custom UI | Custom UI |
| Notifications | GitHub and Claude app notifications, already there | Limited | Build my own |
| Effort | **Low** | Medium | High |
| Weakness | The UI is GitHub's, not a custom one | Weak agent integration | Lots to build and run |

**Recommendation:** start with **A**. Issues already give me storage, history, comments (the question-and-answer channel), labels (priority and status) and notifications on every device. Claude Code on the web can already read and write issues in this repo. Later, add **B** as a nicer front end in phase 3: a private claude.ai Artifact dashboard that reads the same issues.

### 6.1 How option A maps to the requirements

| Requirement | Implementation |
|-------------|----------------|
| Task | GitHub Issue |
| Priority | Labels `P0`–`P3` |
| Status | Open/closed plus labels `queued`, `in-progress`, `needs-input`, `review` |
| Assignee | Label `assignee:claude` (absent means me) |
| Details | Issue body, using a template with Context, Done-when and Allowed-workspace sections |
| Agent question | Issue comment in the FR-20 format, plus the `needs-input` label |
| Answer | My reply comment, which triggers the agent to resume |
| Result | Final comment with a summary and links, plus the `review` label |
| Agent rules | `CLAUDE.md` in this repo sets the FR-18/19/20 behaviour for every agent run |

## 7. Example flows

**Capture on mobile**
> Me (Claude app): "Add task: compare 3 home insurance quotes, P1, due Oct 20, delegate to Claude."
> Claude: creates issue #12 with labels `P1` and `assignee:claude`, then confirms.

**Agent asks a critical question**
> Agent on #12: "❓ **Needs input:** Which coverage should I compare?
> A) Building only  B) Building + contents (recommended, matches your current policy)  C) Other.
> Reply with a letter."
> Label → `needs-input`; I get a phone notification and reply "B"; the agent carries on.

**Review**
> Agent: posts a comparison table and a recommendation, sets `review`. I close the issue (done) or comment "also include X" (back to in-progress).

## 8. Phased roadmap

| Phase | Scope | Exit criteria |
|-------|-------|---------------|
| **0. Setup** | Labels, issue template, `CLAUDE.md` with the agent's question rules | I can add a labelled task from my phone through Claude |
| **1. MVP** | FR-1, 4, 5, 7, 10–13, 16–20, 23, 24 | One real task delegated end to end, including one question answered from my phone |
| **2. Polish** | Plain-language capture, "focus today", stop/cancel, limits, batched questions | Used every day for 2 weeks |
| **3. Dashboard** | claude.ai Artifact showing the Needs-input, Open and Agent-activity views | Can triage everything from one screen on my phone |
| **4. Extras** | Voice, attachments, recurring tasks, calendar/email capture | Nice to have |

## 9. Open questions for me (the owner)

1. **What kinds of tasks will I delegate?** Coding in my repos, research and writing, or actions in my accounts (email, calendar)? This decides what access the agent needs and how strict approvals must be.
2. **Agent runtime:** start a Claude Code cloud session per task, or use the Claude GitHub Action triggered by labels or mentions? The first is more interactive; the second is more automatic.
3. **Is the GitHub UI acceptable** as the phase 1 interface, alongside talking to Claude, or do I need a custom screen from day one?
4. **Notifications:** are GitHub and Claude app notifications enough, or do I want push, email or another channel for `needs-input`?
5. **Cost limits:** maximum agent tasks per day or week, and maximum run time per task.
6. **Approval policy:** besides FR-18.2, are there areas where the agent may act alone (e.g. open PRs freely, never merge)?
7. **Do I need recurring tasks or due-date reminders** in the MVP?
