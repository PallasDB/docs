# Getting Started

## Installation

Requires **Go 1.25** or later.

```sh
git clone https://github.com/teddymalhan/pallasdb.git
cd pallasdb
go mod download
go build -o pallasdb ./cmd/pallasdb
```

Run the test suite:

```sh
go test ./...
```

---

## Configuration

PallasDB reads configuration from three sources with the following precedence:

```
CLI flags  >  environment variables  >  config file  >  built-in defaults
```

Load an explicit config file:

```sh
pallasdb --config ./pallasdb.example.yaml serve grpc
```

Environment variables use the `PALLASDB_` prefix; dots and dashes become underscores:

```sh
PALLASDB_LOG_FORMAT=json \
PALLASDB_SERVE_GRPC_ADDR=:50052 \
pallasdb serve grpc
```

### Config key reference

| Config key | Env variable | Default | Description |
|---|---|---|---|
| `log.format` | `PALLASDB_LOG_FORMAT` | `text` | Log handler format: `text` or `json` |
| `shutdown.timeout` | `PALLASDB_SHUTDOWN_TIMEOUT` | `15s` | Graceful shutdown timeout |
| `local.data_dir` | `PALLASDB_LOCAL_DATA_DIR` | `data` | Data directory for local commands |
| `serve.grpc.addr` | `PALLASDB_SERVE_GRPC_ADDR` | `:50051` | Listen address for the gRPC server |
| `serve.grpc.data_dir` | `PALLASDB_SERVE_GRPC_DATA_DIR` | `data` | Data directory for the gRPC server |
| `cluster.grpc_addr` | `PALLASDB_CLUSTER_GRPC_ADDR` | `:50051` | gRPC address advertised to peers |
| `cluster.data_dir` | `PALLASDB_CLUSTER_DATA_DIR` | `data` | KV data directory for the cluster node |
| `cluster.raft_addr` | `PALLASDB_CLUSTER_RAFT_ADDR` | `:7001` | Raft TCP transport address |
| `cluster.raft_dir` | `PALLASDB_CLUSTER_RAFT_DIR` | `raft` | Directory for Raft BoltDB log and snapshots |
| `cluster.node_id` | `PALLASDB_CLUSTER_NODE_ID` | `node-1` | Unique node identifier |
| `cluster.join` | `PALLASDB_CLUSTER_JOIN` | `""` | gRPC address of an existing node to join |
| `cluster.apply_timeout` | `PALLASDB_CLUSTER_APPLY_TIMEOUT` | `10s` | Raft apply timeout |
| `cluster.serf.enabled` | `PALLASDB_CLUSTER_SERF_ENABLED` | `true` | Enable Serf gossip discovery |
| `cluster.serf.addr` | `PALLASDB_CLUSTER_SERF_ADDR` | `:7946` | Serf bind address |
| `cluster.serf.advertise_addr` | `PALLASDB_CLUSTER_SERF_ADVERTISE_ADDR` | `""` | Serf advertise address (for NAT) |
| `cluster.serf.join` | `PALLASDB_CLUSTER_SERF_JOIN` | `[]` | Serf peers to join on startup |
| `cluster.serf.event_buffer` | `PALLASDB_CLUSTER_SERF_EVENT_BUFFER` | `64` | Serf event channel buffer size |

### Example YAML config

```yaml
log:
  format: text

shutdown:
  timeout: 15s

local:
  data_dir: data

serve:
  grpc:
    addr: ":50051"
    data_dir: data

cluster:
  grpc_addr: ":50051"
  data_dir: data
  raft_addr: ":7001"
  raft_dir: raft
  node_id: node-1
  join: ""
  apply_timeout: 10s
  serf:
    enabled: true
    addr: ":7946"
    advertise_addr: ""
    join: []
    event_buffer: 64
```

---

## CLI Reference

### `pallasdb local` - local key-value operations

All local commands accept `--data-dir` to specify where data is stored.

```sh
# Insert or update a key
pallasdb local put <key> <value> --data-dir ./data

# Fetch a key
pallasdb local get <key> --data-dir ./data

# Scan a key range [start, stop]
pallasdb local range <start> <stop> --data-dir ./data

# Delete a key
pallasdb local delete <key> --data-dir ./data

# Trigger compaction
pallasdb local compact --data-dir ./data
```

### `pallasdb local benchmark` - disk-backed benchmark

```sh
pallasdb local benchmark \
  --data-dir /tmp/bench \
  --reset \
  --keys 2000000 \
  --value-size 128 \
  --batch-size 1000 \
  --read-ops 500000 \
  --compact \
  --format text \
  --output results.txt
```

| Flag | Default | Description |
|---|---|---|
| `--data-dir` | `data` | Benchmark data directory |
| `--reset` | `false` | Wipe existing data before running |
| `--keys` | `10000` | Number of keys to populate |
| `--value-size` | `128` | Value size in bytes |
| `--key-size` | `16` | Key size in bytes |
| `--batch-size` | `500` | Keys written per transaction |
| `--read-ops` | `10000` | Number of random-read operations |
| `--scan-limit` | `0` | Number of keys to scan (0 = skip) |
| `--compact` | `false` | Run explicit compaction after populate |
| `--format` | `text` | Output format: `text` or `json` |
| `--output` | `""` | Write output to file instead of stdout |

### `pallasdb serve grpc` - standalone gRPC server

```sh
pallasdb serve grpc --addr :50051 --data-dir ./data
```

### `pallasdb cluster start` - Raft cluster node

Bootstrap the first node:

```sh
pallasdb cluster start \
  --node-id node-1 \
  --grpc-addr :50051 \
  --raft-addr :7001 \
  --serf-addr :7946 \
  --data-dir ./data/node-1 \
  --raft-dir ./raft/node-1
```

Join a running cluster via Serf gossip:

```sh
pallasdb cluster start \
  --node-id node-2 \
  --grpc-addr :50052 \
  --raft-addr :7002 \
  --serf-addr :7947 \
  --serf-join localhost:7946 \
  --data-dir ./data/node-2 \
  --raft-dir ./raft/node-2
```

Join explicitly via gRPC (no Serf):

```sh
pallasdb cluster start \
  --node-id node-2 \
  --grpc-addr :50052 \
  --raft-addr :7002 \
  --serf-enabled=false \
  --join localhost:50051 \
  --data-dir ./data/node-2 \
  --raft-dir ./raft/node-2
```

### `pallasdb completion` - shell completions

```sh
pallasdb completion bash
pallasdb completion zsh
pallasdb completion fish
pallasdb completion powershell
```

### `pallasdb version`

```sh
pallasdb version
# pallasdb dev
# commit: none
# built: unknown
```
