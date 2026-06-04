# Go API Reference

Import path: `github.com/teddymalhan/pallasdb/db`

---

## KV - key-value store

### Opening a store

```go
kv, err := db.NewKV("path/to/data",
    db.WithLogThreshold(2000),
    db.WithGrowthFactor(2.0),
    db.WithAutoCompact(true),
    db.WithCache(64<<20, 640000),  // 64 MiB cache, 640k counters
)
if err != nil {
    log.Fatal(err)
}
defer kv.Close()
```

### Options

| Function | Default | Description |
|---|---|---|
| `WithLogThreshold(n int)` | 1000 | Memtable entry count before flush to SSTable |
| `WithGrowthFactor(f float32)` | 2.0 | Size ratio threshold for SSTable merge (minimum 2.0) |
| `WithAutoCompact(enabled bool)` | false | Run compaction in a background goroutine after every write |
| `WithCache(maxCostBytes, numCounters int64)` | disabled | Enable Ristretto LRU read cache |

### Point operations

```go
// Get
val, ok, err := kv.Get([]byte("hello"))

// Set (upsert)
updated, err := kv.Set([]byte("hello"), []byte("world"))

// SetEx - explicit mode
updated, err := kv.SetEx([]byte("hello"), []byte("world"), db.ModeInsert)
updated, err := kv.SetEx([]byte("hello"), []byte("world"), db.ModeUpdate)
updated, err := kv.SetEx([]byte("hello"), []byte("world"), db.ModeUpsert)

// Delete
deleted, err := kv.Del([]byte("hello"))
```

### Iteration

```go
// Iterate all live keys
iter, cleanup, err := kv.IterAll()
defer cleanup()
for ; iter.Valid(); iter.Next() {
    fmt.Printf("%s = %s\n", iter.Key(), iter.Val())
}
```

### Compaction

```go
err := kv.Compact()
```

Flushes the memtable if it exceeds the threshold and merges adjacent SSTables if the size ratio triggers. Safe to call concurrently with reads and writes.

### Close

```go
err := kv.Close()
```

Waits for background goroutines (compaction thread, open transactions), closes the cache, and closes all open files.

---

## KVTX - key-value transaction

```go
tx := kv.NewTX()
defer tx.Abort()  // no-op if already committed

// Read
val, ok, err := tx.Get([]byte("key"))

// Write
updated, err := tx.Set([]byte("key"), []byte("val"))
updated, err := tx.SetEx([]byte("key"), []byte("val"), db.ModeInsert)
deleted, err := tx.Del([]byte("key"))

// Range scan
iter, err := tx.Range([]byte("a"), []byte("z"), false /* ascending */)
for ; iter.Valid(); iter.Next() {
    fmt.Printf("%s\n", iter.Key())
}

// Seek
iter, err := tx.Seek([]byte("prefix"))

// Commit
if err := tx.Commit(); err == db.ErrTXConflict {
    // retry
}
```

Nested transactions:

```go
inner := tx.NewTX()
inner.Set(...)
inner.Commit()  // merges into parent tx.updates, not the store
```

---

## DB - higher-level table database

`DB` wraps `KV` and adds schema management and SQL execution.

### Opening

```go
d, err := db.NewDB("path/to/data")
defer d.Close()
```

`NewDB` accepts the same `KVOption` functions as `NewKV`.

### Schema definition

```go
schema := &db.Schema{
    Table: "users",
    Cols: []db.Column{
        {Name: "id",   Type: db.TypeI64},
        {Name: "name", Type: db.TypeStr},
        {Name: "age",  Type: db.TypeI64},
    },
    Indices: [][]int{
        {0},    // primary key: id
        {2, 0}, // secondary index: (age, id)
    },
}
```

### Row operations

```go
row := schema.NewRow()
row[0] = db.Cell{Type: db.TypeI64, I64: 42}
row[1] = db.Cell{Type: db.TypeStr, Str: []byte("alice")}
row[2] = db.Cell{Type: db.TypeI64, I64: 30}

tx := d.NewTX()
defer tx.Abort()

updated, err := tx.Insert(schema, row)   // fails if id=42 exists
updated, err := tx.Upsert(schema, row)   // create or replace
updated, err := tx.Update(schema, row)   // fails if id=42 absent
deleted, err := tx.Delete(schema, row)   // deletes all index entries

ok, err := tx.Select(schema, row)        // fetch by primary key (row[0] must be set)

tx.Commit()
```

### Range iteration

```go
iter, err := tx.Seek(schema, row)  // forward scan from row's primary key
for ; err == nil && iter.Valid(); err = iter.Next() {
    r := iter.Row()  // []db.Cell
}
```

```go
req := &db.RangeReq{
    StartCmp: db.OP_GE,
    StopCmp:  db.OP_LE,
    Start:    []db.Cell{{Type: db.TypeI64, I64: 10}},
    Stop:     []db.Cell{{Type: db.TypeI64, I64: 50}},
    IndexNo:  0,  // primary key index
}
iter, err := tx.Range(schema, req)
```

### SQL execution

```go
p := db.NewParser(`
    CREATE TABLE users (
        id int64,
        name string,
        age int64,
        PRIMARY KEY (id),
        INDEX (age)
    );
`)
stmt, err := p.parseStmt()
_, err = d.ExecStmt(stmt)

p = db.NewParser(`SELECT id, name FROM users WHERE id >= 1 AND id <= 100;`)
stmt, _ = p.parseStmt()
result, err := d.ExecStmt(stmt)
for _, row := range result.Values {
    fmt.Printf("id=%d name=%s\n", row[0].I64, row[1].Str)
}
```

`SQLResult`:

```go
type SQLResult struct {
    Updated int      // rows affected by INSERT / UPDATE / DELETE
    Header  []string // column names for SELECT
    Values  []Row    // result rows for SELECT
}
```

---

## UpdateMode

```go
const (
    ModeUpsert UpdateMode = iota + 1  // insert or update
    ModeInsert                         // insert only (no-op if exists)
    ModeUpdate                         // update only (no-op if absent)
)
```

---

## Error values

| Error | Description |
|---|---|
| `db.ErrTXConflict` | Transaction committed after another write to the same key |
| `db.ErrOutOfRange` | Key prefix does not match expected schema table/index |
| `db.ErrBadSum` | WAL entry failed CRC32 checksum (treated as EOF during recovery) |

---

## SortedKV interface

For advanced use, you can implement `SortedKV` to plug custom data sources into the merge iterator:

```go
type SortedKV interface {
    EstimatedSize() int
    Iter() (SortedKVIter, error)
    Seek(key []byte) (SortedKVIter, error)
}

type SortedKVIter interface {
    Valid() bool
    Key() []byte
    Val() []byte
    Deleted() bool
    Next() error
    Prev() error
}
```
