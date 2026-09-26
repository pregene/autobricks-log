# Autobricks Log · Autobricks True Log

Autobricks는 [WORM 저장소](https://github.com/pregene/autobricks-worm)를 기반으로 로그를 보관합니다. **Autobricks Log**는 기존 syslog 메시지를 프로그램별로 수집·보관하고, **Autobricks True Log**는 연속 체크섬 체인과 쓰기 영수증을 통해 애플리케이션의 기록 전후 상태를 확인할 수 있도록 합니다.

| 사용 목적 | 제품 | 안내 |
| --- | --- | --- |
| 기존 syslog 로그를 프로그램별 일별 파일로 보관하고 무결성 확인 | Autobricks Log | [설치](INSTALL.md) · [사용법](HOWTO.md) |
| 애플리케이션 기록을 체크섬 체인으로 관리하고 원격 쓰기 영수증 수신 | Autobricks True Log | [설치 및 사용 통합 안내](TRUELOG.md) |

## Autobricks Log

Autobricks Log는 애플리케이션이 운영체제의 표준 syslog API로 보낸 메시지를 rsyslog를 통해 수집합니다. rsyslog에서 지정한 프로그램의 메시지만 저장하며, 프로그램마다 일별 로그 파일을 관리합니다.

- 기존 애플리케이션의 표준 syslog 인터페이스를 사용할 수 있습니다.
- 프로그램 이름, facility, severity 조건으로 저장할 메시지를 선택할 수 있습니다.
- 저장된 바이트는 WORM 인터페이스를 통해 덮어쓰거나 보존 기한 전에 삭제할 수 없습니다.
- 파일 무결성을 확인하고 보존 기한이 지난 파일을 정리합니다.
- rsyslog의 디스크 큐와 메시지별 확인 응답을 통해 저장 실패 시 재시도를 지원합니다.

```text
애플리케이션 → syslog → rsyslog 필터·큐 → Autobricks Log → WORM 저장소
```

### 패키지와 사용 예시

기존 Log 배포판의 패키지명은 `autobricks-log`, 실행 명령은 `ablog`입니다.

rsyslog에서 `example-service` 프로그램을 전달하도록 설정한 뒤 메시지를 전송합니다.

```sh
logger --tag example-service --priority local0.notice 'policy decision completed'
```

저장 경로는 다음과 같습니다.

```text
/mnt/worm-storage/example-service/ablog-YYYY-MM-DD.log
```

서버에서 설치 상태와 파일 무결성을 확인합니다.

```sh
sudo ablog check
sudo ablog verify example-service
```

전체 rsyslog 설정과 운영 방법은 [HOWTO.md](HOWTO.md), 패키지 설치는 [INSTALL.md](INSTALL.md)를 참고하세요. 애플리케이션의 `syslog()` 호출 성공 자체가 최종 저장 확인을 뜻하지는 않습니다.

## Autobricks True Log

Autobricks True Log는 로그를 연속된 체크섬 체인으로 관리합니다. 애플리케이션은 클라이언트로 기록을 보내고, 서버가 저장 내구성 경계를 완료한 뒤 반환하는 **쓰기 전후의 파일명·크기·체크섬**을 받을 수 있습니다.

- 프로그램별 체크섬 체인을 일별 파일 회전 이후에도 이어갑니다.
- 쓰기 전후의 체크포인트를 영수증으로 반환합니다.
- 업무 이벤트 ID와 영수증을 함께 저장하여 애플리케이션 기록과 로그를 연결할 수 있습니다.
- TCP 또는 mTLS 연결로 원격 서버에 로그를 전달합니다.
- 클라이언트 데몬이 연결과 인증서를 관리하므로 애플리케이션은 로컬 명령으로 기록합니다.
- 서버에서 현재 체인 상태, 봉인된 파일 이력, 보관 중인 파일의 체인 무결성을 조회합니다.
- 기존 syslog·rsyslog 연동도 사용할 수 있습니다.

```text
애플리케이션 / Express
  → ab-truelog-cli write
  → 클라이언트 데몬
  → TCP 또는 mTLS
  → RPC 서비스
  → True Log · WORM 저장소
```

### 서버와 클라이언트

| 구성 | 패키지 | 역할 |
| --- | --- | --- |
| 서버 | `autobricks-truelog` | WORM 저장소, 로그 기록, 체크섬 체인 및 영수증 관리 |
| 클라이언트 | `autobricks-truelog-cli` | 로컬 애플리케이션 요청을 서버로 전달하고 영수증 반환 |

서버의 `ab-truelog.service`가 WORM과 로깅을 함께 관리합니다. 원격 접속은 별도 `ab-truelog-rpc.service`가 담당하며, 클라이언트는 애플리케이션 머신의 `ab-truelog-cli.service`로 실행됩니다.

TCP/mTLS 모드와 서버 주소·포트는 설치 시 설정합니다. mTLS에서는 서버의 Pairing Code로 클라이언트를 등록합니다. 코드는 입력 중 화면에 표시되고, 하이픈 없이 숫자 8자리로 입력할 수도 있습니다.

### 로그 쓰기

클라이언트 설치와 실행 계정 권한 설정을 마친 뒤 다음과 같이 사용합니다.

```sh
ab-truelog-cli write --service example-service \
  --data '{"event_id":"example-event-001","action":"policy-update"}'
```

쓰기 명령에 sudo나 `--hostname`을 넣지 않습니다. 클라이언트 데몬이 설치 시 설정한 hostname을 사용합니다. Express 등 다른 계정으로 실행되는 애플리케이션에는 해당 **실행 계정**의 클라이언트 소켓 권한이 필요합니다.

영수증에는 다음 정보가 포함됩니다.

```text
hostname, service
before.file, before.filesize, before.checksum
after.file, after.filesize, after.checksum
```

애플리케이션은 이벤트 ID와 영수증 전체를 보관할 수 있습니다. 타임아웃이나 연결 끊김에서는 서버에 저장된 뒤 응답만 유실됐을 수도 있으므로, 무조건 재전송하지 말고 기록을 대조해야 합니다. 클라이언트는 불확실한 쓰기를 자동 재전송하지 않으며, 이벤트 ID 자체가 자동 중복 제거를 제공하지는 않습니다.

새 일별 파일의 경로는 다음과 같습니다.

```text
/mnt/worm-storage/example-service/truelog-YYYY-MM-DD.log
```

서버 관리자는 다음 명령으로 상태와 이력을 확인할 수 있습니다.

```sh
sudo ab-truelog status --service example-service
sudo ab-truelog history --service example-service
sudo ab-truelog checksum --service example-service --date YYYY-MM-DD
```

`YYYY-MM-DD`는 확인할 실제 날짜로 바꿉니다. 일반 파일의 SHA256 값과 True Log 체인 체크섬은 다르므로, 체인 검증에는 `ab-truelog checksum`을 사용합니다.

**[TRUELOG.md](TRUELOG.md)**에서 서버·클라이언트 설치, Pairing Code 재설정, Express 계정 설정과 Node.js 예시, syslog 연동, 장애 확인 방법을 한 문서로 볼 수 있습니다.

## 저장과 보존

두 제품 모두 설치 시 하나의 SOURCE 경로와 보존 기간을 설정합니다. 프로그램은 저장 경로나 보존 기간을 개별적으로 선택할 수 없습니다. 첫 메시지가 수신될 때 해당 프로그램의 일별 파일이 만들어지며, 메시지 내용이나 생산자가 지정한 시각으로 저장 경로를 결정하지 않습니다.

WORM은 마운트 인터페이스를 통한 덮어쓰기와 조기 삭제를 제한합니다. 원본 저장소·커널·실행 파일을 직접 변경할 수 있는 관리자까지 막는 기능은 아닙니다. 보존 기한이 지나 삭제된 로그는 이력 정보만으로 복원할 수 없습니다.

## 기존 Log에서 True Log로 전환

| 항목 | Autobricks Log | Autobricks True Log |
| --- | --- | --- |
| 서버 패키지 | `autobricks-log` | `autobricks-truelog` |
| 서버 명령 | `ablog` | `ab-truelog` |
| 저장 서비스 | `ab-worm.service`, `autobricks-log.service` | `ab-truelog.service` |
| 원격 접속 | 기존 Log의 로컬 syslog 연동 | 별도 RPC 서비스와 TCP/mTLS 클라이언트 |
| 새 일별 파일 | `ablog-YYYY-MM-DD.log` | `truelog-YYYY-MM-DD.log` |

패키지명·서비스·명령 형식이 다르므로 기존 Log 위에 True Log를 설치하는 것을 단순한 동일 패키지 업그레이드로 취급하지 마세요. 기존 데이터와 관리 상태를 보전하고, [전환 안내](TRUELOG.md#10-기존-log-전환과-업그레이드)를 먼저 확인하세요.

호환되는 관리 상태가 있는 기존 체인의 `ablog-*.log`는 이름을 바꾸거나 덮어쓰지 않습니다. 해당 일자의 활성 파일을 이어 쓰고 다음 일별 회전부터 새 파일명을 사용합니다. 관리 상태가 없는 임의의 로그 디렉터리를 자동 채택하는 기능은 아닙니다.

True Log 서버에서 `remove`는 설정·관리 상태·SOURCE를 보존하지만, `purge`는 서버 설정·TLS 자료·체인 관리 상태를 제거합니다. 보관된 바이트가 남아 있어도 삭제된 체인 상태가 복구되지는 않습니다. 자세한 차이는 [제거 안내](TRUELOG.md#11-remove와-purge)를 참고하세요.

## 다운로드와 설명서

[Releases](https://github.com/pregene/autobricks-log/releases)에서 원하는 제품과 Ubuntu 버전·CPU 아키텍처에 맞는 패키지를 선택하세요.

- **Autobricks Log:** [설치 안내](INSTALL.md) · [사용 및 rsyslog 설정](HOWTO.md)
- **Autobricks True Log:** [설치·사용 통합 안내](TRUELOG.md)
- **WORM 저장소:** [기능 및 저장 정책](docs/WORM_README.md)

## 라이선스

[LICENSE](LICENSE)와 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)를 참고하세요.
