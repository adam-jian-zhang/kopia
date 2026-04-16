# Kopia Documentation

Kopia is a fast, secure, open-source backup tool. This documentation is generated from source code analysis of the [kopia/kopia](https://github.com/kopia/kopia) repository.

## Contents

### Architecture

- [Architecture Overview](architecture-overview.md) – High-level system design, data flows, and storage layout

### Core Repository Layer

- [repo/blob – Storage Backends](pkg-repo-blob.md) – Blob storage interface and all backend implementations
- [repo/content – Content Manager](pkg-repo-content.md) – Content-addressable storage, packing, indexing
- [repo/object – Object Manager](pkg-repo-object.md) – Large object splitting and assembly
- [repo/manifest – Manifest Manager](pkg-repo-manifest.md) – JSON metadata store
- [repo/format – Format & Configuration](pkg-repo-format.md) – Repository configuration, key derivation, format upgrades
- [repo/maintenance – Maintenance](pkg-repo-maintenance.md) – Scheduled GC, index compaction, blob cleanup

### Cryptography & Data Integrity

- [Cryptography Packages](pkg-crypto.md) – Hashing, encryption, compression, ECC, content splitting

### Snapshot Layer

- [snapshot & sub-packages](pkg-snapshot.md) – Snapshot manifests, policies, upload, restore, GC

### Filesystem Abstraction

- [fs & sub-packages](pkg-fs.md) – Filesystem interfaces, local FS, ignore rules, mount

### User Interfaces

- [cli – Command-Line Interface](pkg-cli.md) – All CLI commands and operating modes
- [internal/server – HTTP API Server](pkg-internal-server.md) – REST API, gRPC session, HTML UI, auth

### Internal Infrastructure

- [internal/epoch – Epoch Index Manager](pkg-internal-epoch.md) – Lock-free concurrent index management
- [internal/cache – Content Cache](pkg-internal-cache.md) – Local LRU cache for content and blobs
- [notification – Notification System](pkg-notification.md) – Email, webhook, Pushover notifications
- [Internal Support Packages](pkg-internal-misc.md) – gather, metrics, auth, ACL, scheduler, uitask, and more

## Quick Reference: Package Dependency Map

```mermaid
graph BT
    blob["repo/blob"]
    format["repo/format"]
    crypto["hashing · encryption · ecc · splitter · compression"]
    epoch["internal/epoch"]
    cache["internal/cache"]
    content["repo/content"]
    object["repo/object"]
    manifest["repo/manifest"]
    repo["repo (DirectRepository)"]
    fs["fs"]
    snapshot["snapshot"]
    policy["snapshot/policy"]
    snapshotfs["snapshot/snapshotfs"]
    restore["snapshot/restore"]
    maintenance["repo/maintenance"]
    server["internal/server"]
    cli["cli"]
    notification["notification"]

    blob --> format
    blob --> crypto
    format --> content
    crypto --> content
    epoch --> content
    cache --> content
    content --> object
    content --> manifest
    object --> repo
    manifest --> repo
    format --> repo
    repo --> snapshot
    snapshot --> policy
    snapshot --> snapshotfs
    snapshot --> restore
    repo --> maintenance
    fs --> snapshotfs
    fs --> restore
    repo --> server
    repo --> cli
    snapshotfs --> cli
    snapshotfs --> server
    repo --> notification
    server --> notification
```
