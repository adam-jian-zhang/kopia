# Kopia Architecture Overview

Kopia is a fast, secure, open-source backup tool written in Go. It provides end-to-end encrypted, deduplicated, compressed snapshots of files and directories stored in a variety of cloud and local backends.

## High-Level Architecture

At the highest level, Kopia is structured around three major concerns:

1. **Frontend** – how users interact with Kopia (CLI, GUI, HTTP API, gRPC).
2. **Repository** – the core abstraction that orchestrates storage, content management, manifests, and format.
3. **Storage Backends** – the pluggable blob storage layer (S3, GCS, Azure, filesystem, …).

```mermaid
graph TD
    User["User / Client"]

    subgraph Frontend
        CLI["cli (CLI commands)"]
        App["app (Electron GUI)"]
        Server["internal/server (HTTP/REST API)"]
        GRPC["internal/grpcapi (gRPC API)"]
    end

    subgraph Repository["repo – Repository Layer"]
        RepoPkg["repo (DirectRepository / RepositoryWriter)"]
        Format["repo/format (Format & Config Manager)"]
        Content["repo/content (Content Manager / WriteManager)"]
        Object["repo/object (Object Manager)"]
        Manifest["repo/manifest (Manifest Manager)"]
        Epoch["internal/epoch (Epoch Index Manager)"]
        Cache["internal/cache (Content & Blob Cache)"]
    end

    subgraph Crypto["Cryptography"]
        Hashing["repo/hashing (BLAKE2/SHA)"]
        Encryption["repo/encryption (AES-256-GCM / ChaCha20)"]
        ECC["repo/ecc (Error Correction)"]
        Splitter["repo/splitter (Content Splitting)"]
        Compression["repo/compression (zstd, lz4, gzip, …)"]
    end

    subgraph SnapshotLayer["Snapshot Layer"]
        Snapshot["snapshot (Snapshot Manifests)"]
        Policy["snapshot/policy (Backup Policies)"]
        SnapshotFS["snapshot/snapshotfs (Upload / Tree Walker)"]
        Restore["snapshot/restore (Restore Engine)"]
        GC["snapshot/snapshotgc (Garbage Collection)"]
        Maintenance["repo/maintenance (Maintenance Tasks)"]
    end

    subgraph FSLayer["Filesystem Abstraction"]
        FS["fs (Entry / File / Directory)"]
        LocalFS["fs/localfs (Local FS)"]
        CacheFS["fs/cachefs (Cached FS)"]
        IgnoreFS["fs/ignorefs (Ignore Rules)"]
        VirtualFS["fs/virtualfs (Virtual FS)"]
    end

    subgraph BlobStorage["repo/blob – Storage Backends"]
        S3["s3"]
        GCS["gcs"]
        Azure["azure"]
        B2["b2"]
        Filesystem["filesystem"]
        WebDAV["webdav"]
        SFTP["sftp"]
        Rclone["rclone"]
        GDrive["gdrive"]
    end

    subgraph Notification["Notification"]
        NotifSender["notification/sender"]
        Email["email"]
        Webhook["webhook"]
        Pushover["pushover"]
    end

    User --> CLI
    User --> App
    User --> Server
    CLI --> RepoPkg
    App --> Server
    Server --> RepoPkg
    Server --> GRPC
    GRPC --> RepoPkg

    RepoPkg --> Format
    RepoPkg --> Content
    RepoPkg --> Object
    RepoPkg --> Manifest

    Content --> Epoch
    Content --> Cache
    Content --> Hashing
    Content --> Encryption
    Content --> ECC
    Content --> Splitter
    Content --> Compression

    Object --> Content
    Manifest --> Content

    RepoPkg --> BlobStorage
    Content --> BlobStorage
    Epoch --> BlobStorage
    Cache --> BlobStorage

    SnapshotFS --> Object
    SnapshotFS --> FS
    SnapshotFS --> Policy
    SnapshotFS --> Snapshot
    Restore --> Object
    Restore --> FS
    GC --> Manifest
    Maintenance --> Content
    Maintenance --> BlobStorage

    FS --> LocalFS
    FS --> CacheFS
    FS --> IgnoreFS
    FS --> VirtualFS

    Server --> NotifSender
    NotifSender --> Email
    NotifSender --> Webhook
    NotifSender --> Pushover
```

## Data Flow: Creating a Snapshot

When a user runs `kopia snapshot create`, the following pipeline executes:

