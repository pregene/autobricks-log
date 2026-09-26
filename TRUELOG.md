# Autobricks True Log 0.3.55 설치 및 사용 안내

Autobricks True Log는 프로그램별 로그를 WORM 저장소에 보관하고, 로그 쓰기 전후의 파일 크기와 체크섬을 연결하여 True Log 기록을 처리합니다. 애플리케이션은 클라이언트를 통해 로그를 전송하고, 서버의 저장 내구성 경계를 완료한 뒤 반환되는 쓰기 영수증을 받을 수 있습니다.

이 문서는 **0.3.55의 서버와 클라이언트**를 설명합니다. 기존 `autobricks-log` 0.2.31의 설치 명령과 서비스 이름을 그대로 사용하면 안 됩니다. 이 저장소의 기존 README, INSTALL, HOWTO는 이전 Log 배포판을 설명할 수 있으므로, **True Log 설치에는 이 문서를 기준으로 사용하세요.**

- [True Log 0.3.55 다운로드](https://github.com/pregene/autobricks-log/releases/tag/v0.3.55)
- [기존 Autobricks Log 0.2.31](https://github.com/pregene/autobricks-log/releases/tag/v0.2.31)

## 1. 기존 Log와 무엇이 다른가

| 항목 | 기존 Autobricks Log 0.2.31 | Autobricks True Log 0.3.55 |
| --- | --- | --- |
| 서버 패키지 | `autobricks-log` | `autobricks-truelog` |
| 주요 명령 | `ablog` | `ab-truelog` |
| 저장 서비스 | `ab-worm.service`, `autobricks-log.service` | WORM과 로깅을 함께 관리하는 `ab-truelog.service` |
| 원격 전송 | 기존 릴리스의 로컬 rsyslog 연동 안내 | 별도 `ab-truelog-rpc.service`와 TCP/mTLS 클라이언트 |
| 애플리케이션 클라이언트 | 기존 릴리스에 별도 True Log 클라이언트 없음 | `autobricks-truelog-cli` 패키지, `ab-truelog-cli` 명령 |
| 새 일별 파일 | `ablog-YYYY-MM-DD.log` | `truelog-YYYY-MM-DD.log` |
| True Log 처리 | 기존 릴리스 사용법과 구분 필요 | 연속 체크섬 체인, 쓰기 전후 영수증, 현재 상태·봉인 이력·파일 검증 |

WORM은 저장 정책을 담당하고, True Log는 프로그램별 기록·회전·체크섬 체인·영수증을 관리합니다. RPC는 별도의 네트워크 입구입니다.

```text
Express / 애플리케이션
  -> ab-truelog-cli write
  -> 로컬 Unix 소켓
  -> ab-truelog-cli.service
  -> 지속 TCP 또는 mTLS 연결
  -> ab-truelog-rpc.service
  -> ab-truelog.service
  -> WORM 저장소
```

서버만 로컬에서 사용할 때는 RPC를 `none`으로 설정할 수 있습니다. 원격 클라이언트를 사용할 때는 서버와 클라이언트의 TCP/mTLS 모드가 일치해야 합니다.

## 2. 다운로드 파일 선택 및 확인

각 머신의 Ubuntu 버전과 CPU 아키텍처에 맞는 파일을 선택합니다.

```sh
. /etc/os-release
printf '%s\n' "$VERSION_ID"
dpkg --print-architecture
```

| Ubuntu | 아키텍처 | 서버 패키지 | 클라이언트 패키지 |
| --- | --- | --- | --- |
| 22.04 | amd64 | `autobricks-truelog-0.3.55-ubuntu-22.04-amd64.deb` | `autobricks-truelog-cli-0.3.55-ubuntu-22.04-amd64.deb` |
| 22.04 | arm64 | `autobricks-truelog-0.3.55-ubuntu-22.04-arm64.deb` | `autobricks-truelog-cli-0.3.55-ubuntu-22.04-arm64.deb` |
| 24.04 | amd64 | `autobricks-truelog-0.3.55-ubuntu-24.04-amd64.deb` | `autobricks-truelog-cli-0.3.55-ubuntu-24.04-amd64.deb` |
| 24.04 | arm64 | `autobricks-truelog-0.3.55-ubuntu-24.04-arm64.deb` | `autobricks-truelog-cli-0.3.55-ubuntu-24.04-arm64.deb` |

`amd64`는 x86-64, `arm64`는 AArch64입니다. 서버와 클라이언트는 서로 다른 머신에 설치할 수 있으며, 각 머신에 맞는 패키지를 선택합니다. GitHub의 자동 `Source code` 압축 파일은 이 배포 저장소의 내용이며, 설치용 `.deb`를 대신하지 않습니다.

패키지와 `SHA256SUMS`를 같은 디렉터리에 다운로드한 뒤 확인합니다. 아래 예시는 22.04 amd64이며, 다른 환경에서는 파일명 전체를 변경하세요.

```sh
grep -F '  autobricks-truelog-0.3.55-ubuntu-22.04-amd64.deb' SHA256SUMS | sha256sum -c -
grep -F '  autobricks-truelog-cli-0.3.55-ubuntu-22.04-amd64.deb' SHA256SUMS | sha256sum -c -
```

다운로드한 파일마다 `OK`가 출력되어야 합니다. 8개 패키지를 모두 다운로드했다면 `sha256sum -c SHA256SUMS`로 한 번에 확인할 수 있습니다.

## 3. 서버 설치

아래는 **새 서버 설치** 예시입니다. 기존 Log가 설치되어 있거나 보관 중인 데이터가 있으면 먼저 10절의 전환 주의사항을 읽으세요.

다운로드 디렉터리에서 의존성을 설치한 뒤 서버 패키지를 설치합니다.

```sh
sudo apt-get update
sudo apt-get install -y rsyslog fuse3 debconf systemd openssl python3
sudo dpkg -i ./autobricks-truelog-0.3.55-ubuntu-22.04-amd64.deb
```

`dpkg`는 의존성을 자동 다운로드하지 않습니다. 의존성 오류가 나오면 해당 의존성을 APT로 설치하고 같은 `dpkg -i` 명령을 다시 실행하세요.

### 설치 화면의 입력 항목

| 항목 | 설명 |
| --- | --- |
| SOURCE path | WORM 원본 데이터가 저장될 절대 경로. 예: `/var/lib/autobricks-log-source`. 마운트 경로와 달라야 합니다. |
| Retention days | 전체 저장소에 적용하는 보존 일수. 프로그램별로 별도 지정하지 않습니다. |
| RPC support | `none`, `tcp`, `mtls` 중 선택합니다. |
| Bind address | 서버가 수신할 로컬 주소. 클라이언트가 접속할 주소와 구분합니다. |
| Server address | 클라이언트가 실제로 접속할 서버 IP 또는 DNS 이름. mTLS 인증서의 서버 주소에도 사용합니다. |
| Port | RPC 포트. 기본값은 `5544`입니다. |
| 인증서 정보 | mTLS 선택 시 국가·지역·조직 등 설치 화면에서 요구하는 정보를 입력합니다. |

문서의 `192.0.2.10`, `app01.example.test`, `1234-5678` 등은 예시입니다. 실제 배포 설정으로 바꿔야 합니다. `0.0.0.0`은 수신 주소로 사용할 수 있지만 클라이언트의 목적지 주소로 사용하지 않습니다. 네트워크와 방화벽에서 선택한 TCP 포트가 클라이언트에 허용되어 있어야 합니다.

mTLS 설치 완료 화면의 **Pairing Code를 확인하여 클라이언트 설치에 사용**합니다. 코드는 등록 자격 정보이므로 공개 문서나 애플리케이션 로그에 남기지 마세요. 이 버전에서는 코드의 자동 만료·사용 횟수 제한이 없고, 여러 클라이언트가 같은 공유 클라이언트 인증서 자료를 받습니다. 코드와 인증서를 클라이언트별 독립 신원 관리 기능으로 간주하지 마세요.

### 서버 상태 확인

```sh
sudo ab-truelog check
systemctl status ab-truelog.service --no-pager
findmnt /mnt/worm-storage
```

TCP 또는 mTLS를 선택했다면 RPC도 확인합니다.

```sh
systemctl status ab-truelog-rpc.service --no-pager
sudo ss -ltnp
```

`ab-truelog.service`가 WORM과 로깅을 함께 관리합니다. 별도의 WORM 서비스를 추가로 실행하지 않습니다. RPC는 `ab-truelog-rpc.service`로 분리되어 있습니다.

## 4. 클라이언트 설치와 페어링

클라이언트는 Express 등 애플리케이션이 실행되는 머신에 설치합니다. 설치 사용자를 자동 등록하려면 해당 사용자로 로그인한 상태에서 `sudo`를 사용합니다.

```sh
sudo apt-get update
sudo apt-get install -y debconf systemd acl
sudo dpkg -i ./autobricks-truelog-cli-0.3.55-ubuntu-22.04-amd64.deb
```

| 입력 순서 | 입력값 |
| --- | --- |
| Connection mode | 서버와 동일한 `tcp` 또는 `mtls` |
| Client hostname | 로그에 기록할 클라이언트 이름. 기본값은 현재 머신의 hostname이며 서버 주소가 아닙니다. |
| Server address | 서버의 접속 가능한 IP 또는 DNS 이름 |
| RPC port | 서버에서 설정한 포트. 기본값 `5544` |
| Pairing Code | mTLS일 때만 서버에서 확인한 8자리 코드 입력 |

**Pairing Code는 입력할 때 화면에 보입니다.** `12345678`과 `1234-5678` 형식을 모두 허용하고 공백은 무시합니다. 예시 코드가 아닌 서버에서 확인한 코드를 입력하세요. 첫 mTLS 등록에서는 필수이며, 이미 인증서가 있는 클라이언트에서 같은 서버의 인증서를 재사용할 때만 빈칸으로 둘 수 있습니다. TCP에서는 코드 입력과 인증서 등록을 건너뜁니다. TCP는 TLS 암호화와 인증서 인증을 제공하지 않습니다.

설정은 `/etc/autobricks-truelog-cli/client.toml`, 인증서는 `/etc/autobricks-truelog-cli/clients/`에 저장됩니다. 서버 설정 디렉터리와 분리되어 있습니다. mTLS 등록에 성공하면 설치 과정에서 클라이언트 서비스를 활성화하고 시작합니다.

```sh
systemctl status ab-truelog-cli.service --no-pager
```

### Pairing Code 입력 실패로 설치가 중단된 경우

코드 누락·형식 오류·서버 거부 또는 등록 실패로 설치가 중단되면 다음 명령으로 미완료 설정을 다시 진행합니다.

```sh
sudo dpkg --configure autobricks-truelog-cli
```

Pairing Code 질문이 다시 나오면 올바른 코드를 입력합니다. 다른 저장된 답변은 유지됩니다. **코드 수정 때문에 purge하거나 재설치할 필요가 없습니다.** 코드가 맞아도 실패하면 서버 모드·주소·포트·서버 실행 상태를 확인하세요.

### 설치 완료 후 설정을 변경하는 경우

```sh
sudo dpkg-reconfigure autobricks-truelog-cli
```

모드, hostname, 서버 주소, 포트 및 Pairing Code를 다시 설정합니다. 새로운 코드로 다시 등록하거나, 같은 서버의 기존 인증서를 재사용한다면 코드를 비워둘 수 있습니다. 설정 완료 시 클라이언트 서비스가 재시작됩니다. 원래 설치가 미완료이면 앞의 `dpkg --configure`를 먼저 사용합니다.

## 5. 로그 쓰기와 쓰기 영수증

클라이언트 설치 시 sudo를 실행한 사용자에게는 소켓 ACL이 부여되어 재로그인 없이 다음 명령을 사용할 수 있습니다.

```sh
ab-truelog-cli write --service example-service \
  --data '{"event_id":"example-event-001","action":"policy-update"}'
```

이 명령에는 **sudo와 `--hostname`이 필요하지 않습니다.** hostname은 설치 설정에서 클라이언트 데몬이 삽입합니다. `--service`는 프로그램/서비스 이름이며, 파일 경로나 사용자 권한을 지정하는 옵션이 아닙니다. 애플리케이션에서 고정된 허용 서비스 이름을 사용하세요.

응답에는 다음 정보가 포함됩니다.

```text
hostname, service
before.file, before.filesize, before.checksum
after.file, after.filesize, after.checksum
```

`before`와 `after`는 해당 쓰기 전후의 일관된 체크포인트입니다. 애플리케이션은 이벤트 ID와 전체 영수증을 함께 저장하여 업무 기록과 로그 기록을 연결할 수 있습니다. RPC는 서버 영수증을 전달하며, 클라이언트가 별도로 체크섬을 계산하지 않습니다.

로컬 소켓 접속 성공만으로 저장 성공을 판단하지 마세요. 명령의 성공 응답은 서버의 저장 내구성 경계가 완료된 뒤 반환됩니다. 연결 끊김·타임아웃·오류 종료에서는 서버에 저장되었으나 응답만 유실됐을 수도 있습니다. 클라이언트에는 내구성 재시도 큐나 자동 중복 제거가 없으며 불확실한 쓰기를 자동 재전송하지 않습니다. 동일 이벤트 ID로 결과를 대조한 뒤 재시도를 결정하세요. 업무 DB 트랜잭션과 로그 쓰기는 하나의 원자적 트랜잭션이 아닙니다.

### 저장 위치

```text
/mnt/worm-storage/example-service/truelog-YYYY-MM-DD.log
```

첫 메시지를 수신할 때 파일이 만들어집니다. 경로와 일별 파일명은 서버의 신뢰할 수 있는 수신 시각을 기준으로 결정하며, 메시지에 포함된 시각이나 내용으로 파일 경로를 선택하지 않습니다. 애플리케이션은 SOURCE나 마운트 파일에 직접 쓰지 않습니다.

## 6. Express / Node.js 연동

### 실제 Express 실행 계정에 권한 부여

Express는 `/usr/bin/ab-truelog-cli`를 자식 프로세스로 실행하므로, **Node.js 프로세스의 OS 계정**에 소켓 접근 권한이 있어야 합니다. HTTP 로그인 사용자나 `--service` 값은 OS 권한을 부여하지 않습니다.

Express가 클라이언트 설치 시 sudo를 실행한 사용자로 실행된다면 설치 ACL을 사용할 수 있습니다. 다른 전용 계정으로 실행된다면 관리자가 배포 시 그 계정에 권한을 부여합니다. 기존 계정에는 `useradd` 대신 다음을 사용합니다.

```sh
sudo usermod -aG autobricks-truelog-cli example-service
```

`example-service`를 실제 Node.js 실행 계정으로 바꾸세요. `-aG`는 기존 그룹을 유지합니다. 변경된 그룹이 적용되도록 프로세스 관리자를 갱신하여 Express를 다시 시작해야 합니다. 기존 권한을 가진 부모 프로세스가 워커만 재시작하면 예전 그룹이 유지될 수 있습니다.

새 전용 계정이 필요한 경우에만, 클라이언트 패키지를 설치한 뒤 계정을 생성합니다.

```sh
sudo useradd --system --user-group --no-create-home \
  --shell /usr/sbin/nologin --groups autobricks-truelog-cli example-service
```

계정 생성만으로 기존 Express의 실행 사용자가 바뀌지는 않습니다. 프로세스 관리자에서 해당 계정으로 실행하도록 설정하고 애플리케이션 파일 접근 권한을 부여하세요.

systemd를 사용한다면 계정의 그룹 목록을 변경하는 대신 애플리케이션 유닛의 기존 `User=`를 유지하면서 다음을 추가할 수도 있습니다.

```ini
[Service]
SupplementaryGroups=autobricks-truelog-cli
```

```sh
sudo systemctl daemon-reload
sudo systemctl restart example-service.service
```

root로 직접 설치하거나 무인 설치하여 sudo 호출 사용자가 없는 경우에는 애플리케이션 계정을 자동 추정하지 않으므로 위 권한 설정이 필요합니다. 컨테이너에서 실행하는 애플리케이션은 소켓 디렉터리 공유와 컨테이너 내부 UID/GID 접근 권한도 맞춰야 합니다. 소켓을 world-writable로 바꾸거나, Express를 root로 실행하거나, 요청마다 sudo를 실행하지 마세요.

### Node.js 호출 예시

아래는 ES module 예시입니다. `execFile()`의 인자 배열을 사용하며 HTTP 입력으로 셸 명령을 조합하지 않습니다.

```js
import { execFile } from 'node:child_process';
import { promisify } from 'node:util';
import { randomUUID } from 'node:crypto';

const execFileAsync = promisify(execFile);
const service = 'example-service';

export async function writeTrueLog(fields) {
  const eventId = randomUUID();
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

Express 핸들러에서 `await writeTrueLog(...)`를 호출하고 오류를 처리하세요. 영수증을 확인하기 전에 기록 성공으로 응답하지 않습니다. 불확실한 결과의 복구가 필요하면 호출 전에 이벤트 ID를 생성·보존하도록 애플리케이션에 맞게 구성하세요. 입력 크기와 필드를 검증하고 비밀번호·토큰 등 비밀은 기록하지 마세요. 명령 인자는 로컬 프로세스 조회에서 보일 수 있습니다.

## 7. 기존 syslog 애플리케이션 연동

일반 syslog API도 계속 사용할 수 있습니다. 서버의 rsyslog가 선택한 프로그램의 메시지를 `ab-truelog ingest --service example-service`로 전달하도록 설정합니다. `openlog()` 자체는 메시지를 전송하지 않습니다. syslog 호출 성공은 True Log 쓰기 영수증 반환과 동일하지 않습니다.

서버에서 `/etc/rsyslog.d/60-autobricks-log.conf`에 다음 설정을 작성합니다. `example-service`는 실제 허용할 프로그램 이름으로 바꾸세요. 이미 `omprog` 모듈을 로드했다면 중복으로 로드하지 않습니다. 이전 0.2.31 릴리스의 `/usr/bin/ablog ingest example-service` 명령을 그대로 사용하지 마세요.

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

선택된 프로그램만 True Log에 전달됩니다. 다른 프로그램의 로그는 기존 rsyslog 규칙을 따릅니다.

설정 후 검증하고 테스트 메시지를 전송합니다.

```sh
sudo rsyslogd -N1
sudo systemctl restart rsyslog.service
logger --tag example-service --priority local0.notice 'first syslog test'
```

프로그램별 필터가 구성되어 있어야 해당 메시지가 저장됩니다. per-message confirmation과 디스크 큐를 사용하는 설정을 유지하세요. 메시지별 영수증을 업무 DB에 저장해야 한다면 5~6절의 클라이언트 호출을 사용합니다.

## 8. 서버에서 체인 상태 및 파일 검증

아래는 서버 관리자가 서버 머신에서 실행하는 명령입니다. 클라이언트 CLI에는 `status`, `history`, `checksum` 조회 기능이 없습니다.

```sh
sudo ab-truelog status --service example-service
sudo ab-truelog history --service example-service
sudo ab-truelog checksum --service example-service --date YYYY-MM-DD
```

마지막 명령의 `YYYY-MM-DD`는 확인할 실제 로그 날짜로 바꾸세요.

- `status`: 현재 체인의 파일명·크기·체크섬을 조회합니다.
- `history`: 회전으로 봉인된 일별 파일의 기록을 조회합니다. 활성 파일은 `status`로 확인합니다.
- `checksum`: 보관 중인 파일의 크기와 기록된 쓰기 경계에 따른 체인 체크섬을 검증합니다.

일반 파일의 `sha256sum` 결과는 True Log 체인 체크섬과 같은 값이 아닙니다. 2절의 SHA256 검증은 배포 파일 무결성 확인용입니다. 보존 기한이 지나 삭제된 로그의 이력이 남아 있어도 삭제된 바이트를 다시 검증하거나 복원할 수는 없습니다.

## 9. 문제 해결

```sh
sudo journalctl -u ab-truelog-cli.service -n 100 --no-pager
sudo journalctl -u ab-truelog-rpc.service -n 100 --no-pager
sudo journalctl -u ab-truelog.service -n 100 --no-pager
```

클라이언트 로그는 클라이언트 머신에서, RPC·저장 로그는 서버에서 확인합니다.

| 증상 | 확인할 내용 |
| --- | --- |
| Pairing Code 오류로 설치 중단 | `sudo dpkg --configure autobricks-truelog-cli`로 코드를 다시 입력 |
| 접속 실패 | 서버 실행 상태, 서버·클라이언트 모드 일치, 목적지 주소·포트, 네트워크 허용 여부 |
| mTLS 인증 오류 | 서버 주소와 인증서의 이름/IP 일치, 인증서 유효성, 서버의 신뢰 체인 |
| Unix 소켓 Permission denied | 실제 Node.js 실행 계정, 클라이언트 그룹 또는 ACL, 실행 프로세스의 갱신된 그룹 |
| root 테스트는 되지만 Express는 실패 | Express의 실행 계정·프로세스 관리자·systemd 접근 제한 확인 |
| `unmanaged service` | 데이터 디렉터리는 있지만 True Log 관리 상태가 없는지 확인. 자동 채택이나 체인 초기화로 우회하지 않음 |
| 쓰기 타임아웃 | 서버 저장 여부가 불확실할 수 있으므로 이벤트 ID·기록·영수증을 대조 |

## 10. 기존 Log 전환과 업그레이드

**0.2.31의 `autobricks-log`에서 0.3.55의 `autobricks-truelog`로의 전환을 일반적인 동일 패키지 업그레이드로 취급하지 마세요.** 패키지명·실행 파일·서비스·명령 형식이 달라졌으며, 배포 패키지에 구 패키지를 자동 대체하는 `Conflicts/Replaces` 선언은 없습니다. 기존 시스템 위에 두 서버 패키지를 무조건 겹쳐 설치하는 절차는 검증되지 않았습니다.

새 설치는 별도 머신 또는 기존 서비스와 충돌하지 않는 환경에서 먼저 검증하세요. 기존 시스템 전환 전에는 다음을 확인해야 합니다.

1. 기존 설치 버전, 서비스, rsyslog 설정, SOURCE, 보존 정책 및 관리 상태를 파악하고 운영 정책에 맞게 보전합니다.
2. 로그 생산자·큐를 포함한 전환 시간을 계획합니다. 기존 서비스와 새 서비스를 같은 마운트·설정·저장소에 동시에 실행하지 않습니다.
3. 이전 명령과 rsyslog 경로를 새 명령으로 전환하고 실제 서비스 계정의 권한을 확인합니다.
4. 관리 상태와 데이터가 일치하는지, 기존 체인을 계속할 수 있는지 확인합니다. 단순히 파일을 복사하거나 기존 디렉터리를 지정한다고 True Log 체인이 생기지는 않습니다.
5. 서버 점검과 시험 쓰기·영수증·상태·체크섬 검증을 마친 뒤 운영 트래픽을 전환합니다.

호환 관리 상태가 있는 기존 체인의 `ablog-*.log`는 이름을 바꾸거나 덮어쓰지 않습니다. 활성 legacy 파일을 해당 일자에 계속 사용하고, 다음 일별 회전부터 `truelog-*.log` 이름으로 체인을 이어갑니다. 이는 관리 상태 없는 임의의 구 로그를 자동 채택한다는 뜻이 아닙니다.

이미 True Log 패키지를 사용하는 경우에는 설정과 상태를 유지한 채 일치하는 새 `.deb`로 업그레이드합니다. 같은 서버의 기존 클라이언트 인증서를 재사용할 때는 Pairing Code를 비워둘 수 있습니다. 이전 클라이언트 설정 디렉터리를 사용하는 버전에서의 자동 설정·인증서 이전은 별도 확인이 필요합니다.

## 11. remove와 purge

| 작업 | 서버 동작 |
| --- | --- |
| `sudo apt remove autobricks-truelog` | 프로그램과 서비스를 제거하고 설정·관리 상태·SOURCE를 보존합니다. 데이터와 상태가 일치하면 재설치 후 체인을 이어갈 수 있습니다. |
| `sudo apt purge autobricks-truelog` | 서버 설정·TLS 자료와 True Log 활성/대기/이력 관리 상태를 제거합니다. SOURCE·저장된 로그 바이트·WORM 메타데이터 및 별도 클라이언트 설정은 보존합니다. |

**체인을 이어갈 재설치에는 purge를 사용하지 마세요.** purge 후 데이터 디렉터리만 남은 서비스는 비어 있어도 `unmanaged service`로 거부됩니다. 보관 데이터를 삭제하거나 체인을 초기화하여 해결하지 말고, 새 로그에는 승인된 새 서비스 이름을 사용하세요. 바이트가 남아 있다는 사실만으로 삭제된 체인 이력과 검증 상태가 복구되지는 않습니다.

이 설명은 True Log 0.3.55 서버의 제거 정책입니다. 구 `autobricks-log` 패키지를 purge하면 해당 구버전의 제거 스크립트가 실행되므로 같은 보존 동작을 가정하지 마세요. 클라이언트 패키지를 purge하면 클라이언트 설정·인증서·저장된 소켓 사용자 설정이 제거됩니다.

## 12. 이번 배포의 검증 범위

- 4개 Ubuntu/아키텍처 조합의 서버·클라이언트 패키지, 총 8개를 제공합니다. 버전은 모두 **0.3.55**입니다.
- 패키지 버전·ELF 아키텍처·SHA256 무결성을 확인했습니다. ARM64 패키지는 교차 컴파일했습니다.
- Ubuntu 22.04 amd64 CI의 포맷·workspace 테스트·lint·panic 검사·네이티브 빌드 및 버전 잠금·설치/서비스 회귀 검사가 통과했습니다. 설치 회귀 검사는 실제 호스트 설치와 구분됩니다.
- Ubuntu 22.04 amd64의 설치/TCP 기준 동작이 확인되었으며 Pairing Code 표시와 하이픈 없는 페어링 동작도 확인되었습니다.
- Ubuntu 24.04와 ARM64 실제 머신의 설치·실행, 서버/클라이언트 동시 설치, mTLS 원격 쓰기부터 저장까지의 전체 운영 검증은 완료로 간주하지 않습니다. 패키지 빌드 성공과 운영 검증은 구분됩니다.

추가 상세 자료는 설치된 서버의 `/usr/share/doc/autobricks-truelog/` 아래 `INSTALL.md`, `RPC.md`, `INTEGRATION.md`, `HOWTO.md`에서 확인할 수 있습니다. 클라이언트에는 `/usr/share/doc/autobricks-truelog-cli/RPC.md`가 포함됩니다.
