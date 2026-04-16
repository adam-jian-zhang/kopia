# Internal Support Packages

This document covers the supporting `internal/` packages that underpin the core Kopia stack.

---

## `internal/gather` – Non-Contiguous Byte Buffers

### Purpose

Provides a `WriteBuffer` that accumulates bytes in a linked list of chunks rather than a single contiguous slice. This avoids large memory allocations and copies when assembling pack blobs.

### Key Types

| Type | Description |
|---|---|
| `WriteBuffer` | Appendable, non-contiguous byte buffer; implements `io.Writer` |
| `Bytes` | Read-only view over a `WriteBuffer` or slice; implements `blob.Bytes` |
| `WriteBufferChunk` | Fixed-size chunk (4 KB) pooled to reduce GC pressure |

`WriteBuffer.Bytes()` returns a `Bytes` view that can be passed to `blob.Storage.PutBlob` without copying.

---

## `internal/metrics` – Observability

Provides a lightweight, structured metrics system. All metrics are stored in-process in a `Registry` and can be snapshotted on demand.

### Metric Types

| Type | Use |
|---|---|
| `Counter` | Monotonically increasing integer counts |
| `Throughput` | Bytes/second rate |
| `Distribution[T]` | Histogram of `time.Duration` or `int64` (bytes) |

`Snapshot` captures point-in-time values for all metrics, with start/end timestamps and user/hostname.

---

## `internal/auth` – Authentication & Authorization

### Authentication (`authn.go`, `authn_repo.go`)

`Authenticator` interface:
```go
type Authenticator interface {
    IsValid(ctx, rep, username, password string) bool
}
```

`repositoryAuthenticator` validates users against password hashes stored in repository manifests (`type=user`). Passwords are hashed with scrypt.

### Authorization (`authz.go`, `authz_acl.go`)

`Authorizer` interface:
```go
type Authorizer interface {
    AuthorizeAccess(ctx, rep, username string, op Op, args ...string) bool
}
```

The ACL-based authorizer checks an ordered list of ACL rules (stored in repository manifests). Each rule specifies a user pattern, a resource pattern, and an `AccessLevel` (read / append / full).

---

## `internal/acl` – Access Control Lists

Stores and evaluates fine-grained ACL rules for multi-user repository server deployments.

### `ACL`

```go
type ACL struct {
    User         string
    Resource     string
    AccessLevel  AccessLevel  // Read | Append | Full
}
```

`ACLManager` loads ACL manifests on startup and provides `GetEffectiveAccessLevel(user, resource)`.

---

## `internal/epoch` (covered separately)

See [pkg-internal-epoch.md](pkg-internal-epoch.md).

---

## `internal/scheduler` – Background Task Scheduler

Runs recurring background tasks at policy-defined intervals. Used by `internal/server` to trigger snapshot uploads.

```go
type Scheduler struct {
    // map of sourceID → next run time and task func
}
func (s *Scheduler) Schedule(id string, nextRun time.Time, fn func(ctx))
func (s *Scheduler) Cancel(id string)
```

---

## `internal/uitask` – Async Task Tracking

Provides a registry of **UI-visible background tasks** (snapshots, restores, maintenance). Each task has:
- A unique ID
- A description and start time
- A `LogEntry` stream (log lines produced during execution)
- A progress counter (bytes, files)
- Cancellation support

The `/api/v1/tasks` REST endpoint exposes these to the HTML UI for real-time progress display.

---

## `internal/parallelwork` – Parallel Work Queue

A bounded parallel work queue with error collection. Used internally to run multiple goroutines with a shared semaphore and collect the first error.

---

## `internal/workshare` – Work-Sharing Pool

Provides a dynamic goroutine pool where work items can be stolen by idle workers. Used by `snapshotfs.Uploader` to parallelize file uploads within a directory without unbounded goroutine spawning.

---

## `internal/retry` – Retry with Backoff

Implements exponential backoff retry for transient storage errors. Used by the `blob/retrying` wrapper and the server's repository initialization loop.

---

## `internal/clock` – Mockable Time

Wraps `time.Now()` behind an interface to allow test-controlled time advancement.

---

## `internal/logfile` – Log File Management

Manages Kopia's on-disk log files:
- Rotates log files by size and age.
- Writes log blobs to the repository (`l...` prefix) for remote diagnostic access.
- Attaches structured log capture to CLI commands.

---

## `internal/crypto` – Cryptographic Utilities

Low-level helpers: scrypt key derivation, password hashing, HMAC utilities. Used by `repo/format` and `internal/auth`.

---

## `internal/tlsutil` – TLS Utilities

Generates and manages TLS certificates for the HTTP server. Supports auto-generation of self-signed certificates and certificate pinning (fingerprint verification on the client side).

---

## `internal/grpcapi` – gRPC Protocol Definitions

Contains `repository_server.proto` and its generated Go bindings. Defines the bidirectional streaming RPC protocol used between CLI clients and the repository server.

---

## `internal/serverapi` – REST API Types

Defines the Go structs that are JSON-serialized for the REST API request/response bodies. Shared between the server and the `apiclient` package.

---

## `internal/apiclient` – HTTP API Client

`KopiaAPIClient` wraps `http.Client` with Kopia-specific authentication (cookie + CSRF token), TLS configuration, and JSON marshaling helpers. Used by the CLI in server-proxy mode.

---

## `internal/bigmap` – Large Map

A hash map that spills to disk when the in-memory size exceeds a threshold. Used during GC and index operations that may process millions of content IDs.

---

## `internal/completeset` – Complete Set Tracker

Tracks whether a set of expected items has all been received. Used by the epoch manager to determine when a compaction result set is complete.

---

## `internal/diff` – Snapshot Diff

Computes the difference between two directory trees (or snapshots). Used by `kopia diff` and the `cli/command_diff.go` command.

---

## `internal/sparsefile` – Sparse File Support

Platform-specific helpers for creating sparse files on restore. On filesystems that support holes (ext4, APFS, NTFS), sparse regions in restored files are represented as actual holes rather than zero-filled bytes.

---

## `internal/ospath` – OS Path Utilities

Cross-platform path helpers (long-path support on Windows, path normalization).

---

## `internal/wcmatch` – Wildcard Matching

Implements glob/wildcard pattern matching used by `ignorefs` and `FilesPolicy.Ignore` rules. Supports `*`, `**`, and `?` patterns.
