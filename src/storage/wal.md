# Write-Ahead Log (WAL)

Source: `db/log.go`, `db/kv_entry.go`

The Write-Ahead Log (WAL) provides **crash-safe persistence**. Every committed transaction is written and fsync'd to the WAL before the in-memory state is updated. On restart, PallasDB replays all committed entries to reconstruct the memtable.

---

## Log entries

Each WAL record is a single `Entry`:

```go
type EntryOp uint8

const (
    EntryAdd    EntryOp = 0  // set key → value
    EntryDel    EntryOp = 1  // delete key
    EntryCommit EntryOp = 2  // transaction boundary
)

type Entry struct {
    key []byte
    val []byte
    op  EntryOp
}
```

### Wire format

Each entry is encoded as:

```
[4 bytes] CRC32-IEEE checksum of everything after these 4 bytes
[4 bytes] key length (little-endian uint32)
[4 bytes] value length (little-endian uint32)
[1 byte]  op (EntryAdd=0, EntryDel=1, EntryCommit=2)
[key len] key bytes
[val len] value bytes
```

The CRC32 checksum covers all bytes starting from the key-length field. An `EntryCommit` record has zero-length key and value. The maximum entry size is 64 MiB.

---

## Transaction protocol

A single transaction writes:

```
EntryAdd  key=k1  val=v1
EntryDel  key=k2  val=nil
EntryAdd  key=k3  val=v3
EntryCommit
```

The `Commit()` call:
1. Writes the `EntryCommit` record.
2. Calls `file.Sync()` - the writes are now durable.
3. Advances `writer.committed` to the current offset.

If the process crashes before `Sync()` returns, the uncommitted records are ignored on recovery because the commit marker was never persisted.

`ResetTX()` rolls back an uncommitted transaction by resetting the write offset to `writer.committed`, overwriting any partially written records on the next write.

---

## Recovery on startup

`openLog()` in `kv.go`:

1. Reads entries sequentially from offset 0.
2. Accumulates `EntryAdd` / `EntryDel` records in a buffer.
3. On `EntryCommit`, records the committed count.
4. On EOF, bad checksum, or truncated read, stops - only committed entries survive.
5. Sorts the committed entries by key (stable sort), then replays them into the memtable, deduplicating by keeping the last write per key.

---

## Log truncation

When the memtable is compacted into an SSTable (`compactLog()`), the WAL is truncated to zero with `file.Truncate(0)`. The SSTable now contains the durable representation of what was in the log.

---

## File management

The WAL file is created via `createFileSync` which opens with `O_CREATE | O_RDWR` and calls `Sync()` after creation to ensure the directory entry is durable. Reads use `ReadAt` (positioned reads) via `OffsetReader`, avoiding seek races when reading and writing concurrently.
