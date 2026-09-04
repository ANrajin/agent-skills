---
name: feature-reference-docs
description: Write standing reference documentation for an existing feature, report, service, or system component — the kind meant to outlive the conversation that produced it, not a summary of a fix or investigation. Use this whenever the user asks to "document this for future reference," write docs about how something works, prepare documentation after finishing a fix so the next person doesn't have to re-derive it, or update an already-documented feature with a new change. This is NOT for README/setup docs, user-facing tutorials, or auto-generated API references — it's specifically for "what is this, why does it exist, how does it work today" documentation of something that already exists in the codebase. Trigger even if the user doesn't say "documentation" explicitly — e.g. "can you write up how the vendor sync works so we don't have to figure this out again" or "put together something the next dev can read."
---

# Feature Reference Documentation

## The trap this skill exists to avoid

Requests to document a feature very often arrive right after building or debugging it — the
context is still fresh, so the natural first draft narrates what just happened: what the user
reported, what was investigated, what was found, what changed. That draft reads fine to the
person who just lived through it. It fails the actual audience: someone with zero conversation
context, reading it months later, who wants to know what the thing *is* and how it *works*, not
the story of how it got that way.

The tell is simple: if a sentence only makes sense because the reader was in the room for the
investigation ("the client confirmed X," "we traced this to Y," "after ruling out Z..."), it
does not belong in the descriptive parts of a reference doc. That narrative has a home — a dated
changelog entry — but it is not the document's spine.

Write the descriptive sections as if the feature always worked exactly as described. Save the
story of *how it changed* for the one section built to hold history.

## Default structure

Adapt section names to fit the subject, but this shape covers almost everything:

1. **Business Need / Purpose** — why this exists, in plain terms. What problem does it solve,
   who relies on it. A reader unfamiliar with the feature should understand *why it's worth
   having* before reading how it works.
2. **How It Works (current state)** — the mechanics as they stand today. Parameters, logic
   branches, data flow, output format — whatever a reader would need to reason about or extend
   the thing. Written in present tense, third person, as a specification — not as a report of
   what "we" built.
3. **Known Limitations & Assumptions** — evergreen facts about the feature's behavior, stated as
   characteristics ("X is excluded because Y"), not as findings from an investigation ("we
   discovered X was missing"). If a limitation exists because of a deliberate tradeoff, say what
   the tradeoff was — that context is what stops a future reader from "fixing" something that
   was intentional.
4. **Changelog** — the *only* section where investigation narrative, dates, problem statements,
   and fix descriptions belong. See below.

Sections 1–3 describe the system as it exists right now. Section 4 is the only place time
enters the document. Keeping that boundary sharp is most of what makes this pattern work.

See `assets/template.md` for a ready-to-fill skeleton and `references/example.md` for a
complete worked example (including a before/after of the narrative-trap draft vs. the corrected
version) if you want to see the pattern applied end to end.

## Ground the changelog in git, not memory

Don't reconstruct dates, authors, or "what changed when" from conversation memory — it drifts,
and a reader can't verify it. Pull it from source control instead:

```bash
git log --oneline --follow -- path/to/the/relevant/file
git log -1 --format="%h %ad %an %s" --date=short <commit>
```

Each changelog entry should read like a specification-grade summary of one change: what problem
it addressed, what was actually changed (with a files-touched list if useful), and — if the
change was validated somehow — a short note on how, without re-litigating the whole
investigation. One paragraph of "why," one of "what," is usually enough. Order newest-first.

If a reference doc in this shape already exists for the feature, don't rewrite the whole file —
add a new dated entry at the top of its Changelog, and only touch sections 1–3 if the feature's
actual current-state behavior changed (not just its history).

## Keep the doc scoped to its subject

Investigations are rarely tidy — debugging one feature usually means poking at two or three
adjacent ones along the way. That adjacent context was useful *during* the investigation. It is
not automatically useful *in the deliverable*. If the user asked to document Feature A, and the
investigation also touched Feature B and Feature C, describing B and C's internals in the doc
about A is scope creep, even if B and C came up constantly in conversation.

A quick test while drafting: would this sentence still make sense to someone who only cares
about the named subject and has never heard of the other systems that got mentioned along the
way? If not, cut it — or, if it's genuinely load-bearing context (e.g. "this feature reconciles
against System B's totals"), state the relationship in one line without explaining System B's
internals.

## Before drafting, get oriented

A few minutes of orientation prevents most of the rework:

- **What exactly is in scope?** If the user says "document the X report," confirm (or infer from
  context) whether that means just the report's own logic, or also the surrounding feature
  (admin UI for configuring it, related data flags, etc.) that a reader would need to actually
  *use* it. When genuinely ambiguous and consequential, ask rather than guess.
- **Where does its history live?** Identify the file(s) that define the feature's current
  behavior — that's both your source for "How It Works" and what you'll run `git log` against
  for the changelog.
- **Is there already a doc for this?** Check for an existing file in the same shape before
  writing a new one from scratch (a project may keep these alongside the code, e.g. a markdown
  file next to the feature it documents — check for that convention before assuming there's
  none).
- **What's actually a limitation vs. a bug?** Don't document something as a "known limitation"
  if it's actually just-discovered brokenness nobody has decided to accept yet — that belongs in
  the changelog as a fix, or as an open item, not framed as permanent design.

## Writing style

- Present tense for anything describing current behavior; past tense only inside changelog
  entries.
- Third person, specification voice — describe what the system does, not what "we" did to it.
  Avoid "I," "we," "the user," or "the client" outside the changelog.
- No hedging language ("should," "seems to," "probably") in the descriptive sections — if
  something is genuinely uncertain, that uncertainty is itself worth stating plainly as a
  limitation, not smoothed over with a qualifier.
- Prefer concrete mechanics over vague summary. "Filters to `PaymentStatusId = 30`" beats "only
  includes paid orders" when the reader might need to act on the exact behavior, not just the
  gist.
