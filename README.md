# Autobricks Log

Autobricks Log is an audit-oriented logging service built on the
[Autobricks WORM filesystem](https://github.com/pregene/autobricks-worm).

The service accepts local syslog messages, separates them by program name,
protects stored bytes against modification, and removes expired files according
to the system-wide retention period selected during installation.

## Goals

- Accept messages from applications through the standard system syslog API.
- Create and manage one directory for each syslog program name.
- Create log files as `ablog-<date-time>.log` when a program first sends data.
- Make committed bytes append-only through the WORM storage layer.
- Verify file integrity and retain files for a fixed period.
- Delete expired files without giving producers deletion permission.
- Continue safely across process restarts and partial failures.
- Keep credentials and machine-specific data out of the repository.

This project is not intended to replace a general-purpose observability stack,
provide full-text search, or accept arbitrary client-selected paths.
