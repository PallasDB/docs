# Sorted Array (Memtable)

Source: `db/sorted_array.go`

The **memtable** is an in-memory sorted key-value store backed by three parallel slices. It is the mutable hot tier of the LSM tree - all committed writes land here before being compacted to disk.

---

## Data structure

```go
type SortedArray struct {
    keys    [][]byte
    vals    [][]byte
    deleted []bool
}
```

The three slices are kept in strict lexicographic key order. A `deleted=true` entry is a **tombstone** - it records that a key was deleted without removing it from the array, so that merging with older SSTables correctly shadows the older value.

---

## Interface

`SortedArray` implements the `SortedKV` interface:

```go
type SortedKV interface {
    EstimatedSize() int
    Iter() (SortedKVIter, error)
    Seek(key []byte) (SortedKVIter, error)
}
```

And additionally implements `sortedKVExactGetter` for O(log n) point lookups without creating a full iterator.

### Key operations

| Method | Complexity | Notes |
|---|---|---|
| `Set(key, val)` | O(log n) search + O(n) insert | `slices.BinarySearchFunc` + `slices.Insert` |
| `Del(key)` | O(log n) search + O(n) insert | Marks existing entry as deleted, or inserts tombstone |
| `getExact(key)` | O(log n) | Binary search, returns `(val, found, deleted)` |
| `Seek(key)` | O(log n) | Returns iterator positioned at or after `key` |
| `Iter()` | O(1) | Iterator starting at position 0 |

### Iterator

`SortedArrayIter` holds a snapshot of the slice headers at creation time. It supports `Next()` and `Prev()` for bidirectional iteration.

---

## Fast-path append during commit

When committing a transaction and there are no concurrent transactions, `appendSortedArray` is attempted first:

```go
// If all keys in src sort after all keys in dst, just append.
if dst.Size() != 0 && bytes.Compare(dst.keys[last], src.keys[0]) >= 0 {
    return false  // fall back to merge
}
dst.keys = append(dst.keys, src.keys...)
// ...
```

This is a common case for sequential key populations and avoids the O(n) merge path.

---

## Merge into memtable

When the fast-path append is not safe (e.g., overlapping key ranges or concurrent transactions), the commit merges the transaction updates with the current memtable using `MergedSortedKV`:

```go
merged := SortedArray{}
iter, _ := MergedSortedKV{&tx.updates, &kv.mem}.Iter()
for ; iter.Valid(); iter.Next() {
    merged.Push(iter.Key(), iter.Val(), iter.Deleted())
}
kv.mem = merged
```

The merge iterator gives priority to the first (newer) level, so transaction updates shadow older memtable entries.
