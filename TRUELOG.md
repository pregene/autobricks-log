# Autobricks True Log 0.3.55 — Installation and Usage

**Applies to `autobricks-truelog` and `autobricks-truelog-cli` 0.3.55.**

[Product overview](README.md) · [Download True Log 0.3.55](https://github.com/pregene/autobricks-log/releases/tag/v0.3.55)

Autobricks Log users should follow [Log installation](INSTALL.md) and [Log usage](HOWTO.md). This guide covers True Log's own server, client, commands, and services.

## Choose your True Log setup

| Use case | Required components | Follow these sections |
| --- | --- | --- |
| Collect local syslog messages on a server | True Log server; RPC can be `none` | [Server installation](#install-the-true-log-server) → [Local syslog integration](#local-syslog-integration) |
| Submit records from a remote application | True Log server with TCP/mTLS; client on the application machine | [Server installation](#install-the-true-log-server) → [Client installation](#install-the-true-log-client) → [Client writes](#write-records-through-the-client) |
| Submit records from Express | Same server/client setup; authorize the Node.js runtime account | [Client installation](#install-the-true-log-client) → [Express integration](#express-and-nodejs-integration) |
| Inspect stored chains | Administrator access to the True Log server | [Server-side inspection](#server-side-chain-inspection) |

The server stores records and maintains checksum chains. The client forwards requests over a persistent TCP or mTLS connection and returns the server's write receipt.

```text
Application → ab-truelog-cli → local client daemon → TCP/mTLS
            → ab-truelog-rpc.service → ab-truelog.service → WORM storage
```

## Select and verify packages

Select packages for each machine's Ubuntu release and architecture. The server and client may run on different machines and architectures.

```sh
. /etc/os-release
printf '%s\n' "$VERSION_ID"
dpkg --print-architecture
```

| Ubuntu | Architecture | Server | Client |
| --- | --- | --- | --- |
| 22.04 | amd64 | `autobricks-truelog-0.3.55-ubuntu-22.04-amd64.deb` | `autobricks-truelog-cli-0.3.55-ubuntu-22.04-amd64.deb` |
| 22.04 | arm64 | `autobricks-truelog-0.3.55-ubuntu-22.04-arm64.deb` | `autobricks-truelog-cli-0.3.55-ubuntu-22.04-arm64.deb` |
| 24.04 | amd64 | `autobricks-truelog-0.3.55-ubuntu-24.04-amd64.deb` | `autobricks-truelog-cli-0.3.55-ubuntu-24.04-amd64.deb` |
| 24.04 | arm64 | `autobricks-truelog-0.3.55-ubuntu-24.04-arm64.deb` | `autobricks-truelog-cli-0.3.55-ubuntu-24.04-arm64.deb` |

`amd64` means x86-64; `arm64` means AArch64. Download the required packages and `SHA256SUMS` from the release. The automatically generated source archives contain this distribution repository's documents and do not replace the installable packages.

For the 22.04 amd64 example, verify the files you downloaded:

```sh
grep -F '  autobricks-truelog-0.3.55-ubuntu-22.04-amd64.deb' SHA256SUMS | sha256sum -c -
grep -F '  autobricks-truelog-cli-0.3.55-ubuntu-22.04-amd64.deb' SHA256SUMS | sha256sum -c -
```

Each downloaded file must report `OK`. If you download all eight packages, run `sha256sum -c SHA256SUMS`.

## Install the True Log server

Run this section **on the storage server**. It installs `autobricks-truelog`, not the client. For an existing Autobricks Log installation, read [transition guidance](#transition-from-autobricks-log) first.

From the download directory, install dependencies and the server package. Replace the filename for other platforms.

```sh
sudo apt-get update
sudo apt-get install -y rsyslog fuse3 debconf systemd openssl python3
sudo dpkg -i ./autobricks-truelog-0.3.55-ubuntu-22.04-amd64.deb
```

`dpkg` does not download missing dependencies. If it reports an unmet dependency, install it with APT and rerun the package installation.

### Server installation questions

| Question | Value |
| --- | --- |
| SOURCE path | Absolute path for private WORM backing data, for example `/var/lib/autobricks-log-source`. It must differ from the mount path. |
| Retention days | One retention period for the complete storage mount. Programs cannot override it. |
| RPC support | `none` for local-only use, or `tcp` / `mtls` for client connections. |
| Bind address | Local address on which the RPC server listens. |
| Server address | IP address or DNS name clients will actually use. Also used for the mTLS server certificate. |
| Port | RPC TCP port; default `5544`. |
| Certificate fields | In mTLS mode, supply the country, region, organization, and other requested certificate information. |

Values such as `192.0.2.10` and `app01.example.test` are examples. Replace them with your deployment values. A wildcard bind address such as `0.0.0.0` is not a client destination. Allow the selected port between client and server through your network controls.

For mTLS, record the **Pairing Code shown on the installation-complete screen** and use it during client installation. Treat it as an enrollment credential. In this version it has no automatic expiry or use-count limit; enrolled clients receive shared client certificate material rather than distinct per-client certificates.

### Check the server

```sh
sudo ab-truelog check
systemctl status ab-truelog.service --no-pager
findmnt /mnt/worm-storage
```

If TCP or mTLS is enabled:

```sh
systemctl status ab-truelog-rpc.service --no-pager
sudo ss -ltnp
```

`ab-truelog.service` manages WORM and logging together. Do not start an additional standalone WORM service. RPC runs separately as `ab-truelog-rpc.service`.

## Install the True Log client

Run this section **on the application machine**. It installs `autobricks-truelog-cli`. The server must already be reachable in the selected TCP/mTLS mode.

Run installation through sudo as the user who will send initial test messages:

```sh
sudo apt-get update
sudo apt-get install -y debconf systemd acl
sudo dpkg -i ./autobricks-truelog-cli-0.3.55-ubuntu-22.04-amd64.deb
```

### Client installation questions

| Question | Value |
| --- | --- |
| Connection mode | `tcp` or `mtls`, matching the server. |
| Client hostname | Hostname recorded in forwarded logs. Defaults to the application machine's hostname; this is not the server address. |
| Server address | Reachable server IP address or DNS name. |
| RPC port | The server's configured port; default `5544`. |
| Pairing Code | mTLS only: the eight-digit code displayed by the server. |

**Pairing Code input is visible.** Both `12345678` and `1234-5678` formats are accepted, and whitespace is ignored. These are format examples; enter the actual code from your server.

First-time mTLS enrollment requires a code. Leave it blank only to reuse existing client certificates for the same server during an upgrade or reconfiguration. TCP mode skips enrollment and certificates; it provides no TLS encryption or certificate authentication.

Client settings are stored in `/etc/autobricks-truelog-cli/client.toml`, with certificates under `/etc/autobricks-truelog-cli/clients/`. After configuration and any required enrollment succeed, installation enables and starts the client service.

```sh
systemctl status ab-truelog-cli.service --no-pager
```

### Retry a failed Pairing Code or enrollment

If installation stops because the code is missing, malformed, rejected, or enrollment fails:

```sh
sudo dpkg --configure autobricks-truelog-cli
```

The installer asks for the Pairing Code again. Replace the previous entry with the correct eight digits and complete configuration. Other saved answers are retained. **Purge and reinstallation are not required to correct the code.**

If the correct code still fails, check server availability, connection mode, address, and port.

### Change a configured client's settings

For an already configured package:

```sh
sudo dpkg-reconfigure autobricks-truelog-cli
```

This reopens the mode, hostname, server address, port, and mTLS code questions. Enter a code to enroll again, or leave it blank to reuse certificates for the same server. Completing configuration restarts the client. Use `dpkg --configure` for an installation that is still incomplete.

## Write records through the client

Run this section **on the application machine**, after client installation. The sudo-invoking installation user receives immediate socket access through an ACL without a new login.

```sh
ab-truelog-cli write --service example-service \
  --data '{"event_id":"example-event-001","action":"policy-update"}'
```

Do not add sudo or `--hostname` to application writes. The client daemon inserts its configured hostname. `--service` identifies the program's log chain; it is not a path or an authorization setting. Keep the service name fixed by deployment configuration.

The response contains:

```text
hostname, service
before.file, before.filesize, before.checksum
after.file, after.filesize, after.checksum
```

The before and after fields are consistent checkpoints for that write. Store the complete receipt with the application event ID. The server returns success after its configured storage durability boundary; merely connecting to the local socket is not proof of storage.

A timeout, disconnect, or error exit may leave the result uncertain: the server could have committed the record before the response was lost. The client has no durable retry queue or automatic deduplication and does not replay uncertain writes. Preserve the event ID and reconcile before retrying. A business database transaction and a True Log write are not one atomic transaction.

New daily files are created on the server:

```text
/mnt/worm-storage/example-service/truelog-YYYY-MM-DD.log
```

The first received message creates the file. Trusted server receive time determines the daily filename. Applications do not write directly to SOURCE or the WORM mount.

## Express and Node.js integration

Install the True Log client on the machine running Node.js. Express invokes `/usr/bin/ab-truelog-cli` with its own operating-system credentials.

### Authorize the Express runtime account

If Express runs as the user who installed the client through sudo, that user's socket ACL provides access. A different service account must be authorized during deployment.

For an **existing** account, replace `example-service` with the actual Node.js runtime account:

```sh
sudo usermod -aG autobricks-truelog-cli example-service
```

`-aG` preserves existing group memberships. Restart the process manager with updated credentials and then restart Express. Restarting only a worker under an unchanged parent may retain the old group list.

Only if you need a **new** dedicated account, create it after installing the client package:

```sh
sudo useradd --system --user-group --no-create-home \
  --shell /usr/sbin/nologin --groups autobricks-truelog-cli example-service
```

Configure the process manager to run Node.js as that account and grant access to the application files. Creating an account does not change the user of an existing process.

For systemd, you may instead keep the application's existing `User=` setting and add this to its unit or drop-in:

```ini
[Service]
SupplementaryGroups=autobricks-truelog-cli
```

```sh
sudo systemctl daemon-reload
sudo systemctl restart example-service.service
```

Root-only or unattended client installations cannot infer an application account. Configure its access explicitly. Applications in containers also need the socket directory and appropriate UID/GID access inside the container. Do not make the socket world-writable, run Express as root, or invoke sudo for each request.

### Invoke the client from Node.js

This example uses ES modules and an argument array, not a shell command assembled from HTTP input. Generate and retain the event ID before calling the function so it remains available if the write result is uncertain.

```js
import { execFile } from 'node:child_process';
import { promisify } from 'node:util';

const execFileAsync = promisify(execFile);
const service = 'example-service';

export async function writeTrueLog(eventId, fields) {
  const data = JSON.stringify({ ...fields, event_id: eventId });
  const { stdout } = await execFileAsync(
    '/usr/bin/ab-truelog-cli',
    ['write', '--service', service, '--data', data],
    { encoding: 'utf8', timeout: 30000, maxBuffer: 1024 * 1024 }
  );
  const receipt = JSON.parse(stdout);
  if (receipt.service !== service) throw new Error('Unexpected receipt service');
  return { event_id: eventId, receipt };
}
```

Call `await writeTrueLog(eventId, fields)` from your handler, handle errors, and store the returned receipt. Validate input size and fields. Do not log passwords or tokens; command-line arguments may be visible to local process inspection.

## Local syslog integration

This section runs **on the True Log server** and uses `ab-truelog ingest`. It does not require the remote client package. Applications keep using the standard syslog API, while rsyslog forwards selected programs.

Create `/etc/rsyslog.d/60-autobricks-log.conf` with the following configuration. Replace `example-service` with the selected program name. Do not load `omprog` twice if another configuration already loads it.

```text
module(load="omprog")

template(name="AutobricksLogFileFormat" type="list") {
    property(name="timegenerated" dateFormat="year")
    constant(value="-")
    property(name="timegenerated" dateFormat="month")
    constant(value="-")
    property(name="timegenerated" dateFormat="day")
    constant(value="T")
    property(name="timegenerated" dateFormat="hour")
    constant(value=":")
    property(name="timegenerated" dateFormat="minute")
    constant(value=":")
    property(name="timegenerated" dateFormat="second")
    constant(value=".")
    property(name="timegenerated" dateFormat="subseconds" position.from="1" position.to="3")
    constant(value=" ")
    property(name="hostname")
    constant(value=" ")
    property(name="syslogtag")
    property(name="msg" spIfNo1stSp="on" dropLastLf="on")
    constant(value="\n")
}

if $programname == "example-service" then {
    action(
        type="omprog"
        name="autobricks_log"
        binary="/usr/bin/ab-truelog ingest --service example-service"
        template="AutobricksLogFileFormat"
        confirmMessages="on"
        confirmTimeout="30000"
        reportFailures="on"
        killUnresponsive="on"
        forceSingleInstance="on"
        action.resumeRetryCount="-1"
        action.resumeInterval="5"
        queue.type="Disk"
        queue.filename="autobricks_log"
        queue.maxDiskSpace="1g"
        queue.saveOnShutdown="on"
        queue.syncQueueFiles="on"
    )
}
```

Validate and activate the configuration:

```sh
sudo rsyslogd -N1
sudo systemctl restart rsyslog.service
logger --tag example-service --priority local0.notice 'first syslog test'
```

The filter must match the submitted tag. Keep the disk queue and per-message confirmation settings. `openlog()` does not send a message, and successful `syslog()` submission is not a returned True Log write receipt. Use the client write interface when the application needs per-record receipts.

## Server-side chain inspection

Run these commands **on the True Log server as an administrator**. The client CLI does not provide status, history, or checksum commands.

```sh
sudo ab-truelog status --service example-service
sudo ab-truelog history --service example-service
sudo ab-truelog checksum --service example-service --date YYYY-MM-DD
```

Replace `YYYY-MM-DD` with the actual log date.

- `status` returns the current chain head: file, size, and checksum.
- `history` returns sealed daily-file records; use `status` for the active file.
- `checksum` verifies the retained file using recorded write boundaries and chain state.

A plain file SHA256 digest is not the True Log chain checksum. Package checksums verify downloaded packages only. History remaining after retention cleanup cannot reconstruct or revalidate deleted log bytes.

## Troubleshooting

On the client:

```sh
sudo journalctl -u ab-truelog-cli.service -n 100 --no-pager
```

On the server:

```sh
sudo journalctl -u ab-truelog-rpc.service -n 100 --no-pager
sudo journalctl -u ab-truelog.service -n 100 --no-pager
```

| Symptom | What to check |
| --- | --- |
| Installation stopped at pairing | Retry with `sudo dpkg --configure autobricks-truelog-cli`. |
| Connection failure | Server state, matching modes, reachable destination address and port, network access. |
| mTLS authentication failure | Certificate validity, trust chain, and server address matching the certificate's name/IP. |
| Socket permission denied | Actual Node.js account, client group or ACL, and running process credentials. |
| Root test succeeds but Express fails | Express account, process manager, group refresh, and service sandbox restrictions. |
| `unmanaged service` | Data directory exists without matching management state; do not silently adopt it or reset its chain. |
| Write timeout | Outcome may be uncertain; reconcile using the event ID and stored records. |

## Transition from Autobricks Log

Autobricks Log 0.2.31 and True Log 0.3.55 have different package names, commands, services, and record-management behavior.

| Item | Log 0.2.31 | True Log 0.3.55 |
| --- | --- | --- |
| Server package | `autobricks-log` | `autobricks-truelog` |
| Command | `ablog` | `ab-truelog` |
| Storage services | `ab-worm.service`, `autobricks-log.service` | `ab-truelog.service` |
| Remote application writes | Existing local syslog integration | Separate RPC service and `ab-truelog-cli` |
| New daily files | `ablog-YYYY-MM-DD.log` | `truelog-YYYY-MM-DD.log` |

Do not treat changing products as a routine same-package upgrade. The True Log package does not declare an automatic `Conflicts/Replaces` transition from the old Log package. Installing both server packages over the same storage and settings is not a verified migration procedure.

Before a transition:

1. Identify and preserve the installed version, service configuration, rsyslog queues, SOURCE, retention settings, and management state.
2. Plan the logging handoff. Do not run old and new services concurrently against the same mount and settings.
3. Update producer commands and rsyslog paths and verify the runtime account's access.
4. Check that stored data and management state are compatible. Copying files or pointing at an existing directory does not create a True Log chain.
5. Verify a test write, its receipt, chain state, and retained-file integrity before switching operational traffic.

For a compatible managed chain, legacy `ablog-*.log` files are never renamed or overwritten. The active legacy file continues for its current day; the next daily file uses the new prefix while preserving the chain. This is not automatic adoption of unmanaged old logs.

For an existing True Log installation, use the matching new package while preserving settings and state. A client may reuse certificates for the same server by leaving the Pairing Code blank. Automatic migration from older client configuration directories requires separate verification.

## Remove, purge, and reinstall

The following applies to the **True Log server package**, not the old Log package.

| Command | Effect |
| --- | --- |
| `sudo apt remove autobricks-truelog` | Stops and removes program services, preserving configuration, management state, and SOURCE. Matching data and state allow chains to resume after reinstallation. |
| `sudo apt purge autobricks-truelog` | Also removes server settings, TLS material, and active/pending/history management state. Preserves SOURCE, stored bytes, WORM metadata, and separately installed client settings. |

Do not purge when the intention is to reinstall and continue the same chain. After purge, a remaining service directory is unmanaged even if empty. Use an approved new service name for new logs; do not delete retained data or reset chains to bypass the check.

Purging an older Log package executes that package's own removal script. Do not assume it follows the True Log policy. Purging the client package removes its configuration, certificates, and saved socket-user setting.

## Platform use

Packages are provided for Ubuntu 22.04 and 24.04 on amd64 and arm64. Select the correct file for each machine. Actual installation and operation on Ubuntu 24.04 and ARM64, server/client coexistence, and the complete mTLS remote-write-to-storage path still require validation in the deployment environment.

Installed server reference documents are under `/usr/share/doc/autobricks-truelog/`. The client includes `/usr/share/doc/autobricks-truelog-cli/RPC.md`.
