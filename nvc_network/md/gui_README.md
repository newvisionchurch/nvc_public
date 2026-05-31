# NVC Network GUI

GUI Version 0.3 — 05/30/2026  
뉴비전 교회 네트워크 관리 도구 — Python + Tkinter 기반 로컬 GUI

---

## 목차

- [실행](#실행)
- [사전 요구사항](#사전-요구사항)
- [구성](#구성)
- [탭 구성](#탭-구성)
- [AP Status 탭 상세](#ap-status-탭-상세)
- [AP Reset 후보 분류 ResetScore](#ap-reset-후보-분류-resetscore)
- [AP Reset EFG Relay 구조](#ap-reset-efg-relay-구조)
- [GitHub 연동 기능](#github-연동-기능)
- [권한 구조](#권한-구조)
- [런타임 데이터](#런타임-데이터)
- [개발자 모드](#개발자-모드)
- [변경 이력](#변경-이력)

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
> 앱 실행 별칭에서 `python.exe`, `python3.exe` 별칭을 끄세요.

---

## 사전 요구사항

- VPN 연결 (GUI 실행 전 필수 — 192.168.11.x 대역 접근)
- Google Authenticator 또는 Authy (TOTP 2FA)
- `gui/config/ssh_targets.json` (예제: `ssh_targets.example.json` 참고)
- `gui/config/users.json` (초기 생성: `python gui\init_users.py`)

---

## 구성

```
gui/
├── main.py                    ← Tkinter 시작점 (NetworkGuiApp)
├── assets/
│   └── nvc_logo.png           ← 교회 로고 (상단 바 표시)
├── modules/
│   ├── auth.py                ← bcrypt 로그인 + TOTP 2FA + 개발자 모드
│   ├── vpn_check.py           ← VPN TCP 소켓 확인
│   ├── nas_sync.py            ← NAS JSONL → PC 캐시 동기화 (SFTP)
│   ├── ap_count.py            ← AP Stuck 이벤트 집계
│   ├── ap_detail.py           ← 문제 AP 상세 분석 및 Stuck 유형 분류
│   ├── ap_reset.py            ← AP 원격 reboot (EFG relay 경유)
│   ├── ap_remote.py           ← AP SSH 스캔 (mca-dump + logread + ResetScore)
│   ├── efg_remote.py          ← EFG 시스템 대시보드 조회
│   ├── unifi_client.py        ← UniFi Local API 클라이언트
│   ├── log_export.py          ← 조건별 로그 필터링 후 파일 저장
│   ├── ap_inventory.py        ← AP 인벤토리 로드 및 검색
│   ├── local_storage.py       ← 런타임 폴더 및 app_config.json 관리
│   ├── report_writer.py       ← operations.log 기록
│   ├── keychain.py            ← Windows Credential Manager (keyring 래퍼)
│   ├── ssh_client.py          ← paramiko 공통 (패스워드/키 인증, PTY, relay)
│   ├── font_utils.py          ← OS 폰트 감지 및 UI 전체 폰트 적용
│   ├── models.py              ← 핵심 데이터 모델
│   ├── github_auth.py         ← GitHub API (HTTPS + PAT) 인증
│   ├── github_schedule.py     ← AP Reset 예약 관리 (ap_reset_schedule.json)
│   ├── github_feedback.py     ← 팀원 게시판 (feedback.json)
│   └── credentials.py         ← NAS 자격증명 암호화 저장
├── config/
│   ├── app_config.json        ← 앱 경로, NAS 설정, stuck_thresholds, unifi_api
│   ├── ap_inventory.json      ← AP 29대 목록 (이름, IP, 모델, 위치)
│   ├── ssh_targets.example.json ← SSH 설정 예제
│   ├── users.json             ← 사용자 계정 (bcrypt 해시, TOTP secret) — git 제외
│   └── ssh_targets.json       ← NAS/EFG/AP SSH 접속 정보 — git 제외
├── MANUAL.md                  ← 사용자 매뉴얼 (배포용)
├── README.md                  ← 이 문서 (기술 개요)
└── DESIGN.md                  ← 설계 구상서 및 구현 이력
```

---

## 탭 구성

| 탭 | 서브탭 | 핵심 기능 | 권한 |
|---|---|---|---|
| **AP Status** | — | mca-dump/logread 진단 + ELK Stuck 집계 + ResetScore 분류 + AP Reset | 모든 사용자 |
| **EFG Remote** | EFG SSH | EFG 시스템 대시보드 (SSH) | `can_view_efg_tab` |
| **EFG Remote** | EFG API | UniFi Controller API 탐색기 (read-only GET) | `can_view_efg_tab` |
| **Log Export** | — | 날짜·AP·유형·키워드 조건 로그 파일 저장 | `can_export` |
| **동기화** | NAS ELK Sync | NAS JSONL → PC 로컬 캐시 동기화 (SFTP) | `can_sync_nas` |
| **동기화** | 연결 현황 | SSH/SFTP 연결 상태 확인 | — |
| **설정** | 일반 | 폰트, AP 등록 수, 기본 정렬, mca 덤프 | 모든 사용자 |
| **설정** | 보안 관리 | UniFi API 설정/Probe, GitHub PAT, 인증 방식 | 모든 사용자 (PAT는 Admin) |
| **설정** | 사용자 관리 | 계정·권한 관리 | `can_manage_users` |

---

## AP Status 탭 상세

### 컨트롤 바 (상단)

```
[등록 카드] [온라인 카드] [오프라인 카드]
│
[전체 조회] [선택 조회] [중지]
[동작 설명] [경고 설정] [컬럼 설정]
│
○전체 ●범위 [오늘] [목록] (N일)  From ~ To  [ELK Stuck Count]
│
분류규칙  분류설정  Reset후보  [AP Reset 후보 분류]
```

### 하단 버튼 바

```
[상세 보기]  ···  [AP Reset 선택] [AC Pro 선택] [AP 선택 해제] | [모두 해제] | [분류 선택] [예약 관리] | 선택N | [AP Reset 실행]
```

### 진단 컬럼

| 컬럼 | 소스 | 설명 |
|------|------|------|
| ELK Stuck | ELK JSONL 캐시 | 날짜 범위 내 Stuck 이벤트 수 |
| 로그오류 | `logread \| grep -cE` | 현재 부팅 세션 내 오류 패턴 수 |
| DevReset | `logread WAL_DBGID_DEV_RESET` | AC Pro 6.8.x 버그 감지 수 |
| 재시작2G/5G | `athstats.ast_ath_reset` | 드라이버 레벨 무선 리셋 누적 |
| VAP지연2G/5G | `athstats.timeout_waiting_for_vap_cnt` | VAP 초기화 타임아웃 누적 |
| CU2G%/CU5G% | `athstats.cu_total` | 채널 이용률 |
| Client2G/5G | `radio_table[*].num_sta` | 연결 클라이언트 수 |
| Ch 2G/5G/6G | UniFi API | 채널 번호 |
| BW 2G/5G/6G | UniFi API | 채널 폭 MHz |
| CPU% / Mem% | mca-dump | CPU / 메모리 사용률 |
| Uptime | mca-dump | 마지막 재부팅 이후 경과 시간 |
| 응답(ms) | SSH 왕복 시간 | mca-dump 전체 응답 시간 |

임계값 초과 시 셀 앞에 `⚠` 표시. [경고 설정] 버튼으로 임계값 변경 → `app_config.json`의 `stuck_thresholds`에 저장.

---

## AP Reset 후보 분류 ResetScore

SSH 조회 후 [AP Reset 후보 분류] 버튼으로 점수를 계산합니다.

### 점수 파라미터

| 항목 | 조건 | 점수 |
|------|------|------|
| 로그오류 | 1~2 | 30 |
| 로그오류 | 3~9 | 40 |
| 로그오류 | ≥10 | 50 |
| ELK Stuck | 500~1999 | 15 |
| ELK Stuck | 2000~4999 | 30 |
| ELK Stuck | ≥5000 | 50 |
| VAP 지연 | 1~20 | 15 |
| VAP 지연 | ≥21 | 30 |
| 채널이용률 | 70~84% | 10 |
| 채널이용률 | ≥85% | 15 |

### 분류 기준

| 색상 | 점수 조건 | 의미 |
|------|----------|------|
| 🔴 빨강 | ≥60 + Reset 근거 | Reset 권장 |
| 🟡 노랑 | ≥20 | 관찰 필요 |
| 🟢 초록 | <20 | 정상 |
| ⬜ 회색 | 미조회 | 데이터 없음 |

---

## AP Reset EFG Relay 구조

AP는 Dropbear SSH를 사용하므로 paramiko 직접 연결이 불가합니다.  
EFG(OpenSSH)를 경유해 sshpass로 접속합니다.

```
GUI (paramiko) → EFG (OpenSSH) → sshpass → AP (Dropbear)
```

- EFG에 `sshpass` 설치 필요
- `ap_reset.py` → `run_ssh_via_relay()` 사용

### SSH 접속 방식 비교

| 대상 | 방식 | PTY |
|------|------|-----|
| NAS (Synology) | paramiko SSHClient, 키 인증 | 불필요 |
| EFG (OpenSSH) | paramiko Transport, 패스워드 인증 | 불필요 |
| AP (Dropbear) 직접 | paramiko Transport, 패스워드 인증 | **필요** |
| AP via EFG relay | EFG에서 sshpass+ssh 실행 | 불필요 |

---

## GitHub 연동 기능

GitHub `newvisionchurch/nvc-auth` 저장소를 팀 공유 데이터 저장소로 사용합니다.  
PAT(Personal Access Token)를 Windows Keychain에 저장하고 HTTPS API로 통신합니다.

### ap_reset_schedule.json

AP Reset 예약을 팀 전체가 공유합니다.

```json
{
  "schedules": [
    {
      "id": "20260531-020000-chris",
      "created_by": "chris",
      "created_at": "2026-05-30T16:13:00",
      "target_time": "2026-05-31T02:00:00",
      "ap_ids": ["AP13", "AP14"],
      "interval_min": 4,
      "order": "yellow_first",
      "status": "pending"
    }
  ],
  "history": [ ... ]
}
```

### feedback.json

팀원 게시판 데이터를 저장합니다.

```json
{
  "items": [
    {
      "id": "20260530-161300-chris",
      "type": "bug",
      "title": "제목",
      "body": "내용",
      "author": "chris",
      "created_at": "2026-05-30T16:13:00",
      "version": "0.3",
      "status": "open",
      "comments": []
    }
  ]
}
```

---

## 권한 구조

### 권한 플래그

| 플래그 | 설명 |
|--------|------|
| `can_reset_ap` | AP 원격 재부팅 |
| `can_sync_nas` | NAS 로그 동기화 |
| `can_export` | Log Export 실행 |
| `can_view_efg_tab` | EFG Remote 탭 접근 |
| `can_manage_users` | 사용자 관리 탭 (Admin 전용) |

### 역할별 기본 권한

| 권한 | admin | operator | viewer |
|------|:---:|:---:|:---:|
| can_reset_ap | ✓ | 설정에 따름 | ✗ |
| can_sync_nas | ✓ | ✓ | ✗ |
| can_export | ✓ | ✓ | ✓ |
| can_view_efg_tab | ✓ | ✓ | ✓ |
| can_manage_users | ✓ | ✗ | ✗ |

---

## 런타임 데이터

`app_config.json`의 `runtime_root` 경로 아래에 저장됩니다 (기본: `C:\Projects\nvc_network\runtime\`).

```
runtime/
├── logs/raw/          ← NAS 동기화 JSONL 캐시 (날짜별 파티션)
├── exports/           ← Log Export 결과 파일
├── mca_dumps/         ← AP Remote mca-dump 원본
├── api_probe/         ← EFG API GET 응답 JSON (YYYYMMDD_HHMMSS_<preset>.json)
└── operations.log     ← 사용자명·작업 기록
```

---

## 개발자 모드

Windows `MachineGuid` 레지스트리 값으로 PC를 식별합니다.  
Admin이 설정 탭 → 사용자 관리 → [이 컴퓨터 개발자 등록]을 클릭하면 해당 PC에서 자동 로그인됩니다.

타이틀 바에 `⚙ 개발자 모드` 뱃지가 표시됩니다.

---

## 변경 이력

### v0.3 — 2026-05-30

- **AP Reset 예약 관리** 추가 — GitHub nvc-auth 저장소 연동 (예약 목록 / AP Reset 이력)
- **팀원 게시판** 추가 — 버그/개선/공지 피드백, 댓글, 공지 자동 팝업
- 하단 바 [예약 관리] 버튼 추가 (Action 스타일)
- 상단 타이틀 바 [💬 팀원 게시판] 버튼 추가
- `github_schedule.py` / `github_feedback.py` 모듈 신규 추가

### v0.2 — 2026-05-29

- NAS 인증 안정성 개선
- EFG SSH 대시보드 컬럼 정렬 기능 추가
- ResetScore 파라미터 설정 UI 추가
- AP Reset 포그라운드/백그라운드 모드 분리
