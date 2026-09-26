# Autobricks Log 설치 안내

**적용 제품: `autobricks-log` 0.2.31 / 명령: `ablog`**

이 문서는 Autobricks Log를 설치하고 계속 운영하는 사용자를 위한 안내입니다. True Log 서버·클라이언트를 설치하려면 [TRUELOG.md](TRUELOG.md)를 사용하세요. 제품별 패키지·서비스·명령이 다르므로 해당 제품의 절차를 따릅니다.

[제품 선택과 문서 목록](README.md#내-제품의-문서-찾기) · **Log 설치** → [Log 사용·검증](HOWTO.md)

## Install the package

Autobricks Log runs on Linux with systemd, rsyslog, FUSE 3, and the Autobricks
WORM filesystem.

[Log 0.2.31 릴리스](https://github.com/pregene/autobricks-log/releases/tag/v0.2.31)에서 머신의 Ubuntu 버전(22.04 또는 24.04)과 아키텍처(amd64 또는 arm64)에 맞는 `autobricks-log-0.2.31-ubuntu-버전-아키텍처.deb`를 다운로드하세요. 다음은 Ubuntu 22.04 amd64 예시이며, 다운로드한 디렉터리에서 실행합니다. 다른 환경에서는 패키지 파일명을 바꾸세요.

```sh
sudo apt-get update
sudo apt-get install -y rsyslog fuse3
sudo apt-get install ./autobricks-log-0.2.31-ubuntu-22.04-amd64.deb
```

## Select storage and retention

The installer asks for two system-wide values:

```text
SOURCE path: /worm-storage
Retention days: 365
```

- `SOURCE` is the root-only Autobricks WORM backing directory.
- The retention period applies to every program directory and log file.
- Programs cannot select another path or retention period.
- Existing files keep the deadline assigned when they were created.

The WORM filesystem exposes `SOURCE` at the fixed mount point:

```text
/mnt/worm-storage
```

The resulting layout is:

```text
<SOURCE>/                           root-only WORM backing data
/mnt/worm-storage/                  Autobricks Log WORM mount
└── <program>/
    └── ablog-YYYY-MM-DD.log
```

`SOURCE` and the mount point must be different directories. Applications receive
no direct write permission on either directory. Only the `autobricks-log`
identity and members of its group can write through the WORM mount. The
installer adds the system rsyslog user to that group.

WORM `.meta` sidecars are internal. They remain in the root-only `SOURCE`, but
ordinary log users cannot see them in mounted-directory listings or open them
by path. Only the WORM service identity and root can access them for cleanup and
integrity verification.

The installation values are stored in `/etc/default/autobricks-log`:

```sh
AB_WORM_RETAIN_DAYS=365
AB_WORM_SOURCE_PATH=/worm-storage
AB_WORM_MOUNT_PATH=/mnt/worm-storage
AUTOBRICKS_LOG_MOUNT=/mnt/worm-storage
AUTOBRICKS_LOG_CLEANUP_INTERVAL_SECONDS=3600
AUTOBRICKS_LOG_MAX_MESSAGE_BYTES=65536
```

`AUTOBRICKS_LOG_MOUNT` is package-managed and is not an installer choice.

## Installed components

```text
/usr/bin/ablog
/etc/default/autobricks-log
/usr/lib/systemd/system/ab-worm.service
/usr/lib/systemd/system/autobricks-log.service
```

An `sshd` rsyslog example is installed under
`/usr/share/doc/autobricks-log/examples/` and is not activated automatically.

## Start the services

```sh
sudo ablog check
sudo rsyslogd -N1
sudo systemctl enable --now ab-worm.service
sudo systemctl enable --now autobricks-log.service
sudo systemctl restart rsyslog.service
```

The startup order is:

```text
ab-worm.service
    -> autobricks-log.service
        -> rsyslog delivery action
```

If storage is unavailable, Autobricks Log returns a negative confirmation and
rsyslog retains the message in its action queue.

## Verify the installation

```sh
systemctl status ab-worm.service autobricks-log.service rsyslog.service
findmnt /mnt/worm-storage
sudo ablog check
```

Follow [HOWTO.md](HOWTO.md) to send a standard syslog message and verify the
program directory and WORM file.

## Upgrade and removal

이 절은 **Autobricks Log 패키지**에 적용됩니다. True Log의 제거·관리 상태 정책은 [별도 안내](TRUELOG.md#11-remove와-purge)를 확인하세요. 제품을 바꾸려는 경우에는 [Log에서 True Log로 전환](TRUELOG.md#10-기존-log-전환과-업그레이드)을 먼저 읽으세요.


An upgrade preserves the installation settings, WORM data and metadata,
retention deadlines, and rsyslog queues. Package removal does not delete retained
logs or queued messages. Data removal is a separate administrative operation and
remains subject to WORM retention enforcement.
