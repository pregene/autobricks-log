# Linux dependency licenses

Reviewed on 2026-09-25 against the official project sources.

| Component | License | Integration |
| --- | --- | --- |
| Rust fuser 0.16.0 | MIT | Linux Rust FUSE interface |
| Linux libfuse | LGPL 2.1 / GPL 2 by component | FUSE libraries and tools |
| Rust standard library | MIT OR Apache-2.0, with component-specific notices | Rust runtime |
| Rust libc 0.2.189 | MIT OR Apache-2.0 | Linux system calls and error constants |

## fuser and libfuse

fuser 0.16.0 uses MIT, with its original copyright and permission notice retained
in [the local license copy](../licenses/fuser-0.16.0-MIT.txt).

The [libfuse LICENSE](https://github.com/libfuse/libfuse/blob/master/LICENSE)
assigns LGPL 2.1 to `include/`, `lib/`, and `meson.build`, and GPL 2 to the
remaining files.

## Persistent storage dependencies

- `sha2` 0.11.0, `digest`, `crypto-common`, `block-buffer`, `hybrid-array`,
  `typenum`, `cpufeatures`, and `const-oid`: MIT OR Apache-2.0. SHA-256
  serialization is pinned to the sha2 0.11 format.
- `serde`, `serde_core`, `serde_derive`, `serde_json`, and `itoa`: MIT OR
  Apache-2.0. `memchr` offers MIT or Unlicense, and `zmij` uses MIT.
- `fs2` 0.4.3: MIT OR Apache-2.0, for process-level backing-store locks.
- `proc-macro2`, `quote`, and `syn`: MIT OR Apache-2.0; `unicode-ident`
  additionally includes Unicode-3.0 data terms.

Version-specific upstream notices for the Linux Rust dependency graph are
preserved in [licenses/rust](../licenses/rust/).
