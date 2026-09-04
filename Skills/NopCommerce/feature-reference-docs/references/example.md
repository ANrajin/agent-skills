# Worked Example

This shows the pattern applied to a hypothetical feature — a nightly job that reconciles
warehouse stock counts against the storefront. It's a different domain from any real project on
purpose, so it reads as a template rather than something to copy verbatim.

## The trap: first-draft narrative

This is the kind of draft that comes naturally right after fixing a bug — accurate, but shaped
like a bug report, not a reference doc:

> ## What happened
>
> The client noticed stock levels on the site didn't match the warehouse system. We
> investigated and found that the nightly sync job was skipping any SKU with a pending
> transfer between warehouses, because the query used `WHERE transfer_status IS NULL`, which
> excluded rows where a transfer had *completed* but the status field hadn't been cleared. We
> confirmed with the client that completed transfers should count, and fixed the query to check
> `transfer_status != 'in_progress'` instead. Tested against last Tuesday's data and confirmed
> the previously-missing 340 SKUs now sync correctly.

Everything in that paragraph is true and was worth knowing *during* the fix. None of it belongs
in a reference doc's description of how the sync job works — a reader six months from now
doesn't care what the client noticed or what "we" tried first. They want to know what the job
does now, and if they want the history, it should be clearly separated as history.

## The corrected shape

```markdown
# Nightly Warehouse Stock Sync

## Business Need

The storefront's displayed stock counts must reflect warehouse reality closely enough that
customers aren't sold out-of-stock items. Warehouse systems update stock in near-real-time
internally, but the storefront only reflects it after this nightly job runs — this document
covers what that job does and does not account for, so gaps in displayed stock can be
diagnosed quickly.

## How It Works

### Schedule

Runs nightly at 02:00 local warehouse time, before storefront traffic ramps up.

### Selection logic

For each SKU, the job pulls the warehouse system's on-hand quantity and writes it to the
storefront's `Product.StockQuantity`, excluding any SKU with an **in-progress** inter-warehouse
transfer (`transfer_status = 'in_progress'`) — a transfer mid-flight has an ambiguous true
quantity until it completes, so those SKUs are left at their last known value rather than
synced to a potentially wrong number. Completed and not-yet-started transfers are treated as
normal and synced.

### Output

Updates `Product.StockQuantity` in place; does not create any audit record of the previous
value.

## Known Limitations & Assumptions

- **Up to 24 hours stale.** Stock shown on the storefront reflects the previous night's run,
  not live warehouse state.
- **In-progress transfers are frozen, not zeroed.** A SKU with a transfer mid-flight keeps
  showing its pre-transfer quantity until the transfer resolves, which can overstate or
  understate availability for the duration of the transfer.
- **No change history.** Because the update is in-place, there's no built-in way to see what a
  SKU's stock count was before last night's run.

## Changelog

### 2026-03-11 — Completed transfers no longer excluded from sync

**Problem:** The selection query used `transfer_status IS NULL` to exclude in-progress
transfers, but this also excluded SKUs whose transfer had *completed* without the status field
being cleared — those SKUs silently stopped syncing entirely rather than just being delayed one
cycle, compounding over time. Roughly 340 SKUs were affected as of the last full run before the
fix.

**Change:** Selection logic changed to explicitly check `transfer_status != 'in_progress'`
instead of relying on `IS NULL`, so completed transfers are treated as normal, synced SKUs.

**Validation:** Re-ran the job against a snapshot of the affected date; confirmed all 340
previously-stuck SKUs synced correctly and no other SKU's behavior changed.

### 2025-08-02 — Initial version

Nightly sync job created, syncing all SKUs except those with an in-progress transfer.
```

## What changed between the two versions

- The investigation narrative ("the client noticed," "we investigated," "we confirmed") moved
  entirely into the Changelog, and only there.
- "How It Works" now describes the *current*, correct behavior in present tense, as if it always
  worked this way — because from a reader's perspective starting today, it did.
- The limitation that in-progress transfers freeze rather than sync is now stated as a
  deliberate characteristic with its reasoning ("ambiguous true quantity"), not as something
  that was merely observed.
- The Changelog entry keeps just enough of the original story (problem, change, validation) to
  be useful to someone debugging a *future* stock discrepancy — without it, the doc would look
  like the exclusion logic just appeared unchanged since day one.
