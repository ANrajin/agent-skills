---
name: trello-worklog-effort-report
description: Analyze Trello worklog data and generate a standalone HTML project effort report covering user-level estimation versus actual time, overspent and underspent resources, overall project summary, and overspent and underspent cards. Use when the user provides Trello worklog data in spreadsheet, CSV, TSV, copied table, or structured text format and requests an effort, variance, utilization, estimation-accuracy, or project-spend report.
---

# Trello Worklog Effort Report

## Purpose

Analyze Trello worklog data and generate a polished, stakeholder-ready, standalone HTML report named `ProjectEffortReport.html`.

The report must contain, in this exact order:

1. Overall Project Summary
2. Estimation vs Actual Spent Report (By User)
3. Overspent Resources
4. Underspent Resources
5. Overspent Cards
6. Underspent Cards

The report may also include on-target records, top-five insights, data-quality notes, and key findings when useful, but it must not omit any of the six required sections.

## When to Use This Skill

Use this skill when the user:

- Provides Trello worklog data as `.xlsx`, `.xls`, `.csv`, `.tsv`, pasted text, or a copied table.
- Requests an estimation-versus-actual report.
- Requests an overspent/underspent report by user or card.
- Requests a project-effort, utilization, worklog, estimation-accuracy, or variance report.
- Requests a reusable HTML report from Trello card and user worklogs.

Do not use this skill for financial cost reporting unless the input explicitly includes hourly rates or monetary amounts.

## Expected Input

Preferred columns:

- `Card Name`
- `Estimate Time`
- `User Name`
- `User Estimation`
- `Total STLog/User`

The minimum fields required for the full report are:

- Card identifier or card name
- User name
- User-level estimate
- User-level actual spent time

Accept close column-name variations, including:

- `Card`, `Task`, `Issue`, or `Trello Card` for `Card Name`
- `Estimated Time`, `Estimate`, or `Card Estimate` for `Estimate Time`
- `Assignee`, `Resource`, or `Member` for `User Name`
- `User Estimate`, `Assigned Estimate`, or `Estimation` for `User Estimation`
- `Actual`, `Spent`, `Logged Time`, `STLog`, or `Total Logged` for `Total STLog/User`

Before calculating, map detected input columns to the canonical names. Preserve the original card and user text in the report.

## Input Normalization

### Fill-Down Rules

Trello or spreadsheet exports may show a card name only on the first row when multiple users belong to the same card. Fill down the most recent non-empty `Card Name` and `Estimate Time` values until a new card begins.

Do not fill down `User Name`, `User Estimation`, or `Total STLog/User`.

Ignore summary rows such as:

- `Total Summary`
- `Total Summery`
- `Grand Total`
- `Subtotal`

Use detail rows to recalculate totals. Retain source-provided totals only for reconciliation and data-quality notes.

### HTML Entity Cleanup

Decode HTML entities in card names, including repeated encoding. For example:

- `&amp;` becomes `&`
- `&amp;amp;` becomes `&`

### User Name Normalization

For grouping only:

- Trim leading and trailing spaces.
- Collapse repeated internal spaces.
- Compare names case-insensitively.
- Preserve employee IDs and meaningful punctuation.
- Do not merge two users merely because their names are similar.

Display one consistent original form for each grouped user, preferably the first non-empty occurrence.

### Duplicate Handling

Do not automatically remove duplicate-looking rows. Multiple identical rows may represent legitimate worklog allocations.

Only deduplicate when there is a reliable unique key or when the user explicitly requests deduplication. If exact duplicates appear suspicious, mention them in data-quality notes.

## Time Parsing

Convert time values to integer minutes before performing calculations. Do not use binary floating-point hours as the primary calculation unit.

Support at least these formats:

- `2h 0m`
- `2h 30m`
- `2h`
- `30m`
- `2:30`
- `2.5` meaning 2.5 hours
- Numeric spreadsheet values representing hours
- Blank, `null`, `N/A`, or `-`

Conversions:

- Hours and minutes: `(hours * 60) + minutes`
- Decimal hours: `round(decimal_hours * 60)`
- `H:MM`: `(H * 60) + MM`

Reject or flag negative durations unless the input clearly defines them as adjustments.

Format calculated output as `Hh Mm`, for example:

- `120` minutes -> `2h 0m`
- `150` minutes -> `2h 30m`

Format signed variance as:

- Positive: `+2h 30m`
- Negative: `-2h 30m`
- Zero: `0h 0m`

## Calculation Policy

### Authoritative Estimate

