# NVC Network GUI

GUI Version 0.2 — 05/29/2026  
뉴비전 교회 네트워크 관리 도구 — Python + Tkinter 기반 로컬 GUI

---

## 실행

Python 3.10 이상 필요. 설치 시 `Add python.exe to PATH` 옵션을 켜야 합니다.

```powershell
cd C:\Projects\nvc_network
pip install -r gui\requirements.txt
python gui\main.py
```

> VPN을 먼저 연결한 후 실행하세요.  
> `python --version`이 Microsoft Store 메시지를 보이면 Windows 설정 →  
> App execution aliases에서 `python.exe`, `python3.exe` 별칭을 끄세요.

---

## 사전 요구사항

- VPN 연결 (GUI 실행 전 필수 — 192.168.11.1:22 접속 확인)
- Google Authenticator 또는 Authy (TOTP 2FA)
- `gui/config/ssh_targets.json` (예제: `ssh_targets.example.json` 참고)
- `gui/config/users.json` (초기 생성: `python gui\init_users.py`)

---

## 구성

```
gui/
├── main.py                  ← Tkinter 시작점 (NetworkGuiApp)
├── assets/
│   └── nvc_logo.png         ← 교회 로고 (상단 바 표시)
├── modules/
│   ├── auth.py              ← bcrypt 로그인 + TOTP 2FA + 개발자 모드
│   ├── vpn_check.py         ← VPN TCP 소켓 확인
│   ├── nas_sync.py          ← NAS JSONL → PC 캐시 동기화
│   ├── ap_count.py          ← AP Stuck 이벤트 집계
│   ├── ap_detail.py         ← 문제 AP 상세 분석
│   ├── ap_reset.py          ← AP 원격 reboot (EFG relay 경유)
│   ├── ap_remote.py         ← AP SSH 스캔 (mca-dump + logread 진단)
│   ├── efg_remote.py        ← EFG 시스템 대시보드 조회
│   ├── unifi_client.py      ← UniFi Local API 클라이언트
│   ├── log_export.py        ← 조건별 로그 export
│   ├── ap_inventory.py      ← AP 인벤토리 관리
│   ├── local_storage.py     ← 런타임 폴더 관리
│   ├── report_writer.py     ← operations.log 기록
│   ├── keychain.py          ← Windows Credential Manager 래퍼
│   ├── ssh_client.py        ← paramiko 공통 (패스워드/키, PTY, relay)
│   ├── font_utils.py        ← UI 폰트 로드/적용
│   └── models.py            ← 핵심 데이터 모델
├── config/
│   ├── app_config.json      ← 앱 경로, NAS 설정, stuck_thresholds, unifi_api (git 포함)
│   ├── ap_inventory.json    ← AP 29대 목록 (git 포함)
│   ├── ssh_targets.example.json  ← SSH 설정 예제 (git 포함)
│   ├── ssh_targets.json     ← 실제 SSH 접속 정보 (git 제외)
│   └── users.json           ← 계정·bcrypt 해시·TOTP secret (git 제외)
├── DESIGN.md                ← 설계 구상서
├── MANUAL.md                ← 사용자 매뉴얼
└── requirements.txt
```

---

## 탭 구성

| 탭 | 기능 요약 | 필요 권한 |
|---|---|---|
| **AP Status** | AP SSH 스캔(mca-dump/logread) + ELK Stuck 집계 + AP Reset | 모든 사용자 |
| **EFG Remote > EFG SSH** | EFG 시스템·네트워크·트래픽·ARP 대시보드 | `can_view_efg_tab` |
| **EFG Remote > EFG API** | UniFi Controller API 탐색기 (read-only GET) | `can_view_efg_tab` |
| **Log Export** | 날짜·AP·유형·키워드 조건 로그 파일 저장 | `can_export` |
| **동기화** | NAS JSONL → PC 캐시 동기화 (paramiko SFTP) | `can_sync_nas` |
| **설정** | UniFi API 설정/Probe, AP 등록수, 정렬, mca 덤프 관리 | 모든 사용자 |
| **사용자 관리** | 계정 추가·편집·삭제, TOTP 초기화 (admin 전용) | `can_manage_users` |

---

## AP Status 진단 컬럼

AP와 SSH 연결(EFG relay 경유)하여 `mca-dump` + `logread` 실행:

| 컬럼 | 소스 | 의미 |
|------|------|------|
| ELK Stuck | ELK Log 캐시 | ELK 기반 누적 Stuck 수 |
| 로그오류 | logread grep | 현재 부팅 중 오류 패턴 수 |
| 재시작2G/5G | `ast_ath_reset` | 드라이버 무선 리셋 횟수 |
| VAP지연2G/5G | `timeout_waiting_for_vap_cnt` | VAP 초기화 타임아웃 횟수 |
| CU2G%/CU5G% | `cu_total` | 채널 이용률 |
| Client2G/5G | `radio_table[*].num_sta` | 연결 클라이언트 수 |
| Ch 2G/5G/6G | UniFi API | 채널 번호 |
| BW 2G/5G/6G | UniFi API | 채널 폭 MHz |
| 응답(ms) | SSH 응답 시간 | mca-dump 왕복 시간 |

임계값 초과 셀에 `⚠` 표시. `⚠ 설정` 버튼으로 변경 가능 (`app_config.json` 저장).

---

## AP Reset — EFG Relay

AP Dropbear SSH에 paramiko 직접 패스워드 인증 불가 → EFG 경유 relay 구조:

```
GUI (paramiko) → EFG (OpenSSH) → sshpass → AP (Dropbear)
```

EFG에 `sshpass` 설치 필요.

---

## EFG API 탭 — Controller 탐색기

UniFi Controller API를 read-only GET으로 탐색. AP Stuck 분석 보조 용도.

**좌측 패널 (30%)**:
- API 연결 상태 / Site 정보 / 장비 현황
- WLAN 카드: SSID 드롭다운 → 상세 (밴드/VLAN/QoS/Guest 등)
- 장비 목록: AP / SW / EFG 라디오 버튼 필터

**우측 패널 (70%)**:
- Preset 드롭다운 (20개) + GET 버튼
- AP 선택 드롭다운 (device detail/stats 시 자동 표시)
- JSON 응답 뷰어 (캐시 자동 로드)
- 저장 경로: `runtime/api_probe/YYYYMMDD_HHMMSS_<preset>.json`

---

## 개발자 모드

등록된 PC에서 비밀번호·TOTP 없이 자동 로그인:
- Machine ID: Windows 레지스트리 MachineGuid 기반
- 설정 → 사용자 관리 → "이 컴퓨터 개발자 등록"
- 상단 타이틀 바에 `⚙ 개발자 모드` 배지 표시

---

## 런타임 데이터

`app_config.json`의 `workspace_root_windows` (기본: `C:\Projects\nvc_network\runtime\`):

```
runtime/
├── logs/raw/          ← NAS 동기화 JSONL 캐시
├── exports/           ← Log Export 결과
├── mca_dumps/         ← AP Remote mca-dump 원본
├── api_probe/         ← EFG API GET 응답 JSON
└── operations.log     ← 작업 기록
```

---

## SSH 설정

`ssh_targets.example.json` → `ssh_targets.json` 복사 후 실제 정보 입력:

```json
{
  "nas":  { "host": "...", "username": "...", "port": 22 },
  "efg":  { "host": "...", "username": "...", "port": 22 },
  "ap_defaults": { "username": "ubnt", "port": 22, "reboot_command": "reboot" }
}
```

비밀번호는 파일에 저장하지 않습니다. 첫 실행 시 입력하면 Windows Credential Manager에 저장됩니다.
