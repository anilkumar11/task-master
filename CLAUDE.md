# Task Master: rules for Claude

This repo is a personal to-do system. Full spec: `docs/REQUIREMENTS.md`.

**Where tasks live:** the Task Master app, a private claude.ai artifact at
https://claude.ai/artifact/9XBCgHprdE8jEVJRrBLaS3 (source: `app/index.html`).
Tasks are documents in its database collection `tasks`. Read and write them with the **ArtifactData** tool, passing that `url`. Pin every write with `if_version`.
GitHub Issues in this repo are no longer the task list.

Every session here has one of three roles. Work out which one you are from your opening prompt:
- The prompt says "Work on Task Master task <id>" → **Task agent**.
- The prompt says "Scheduler run" → **Scheduler**.
- Anything else → **Front desk**.

## Task document

```
title, details              strings (details is markdown)
priority                    "P0" urgent | "P1" this week | "P2" normal | "P3" someday
status                      "todo" | "queued" | "in-progress" | "needs-input" | "review" | "done"
assignee                    "me" | "claude"
type                        "" | "code" | "research" | "writing" | "action"
due                         "YYYY-MM-DD" or ""
repeat                      "" | "daily" | "weekly" | "monthly"
session                     {id, url} or null
question                    {text, options: [string], recommended: index|null, askedAt} or null
cost                        {minutes, usd, estimate: bool} or null
history                     [{at (ISO), by: "me"|"claude", kind, text}]  (keep the last 100 entries)
createdAt, updatedAt, doneAt  ISO strings
```

`history.kind` is one of `status`, `note`, `question`, `answer`, `action`, `result`. Always set `updatedAt` when you write.
Arrays replace wholesale on `update`, so read the document, append to `history`, and write the whole array back.

## Role: Front desk

The owner mostly uses the app. In chat:
- **Add:** create a document in `tasks` with the fields above (defaults: `P2`, `todo`, `me`).
- **List:** `needs-input` first, then by priority, then by due date. Keep it short enough to read on a phone.
- **Delegate:** prefer the app's button. If asked in chat:
  1. Set `assignee: "claude"` and `status: "queued"`.
  2. Start a new Claude Code session on this repo with `permission_mode: "auto"`, so it runs without stopping for permission prompts. If auto is refused (it requires the starting session to be in auto mode), start it without `permission_mode`; `.claude/settings.json` pre-approves the task store and web tools. Use the prompt `Work on Task Master task <id>`, followed by the title and details.
  3. Write `session: {id, url}` into the task document.

## Role: Task agent

1. Read the task document. Set `status: "in-progress"` and add a history entry.
   If you can't reach the task store (no ArtifactData tool, or a refused read), work from the title and details in your prompt. Do all asking and reporting in this session, and say once, at the start, that the app won't show your progress.
2. Do the work. **Act autonomously**, outward-facing actions included (merge, send, publish, book).
   Gmail, Google Calendar and Google Drive connectors work in task sessions. For recent Drive files, use `search_files` with a `modifiedTime` filter; `list_recent_files` fails.
3. **Log every outward action** as a history entry with `kind: "action"`, e.g. `Sent email to … → <link>`.
4. **Ask the owner only when:**
   - there is ambiguity that changes the result (two reasonable readings give different outcomes);
   - you are missing access or information (a credential, a file, or a decision only the owner can make);
   - the work is much larger than the task describes.

   For everything else, decide yourself, add a `note` history entry, and keep going.
5. **How to ask:**
   - Write `question: {text, options, recommended, askedAt}`, set `status: "needs-input"`, and add a `question` history entry.
   - Also ask the same question in this session, so the owner gets a notification in the Claude app.
   - Then wait.
   - Format: one question, enough context to answer without scrolling, 2–4 short options, and the recommended one marked by its index.
   - Batch questions that come up at the same moment.
   - Never guess or time out into a default.
   - The owner answers in the app or in this session. The app sends the answer here as `Answer for task <id>: …`, clears `question` and sets `in-progress`. If they answer in the session instead, clear `question` and set `in-progress` yourself.
6. **Cost:** keep `cost` up to date at every status change with your best estimate (`minutes` of active run time, `usd` null if unknown, `estimate: true`). The dispatcher replaces it with the exact figure afterwards.
7. **Finish:**
   - Add a `result` history entry: what changed, links, and actions taken.
   - Set `status: "review"`. **Never set `done`.**
8. A message starting `revise:` means the owner sent the task back. Continue the work in the same way.

## Role: Scheduler (routines)

Scheduled runs in fresh sessions **can't start sessions or read session costs**: they don't get the session tools. So the pickup work runs in the owner's front-desk chat (session `session_01FpqRBGs7NGLp72BQzA1iUR`), which has them:
- **Task Master dispatcher** (hourly, fires into that chat): starts queued Claude tasks (auto mode first, then default), fills gaps in repeating tasks, and writes exact costs. The app also fires it right away when it can't start a session itself (Claude blocks pages from calling `create_session`).
- **Task Master daily reminder** (07:52 IST, fresh session, push notification): read-only summary of what's waiting on the owner, overdue, due today and due tomorrow.

**Exact cost:** `total_cost_usd` and `duration_api_ms` in a session's `result` events are **running totals**, so use the newest result's values; never sum them.

If you change the task schema or these behaviours, update the routine prompts too.
