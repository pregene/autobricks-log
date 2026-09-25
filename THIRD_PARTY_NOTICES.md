# Third-party notices

## FUSE development references

Autobricks WORM Filesystem by Autobricks, Co. contains a Rust Appendable WORM policy core for LOCK advancement, creation-based retention, and deletion eligibility. Its Linux FUSE adapter adds reserved metadata name checks and rename restrictions to filesystem callbacks.

## Rust fuser — MIT

- Project: https://github.com/cberner/fuser
- Version: 0.16.0
- Upstream license: https://github.com/cberner/fuser/blob/v0.16.0/LICENSE.md
- Original notice: [licenses/fuser-0.16.0-MIT.txt](licenses/fuser-0.16.0-MIT.txt)

The callback signatures in `src/fuse/` follow the fuser Filesystem interface.

## Rust libc — MIT OR Apache-2.0

- Project: https://github.com/rust-lang/libc
- Version: 0.2.189
- License: MIT OR Apache-2.0
- Purpose: Unix error constants and system calls in mount tests

## libfuse — LGPL 2.1 / GPL 2 by component

- Project: https://github.com/libfuse/libfuse
- Component license assignments: https://github.com/libfuse/libfuse/blob/master/LICENSE
- LGPL text: https://github.com/libfuse/libfuse/blob/master/LGPL2.txt
- GPL text: https://github.com/libfuse/libfuse/blob/master/GPL2.txt

The upstream LICENSE assigns LGPL 2.1 to `include/`, `lib/`, and `meson.build`, and GPL 2 to the remaining files.

## Persistent storage

The Rust storage layer uses RustCrypto SHA-256, Serde JSON, and fs2. Their direct and transitive licenses are recorded in [the dependency review](docs/DEPENDENCY_LICENSES.md#persistent-storage-dependencies), with upstream license notices in [licenses/rust](licenses/rust/).
