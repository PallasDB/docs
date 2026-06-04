# gRPC API

Source: `grpc/server.go`, `grpc/cluster_server.go`, `proto/pallasdb/v1/kv.proto`

PallasDB exposes its key-value store and cluster management over gRPC using Protocol Buffers v3.

---

## Services

```protobuf
service KVService {
  rpc Get(GetRequest) returns (GetResponse);
  rpc Put(PutRequest) returns (PutResponse);
  rpc Delete(DeleteRequest) returns (DeleteResponse);
  rpc Range(RangeRequest) returns (stream RangeResponse);
}

service ClusterService {
  rpc Join(JoinRequest) returns (JoinResponse);
  rpc ListMembers(ListMembersRequest) returns (ListMembersResponse);
  rpc GetLeader(GetLeaderRequest) returns (GetLeaderResponse);
}
```

---

## KVService

### Get

```protobuf
message GetRequest  { bytes key = 1; }
message GetResponse { bytes value = 1; }
```

Fetches the value for `key`. Returns `codes.NotFound` if the key does not exist. Returns `codes.InvalidArgument` if `key` is empty.

### Put

```protobuf
enum PutMode {
  PUT_MODE_UNSPECIFIED = 0;  // treated as UPSERT
  PUT_MODE_UPSERT = 1;
  PUT_MODE_INSERT = 2;
  PUT_MODE_UPDATE = 3;
}

message PutRequest  { bytes key = 1; bytes value = 2; PutMode mode = 3; }
message PutResponse { bool updated = 1; }
```

Inserts or updates `key` according to `mode`:

| Mode | Behavior |
|---|---|
| `UPSERT` / `UNSPECIFIED` | Create or overwrite |
| `INSERT` | Only create; no-op if key exists |
| `UPDATE` | Only update; no-op if key does not exist |

`updated=true` if the store was actually modified.

### Delete

```protobuf
message DeleteRequest  { bytes key = 1; }
message DeleteResponse { bool deleted = 1; }
```

Deletes `key`. Returns `codes.NotFound` if the key does not exist.

### Range (server-streaming)

```protobuf
message RangeRequest  { bytes start = 1; bytes stop = 2; bool descending = 3; }
message RangeResponse { bytes key = 1; bytes value = 2; }
```

Streams all live key-value pairs in `[start, stop]` (ascending) or `[stop, start]` (descending). Both `start` and `stop` are required. The stream ends when the key range is exhausted or the client cancels.

---

## ClusterService

### Join

```protobuf
message JoinRequest  { string node_id = 1; string raft_addr = 2; }
message JoinResponse {}
```

Called by a new node to register itself with the leader. The leader calls `raft.AddVoter` to add the joining node to the Raft configuration.

### ListMembers

```protobuf
message ListMembersResponse {
  repeated ClusterMember members = 1;
}

message ClusterMember {
  string node_id = 1;
  string grpc_addr = 2;
  string raft_addr = 3;
  string serf_addr = 4;
  ClusterMemberStatus status = 5;
  bool raft_voter = 6;
  bool leader = 7;
}
```

Returns all known cluster members, merging information from both Raft's server list and Serf's member list.

### GetLeader

```protobuf
message GetLeaderResponse { string node_id = 1; string raft_addr = 2; }
```

Returns the current Raft leader's node ID and Raft transport address.

---

## Server implementation

### KVServer

`KVServer` in `grpc/server.go` wraps a `*db.KV` and implements `KVServiceServer`.

For **Get / Put / Delete**, the flow is:
1. Validate that `key` is non-empty.
2. Check `ctx.Err()` (cancelled/deadline exceeded).
3. Call the corresponding `db.KV` method.
4. Map errors to gRPC status codes.

For **Range**, a transaction is opened (`store.NewTX()`) and held for the duration of the stream. The transaction provides a point-in-time consistent view of the store.

### Health check

`NewGRPCServer` also registers the standard gRPC health service (`grpc/health/grpc_health_v1`):

```go
healthSrv := health.NewServer()
healthSrv.SetServingStatus("", healthpb.HealthCheckResponse_SERVING)
```

### Logging interceptor

`LoggingUnaryInterceptor` is available for unary RPCs. It logs the method name, gRPC status code, and duration using `log/slog`.

```go
srv := grpcapi.NewGRPCServer(store, grpc.ChainUnaryInterceptor(
    grpcapi.LoggingUnaryInterceptor(logger),
))
```

### Graceful shutdown

`Serve(ctx, lis, srv)` blocks until `ctx` is cancelled. On cancellation, `GracefulStop` is called with a 15-second timeout. If graceful stop does not complete within the timeout, `srv.Stop()` is called.

---

## Regenerating protobuf code

```sh
go install github.com/bufbuild/buf/cmd/buf@latest
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
buf generate
```

The `buf.yaml` and `buf.gen.yaml` in the repository root configure the buf toolchain.
