# Storage Engine

The storage engine lives entirely in `db/`. It is a classic **LSM-tree** (Log-Structured Merge-tree) with a custom binary encoding layer.

## Components at a glance

| Component | File(s) | Role |
|---|---|---|
| Binary encoding | `cell.go` | Typed value encoding for keys and values |
| Row/schema encoding | `row.go` | Multi-column key and value encoding |
| Write-Ahead Log | `log.go`, `kv_entry.go` | Crash-safe mutation log |
| Memtable | `sorted_array.go` | In-memory sorted key-value buffer |
| SSTables | `sorted_file.go` | Immutable on-disk sorted key-value files |
| Merge iterator | `merge.go` | Multi-level sorted merge |
| KV store | `kv.go` | LSM engine: transactions, compaction, cache |
| Metadata | `metadata.go` | Double-buffered SSTable version tracking |
| SQL layer | `sql_parser.go`, `eval.go`, `table.go` | SQL parsing, expression evaluation, table ops |

## On-disk layout

For a data directory `./data`, PallasDB creates:

```
./data/
  kv_log           ← Write-Ahead Log
  meta0            ← Metadata slot 0 (double-buffered)
  meta1            ← Metadata slot 1 (double-buffered)
  sstable_1        ← SSTable (newest first after compaction)
  sstable_2
  ...
```

SSTable filenames are `sstable_<version>` where `version` is a monotonically increasing `uint64` tracked in the metadata.

## Data flow summary

```
Write:
  client → KVTX.updates (SortedArray)
         → WAL (fsync)
         → kv.mem (SortedArray, merged)
         → compaction → SortedFile (SSTable)

Read:
  client → cache? → MergedSortedKV[updates, mem, sstable_1, sstable_2, ...]
                                  ↑ bloom filter ↑ binary search
```

The sections below document each component in detail.
