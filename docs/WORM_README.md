# Autobricks WORM Filesystem (upstream reference)

Autobricks WORM is an appendable WORM (Write Once, Read Many) filesystem.
Committed bytes cannot be overwritten, while new bytes can be appended at the
current lock boundary.

The command-line executable is `ab-worm`.

## Status

- Linux: FUSE mount, storage policy, test workflow, and Debian packaging.

For Autobricks Log installation, see [INSTALL.md](../INSTALL.md). For rsyslog
routing and program-directory operation, see [HOWTO.md](../HOWTO.md).

## Development Setup

Ubuntu:

```sh
sudo apt-get update
sudo apt-get install -y build-essential pkg-config fuse3
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
. "$HOME/.cargo/env"
```

Clone, build, and test:

```sh
git clone https://github.com/pregene/autobricks-worm.git
cd autobricks-worm
cargo test --locked
./build.sh --release
./target/release/ab-worm --version
```

Linux mount integration test:

```sh
tests/linux-test.sh
```

The integration test builds `bin/ab-worm`, mounts `worm-storage` at
`worm-mount`, checks append-only behavior, directory behavior, metadata
protection, delete rejection during retention, and delete success with
zero-day retention.

## CLI

```sh
ab-worm --help
ab-worm --version
ab-worm mount SOURCE MOUNTPOINT --retain DAYS [--allow-other]
ab-worm unmount MOUNTPOINT
```

Example:

```sh
sudo mkdir -p /worm-storage /mnt/worm-storage
sudo ab-worm mount /worm-storage /mnt/worm-storage --retain 365 --allow-other
```

## Appendable WORM Structure

Each file has data plus a metadata record. The metadata stores the creation
time, fixed retention deadline, committed lock offset, checksum, and incremental
hash state.

```mermaid
flowchart LR
    subgraph DataFile["audit.log"]
        A["Record A"]
        B["Record B"]
        C["Record C"]
        L["LOCK offset"]
    end

    A --> B --> C --> L

    subgraph MetaFile["audit.log.meta"]
        M1["created_at"]
        M2["retain_until"]
        M3["lock_offset"]
        M4["sha256 / hash state"]
    end

    DataFile -. "described by" .-> MetaFile
```

Append handling:

```mermaid
sequenceDiagram
    participant App as Writer
    participant WORM as WORM policy
    participant Data as Data file
    participant Meta as Metadata

    App->>WORM: write(offset, bytes)
    WORM->>Meta: read lock_offset
    alt offset == lock_offset
        WORM->>Data: append bytes
        WORM->>Meta: update lock_offset and hash state
        WORM-->>App: success
    else offset < lock_offset
        WORM-->>App: reject overwrite
    else offset > lock_offset
        WORM-->>App: reject gap
    end
```

Retention is fixed when the file is created. Appending changes `lock_offset` and
hash state, but it does not extend `retain_until`.

```mermaid
flowchart TD
    Create["create file"] --> Policy["created_at + retain_seconds = retain_until"]
    Policy --> Append1["append Record A"]
    Append1 --> Append2["append Record B"]
    Append2 --> Append3["append Record C"]
    Append3 --> DeleteCheck{"delete requested"}
    DeleteCheck -->|now < retain_until| Deny["deny delete"]
    DeleteCheck -->|now >= retain_until| Allow["allow delete"]
```

On Linux, retention checks use a monotonic boot clock so wall-clock changes do
not make retained files expire early.

## FUSE Policy Flow

On Linux, `ab-worm mount` opens the backing storage as root and exposes it
through a FUSE mount. Users interact with the mount point; policy checks happen
before backing files are changed.

```mermaid
flowchart TD
    User["user process"] --> VFS["Linux VFS"]
    VFS --> FUSE["FUSE request"]
    FUSE --> Adapter["ab-worm FUSE adapter"]
    Adapter --> Namespace["namespace policy"]
    Adapter --> FilePolicy["file policy"]
    Adapter --> Store["backing storage"]

    Namespace --> N1["reject .meta creation"]
    Namespace --> N2["reject rename and move"]
    Namespace --> N3["reject invalid path names"]

    FilePolicy --> P1["accept append at LOCK"]
    FilePolicy --> P2["reject overwrite or gap"]
    FilePolicy --> P3["reject delete before retention"]

    Store --> Disk["source directory"]
```

Common request handling:

```mermaid
sequenceDiagram
    participant U as User
    participant K as Kernel/FUSE
    participant A as ab-worm adapter
    participant P as Policy
    participant S as Store

    U->>K: create/write/read/delete/rename
    K->>A: FUSE callback
    A->>P: validate namespace and WORM rules
    alt allowed
        A->>S: apply durable storage operation
        S-->>A: result
        A-->>K: success
        K-->>U: success
    else denied
        A-->>K: EPERM/EACCES/EROFS
        K-->>U: failure
    end
```

Read-only metadata is exposed as a sibling `.meta` file. Applications can read
metadata through the mount, but user writes, truncates, deletes, renames, and
manual `.meta` creation are rejected.

## Packaging

Build local Ubuntu packages:

```sh
scripts/package_deb.sh
```

Build Ubuntu 22.04 and 24.04 packages for `amd64` and `arm64` with Docker:

```sh
scripts/package_deb_docker.sh
AB_WORM_PACKAGE_IMAGE=ubuntu:24.04 AB_WORM_PACKAGE_OS_VERSION=24.04 scripts/package_deb_docker.sh
```

Generated packages are written to `build/`.

## Verification

```sh
cargo test --locked
cargo fmt --check
cargo clippy --locked --all-targets -- -D warnings
python tests/build_version.py
```

Linux FUSE tests need `/dev/fuse` and mount permission:

```sh
cargo test --locked --test fuse_namespace -- --ignored
tests/linux-test.sh
```

## Licenses

FUSE and Rust `fuser` references, component licenses, and the original MIT notice
are recorded in [THIRD_PARTY_NOTICES.md](../THIRD_PARTY_NOTICES.md).

The Linux FUSE and Rust dependency review is recorded in
[Linux dependency licenses](DEPENDENCY_LICENSES.md).
