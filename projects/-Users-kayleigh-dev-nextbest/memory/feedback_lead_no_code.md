---
name: feedback_lead_no_code
description: When orchestrating agents via linear-todo-runner, the lead must never write code directly — always delegate to agents
type: feedback
---

When running the linear-todo-runner skill, the lead must NEVER write code directly — even for "quick" fixes like adding nav links, fixing a query fallback, or triggering rebuilds. Always spawn a subagent for any code change.

**Why:** The user caught me making direct code edits (adding nav links to layout.tsx, fixing batchFetchTagsForProducts) instead of delegating. The whole point of the orchestration is that agents do the work — the lead coordinates.

**How to apply:** When a bug or change request comes in during a todo-runner session, spawn a subagent with the fix description instead of editing files yourself. This applies even for one-line changes.
