---
name: create-ticket
description: >
  Create a ticket from conversation context, a proposal doc, or a verbal
  description. Outputs a concise Definition of Ready markdown file.
  Use when work needs to be tracked — especially follow-up tickets discovered
  mid-implementation, or when turning a design proposal into actionable work.
---

# Create Ticket

Write a concise Definition of Ready ticket. The ticket defines *what* is wrong
and *where* — it does **not** prescribe *how* to fix it. Leave room for the
implementing engineer to discover and propose an appropriate solution.

## Contract

**Input** — one of:

- A verbal description from the user (in conversation).
- A path to a proposal or investigation document to extract work from.
- Context from the current session (e.g. a blocker discovered during implementation).

The user may also supply a ticket title or key. If not, infer a short one.

**Output** — a markdown file written to disk. Default location is `docs/tickets/`
relative to the project root. The user may override the path.

Ask the user to confirm the ticket content before writing.

## Template

Use this structure exactly. Every section heading must appear. Leave a section
blank (with a single `-` or `N/A`) rather than omitting it.

```markdown
# <Title>

## Problem / change

What is wrong or what needs to change?

How does it work now?

## Expected behavior

What should happen instead?

## Scope

Where does it apply? (screen / flow / API / service)

## Acceptance criteria / test cases

- [ ] ...

## Dependencies / blockers

- ...

## Additional links

- ...

## Technical details

- Engineering context: relevant code pointers, constraints, prior art.
- Keep it short — just enough for the engineer to orient.
- Use plain text / bullet points, not HTML comments.
```

## Guidelines

1. **Be brief.** A ticket is a starting point, not a design doc. One or two
   sentences per section is ideal. Bullet points over paragraphs.
2. **State the problem, not the solution.** Describe the gap between current
   and expected behavior. Don't dictate implementation unless there is a hard
   constraint (e.g. "must use existing bulk API").
3. **Acceptance criteria are observable.** Each criterion describes something
   a human or test can verify. Avoid vague criteria like "works correctly".
4. **Link, don't copy — but only internet-available links.** The ticket may be
   read by another engineer in a different context. Only include links that are
   accessible over the internet (Notion pages, MR URLs, deployed docs). Do not
   include local file paths in Additional links. If the source material is only
   available locally (e.g. a proposal doc in the repo), summarise the relevant
   design context in Technical details instead.
5. **Technical details are for orientation.** Point the engineer at the right
   files, services, or prior art. A few lines, not a mini-RFC.
6. **Follow-up tickets are first-class.** When a blocker or tangent surfaces
   during implementation, create a separate ticket immediately rather than
   expanding the current one. Reference the parent ticket if relevant.

## Workflow

1. Gather context — read the referenced doc or use conversation history.
2. Draft the ticket using the template above.
3. Present the draft to the user for confirmation.
4. On confirmation, write the file to `docs/tickets/<slug>.md` (or the
   user-specified path) and report the path.
