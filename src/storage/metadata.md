# Metadata Management

Source: `db/metadata.go`

The metadata subsystem tracks which SSTable files currently make up the persistent storage. It uses a **double-buffered write** strategy to guarantee that a metadata update is either fully applied or fully absent - never partially written - even if the process crashes mid-write.

---

## Data model

```go
type KVMetaData struct {
    Version  uint64   // monotonically increasing, identifies the current generation
    SSTables []string // ordered list of SSTable filenames, newest first
}
```

`Version` starts at 0 and increments on every compaction. SSTable filenames are relative to the data directory (e.g., `"sstable_7"`).

---

## Double-buffered files

Two metadata files (`meta0`, `meta1`) are maintained. At any point, the file with the higher `Version` is the authoritative current state:

```go
func (meta *KVMetaStore) current() int {
    if meta.slots[0].data.Version > meta.slots[1].data.Version {
        return 0
    }
    return 1
}
```

### Write protocol

`Set(data)`:
1. Determine the current slot (`c`).
2. Write `data` to the **other** slot (`1-c`).
3. Update the in-memory state of that slot.

The current slot is never overwritten until the new slot has been successfully fsynced. This means:

- If a crash occurs during step 2, the current slot is still valid.
- After recovery, `current()` returns the slot with the higher version, which is always a consistent state.

### Metadata file format

Each metadata file holds:

```
[4 bytes]  CRC32-IEEE checksum of bytes 4 onward
[4 bytes]  JSON payload length (little-endian uint32)
[N bytes]  JSON-encoded KVMetaData
```

On read, if the checksum does not match or the file is too short, `KVMetaData{}` (zero value with `Version=0`) is returned - this is treated as an empty/uninitialized slot.

---

## Integration with compaction

During `compactLog()` (memtable flush):

```go
meta := kv.meta.Get()            // read current metadata
meta.Version = kv.version        // increment version
meta.SSTables = slices.Insert(meta.SSTables, 0, sstable)  // prepend new SSTable
kv.meta.Set(meta)                // write to the other slot (fsynced)
```

During `compactSSTable(level)` (SSTable merge):

```go
meta.SSTables = slices.Replace(meta.SSTables, level, level+2, sstable)
kv.meta.Set(meta)
```

Both operations complete atomically from the metadata perspective: the old SSTable names are still referenced until `meta.Set` succeeds, so the data files are not deleted until after the metadata update.

---

## Recovery on startup

`openSSTable()` calls `kv.meta.Get()` to find the current SSTable list, then opens each file in order. If a file listed in metadata does not exist on disk, `Open()` returns an error.
