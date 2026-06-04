# Architecture Overview

PallasDB is structured as three independent layers that can be used separately or stacked together.

```
┌─────────────────────────────────────────────────────────┐
│                    CLI  (cmd/pallasdb)                  │
├─────────────────────────────────────────────────────────┤
│        Cluster / Raft  (cluster/)                       │
│  ┌──────────────────────────────────────────────────┐   │
│  │  FSM  →  db.KV  ←  Snapshot / Restore           │   │
│  │  Serf discovery  →  AddVoter                     │   │
│  └──────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────┤
│        gRPC transport  (grpc/)                          │
│  KVService: Get / Put / Delete / Range (streaming)      │
│  ClusterService: Join / ListMembers / GetLeader         │
├─────────────────────────────────────────────────────────┤
│        SQL layer  (db/table.go, db/sql_parser.go)       │
│  CREATE TABLE / SELECT / INSERT / UPDATE / DELETE       │
│  WHERE clause expression evaluator                      │
├─────────────────────────────────────────────────────────┤
│        KV store  (db/kv.go)                             │
│  MVCC transactions • conflict detection • compaction    │
├───────────────────────────┬─────────────────────────────┤
│    Memtable               │    SSTables (sorted files)  │
│    db/sorted_array.go     │    db/sorted_file.go        │
│    in-memory sorted array │    on-disk sorted KV        │
│    binary search          │    binary search + bloom    │
├───────────────────────────┴─────────────────────────────┤
│        Write-Ahead Log  (db/log.go, db/kv_entry.go)     │
│        CRC32-checksummed entries, fsync on commit       │
├─────────────────────────────────────────────────────────┤
│        Binary encoding  (db/cell.go, db/row.go)         │
│        TypeI64 / TypeStr • key encoding • value encoding│
└─────────────────────────────────────────────────────────┘
```

## Data flow - write path

1. **Client** calls `KV.Set(key, val)` (or via gRPC `Put`, or SQL `INSERT`).
2. A **transaction** (`KVTX`) is opened with a snapshot of the current state.
3. The mutation is staged in `tx.updates` (a `SortedArray`).
4. On `Commit()`:
   a. A **conflict check** compares updated keys against keys modified by concurrent transactions since this snapshot.
   b. Each mutation is appended to the **WAL** (`Log.Write`) as a `EntryAdd` or `EntryDel` record.
   c. A `EntryCommit` record is written and **fsync'd** - the write is now durable.
   d. The transaction's updates are merged into the in-memory **memtable** (`kv.mem`).
5. If **auto-compact** is enabled, a compaction signal is sent on `kv.updated`.

In **cluster mode**, mutating commands are first encoded as a `Command` protobuf, submitted to `raft.Apply`, and only reach the FSM's `Apply()` method after consensus. The FSM then calls `db.KV.SetEx` / `db.KV.Del` directly.

## Data flow - read path

1. `KV.Get(key)` checks the optional **Ristretto LRU cache** first.
2. A transaction is opened, creating a `MergedSortedKV` view spanning:
   - `tx.updates` (pending mutations, if any)
   - `kv.mem` (memtable snapshot)
   - `kv.main[0..n]` (SSTable snapshots, newest first)
3. The merged view performs `getExact(key)`:
   - For `SortedArray`: binary search on the in-memory key slice.
   - For `SortedFile`: bloom filter pre-check, then binary search using the on-disk offset index.
4. The first level that returns `found=true` wins; deleted markers suppress the result.
5. On cache miss the result is inserted into the Ristretto cache.

## Compaction

Compaction has two sub-operations that are triggered by `KV.Compact()`:

| Trigger | Operation |
|---|---|
| `memtable.Size() >= LogThreshold` | **Log compaction**: flush memtable → new SSTable, truncate WAL |
| `sstable[i].size * GrowthFactor >= sstable[i+1].size` | **SSTable merge**: merge two adjacent SSTables into one |

Both operations update the double-buffered **metadata** files atomically before closing old files.

## Key dependencies

| Library | Role |
|---|---|
| `github.com/hashicorp/raft` | Raft consensus |
| `github.com/hashicorp/raft-boltdb/v2` | BoltDB-backed Raft log store |
| `github.com/hashicorp/serf` | Gossip-based member discovery |
| `github.com/bits-and-blooms/bloom/v3` | Bloom filters for SSTable key lookups |
| `github.com/dgraph-io/ristretto/v2` | High-throughput LRU cache |
| `google.golang.org/grpc` | gRPC transport |
| `github.com/spf13/cobra` | CLI framework |
| `github.com/spf13/viper` | Configuration management |
| `go.etcd.io/bbolt` | BoltDB (Raft log persistence) |
