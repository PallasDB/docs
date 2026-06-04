# Cells and Rows

Source: `db/cell.go`, `db/row.go`

PallasDB's lowest encoding layer provides two abstractions: **Cell** (a single typed value) and **Row** (an ordered collection of Cells).

---

## Cell

A `Cell` holds one of two types:

```go
type CellType uint8

const (
    TypeI64 CellType = 1  // int64
    TypeStr CellType = 2  // []byte (variable-length string)
)

type Cell struct {
    Type CellType
    I64  int64
    Str  []byte
}
```

### Key encoding

Keys must sort correctly when compared as raw bytes. PallasDB encodes each type differently:

**TypeI64** - big-endian with sign bit flipped:

```
uint64(i64) XOR (1 << 63)  →  8 bytes, big-endian
```

The XOR flips the sign bit so that negative integers sort before positive integers in unsigned byte order (e.g., `-1` sorts before `0`).

**TypeStr** - null-terminated with escape sequences:

| Input byte | Encoded bytes |
|---|---|
| `0x00` | `0x01 0x01` |
| `0x01` | `0x01 0x02` |
| other | byte unchanged |
| end-of-string | `0x00` |

This scheme preserves lexicographic order and allows variable-length strings to be compared without knowing their length in advance.

### Value encoding

Values do not need to sort, so a simpler encoding is used:

**TypeI64** - 8 bytes, little-endian.

**TypeStr** - 4-byte little-endian length prefix followed by the raw bytes.

---

## Row

A `Row` is a `[]Cell` whose length matches the schema's column count.

```go
type Schema struct {
    Table   string
    Cols    []Column    // column definitions
    Indices [][]int     // each index is a list of column positions
}

type Column struct {
    Name string
    Type CellType
}
```

### Key encoding for rows

```
[table_name] 0x00 [index_number(1 byte)]
  [type_byte(1)] [encoded_cell] ...  (for each indexed column)
0x00
```

The table name prefix and index number isolate keys for different tables and different indices in the same KV namespace.

### Value encoding for rows

Only non-primary-key columns are stored in the value. Primary key columns are already encoded in the key. Each non-PK column appends its `EncodeVal` output in column order.

### Index keys

Secondary indices encode the indexed columns plus the primary key columns (to keep keys unique). The value for a secondary index entry is empty - the primary key is already in the key itself, so a secondary index lookup requires a follow-up primary key lookup to fetch the full row.

---

## Example

For a schema with columns `(id int64, name string)` and primary key `(id)`:

```
Key:   "users" 0x00 0x00  TypeI64 [big-endian id ^ sign]  0x00
Value: [4-byte length] [name bytes]
```

A secondary index on `(name, id)` would produce:

```
Key:   "users" 0x00 0x01  TypeStr [escaped name bytes] 0x00  TypeI64 [big-endian id ^ sign]  0x00
Value: (empty)
```
