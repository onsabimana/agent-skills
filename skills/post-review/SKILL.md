---
name: post-review
description: >
  Format and post an MR review to GitLab from the outputs of design-review and
  mechanical-review. Owns tone, structure, and posting. Does not re-analyse the
  code. Use after running design-review and/or mechanical-review to publish
  findings to the MR.
---

# Post Review

Format and post an MR review from the outputs of design-review and
mechanical-review. This skill owns tone, structure, and posting. It does not
re-analyse the code.

## Contract

Input:
- An MR ref
- A mechanical-review report file (typically `/tmp/mechanical-review-<slug>.md`)
- Design-review notes (from conversation context, a file, or pasted in)
- Either or both — a review can be mechanical-only, design-only, or combined

Output: a formatted review posted to the MR via GitLab, after the reviewer
confirms the draft.

## Tone

Write as a peer, not a gatekeeper. Use "we" framing, be direct and candid.
No imperative language except on `critical` findings — everything else is a
question or a suggestion.

### Scrub list

Remove or rewrite these before posting. They are AI writing tells or
unnecessarily complex language.

| Remove | Replace with |
|---|---|
| Em-dashes (—) | Comma, or split into two sentences |
| "It's worth noting that" | Cut entirely, just state the thing |
| "Notably" | Cut or rephrase |
| "Importantly" | Cut or rephrase |
| "It should be noted" | Cut entirely |
| "Ensure that" | Just state what should happen |
| "Leverage" | "Use" |
| "Utilize" | "Use" |
| "Facilitate" | "Help" or "let" |
| "In order to" | "To" |
| "With respect to" | "For" or "about" |
| "Additionally" | "Also" or cut |
| "Furthermore" | "Also" or cut |
| "However, it is important to consider" | Just say the thing |
| Semicolons joining independent clauses | Period. Start new sentence |
| Parallel constructions listing 3+ abstract nouns | Rephrase concretely |

After drafting every comment, re-read it and ask: would a real person say
this out loud to a colleague? If it sounds like a report, rewrite it.

## Review Structure

The posted review has these sections in order. Omit any section that has no
content.

### 1. What's done well

Name specific patterns or choices the MR got right. Not praise, not
adjectives. Concrete pattern recognition.

Good:
- "Smart to reuse the existing `retryWithBackoff` helper here instead of
  hand-rolling"
- "Keeping the validation in the service layer rather than the handler
  means other consumers get it for free"

Bad:
- "Nice work!"
- "Really clean implementation"
- "Great MR"

If nothing substantive to name, omit the section entirely. Do not pad.

### 2. Design thoughts

Formatted from the design-review notes. Each thought gets a severity prefix:

- `q:` — a question, no expectation of change
- `non-blocking:` — a suggestion, take it or leave it
- `preferably-blocking:` — would really like this addressed, but author's call
- `blocking:` — rare, only when the architecture fundamentally doesn't work

These go in the summary comment as natural language, not in inline comments.
Design thoughts are about the system, not about specific lines.

<example>
q: have you considered moving the delivery settings to be owned by user-profile
instead? i think it would make notification-service a lot simpler since it
wouldn't need to sync state, it would just subscribe to changes.

preferably-blocking: the validation running in the API layer means anything
that hits the service directly (workers, internal tools) skips it. worth
moving it down into the service itself so it's always enforced.

non-blocking: if you extracted the transform logic into a pure function you
could unit test it without mocking the whole service. not urgent but would
make this easier to change later.
</example>

Keep each thought concise. When a short code example or pattern reference
genuinely clarifies the suggestion, include it.

### 3. Mechanical findings

Formatted from the mechanical-review report. Each finding gets a severity
prefix:

- `critical:` — must fix before merge. Security holes, data loss, build
  breakage.
- `major:` — should fix. Bugs, regressions, resource leaks.
- `minor:` — consider fixing. Small improvements, test gaps.
- `nit:` — optional. Take it or leave it.

Mechanical findings post as **inline comments** anchored to the relevant
file and line. The inline body is plain prose, no severity prefix in the
body itself (the summary manifest carries the severity).

