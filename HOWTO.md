# Autobricks Log 사용 안내

**적용 제품: `autobricks-log` 0.2.31 / 명령: `ablog`**

[Log 설치](INSTALL.md)를 완료한 뒤, 이 문서 순서대로 rsyslog 전달 설정 → 메시지 전송 → 저장 파일 검증을 진행하세요. True Log 사용자는 [True Log의 syslog 연동](TRUELOG.md#7-기존-syslog-애플리케이션-연동) 또는 [클라이언트 쓰기](TRUELOG.md#5-로그-쓰기와-쓰기-영수증)를 사용합니다.

[제품 선택과 문서 목록](README.md#내-제품의-문서-찾기) · [Log 설치](INSTALL.md) → **Log 사용·검증**

Applications use the standard system syslog API. rsyslog forwards selected
messages to Autobricks Log, which creates a directory from the program name and
stores the messages under the system-wide WORM mount.

## Configure rsyslog delivery

Create `/etc/rsyslog.d/60-autobricks-log.conf`:

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
        binary="/usr/bin/ablog ingest example-service"
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

Only messages whose normalized `programname` exactly matches
`example-service` enter Autobricks Log. Other system logs continue through the
remaining rsyslog rules. The selected program name is delivered with every
record and determines its directory.

Multiple programs can be allowlisted explicitly:

```text
if $programname == "example-service"
   or $programname == "example-worker" then {
    # the same Autobricks Log action
}
```

Facility and severity conditions can be combined with the program allowlist.
Do not add `stop` after the action when the selected message must also continue
to the host's normal log destinations.

Validate the configuration before restarting rsyslog:

```sh
sudo rsyslogd -N1
sudo systemctl restart rsyslog.service
```

## Write from an application

```c
#include <syslog.h>

int main(void) {
    openlog("example-service", LOG_PID, LOG_LOCAL0);
    syslog(LOG_NOTICE, "policy decision completed");
    closelog();
    return 0;
}
```

`openlog()` sets the program identity but does not itself send a message.
Autobricks Log therefore creates the file when the first `syslog()` message for
that program arrives.

The resulting file is:

```text
/mnt/worm-storage/example-service/ablog-YYYY-MM-DD.log
```

The date comes from the logger's local calendar day. Message timestamps do not
control paths or filenames. All messages for the same program and local day are
appended to the same file without overwriting committed bytes. Restarting the
ingest process on the same day reopens that daily file; the first message after
local midnight opens the next day's file.

## Send a command-line message

```sh
logger --tag example-service --priority local0.notice \
    "policy decision completed"
```

The program does not specify `SOURCE`, the mount point, filename, or retention.

The stored content remains syslog file text; it is not converted to JSON or
wrapped in an Autobricks-specific record. rsyslog supplies local receive time
with millisecond precision:

```text
2026-09-25T21:34:56.123 host example-service[42]: policy decision completed
```

## Verify the file

```sh
systemctl status ab-worm.service autobricks-log.service rsyslog.service
find /mnt/worm-storage/example-service \
    -maxdepth 1 -type f -name 'ablog-*.log' -print
sudo ablog verify example-service
```

Every file uses the retention period selected during installation. Autobricks
Log's janitor requests deletion after the WORM deadline expires. Program
directories remain as stable namespaces after their files expire.

## Program directory rules

Autobricks Log normalizes and validates the program name before using it as a
directory component:

- path separators, `.` and `..` are rejected;
- the name has a fixed maximum length;
- empty or invalid names are rejected;
- symlinks are never followed;
- file creation remains beneath the fixed WORM mount.

These rules prevent an application-controlled syslog field from escaping the
storage root.

## Delivery failures

Autobricks Log confirms a message only after its log data and WORM metadata
reach the configured durability boundary. Disk full, permission errors, failed
integrity validation, or an unavailable WORM mount produce a negative response.
rsyslog retains those messages in its disk queue and retries them.

A successful application `syslog()` return is not proof that WORM storage has
completed. Durable delivery is determined by the confirmation exchanged between
rsyslog and Autobricks Log.

문제가 발생하면 Log 서비스와 rsyslog의 최근 기록을 확인합니다.

```sh
sudo journalctl -u ab-worm.service -u autobricks-log.service -u rsyslog.service -n 100 --no-pager
sudo rsyslogd -N1
findmnt /mnt/worm-storage
```

메시지가 저장되지 않으면 rsyslog의 프로그램 이름 필터가 전송 태그와 일치하는지, 서비스와 WORM 마운트가 실행 중인지, 디스크 여유 공간이 있는지 확인하세요. 설정 수정 후에는 `rsyslogd -N1` 검증을 통과한 다음 rsyslog를 재시작합니다.