Use `User Estimation` for all user-level and card/user-level variance calculations.

`Estimate Time` is the card-level estimate and may repeat for multiple users. Never sum repeated `Estimate Time` values across user rows for the project estimate unless the dataset lacks `User Estimation` and the fallback is explicitly disclosed.

### Row-Level Calculation

Each detail row represents one card/user allocation.

For every valid row:

- `estimated_minutes = User Estimation`
- `actual_minutes = Total STLog/User`
- `variance_minutes = actual_minutes - estimated_minutes`

Classification:

- Variance greater than zero: `Overspent`
- Variance less than zero: `Underspent`
- Variance equal to zero: `On Target`

### Zero or Missing Estimate Policy

Distinguish zero from missing:

- Explicit `0h 0m` is a valid estimate of zero.
- Blank, null, invalid, or unavailable estimate is missing.

For an explicit zero estimate:

- Include its actual time in total actual spent.
- If actual time is greater than zero, classify it as overspent by the full actual amount.
- If actual time is zero, classify it as on target.
- Mark it with a `Zero estimate` note because percentage variance is undefined.

For a missing estimate:

- Include actual time in total actual spent.
- Exclude the row from estimated totals and variance totals.
- Exclude it from overspent/underspent classification.
- List it in data-quality notes as `Unestimated work`.

Do not silently treat missing estimates as zero.

### User-Level Aggregation

Group by normalized `User Name` and calculate:

- `User Estimated = sum of valid User Estimation values`
- `User Actual = sum of all valid actual values`
- `User Variance = sum of row-level variances where estimate is valid`
- `User Unestimated Actual = actual time from rows with missing estimates`

Classify the user from `User Variance`:

- Positive: Overspent
- Negative: Underspent
- Zero: On Target

If a user has only missing estimates, use status `Unestimated` rather than on target.

### Project-Level Aggregation

Calculate:

- `Project Estimated = sum of all valid User Estimation values`
- `Project Actual = sum of all valid Total STLog/User values`
- `Comparable Actual = actual time only from rows with valid estimates`
- `Net Comparable Variance = Comparable Actual - Project Estimated`
- `Unestimated Actual = Project Actual - Comparable Actual`
- `Gross Actual vs Estimated Gap = Project Actual - Project Estimated`

Use `Net Comparable Variance` for the primary project status because it compares like-for-like rows.

Also display `Unestimated Actual` when it is non-zero so stakeholders can reconcile total actual time.

### Overspent and Underspent Totals

Calculate gross movement, not just net variance:

- `Total Overspent = sum of all positive comparable row variances`
- `Total Underspent = sum of absolute values of all negative comparable row variances`
- `Net Comparable Variance = Total Overspent - Total Underspent`

Verify this identity before generating the report.

### Card-Level Reporting

Treat each card/user combination separately. Do not combine different users assigned to the same card.

If the same normalized card/user pair appears in multiple detail rows, aggregate those rows into one card/user result unless the input contains distinct worklog periods that the user wants kept separate.

For each card/user result, calculate:

- Estimated
- Actual
- Variance
- Status

Overspent Cards contains positive variance only.

Underspent Cards contains negative variance only.

On-target cards may be reported in a separate optional section but must not appear in overspent or underspent tables.

## Sorting

Use stable sorting with names as tie-breakers.

- Estimation vs Actual by User: actual descending, then user name ascending.
- Overspent Resources: positive variance descending.
- Underspent Resources: absolute variance descending.
- Overspent Cards: positive variance descending, then card name and user name.
- Underspent Cards: absolute variance descending, then card name and user name.

## Required HTML Artifact

Generate exactly one primary report artifact:

`ProjectEffortReport.html`

The file must be:

- A valid HTML5 document.
- Self-contained and usable offline.
- UTF-8 encoded.
- Responsive on desktop and mobile.
- Printable with sensible page breaks.
- Free of external libraries, CDN assets, remote fonts, or network calls.
- Styled with embedded CSS.
- Accessible, with semantic headings, table captions, scoped headers, adequate contrast, and text labels in addition to color.

Do not return the full report only as Markdown. The downloadable HTML file is the required deliverable.

## Required Report Layout

### Header

Include:

- `Project Effort Analysis Report`
- Project name if supplied; otherwise `Trello Worklog Project`
- Generation date and time
- Input filename or source label when available

### 1. Overall Project Summary

Show KPI cards for:

- Total Estimated Time
- Total Actual Spent
- Net Comparable Variance
- Total Overspent
- Total Underspent
- Unestimated Actual, if non-zero
- Total Resources
- Total Distinct Cards

