# NVC Network GUI

뉴비전 교회 네트워크 관리 도구 — Python + Tkinter 기반 로컬 GUI.

## 실행

Python 3.10 이상 필요. 설치 시 `Add python.exe to PATH` 옵션을 켜야 합니다.

```powershell
cd C:\Projects\nvc_network
pip install -r gui\requirements.txt
python gui\main.py
```

또는 실행 스크립트:

```powershell
.\gui\run-gui.ps1
```

> `python --version`이 Microsoft Store 메시지를 보이면 Windows 설정 →  
> App execution aliases에서 `python.exe`, `python3.exe` 별칭을 끄세요.

## 사전 요구사항

- VPN 연결 (GUI 실행 전 필수 — 192.168.11.1:22 접속 확인)
- Google Authenticator 또는 Authy (TOTP 2FA)
- `gui/config/ssh_targets.json` (예제: `ssh_targets.example.json` 참고)
- `gui/config/users.json` (초기 생성: `python gui\init_users.py`)

## 구성

```
gui/
├── main.py                  ← Tkinter 시작점 (NetworkGuiApp)
├── modules/
│   ├── auth.py              ← bcrypt 로그인 + TOTP 2FA + 개발자 모드 Machine ID
│   ├── vpn_check.py         ← VPN TCP 소켓 확인
│   ├── nas_sync.py          ← NAS JSONL → PC 캐시 동기화
│   ├── ap_count.py          ← AP Stuck 이벤트 집계
│   ├── ap_detail.py         ← 문제 AP 상세 분석
│   ├── ap_reset.py          ← AP 원격 reboot (EFG relay 경유)
│   ├── ap_remote.py         ← AP SSH 스캔 (mca-dump + logread 진단)
│   ├── efg_remote.py        ← EFG 시스템 대시보드 조회
│   ├── log_export.py        ← 조건별 로그 export
│   ├── ap_inventory.py      ← AP 인벤토리 관리
│   ├── local_storage.py     ← 런타임 폴더 관리
│   ├── report_writer.py     ← operations.log 기록
│   ├── keychain.py          ← Windows Credential Manager 래퍼
│   ├── ssh_client.py        ← paramiko 공통 (패스워드/키, PTY, relay)
│   └── models.py            ← 핵심 데이터 모델
├── config/
│   ├── app_config.json      ← 앱 경로, NAS 설정, stuck_thresholds, remote_default_sort (git 포함)
│   ├── ap_inventory.json    ← AP 29대 목록 — 전체 192.168.11.x (git 포함)
│   ├── ssh_targets.example.json  ← SSH 설정 예제 (git 포함)
│   ├── ssh_targets.json     ← 실제 SSH 접속 정보 (git 제외)
│   └── users.json           ← 계정·bcrypt 해시·TOTP secret·dev_machines (git 제외)
├── DESIGN.md                ← 설계 구상서 및 구현 이력
├── MANUAL.md                ← 사용자 매뉴얼
└── requirements.txt
```

## 런타임 데이터

