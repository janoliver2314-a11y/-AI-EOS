# Standard: Data Lifecycle

## Purpose

Define how AI-EOS deletes data and reconciles state between copies, so a
deletion made in one place is honoured everywhere and cannot be silently
undone by another part of the system.

## Scope

Applies to any data that exists in more than one copy or is reconstructed
from a source: multi-device and offline-first sync, caches, mirrors,
incremental ETL, and any importer or ingestion job that decides whether a
record already exists. Applies to soft-delete (`deleted_at`, status flags,
archive tables) and to hard delete where other systems hold references.

Does **not** cover retention policy, backup, or regulatory erasure
obligations — how long to keep data and when the law compels its removal
are separate questions from making a deletion stick.

## Goals

- Ensure a deletion propagates to every copy, rather than only the one that
  issued it.
- Ensure a deletion is not reversed by a second component that recreates
  the record from its source.
- Make the lifetime of every suppression record deliberate, rather than
  inherited from whichever queue happened to hold it.

## Rules

1. **Deletion is an explicit signal, never inferred from absence.** A record
   missing from a payload is ambiguous: it may also mean not-yet-created,
   filtered, paginated out, or created locally and not yet sent. Before
   writing "remove what is missing", enumerate every reason a record can
   legitimately be absent; more than one entry means an explicit tombstone
   is required.
2. **Enumerate every consumer of a record, and the field each one keys on,
   before designing the tombstone.** A tombstone carrying only the primary
   key serves only the consumer keyed on the primary key.
3. **Anything that answers "does this already exist?" is a consumer of
   deletions** — importers, deduplicators, reconcilers, idempotency checks.
   Such components are easy to overlook because they only read, but they
   are precisely the components that will recreate a deleted record.
4. **Ask what the deleted record was the sole holder of.** Whatever that is
   must survive the deletion, or the thing that created the record will
   create it again.
5. **A suppression record's lifetime is set by what it suppresses, never by
   the queue that delivered it.** Confirming a deletion is not permission to
   forget it.
6. **Soft-deleted rows retain their payload.** Once the row is the only
   surviving record of what was deleted, stripping it destroys the evidence
   the rest of this standard depends on.
7. **Deletion is tested with two copies.** A single client cannot observe a
   propagation failure, because it is the copy that already applied the
   change.

## Design Decisions

- **Tombstones are unbounded rather than windowed.** Bounding by date
  silently strands a copy that was offline longer than the window; the list
  grows with lifetime deletions rather than live records, which at the scale
  this repository's systems operate is a bounded cost worth paying.
- **An unpushed local edit wins over a remote delete.** The local queue is
  the only copy that exists nowhere else, so it is never dropped on the
  strength of a remote signal; re-pushing revives the record, which is the
  correct outcome for "edited here, deleted there".
- **Suppression is recorded per occurrence, not per series**, wherever a
  source emits recurring items with distinct identities. Suppressing a
  series because one occurrence was deleted destroys more than the user
  asked to remove.

## Best Practices

- Harvest whatever the tombstone needs at deletion time, in the same
  operation that removes the record — that is the last moment the data
  exists.
- Where dedupe or existence rules decide whether to recreate a record, keep
  them in a pure, directly testable unit rather than inline in a component
  or handler; rules that cannot be tested are where this class of bug hides.
- When this failure is suspected in live data, count deleted records against
  the number of *distinct* keys among them. A count materially higher than
  the distinct count means the same record has been deleted repeatedly and
  something is recreating it.

## Examples

A soft-delete whose consumers key on two different fields — the sync merge
on `id`, the importer on the source system's `sourceId`:

```text
// The tombstone must carry both, or the importer recreates what the
// merge correctly removed.
GET /sync -> {
  items:          [...],
  deleted:        ["<uuid>", ...],        // consumed by the merge
  deletedSources: ["<sourceId>", ...],    // consumed by the importer
}
```

The suppression set built from those sources is never drained when the
deletion is acknowledged — unlike the tombstone queue, which exists only
until the server confirms it:

```text
onDelete(record):   suppressed.add(record.sourceId)   // permanent
onDeleteConfirmed:  pendingDeletes.remove(record.id)  // transient
```

## Common Mistakes

- Returning only primary keys from a tombstone query, when a second consumer
  keys on a natural or foreign identifier.
- Treating an importer as outside the sync contract because it never mutates
  a record.
- Clearing a suppression list when the delete is acknowledged, so the record
  is recreated on the next ingestion run.
- Relying on an incidental guard — a title-and-date match, a uniqueness
  constraint on unrelated fields — that masks the missing tombstone data
  until the colliding record moves or changes.
- Pruning local records that are merely absent from a payload, which deletes
  locally created records before their remote twin arrives.
- Testing deletion only on the device that issued it.

## Future Improvements

- No AI-EOS-owned system currently reconciles more than two copies; if a
  third-party mirror or an outbound integration is added, revisit whether
  tombstones need ordering or vector-clock semantics rather than the
  last-write-wins rule assumed here.
- Tombstone growth is unbounded by design, and no system here is close to a
  scale where that costs anything. If one ever is, the replacement is an open
  question rather than a decision this document has made: a per-copy high-water
  mark would keep the correctness property the rest of this standard rests on,
  where a time window trades that property away for a bound. Neither has been
  measured against a real payload, so the choice should be made against one.

## Related Documents

- `docs/standards/reliability.md` — idempotency and retry semantics, which
  govern the delivery of these signals rather than their content.
- `docs/standards/testing.md` — the two-copy testing requirement in rule 7.
- `memory/lessons-learned.md` — `LL-0081` (a tombstone that never travelled)
  and `LL-0105` (a tombstone that travelled underspecified).