Show project status as:

- `🔴 Overspent` for positive net comparable variance
- `🟢 Underspent` for negative net comparable variance
- `🟡 On Target` for zero

CSS and text must both indicate status; do not depend on emoji or color alone.

### 2. Estimation vs Actual Spent Report (By User)

Columns:

- User Name
- Estimated
- Actual Spent
- Variance
- Status

Optionally add `Unestimated Actual` when any user has unestimated work.

### 3. Overspent Resources

Show users with positive comparable variance.

Columns:

- User Name
- Estimated
- Actual Spent
- Overspent By

If no rows qualify, display a clear empty-state message: `No overspent resources.`

### 4. Underspent Resources

Show users with negative comparable variance.

Columns:

- User Name
- Estimated
- Actual Spent
- Underspent By

Display `Underspent By` as a positive magnitude, while retaining an explanatory status label.

If no rows qualify, display: `No underspent resources.`

### 5. Overspent Cards

Columns:

- Card Name
- User Name
- Estimated
- Actual Spent
- Overspent By

If no rows qualify, display: `No overspent cards.`

### 6. Underspent Cards

Columns:

- Card Name
- User Name
- Estimated
- Actual Spent
- Underspent By

Display `Underspent By` as a positive magnitude.

If no rows qualify, display: `No underspent cards.`

### Optional Key Findings

After the required sections, optionally include:

- Highest overspent resource
- Highest underspent resource
- Most overspent card/user allocation
- Most underspent card/user allocation
- Count of on-target allocations
- Count and time of unestimated work
- Top five overspent cards
- Top five underspent cards

Keep findings factual and derived from the calculated data.

### Data-Quality and Reconciliation Notes

Include this section whenever relevant. Report:

- Missing or invalid user names
- Missing estimates
- Missing actual values
- Negative durations
- Suspicious exact duplicates
- Source total versus recalculated total mismatch
- Use of any fallback column

Never change calculated detail totals merely to match a source summary row.

## HTML Styling Requirements

Use embedded CSS with a professional Microsoft 365-compatible visual style:

- Font stack: `Segoe UI, Arial, sans-serif`
- Neutral page background and white content cards
- Dark blue table headers
- Red accents for overspent
- Green accents for underspent
- Amber accents for on target or warnings
- Zebra-striped table rows
- Sticky table headers where practical
- Horizontal scrolling wrappers for narrow screens
- Print styles that remove decorative shadows and avoid splitting rows

Use CSS-only bars only when they add clarity. Bar width must be normalized against the largest absolute value in that chart and capped between 0 and 100 percent.

No JavaScript is required. If JavaScript is used for optional filtering or sorting, embed it in the document and ensure the full report remains readable when JavaScript is disabled.

## Validation and Consistency Checks

Before saving the HTML, perform all checks below:

1. Sum of user estimated time equals project estimated time.
2. Sum of user actual time equals project actual time.
3. Sum of overspent resource variances minus underspent resource magnitudes equals net comparable variance.
4. Sum of overspent card variances minus underspent card magnitudes equals net comparable variance.
5. Each detail row is classified exactly once as overspent, underspent, on target, or unestimated.
6. No summary row is included as a detail row.
7. Project actual equals comparable actual plus unestimated actual.
8. All rendered time strings match their underlying minute values.
9. All required HTML sections exist even when empty.
10. The output file opens as valid HTML and contains no external dependency URLs.

If a check fails, correct the parsing or aggregation. If it cannot be corrected safely, generate the report with a prominent data-quality warning describing the discrepancy.

## Workflow

1. Load the provided data.
2. Detect and map columns.
3. Fill down card-level fields where necessary.
4. Remove summary rows.
5. Normalize card text, user names, and time values.
6. Flag missing, invalid, zero-estimate, and suspicious duplicate records.
7. Aggregate card/user results.
8. Aggregate user results.
9. Calculate project totals and gross overspent/underspent movement.
10. Run all validation checks.
11. Generate the standalone HTML report.
12. Save it as `ProjectEffortReport.html` in the working/output directory.
13. Return a short response with a download link to the HTML artifact and a one-sentence summary of project status. Do not paste the entire report into chat unless requested.

## Final Response Pattern

Use a concise completion message such as:

`The Trello worklog effort report is ready: [Download ProjectEffortReport.html](sandbox:/mnt/data/ProjectEffortReport.html)`

Then state the overall comparable status and mention any material data-quality warning.
