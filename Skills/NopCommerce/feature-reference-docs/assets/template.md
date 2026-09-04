# [Feature / Report / Service Name]

<!--
Optional metadata table — keep if it helps a reader orient quickly, drop if the project
doesn't use this convention. Fill only fields that are actually useful; delete the rest.
-->

| Field | Value |
|-------|--------|
| **Component** | e.g. plugin, service, module name |
| **Platform** | e.g. framework + version |
| **Entry point** | primary file/class/endpoint a reader would start at |

---

## Business Need

<!--
Why does this exist? What problem does it solve, for whom? A reader with zero context should
finish this section understanding why the feature is worth having — not yet how it works.
2-4 sentences is usually enough. Avoid restating the feature's name as its own justification.
-->

## How It Works

<!--
Present tense, current-state only. Cover whatever a reader would actually need to reason about
or extend this feature. Typical subsections, use what applies:
-->

### [Inputs / Parameters]

### [Core logic / flow]

<!-- Break into named sub-behaviors if there's branching logic worth calling out individually. -->

### [Output / result format]

## Known Limitations & Assumptions

<!--
Evergreen facts, phrased as characteristics: "X is excluded because Y", not "we found X missing".
If a limitation is a deliberate tradeoff, say what the tradeoff was.
-->

- ...

<!-- Optional — include only if the feature has a meaningful operational/config surface. -->
## Operating / Configuring [Feature Name]

<!-- How someone actually uses or adjusts this day to day, if applicable. -->

## Changelog

<!--
Newest entry first. Ground dates/authors in `git log --follow` on the relevant file(s), not
memory. This is the ONLY section where investigation narrative belongs.
-->

### YYYY-MM-DD — [Short title of the change]

**Problem:** What was wrong or missing, and why it mattered.

**Change:** What was actually changed. Include a files-touched table if it's useful to a future
maintainer.

**Validation:** How the change was confirmed correct, briefly — not a re-telling of the whole
investigation.

### YYYY-MM-DD — Initial version

Brief note on what the feature covered when first built.
