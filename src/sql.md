# SQL Parser & Evaluator

Source: `db/sql_parser.go`, `db/eval.go`, `db/table.go`

PallasDB includes a hand-written recursive-descent SQL parser and expression evaluator. The SQL layer runs on top of the KV store and uses the `DB` / `DBTX` types to manage schemas and data.

---

## Supported statements

| Statement | Syntax |
|---|---|
| `CREATE TABLE` | `CREATE TABLE name (col type, ..., PRIMARY KEY (cols), INDEX (cols));` |
| `SELECT` | `SELECT expr, ... FROM table WHERE cond;` |
| `INSERT INTO` | `INSERT INTO table VALUES (val, ...);` |
| `UPDATE` | `UPDATE table SET col = expr, ... WHERE cond;` |
| `DELETE FROM` | `DELETE FROM table WHERE cond;` |

Column types: `int64`, `string`.

---

## Parser

`Parser` is a struct holding the input string and current position:

```go
type Parser struct {
    buf   string
    pos   int
    depth int  // guards against deeply nested expressions
}
```

`NewParser(s string)` creates a parser. `parseStmt()` dispatches to the appropriate parse function based on the first keyword.

### Token model

The parser does not produce a separate token stream - it operates directly on the input string with helper methods:

- `tryKeyword(kws ...string)` - case-insensitive keyword match (requires separator after)
- `tryPunctuation(tok string)` - exact string match
- `tryName()` - identifier: starts with letter or `_`, continues with `[a-zA-Z0-9_]`
- `parseValue(out *Cell)` - integer or string literal

### Expression grammar

```
expr   ::= or
or     ::= and ("OR" and)*
and    ::= not ("AND" not)*
not    ::= "NOT" not | cmp
cmp    ::= add (("=" | "!=" | "<>" | "<=" | ">=" | "<" | ">") add)*
add    ::= mul (("+" | "-") mul)*
mul    ::= neg (("*" | "/") neg)*
neg    ::= "-" neg | atom
atom   ::= "(" expr ")" | name | value
```

The maximum expression depth is 1,000 (`maxExprDepth`) to prevent stack overflow from deeply nested `NOT` or negation chains.

### AST nodes

```go
type ExprBinOp struct { op ExprOp; left, right any }
type ExprUnOp  struct { op ExprOp; kid any }
type ExprTuple struct { kids []any }
// leaf nodes: string (column name) or *Cell (literal value)
```

Operators:

| Constant | Symbol |
|---|---|
| `OP_ADD` | `+` |
| `OP_SUB` | `-` |
| `OP_MUL` | `*` |
| `OP_DIV` | `/` |
| `OP_EQ` | `=` |
| `OP_NE` | `!=`, `<>` |
| `OP_LE` | `<=` |
| `OP_GE` | `>=` |
| `OP_LT` | `<` |
| `OP_GT` | `>` |
| `OP_AND` | `AND` |
| `OP_OR` | `OR` |
| `OP_NOT` | `NOT` |
| `OP_NEG` | unary `-` |

---

## Expression evaluator

`evalExpr(schema *Schema, row Row, expr any) (*Cell, error)` evaluates an AST node against a row:

- **String (column name)** - looks up the column index in the schema and returns the cell from the row.
- **`*Cell` (literal)** - returns the literal directly.
- **`ExprUnOp`** - applies `OP_NEG` (negate integer) or `OP_NOT` (boolean not).
- **`ExprBinOp`** - evaluates both sides and applies the operator.

Arithmetic operations (`+`, `-`, `*`, `/`) require both sides to be `TypeI64`. Division by zero returns an error.

Comparison operations support both `TypeI64` and `TypeStr`. Comparing different types returns an error.

Logical operators (`AND`, `OR`) work on `TypeI64` values: `0` is false, anything else is true.

---

## Schema management

Schemas are stored as JSON-encoded `Schema` values in the KV store under the key `@schema_<table>`:

```go
val, _ := json.Marshal(schema)
tx.kv.Set([]byte("@schema_"+table), val)
```

`GetSchema(table)` checks a per-transaction cache (`tx.tables`) first, then falls back to a KV lookup. This avoids repeated schema deserialization within a transaction.

---

## WHERE clause optimization

The SQL executor does not use a full-table scan. Instead, `makeRange(schema, cond)` analyzes the `WHERE` clause and attempts to convert it into a `RangeReq`:

1. **Point lookup**: if the WHERE is a conjunction of `col = literal` equalities that exactly matches the primary key columns, produces a single-key range `[pk, pk]`.

2. **Range scan**: if the WHERE is a comparison (`<`, `<=`, `>`, `>=`) or a conjunction of two opposite comparisons on a primary-key prefix, produces a bounded range scan. Secondary indices are also tried.

3. **Unimplemented**: if neither pattern matches, returns `"unimplemented WHERE"`. Full table scans are not supported.

This design means SQL queries must always be anchored to an index - a WHERE clause that cannot be resolved to an index range is rejected.

---

## Table operations

`DBTX` wraps `KVTX` and adds schema-aware methods:

| Method | Description |
|---|---|
| `Insert(schema, row)` | Insert a row; fails if primary key already exists |
| `Upsert(schema, row)` | Insert or update |
| `Update(schema, row)` | Update; no-op if row does not exist |
| `Delete(schema, row)` | Delete all index entries for the row |
| `Select(schema, row)` | Fetch a row by its primary key fields |
| `Seek(schema, row)` | Return a forward iterator starting at the given primary key |
| `Range(schema, req)` | Return a range iterator using a `RangeReq` |

All mutating operations maintain secondary indices: an update first deletes all existing index entries (using the old values), then inserts new ones.

---

## Executing SQL

```go
p := db.NewParser("SELECT id, name FROM users WHERE id >= 1 AND id <= 100;")
stmt, err := p.parseStmt()
result, err := dbHandle.ExecStmt(stmt)
```

`ExecStmt` wraps the execution in a transaction. `SQLResult` contains:

```go
type SQLResult struct {
    Updated int    // rows affected (INSERT/UPDATE/DELETE)
    Header  []string // column names (SELECT)
    Values  []Row    // result rows (SELECT)
}
```