For each inline comment body:
- State the issue in plain English
- Give enough context that the author understands why it matters
- Suggest a concrete fix when possible
- Keep it proportional to severity

<example>
This can return null when the user hasn't set a display name yet, but the
caller on line 88 doesn't check for that. Would crash the settings page
for new users.

You could add a fallback: `user.displayName ?? user.email`
</example>

<example>
Missing `await` here, the promise result gets discarded and the error
handler on the next line never fires.
</example>

<example>
Unused import.
</example>

### 4. Summary manifest

A compact list of all findings (mechanical + design) with severity tags,
so the author can see everything at a glance.

```
### Findings

**Design:**
- preferably-blocking: validation placement (inline suggestion above)
- q: delivery settings ownership
- non-blocking: extract transform for testability

**Mechanical:**
- major: (high) `services/user.ts:42` — null return on displayName
- minor: (med) `handlers/settings.ts:88` — missing await
- nit: `utils/format.ts:3` — unused import
```

Mechanical entries carry confidence in parentheses: `(high)`, `(med)`.
Design entries do not carry confidence.

### 5. Verdict

One line, at the end:

- **Approve** — no findings, or only nit/q findings
- **Approve, please address inline comments** — minor or major findings,
  no critical. Trust-based: the inline comments carry the substance.
- **Request changes** — critical finding exists. Rare. Requires explicit
  confirmation before posting with this verdict.

## Posting Mechanics

### Draft and confirm

Always draft the full review (summary + inline comments) and present it
before posting. The reviewer may edit, cut, add, or rephrase anything. Only
post after explicit confirmation.

Exception: the reviewer can waive confirmation ("just post it", "send it").

### GitLab posting

Post inline findings as MR discussions, post the summary as a top-level note,
and approve only when the verdict calls for approval.

For top-level summary:

```bash
glab mr note <ref> -m "<summary>"
```

For inline discussions, use GitLab's MR discussions API with a valid diff
position from the MR versions/diff refs:

```bash
glab api -X POST projects/:fullpath/merge_requests/<iid>/discussions \
  -f body="<inline body>" \
  -f 'position[position_type]=text' \
  -f 'position[base_sha]=<base_sha>' \
  -f 'position[start_sha]=<start_sha>' \
  -f 'position[head_sha]=<head_sha>' \
  -f 'position[new_path]=<path>' \
  -f 'position[new_line]=<line>'
```

Event mapping:
- Verdict "Approve" or "Approve, please address" → post notes, then `glab mr approve <ref>`
- Verdict "Request changes" → post notes only; GitLab has no exact request-changes event

If approval fails on self-authored MRs or permissions, leave the note posted and
log it.

### Inline comment validation

Before posting, validate each inline comment's `(path, line)` against
the diff. Parse `glab mr diff <ref>` and collect valid `+` line numbers per
file. If an inline targets a line outside the diff:
- Demote it to the summary body rather than dropping it silently
- Format as: `<severity>: <file>:<line> — <body>`

### Fingerprinting

Append an invisible HTML marker to every inline comment for future dedup:

```
<!-- review:fp=<hex16>;sev=<severity>;head=<short-sha> -->
```

Fingerprint: `sha1(normalized_path + category + surrounding_code_hash + short_description)[:16]`

Before posting, check existing MR notes/discussions for matching fingerprints.
Skip any finding already posted.

## Quality Bar

The skill succeeds when:
- Every comment reads like a friendly colleague wrote it, not a bot
- No AI writing tells survive the scrub list
- Design thoughts are concise, natural language, with clear severity
- Mechanical findings have enough context to understand without re-reading
  the code
- The author can scan the summary manifest and know exactly what to address
- The reviewer confirmed the draft before it posted

The skill fails when:
- Comments sound formal, robotic, or lecture-y
- Design thoughts read like essays
- Mechanical findings use vague language ("handle this properly")
- Em-dashes, "notably", or other scrub-list items survive into the post
- The review posted without confirmation

Proceed with the inputs the user provided.
