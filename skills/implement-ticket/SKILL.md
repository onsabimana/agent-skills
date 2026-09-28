---
name: implement-ticket
description: >
  Pull a ClickUp ticket, summarise understanding, clarify with the user,
  build an implementation plan, then execute on approval. Use when the user
  says "implement ticket SA-XXXXX", "pick up SA-XXXXX", or similar.
---

# Implement Ticket

End-to-end workflow: read a ClickUp ticket → summarise → clarify → plan → execute.

The user drives every gate. Never advance to the next phase without explicit
approval.

## Contract

**Input**: a ClickUp ticket reference — custom ID (e.g. `SA-12345`) or full URL.

**Output**: working implementation on a feature branch, ready for the user to
test locally and ship.

## Credentials

ClickUp credentials live in `tools/credentials.json5` (relative to the project
root). Read the file to get `clickup.token` and `clickup.teamId`. The token is
used as the `Authorization` header value; the team ID is needed for custom-ID
lookups.

```
API base: https://api.clickup.com/api/v2
```

## Phases

### Phase 1 — Read Ticket

Fetch the ticket from ClickUp:

```bash
# For custom IDs like SA-12345:
curl -s -H "Authorization: $TOKEN" \
  "https://api.clickup.com/api/v2/task/$TASK_ID?custom_task_ids=true&team_id=$TEAM_ID&include_markdown_description=true"
```

Extract and note:
- Title, status, assignees
- Full description (markdown)
- Acceptance criteria / checklist items (if any)
- Subtasks (if any)
- Linked tasks / dependencies
- Comments (fetch separately: `GET /task/{task_id}/comment`)
- Attachments / links

If the ticket is sparse, say so — that's useful signal for Phase 3.

### Phase 2 — Summarise Understanding

Present a concise summary to the user:

1. **What** — the change in plain English, one or two sentences.
2. **Why** — the motivation / user problem being solved.
3. **Where** — which areas of the codebase are likely involved.
4. **Acceptance criteria** — restate from the ticket, or infer if not explicit.

Keep it short. The point is to prove you understood the ticket before going
further.

### Phase 3 — Clarify

Ask the user:

- Any questions you have about ambiguous requirements.
- Pointers you need — e.g. "where does X currently live?", "is there an
  existing pattern for Y?".
- Assumptions you're making that need confirmation.

If the ticket is well-specified and you have no questions, say so and move to
Phase 4. Don't manufacture questions for the sake of it.

Wait for the user's answers before proceeding.

### Phase 4 — Implementation Plan

Aggregate everything (ticket, user clarifications, codebase context) into a
concrete plan. The plan must include:

1. **Branch name** — use the ticket prefix (e.g. `SA-12345`) followed by a
   concise kebab-case description. Example: `SA-12345-add-bulk-export-endpoint`.
2. **Changes** — ordered list of what you'll do, grouped by area (backend, frontend,
   shared, etc.). Each item should be specific enough that the user can agree or
   disagree.
3. **Testing strategy** — unit tests, integration tests, and what each covers.
   Follow existing project test patterns (see AGENTS.md for conventions).
4. **Out of scope** — anything the ticket mentions or implies that you're
   deliberately not doing in this pass, and why.

Present the plan and wait for approval. The user may adjust scope, reorder
steps, or veto parts.

### Phase 5 — Execute

Once the user gives the go-ahead:

1. Create the branch from the current HEAD.
2. Implement the plan step by step.
3. Follow existing project conventions — as defined in AGENTS.md and the
   codebase's own patterns.
4. Write tests alongside implementation, not as an afterthought.
5. Run tests and linting before declaring done.
6. Format changed files (`npm run format:unstaged` if applicable).

**Escalation rules from AGENTS.md apply throughout execution.** If you hit
ambiguity, test failures, or architectural decisions — stop and ask. Do not
guess your way through.

### Phase 6 — Handoff

When implementation is complete:

- Summarise what was done, files changed, tests added.
- Note anything the user should verify manually (UI flows, edge cases you
  couldn't test programmatically).
- The user takes it from here: local testing, MR creation, review.

## Quality Bar

Succeeds when:
- The user felt oriented at every phase and was never surprised by what happened.
- The implementation matches the agreed plan.
- Tests cover the acceptance criteria.
- No silent assumptions were made — every ambiguity was surfaced.

Fails when:
- A phase was skipped or auto-advanced without user approval.
- The plan was vague ("update the frontend") rather than specific.
- Tests were skipped or bolted on without thought.
- AGENTS.md escalation rules were ignored.
