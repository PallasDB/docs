# LSM Tree & Compaction

Source: `db/kv.go`, `db/merge.go`

PallasDB's storage engine is an **LSM-tree** (Log-Structured Merge-tree). Writes go to an in-memory memtable (backed by a WAL) and are periodically flushed and merged to disk as SSTables.

---

## Levels

The LSM structure has a flat list of levels:

```
Level 0: kv.mem      (SortedArray - in-memory, mutable)
Level 1: kv.main[0]  (SortedFile - newest SSTable)
Level 2: kv.main[1]  (SortedFile)
...
Level N: kv.main[N-1] (SortedFile - oldest SSTable)
```

A read must check all levels in order (newest first) because a newer level may contain an update or deletion that shadows an older entry.

---

## Merge iterator

`MergedSortedKV` unifies multiple `SortedKV` sources into a single sorted stream:

```go
type MergedSortedKV []SortedKV
```

`MergedSortedKVIter` maintains one iterator per level and at each step selects the level with the lexicographically smallest current key. On `Next()`, only levels that are at or behind the current key are advanced.

For point lookups (`getExact`), `MergedSortedKV` short-circuits: it queries each level in order and returns the first hit, so lower (older) levels are not accessed if an upper level has the key.

---

## Compaction triggers

`KV.Compact()` is called:
- Explicitly via `pallasdb local compact`
- Automatically after each commit when `AutoCompact=true` (via a background goroutine)
- Automatically after each gRPC write in `serve grpc` mode

```go
func (kv *KV) Compact() error {
    if memSize >= kv.Options.LogThreshold {
        return kv.compactLog()   // flush memtable → new SSTable
    }
    for i := 0; i < len(kv.main)-1; i++ {
        if kv.shouldMerge(i) {
            return kv.compactSSTable(i)  // merge two adjacent SSTables
        }
    }
    return nil
}
```

### Log compaction (memtable flush)

Triggered when `memtable.Size() >= LogThreshold` (default: 1,000 entries).

1. Increment `kv.version`, compute SSTable filename `sstable_<version>`.
2. Write the memtable as a new `SortedFile` at `kv.main[0]` position.
   - If this is not the last SSTable level, tombstones are preserved (older levels may have live entries that need to be hidden).
   - If it would be the only SSTable, tombstones are stripped (`NoDeletedSortedKV` wrapper).
3. Atomically update metadata: prepend the new SSTable name to the list.
4. Insert the new `SortedFile` at position 0 in `kv.main`.
5. Clear the memtable and truncate the WAL.

### SSTable merge

Triggered when `sstable[i].EstimatedSize() * GrowthFactor >= sstable[i+1].EstimatedSize()`.

The default `GrowthFactor` is 2.0, implementing a 2× size ratio between adjacent levels - similar to LevelDB's tiered compaction.

1. Merge `kv.main[i]` and `kv.main[i+1]` using `MergedSortedKV`.
2. Write the merged result as a new `SortedFile`.
   - If the two files are the last in `kv.main`, tombstones are stripped.
3. Update metadata atomically.
4. Close and delete the two old SSTable files.
5. Replace the two entries in `kv.main` with the single new file.

---

## Growth factor and compaction policy

With `GrowthFactor=2.0`:

```
Level 1: ~1,000 entries  (log threshold)
Level 2: ~2,000 entries  (after first merge: 1+2)
Level 3: ~4,000 entries
Level 4: ~8,000 entries
...
```

Each compaction at level `i` merges levels `i` and `i+1`, producing a file roughly 2× the size. This bounds the total number of levels to O(log N) for N total entries.

---

## Tombstone garbage collection

Tombstones (`deleted=true` entries) are only safe to remove when they are in the **last** (oldest) SSTable level. If a tombstone were removed from a non-bottom level, an older SSTable might surface the deleted entry.

`NoDeletedSortedKV` is applied as a wrapper around the source when writing the bottom level SSTable:

```go
type NoDeletedSortedKV struct{ SortedKV }

func (kv NoDeletedSortedKV) Iter() (SortedKVIter, error) {
    iter, _ := kv.SortedKV.Iter()
    return NoDeletedIter{iter}, nil  // skips tombstones
}
```

---

## Options

| Option | Default | Description |
|---|---|---|
| `LogThreshold` | 1,000 | Memtable entry count before flush |
| `GrowthFactor` | 2.0 | Size ratio threshold for SSTable merge |
| `AutoCompact` | `false` | Run compaction in background after every write |
| `CacheEnabled` | `false` | Enable Ristretto LRU read cache |
| `CacheMaxCost` | 0 | Maximum cache size in bytes |
| `CacheNumCounters` | 0 | Ristretto counter slots (typically 10× max cached items) |