```mermaid
sequenceDiagram
    participant User
    participant CLI
    participant SnapshotFS
    participant Policy
    participant FS
    participant Object
    participant Content
    participant Blob as BlobStorage

    User->>CLI: kopia snapshot create /path
    CLI->>SnapshotFS: Uploader.Upload(ctx, dir, policy)
    SnapshotFS->>Policy: resolve effective policy tree
    SnapshotFS->>FS: iterate directory tree
    loop for each file
        FS-->>SnapshotFS: Entry (file metadata + reader)
        SnapshotFS->>Object: ObjectWriter.Write(data chunks)
        Object->>Content: WriteContent(chunk, prefix, compression)
        Content->>Content: hash → dedup check
        Content->>Content: compress (if policy)
        Content->>Content: encrypt (AES-256-GCM / ChaCha20)
        Content->>Blob: PutBlob (pack blob "p...")
    end
    SnapshotFS->>Object: flush directory manifests
    Object->>Content: write dir manifest content ("q..." prefix)
    SnapshotFS->>CLI: snapshot manifest (root OID)
    CLI->>Content: flush index blobs
    Content->>Blob: PutBlob (index blobs "n..." / epoch blobs)
    CLI->>Content: write snapshot manifest ("m..." prefix)
```

## Data Flow: Restoring a Snapshot

```mermaid
sequenceDiagram
    participant User
    participant CLI
    participant Restore
    participant Object
    participant Content
    participant Cache
    participant Blob as BlobStorage

    User->>CLI: kopia snapshot restore <snapshot-id> /dest
    CLI->>Content: load snapshot manifest
    CLI->>Restore: RestoreEntryFromReader(ctx, output, entry)
    Restore->>Object: OpenObject(ctx, rootOID)
    Object->>Content: GetContent(contentID)
    Content->>Cache: lookup content in local cache
    alt cache miss
        Cache->>Blob: GetBlob(packBlobID, offset, length)
        Blob-->>Cache: encrypted bytes
        Cache-->>Content: cached
    end
    Content->>Content: decrypt → decompress
    Content-->>Object: plaintext bytes
    Object-->>Restore: reader over file data
    Restore->>Restore: write to output (localfs / tar / zip)
    Restore-->>User: restored files
```

## Repository Storage Layout

Kopia stores data in a flat blob namespace. Each blob has a well-known prefix that identifies its role:

```mermaid
graph LR
    Repo["Repository (blob store)"]
    Repo --> kopia_repo["kopia.repository (encrypted format blob)"]
    Repo --> kopia_blobcfg["kopia.blobcfg (blob storage config)"]
    Repo --> pack_p["p... blobs (regular pack data)"]
    Repo --> pack_q["q... blobs (special/dir pack data)"]
    Repo --> index_n["n... blobs (index blobs – epoch-based)"]
    Repo --> manifest_m["m... blobs (manifests)"]
    Repo --> log_l["l... blobs (log / diagnostic blobs)"]
    Repo --> epoch_e["e... blobs (epoch checkpoints)"]
```

## Layer Dependencies

```mermaid
graph BT
    BlobStorage["repo/blob (Storage Backends)"]
    Format["repo/format"]
    Crypto["hashing · encryption · ecc · splitter · compression"]
    Content["repo/content (WriteManager)"]
    Epoch["internal/epoch"]
    Cache["internal/cache"]
    Object["repo/object"]
    Manifest["repo/manifest"]
    Repo["repo (DirectRepository)"]
    Snapshot["snapshot"]
    Policy["snapshot/policy"]
    SnapshotFS["snapshot/snapshotfs"]
    Restore["snapshot/restore"]
    Server["internal/server"]
    CLI["cli"]

    BlobStorage --> Format
    BlobStorage --> Crypto
    Format --> Content
    Crypto --> Content
    Epoch --> Content
    Cache --> Content
    Content --> Object
    Content --> Manifest
    Object --> Repo
    Manifest --> Repo
    Format --> Repo
    Repo --> Snapshot
    Snapshot --> Policy
    Snapshot --> SnapshotFS
    Snapshot --> Restore
    Repo --> Server
    Repo --> CLI
    SnapshotFS --> CLI
    SnapshotFS --> Server
```

## Key Design Principles

| Principle | Implementation |
|---|---|
| Content-addressable storage | Every chunk is identified by its cryptographic hash (BLAKE2B / SHA-256 / BLAKE3). Identical content is stored only once. |
| Zero-knowledge encryption | The repository password never leaves the client. Content is encrypted with AES-256-GCM or ChaCha20-Poly1305 using keys derived from the password via scrypt/pbkdf2. |
| Variable-length chunking | Files are split at content-defined boundaries (BuzHash32 / Rabin-Karp64 rolling hash or fixed sizes) enabling sub-file deduplication. |
| Pack blobs | Multiple small content chunks are packed together into a single blob to reduce API call overhead against cloud storage. |
| Epoch-based indexing | The index is maintained in epochs; clients write small per-write index blobs and the epoch manager compacts them asynchronously, enabling concurrent writers without locking. |
| Pluggable backends | The `repo/blob.Storage` interface is implemented independently for each cloud/local provider; the rest of the stack is backend-agnostic. |
| Policy inheritance | Snapshot policies inherit from parent paths and global defaults, resolved at upload time. |