`app_config.json`의 `workspace_root_windows`에 설정 (기본값: `C:\Projects\nvc_network\runtime\`):

```
runtime/
├── logs/ap/           ← NAS에서 동기화한 JSONL 캐시
├── exports/           ← Log export 결과
├── mca_dumps/         ← AP Remote 스캔 시 mca-dump 원본 저장
└── operations.log     ← 작업 기록 (사용자·시각·명령)
```

## 탭 구성

| 탭 | 기능 요약 |
|---|---|
| Log 동기화 | NAS JSONL → PC 캐시 (paramiko SFTP, 차분 동기화) |
| AP ELK Log | ELK 로그 기반 AP Stuck 집계 + 상세 분석 |
| AP Remote | AP SSH 직접 조회 — mca-dump, logread, 진단 컬럼, 다중 AP Reset |
| EFG Remote | EFG 시스템 대시보드 (시스템·네트워크·트래픽·ARP) |
| Log Export | 날짜·AP·유형·키워드 조건 로그 파일 저장 |
| 설정 | AP Remote 기본 정렬, mca 덤프 관리 |
| 사용자 관리 | 계정 추가·편집·삭제, TOTP 초기화 (admin 전용) |
| EFG 참조 | efg/EFG.md 읽기 전용 표시 |

## AP Remote 진단 컬럼

AP와 직접 SSH 연결(EFG relay 경유)하여 `mca-dump` + `logread` 실행:

| 컬럼 | 소스 | 의미 |
|------|------|------|
| ELK Stuck | ELK Log 캐시 | ELK 기반 누적 Stuck 수 |
| 로그오류 | logread grep | 현재 부팅 중 오류 패턴 수 |
| 재시작2G/5G | `ast_ath_reset` | 드라이버 무선 리셋 횟수 |
| VAP지연2G/5G | `timeout_waiting_for_vap_cnt` | VAP 초기화 타임아웃 횟수 |
| CU2G%/CU5G% | `cu_total` | 채널 이용률 |

임계값 초과 셀에 `⚠` 표시. `⚠ 설정` 버튼으로 변경 가능 (`app_config.json` 저장).

## AP Remote — 다중 Reset

Reset 대상 AP를 빨간색으로 미리 지정한 뒤 일괄 실행:

```
하단 버튼 배치:
[상세 보기]    [AP Reset 선택] [AP 선택 해제] | [모두 선택] [모두 해제] ·· [AP Reset 실행]
```

| 버튼 | 동작 |
|------|------|
| AP Reset 선택 | 선택한 AP(Shift 다중 가능)를 빨강으로 Reset 대상 등록 |
| AP 선택 해제 | 선택한 AP를 Reset 대상에서 해제 |
| 모두 선택 | 전체 AP를 Reset 대상으로 지정 |
| 모두 해제 | 전체 Reset 대상 해제 |
| AP Reset 실행 | 지정된 AP 목록 확인 후 일괄 reset, 결과 표시 |

**상세 보기**: Shift 다중 선택 후 클릭하면 최대 3개까지 별도 창으로 표시.

## AP Remote — 기본 정렬

설정 탭 → **AP Remote 기본 정렬** 섹션에서 선택 후 저장:

| 선택지 | 정렬 방향 |
|--------|----------|
| AP | 오름차순 (번호순) |
| Name | 오름차순 (A→Z) |
| Model | 오름차순 (A→Z) |
| ELK Stuck | 내림차순 (큰 값 상단) |
| 로그오류 | 내림차순 (큰 값 상단) |
| 재시작2G | 내림차순 (큰 값 상단) |

- GUI 시작 시 1회만 적용됩니다.
- 이후 수동으로 컬럼을 클릭하여 정렬을 변경하면 해당 정렬이 유지됩니다.
- 설정은 `app_config.json`의 `remote_default_sort` 키에 저장됩니다.

## AP Reset — EFG Relay

AP에 직접 paramiko 패스워드 인증이 불가 (Dropbear 호환성 문제):

```
GUI (paramiko) → EFG (OpenSSH) → sshpass → AP (Dropbear)
```

EFG에 `sshpass`가 설치되어 있어야 합니다.  
relay ssh에 `ConnectTimeout` 이 설정되어 응답 없는 AP에서 무한 대기하지 않습니다.

## 개발자 모드

등록된 PC에서 비밀번호·TOTP 없이 자동 로그인합니다.

- Machine ID는 **Windows 레지스트리 MachineGuid** 기반 (VPN 연결 여부와 무관, 안정적)
- `users.json`의 `dev_machines` 배열에 등록
- 사용자 관리 탭 → "이 컴퓨터 개발자 등록" 버튼으로 등록
- 상단에 `⚙ 개발자 모드` 배지로 표시

## SSH 설정

`ssh_targets.example.json`을 `ssh_targets.json`으로 복사 후 실제 정보 입력:

```json
{
  "nas":  { "host": "...", "username": "...", "port": 22 },
  "efg":  { "host": "...", "username": "...", "port": 22 },
  "ap_defaults": { "username": "ubnt", "port": 22, "reboot_command": "reboot" }
}
```

비밀번호는 파일에 저장하지 않습니다. 첫 실행 시 입력하면 Windows Credential Manager에 저장 여부를 묻습니다.
