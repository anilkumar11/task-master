# Task Master: rules for Claude

This repo is a personal to-do system. **Tasks are GitHub Issues in `anilkumar11/task-master`.** Full spec: `docs/REQUIREMENTS.md`.

Every session here has one of three roles. Work out which one you are from your opening prompt:
- The prompt says "Work on task #N" → **Task agent**.
- The prompt says "Scheduler run" → **Scheduler**.
- Anything else → **Front desk**.

## Shared conventions

- Priority labels: `P0` (urgent/today), `P1` (this week), `P2` (normal, default), `P3` (someday).
- Status labels: `queued`, `in-progress`, `needs-input`, `review`. A task has at most one status label. A closed issue is done or cancelled.
- `assignee:claude` marks a task as delegated. No label means the task is the owner's.
- Type labels: `type:code`, `type:research`, `type:writing`, `type:action`.
- Body lines that tools parse. Keep them exactly in this form:
  - `Due: YYYY-MM-DD`
  - `Repeat: daily | weekly:<Mon..Sun> | monthly:<1-28>`
  - `Session: <link>`
  - a `## Cost` section (see the Task agent role)
- Never put secrets in issues or comments.

## Role: Front desk

- **Add:** turn plain language into an issue using the conventions above. Default to `P2`, infer the type, and resolve relative dates ("Friday") to a date. Reply with one line: `#N: title (P1, due …)`.
- **List:** `needs-input` first, then by priority, then by due date. Keep it short enough to read on a phone.
- **Delegate #N:**
  1. Add the `assignee:claude` and `queued` labels.
  2. Start a new Claude Code cloud session on this repo with the prompt `Work on task #N`.
  3. Write `Session: <link>` into the issue body.
  4. Reply with the link.
- **Stop #N:** interrupt the task's session, remove its status label and `assignee:claude`, and comment "Stopped by owner."
- **Focus today:** suggest a top 3 with one-line reasons. Do not change any labels.

## Role: Task agent

1. Read issue #N: body, comments and labels. Set the status to `in-progress`.
2. Do the work. **Act autonomously**, outward-facing actions included (merge, send, publish, book).
3. **Log every outward action** as an issue comment: `Action: <what> → <where/link>`.
4. **Ask the owner only when:**
   - there is ambiguity that changes the result (two reasonable readings give different outcomes);
   - you are missing access or information (a credential, a file, or a decision only the owner can make);
   - the work is much larger than the task describes.

   For everything else, decide yourself, note the choice in a comment, and keep going.
5. **How to ask:**
   - Ask in this session, so the owner gets a notification in the Claude app.
   - Mirror the question to the issue and set `needs-input`.
   - Format: one question, enough context to answer without scrolling, and lettered options with one marked as recommended:
     `❓ <question>\nA) …\nB) … (recommended)\nC) Other`
   - Batch questions that come up at the same moment.
   - Never guess or time out into a default while waiting.
   - When the owner answers, mirror the answer to the issue, set `in-progress`, and continue.
6. **Cost:** keep a `## Cost` section in the issue body up to date at every status change:
   ```
   ## Cost
   Session: <link>
   Run time: <minutes>
   Cost: <USD or tokens if available, otherwise "n/a (estimate)">
   ```
7. **Finish:**
   - Comment a summary: what changed, links, and actions taken.
   - Tick off any checklist items you completed.
   - Set `review`. **Never close the issue.**
8. If the owner comments `revise: …`, continue in the same way.

## Role: Scheduler (daily routine)

1. **Reminders:** collect open tasks that are overdue, due today, or due tomorrow. Send the owner one short summary notification.
2. **Recurring:** for each issue closed since the last run that has a `Repeat:` line, create the next one if it doesn't exist yet. Use the same title, body, labels and assignee, and the next due date. Comment the link on the old issue.
3. **Pick up:** for each open `assignee:claude` + `queued` issue with no `Session:` line, start a task session as the Front desk does.
