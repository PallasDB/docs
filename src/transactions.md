# Transactions

Source: `db/kv.go`

PallasDB implements **snapshot-isolated, optimistic MVCC transactions**. Each transaction sees a consistent snapshot of the store at the time it was opened. Concurrent transactions can proceed in parallel; conflicts are detected only at commit time.

---

## Transaction types

| Type | Description |
|---|---|
| `KVTX` | Raw key-value transaction |
| `DBTX` | Higher-level table transaction (wraps `KVTX`) |

Both types support nested transactions: `tx.NewTX()` creates a child transaction whose updates are applied to the parent on commit rather than directly to the store.

---

## Opening a transaction

```go
tx := kv.NewTX()
```

`NewTX()` under `kv.mu`:
1. Takes a snapshot of `kv.snapshot` (a monotonically increasing counter).
2. Copies the memtable reference and SSTable list (copy-on-read - the slices are shared but immutable from the transaction's perspective).
3. Constructs a `MergedSortedKV` view: `[tx.updates, memtable_snapshot, sstable_0, ...]`
4. Registers the transaction in `kv.ongoing`.

---

## Reading in a transaction

```go
val, ok, err := tx.Get(key)
```

Delegates to `tx.levels.getExact(key)` which scans levels newest-first. Tombstones (`deleted=true`) cause `ok=false`. The result reflects the state at the snapshot time plus any updates staged in `tx.updates`.

Range scans:

```go
iter, err := tx.Range(start, stop, descending)
```

Returns a `RangedKVIter` that stops when the key moves past `stop` (ascending) or before `stop` (descending). Tombstones are filtered by `NoDeletedIter`.

---

## Writing in a transaction

```go
tx.Set(key, val)          // upsert
tx.SetEx(key, val, mode)  // ModeInsert, ModeUpdate, or ModeUpsert
tx.Del(key)               // delete (stages a tombstone)
```

All mutations go into `tx.updates` (a `SortedArray`) and are invisible to other transactions until `Commit()`.

---

## Committing

```go
err := tx.Commit()
```

Under `kv.commit` (a separate mutex from `kv.mu`):

1. **Conflict check** - if `tx.updates` is non-empty, compare each updated key against `kv.history`. If any key was updated by another transaction *after* this transaction's snapshot, return `ErrTXConflict`.

2. **WAL write** - iterate `tx.updates`, write one `EntryAdd` or `EntryDel` per key, then write `EntryCommit` and call `Sync()`.

3. **Memtable update** (under `kv.mu`):
   - Fast path: if no other transactions are open, try `appendSortedArray` (O(1) if updates sort after existing keys).
   - Merge path: build a new merged `SortedArray` from `MergedSortedKV{tx.updates, kv.mem}`.

4. **History update** - increment `kv.snapshot` and record updated keys in `kv.history` (if other transactions are open and might conflict).

---

## Conflict detection

`kv.history` is a log of `(snapshot, key)` pairs for every key written since the oldest open transaction's snapshot:

```go
type UpdatedKey struct {
    snapshot uint64
    key      []byte
}
```

A conflict occurs if any key in `tx.updates` appears in `kv.history` with a snapshot **newer than** `tx.snapshot`. This is a **write-write conflict**: another transaction committed a write to the same key after this transaction read it.

`kv.history` is pruned when transactions close: entries older than the oldest open transaction's snapshot are removed, as they can no longer cause conflicts.

---

## Aborting

```go
tx.Abort()
```

Removes the transaction from `kv.ongoing` and prunes `kv.history`. No WAL entry is written. The staged `tx.updates` are discarded.

---

## Nested transactions

`tx.NewTX()` creates an inner transaction whose `levels` include the parent's `tx.updates` at the front:

```go
inner := &KVTX{target: tx}
inner.levels = slices.Concat(MergedSortedKV{&inner.updates}, tx.levels)
```

On `inner.Commit()`, the inner updates are merged into `tx.updates` (not written to the WAL). On `inner.Abort()`, the inner updates are silently dropped. This is used by the SQL layer to implement statement-level atomicity within a larger transaction.

---

## Synchronization

| Mutex | Protects |
|---|---|
| `kv.mu` | `kv.mem`, `kv.main`, `kv.snapshot`, `kv.history`, `kv.ongoing` |
| `kv.commit` | Serializes the WAL write + memtable update sequence |

`kv.commit` is held for the duration of the write pipeline, ensuring WAL entries are appended in commit order. `kv.mu` is held only briefly to update in-memory state after the WAL write completes.

---

## UpdateMode

`SetEx` and `KV.SetEx` accept an `UpdateMode`:

| Mode | Behavior |
|---|---|
| `ModeUpsert` | Insert if absent, update if present and value differs |
| `ModeInsert` | Only insert if key does not exist |
| `ModeUpdate` | Only update if key already exists and value differs |

If the mode's precondition is not met, `updated=false` is returned and no WAL entry is written.
