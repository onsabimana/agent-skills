---
name: design-review
description: >
  Build a system-level mental model of a GitLab MR so the reviewer can apply
  engineering judgment at altitude. Surfaces design smells ranked by consequence.
  Does not post comments — use post-review for that. Use when you want to
  understand what an MR does to the system before deciding what to say about it.
---

# Design Review

Build a system-level mental model of an MR so the reviewer can apply their own
engineering judgment at altitude. The skill is the telescope, not the astronomer.

This is not a line-by-line code review — it explains what shape the change gives
the system, then surfaces design smells worth scrutinising. Smells are not
verdicts; they are places where the code may be carrying more complexity,
coupling, or state than it needs to.

Tone and response formatting are out of scope. A separate skill handles how
review comments are written. This skill produces the analysis and collects
the reviewer's notes.

## Contract

Input: an MR reference — number, URL, branch name, or `group/project!number`.

Output: a mental model of the change followed by a ranked list of design smells.
Default mode is read-only. Do not post comments, modify code, or push anything.

## Workflow

### 1. Gather Context

Fetch the MR description, commits, linked tickets, and diff in parallel:

```bash
glab mr view <ref> -F json
glab mr diff <ref>
```

Skim nearby code to understand existing patterns before treating something as
novel. If a linked tracker is inaccessible, note it and work from the MR body
and diff alone.

### 2. Build The Mental Model

Write a plain-English account of the change that the reviewer can read without
opening the diff. Write for someone who knows the product but hasn't seen the
code — no file paths, type names, or function signatures inline. If a code name
is needed to point at something, put it in parentheses; it is evidence, not the
explanation.

Cover whatever is load-bearing for this MR: why the change exists, what now
exists that didn't before, how data moves and who owns it, what triggers the
behaviour, what the system's state looks like, whether this follows or forks an
existing pattern. Skip angles that aren't relevant. A small change may be a
single paragraph; a cross-layer feature may need several.

<examples>

Good — explains the system in plain English:
> The app now treats a report's filter state as a shareable artefact. The
> frontend serialises the active query into a compact versioned envelope,
> compresses it, and base64-encodes it for the URL. Nothing is wired into
> navigation yet — this MR just establishes the codec and the shared schema so
> the frontend and backend can agree on the shape before it's used.

Poor — annotates code artefacts:
> `ts-packages/src/shared/report-url/` adds `ProductsReportUrlPayloadV1` and
> `RollupsReportUrlPayloadV1` schemas. `encodeReportUrlPayload` /
> `decodeReportUrlPayload` wrap the existing gzip+base64url codec.

---

Good — explains data ownership:
> The query shape the app cares about (channels, tags, time range, filters) moves
> from being duplicated in each slice to being owned once by the shared package.
> Transport-level fields like pagination and cursor stay outside this envelope
> because they belong to the request lifecycle, not the saved state.

Poor — describes types:
> `ReportQueryV2` contains channel, tags, time, dimensions, and filters. Runtime
> transport fields (`limit`, `cursor`, `uuid`, `pagination`) are kept outside
> the envelope.

---

Good — explains a flow:
> The chat screen now persists swap results to the conversation rather than
> holding them in component state. When the conversation reloads, the result is
> already there — the UI renders from stored data rather than re-fetching.

Poor — traces function calls:
> `ChatScreen.tsx` calls `loadToolResults()`, then `ToolResultCard` reads
> `result.status`.

</examples>

### 3. Smells

Immediately after the mental model, in the same response, scan for design
smells: structural patterns where the implementation carries more complexity,
coupling, or state than it needs to.

Common design smell categories to check against:

- **Misplaced responsibility** — logic lives in a layer that shouldn't own it
- **Implicit coupling** — components that look independent but break together
- **Redundant state** — the same truth stored or derived in multiple places
- **Leaky abstraction** — callers need to know implementation details
- **Premature generalisation** — abstractions built for one concrete case
- **Missing seam** — two concerns fused where a boundary would help
- **God object/function** — one thing that knows or does too much
- **Temporal coupling** — things that must happen in a specific order but nothing enforces it
- **Shotgun surgery** — a single logical change requires touching many files
- **Feature envy** — code that mostly operates on another module's data

Use only the smells actually present in the MR, rank by consequence, and prefer
a few high-signal observations over a long inventory. If there are no meaningful
smells, say so directly.

Every smell must be a structural concern, not a correctness issue. The test: if
the code produced correct results but had this same shape, would it still smell?

After presenting both the mental model and smells, ask one frame-check question:

> Does this match your understanding of what this MR is doing?

If the reviewer corrects the framing, update the model and revisit the smells
before continuing to notes.

### 4. Collect Notes

When the reviewer's comments lead to decisions for the final review, track which
concerns to raise, skip, soften, or reframe. Capture reasoning in the reviewer's
own words. At the end, present a collected summary faithful to what they actually
said.

If the reviewer had no notes: "No design concerns raised."

## Quality Bar

Succeeds when the reviewer can explain what the MR does and why without opening
the diff. The model addresses architecture, data flow, ownership, and behaviour,
not files. Smells are real and specific to the MR. Notes reflect the reviewer's
actual positions.

Fails when:
- The mental model names types, files, or functions as the substance rather than
  as pointers
- The reader needs codebase familiarity to follow the model
- The smells list reports correctness issues or behavioral gaps rather than
  structural patterns
- The smells list reads like a generic checklist
- Strong opinions are offered before the reviewer has confirmed the model

Proceed with the MR reference the user provided.
