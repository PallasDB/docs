# SSTables (Sorted String Tables)

Source: `db/sorted_file.go`

SSTables are immutable, on-disk sorted key-value files. They represent the durable tier of the LSM tree. Once written, an SSTable is never modified - compaction merges multiple SSTables into a new one and then deletes the old files.

---

## File format

```
Byte offset 0:
  [8 bytes]   nkeys  (little-endian uint64) - number of entries

Bytes 8 to 8 + nkeys*8:
  [8 bytes each]  entry offsets (little-endian uint64)
                  offsetIndex[i] = file offset of entry i

Entry at offset[i]:
  [4 bytes]  key length (little-endian uint32)
  [4 bytes]  value length (little-endian uint32)
  [1 byte]   deleted flag (0 = live, 1 = tombstone)
  [key len]  key bytes
  [val len]  value bytes

After all entries:
  [bloom len bytes]  marshaled bloom filter
  [8 bytes]          bloom filter byte length (little-endian uint64)
  [8 bytes]          magic: 'P' 'D' 'B' 'B' 'L' 'M' '1' '\n'
```

The file starts with an **offset index** - one 8-byte file offset per entry - enabling O(1) random access to any entry given its position. The maximum supported keys per file is `1 << 26` (~67 million).

The offset index is loaded into memory (`file.offsetIndex`) during `Open()` if it fits within 64 MiB. For larger files the offset index falls back to an individual `ReadAt` per lookup.

---

## Bloom filter

Each SSTable embeds a **Bloom filter** (`github.com/bits-and-blooms/bloom/v3`) at the end of the file:

- False positive rate: **1%** (`sortedFileBloomFalsePositiveRate = 0.01`)
- Keys are added during `writeSortedFile`
- On `Open()`, the bloom filter is deserialized from the file footer
- Before any binary search, `mayContainKey(key)` is called - if the bloom filter definitively says the key is absent, the binary search is skipped entirely

This is critical for performance: a `Get` for a key that does not exist in a given SSTable requires only a bloom filter check (a few memory accesses) rather than a full binary search with disk reads.

---

## Seeking and binary search

`findPos(target)` performs a binary search using `compareKeyAt(mid, target)`:

1. Computes the file offset of entry `mid` (from the in-memory offset index or a `ReadAt`).
2. Reads only the key header + key bytes (not the value).
3. Compares with `bytes.Compare(target, key)`.

This means **values are never read during binary search** - only keys. This optimization significantly reduces I/O for large values.

`findPosExact(target)` returns `(pos, found, error)` and is used for exact-match `Get` operations.

`Seek(key)` returns an iterator positioned at or after `key`.

---

## Reading values separately

`valueAt(pos)` reads only the value (and deleted flag) for a given position, by:
1. Looking up the file offset via the offset index.
2. Reading the header to get `klen` and `vlen`.
3. Seeking past the key bytes to read only the value.

This is used in `getExact` after a successful binary search - the key comparison during search never touches value bytes.

---

## Writing an SSTable

`CreateFromSorted(kv SortedKV)` writes a new SSTable from any `SortedKV` source:

1. Pre-allocates an offset index slot for each estimated key.
2. Iterates the source, writing each entry and recording its offset.
3. After all entries, writes the final `nkeys` count at offset 0.
4. Appends the bloom filter and its footer.
5. Calls `file.Sync()` to ensure durability before the caller updates metadata.

---

## Limits

| Constant | Value | Description |
|---|---|---|
| `maxNKeys` | 67,108,864 | Maximum keys per SSTable |
| `sortedFileBloomFalsePositiveRate` | 0.01 | 1% bloom filter false positive rate |
| `sortedFileMaxOffsetIndexBytes` | 64 MiB | Maximum offset index loaded into RAM |
| `maxEntrySize` | 64 MiB | Maximum combined key+value size |
