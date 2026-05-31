# NVC Network GUI — 설계 구상서

GUI Version 0.3 — 05/30/2026

---

## 목차

- [1. 목적](#1-목적)
- [2. 기술 스택](#2-기술-스택)
- [3. 아키텍처 개요](#3-아키텍처-개요)
- [4. 탭 구성](#4-탭-구성)
- [5. AP Status 탭 상세](#5-ap-status-탭-상세)
- [6. ResetScore 분류 시스템](#6-resetscore-분류-시스템)
- [7. AP Reset EFG Relay 구조](#7-ap-reset-efg-relay-구조)
- [8. GitHub 연동 설계](#8-github-연동-설계)
- [9. 인증 설계](#9-인증-설계)
- [10. 모듈 구조](#10-모듈-구조)
- [11. 주요 데이터 모델](#11-주요-데이터-모델)
- [12. 설정 파일](#12-설정-파일)
- [13. 구현 이력](#13-구현-이력)

---

## 1. 목적

뉴비전 교회 네트워크(UniFi EFG / AP 29대)에서 반복 발생하는 AP Stuck 문제를 운영팀이 Kibana나 SSH 수동 작업 없이 GUI 하나로 빠르게 찾고, 분석하고, 원격 제어할 수 있는 로컬 관리 도구.

**핵심 설계 원칙:**
- 로컬 실행 (서버 불필요, Python + Tkinter)
- 보안 자격증명 분리 (keyring, .gitignore)
- 팀 협업은 GitHub private repo 경유 (PAT + HTTPS API)
- 단일 실행 인스턴스 강제 (Windows Mutex)

---

## 2. 기술 스택

| 항목 | 선택 | 이유 |
|------|------|------|
| 언어 | Python 3.10+ | SSH/SFTP 라이브러리 풍부, 로그 처리 용이 |
| GUI | Tkinter | Python 기본 내장, Windows 지원 |
| SSH | paramiko | AP/EFG/NAS 접속 통합 처리 |
| 인증 | bcrypt + TOTP | 비밀번호 해시 + 2FA (pyotp) |
| 자격증명 | keyring (Windows Credential Manager) | OS 레벨 보안 보관 |
| UniFi API | New API Key 방식 | HTTPS, PAT 기반 |
| GitHub API | HTTPS + PAT | 팀 공유 데이터 저장소 |
| 이미지 | Pillow (PIL) | 교회 로고 리사이즈 표시 |
| 암호화 | cryptography (Fernet) | NAS 자격증명 암호화 |

---

## 3. 아키텍처 개요

```
┌──────────────────────────────────────────────────────────────────┐
│                    NetworkGuiApp (main.py)                       │
│  ┌───────────┐ ┌──────────┐ ┌──────────┐ ┌──────┐ ┌────────┐   │
│  │ AP Status │ │EFG Remote│ │Log Export│ │동기화│ │  설정  │   │
│  └─────┬─────┘ └────┬─────┘ └────┬─────┘ └──┬───┘ └───┬────┘   │
└────────│────────────│────────────│───────────│─────────│────────┘
         │            │            │           │         │
    ┌────▼────┐  ┌────▼────┐  ┌───▼───┐  ┌───▼───┐  ┌──▼──────┐
    │ap_remote│  │efg_     │  │log_   │  │nas_   │  │auth.py  │
    │ap_reset │  │remote   │  │export │  │sync   │  │keychain │
    │ap_count │  │unifi_   │  │       │  │       │  │         │
    │ap_detail│  │client   │  │       │  │       │  │         │
    └────┬────┘  └────┬────┘  └───────┘  └───────┘  └─────────┘
         │            │
    ┌────▼────┐  ┌────▼────────────────────────────────────────┐
    │ AP SSH  │  │ UniFi Controller API (HTTPS)               │
    │(paramiko│  │ EFG SSH (paramiko)                          │
    │via EFG) │  └────────────────────────────────────────────┘
    └─────────┘
         │
    ┌────▼──────────────────────────────┐
    │ GitHub API (HTTPS + PAT)          │
    │ newvisionchurch/nvc-auth          │
    │ ├── ap_reset_schedule.json        │
    │ └── feedback.json                 │
    └───────────────────────────────────┘
```

---

## 4. 탭 구성

| 탭 | 서브탭 | 메서드 | 권한 |
|---|---|---|---|
| AP Status | — | `_build_ap_remote_tab()` | 모든 사용자 |
| EFG Remote | EFG SSH | `_build_efg_remote_tab()` | `can_view_efg_tab` |
| EFG Remote | EFG API | `_build_unifi_api_tab()` | `can_view_efg_tab` |
| Log Export | — | `_build_export_tab()` | `can_export` |
| 동기화 | NAS ELK Sync | `_build_sync_tab()` | `can_sync_nas` |
| 동기화 | 연결 현황 | `_build_settings_connection_tab()` | — |
| 설정 | 일반 | `_build_settings_general_tab()` | 모든 사용자 |
| 설정 | 보안 관리 | `_build_data_paths_tab()` | 모든 사용자 |
| 설정 | 사용자 관리 | `_build_admin_tab()` | `can_manage_users` |

---

## 5. AP Status 탭 상세

### 컨트롤 바 구조 (1단, 4섹션)

```
[섹션1: 등록/온라인/오프라인 카드] │ [섹션2: SSH 조회] │ [섹션3: ELK 날짜+Stuck] │ [섹션4: 분류]
```

**섹션2 버튼 (2줄):**
- 1줄: 전체 조회 / 선택 조회 / 중지
- 2줄: 동작 설명 / 경고 설정 / 컬럼 설정

### 하단 버튼 바 (좌→우)

```
[상세 보기]   [AP Reset 선택] [AC Pro 선택] [AP 선택 해제] | [모두 해제] | [분류 선택] [예약 관리] | 선택N | [AP Reset 실행]
```

### AP Reset 실행 흐름

```
_remote_reset_selected()
  ↓ 권한 확인 (can_reset_ap)
  ↓ 대상 AP 목록 수집 (self._reset_target_ap_ids)
  ↓ AP 비밀번호 획득 (_ensure_ap_password)
  ↓ EFG Relay 정보 (_get_efg_relay)
  ↓
_show_reset_schedule_dialog(target_aps, pw, relay)
  ↓ 팝업: 시작 시각 / 순서 / AP 체크박스 / 간격 / 실행 모드
  ↓ [실행 예약] 클릭
  ↓
_execute_reset_scheduled(aps, pw, relay, operator, start_dt, interval_min, mode)
  ↓ 지정 시각까지 대기
  ↓ reset_ap() 순차 실행 (간격 적용)
  ↓ operations.log 기록
  ↓ GitHub ap_reset_schedule.json history 기록
```

---

## 6. ResetScore 분류 시스템

AP의 Reset 필요성을 수치화해 운영자 판단을 보조합니다.

### 점수 계산 (ap_remote.py — SCORE_PARAM_DEFAULTS)

| 항목 | 조건 | 점수 |
|------|------|------|
| 로그오류(log_score) | 1~2 | 30 |
| 로그오류 | 3~9 | 40 |
| 로그오류 | ≥10 | 50 |
| ELK Stuck(elk_score) | 500~1999 | 15 |
| ELK Stuck | 2000~4999 | 30 |
| ELK Stuck | ≥5000 | 50 |
| 무선 리셋(reset5g_score, AC HD) | 100~299 | 10 |
| 무선 리셋 | 300~599 | 20 |
| 무선 리셋 | ≥600 | 30 |
| VAP 지연(vap_score) | 1~20 | 15 |
| VAP 지연 | ≥21 | 30 |
| 채널이용률(cu_score) | 70~84% | 10 |
| 채널이용률 | ≥85% | 15 |

### 분류 임계값

| 색상 | 조건 | 표시 |
|------|------|------|
| 🔴 빨강 | 점수 ≥ 60 AND reset_evidence=True | Reset 후보 |
| 🟡 노랑 | 점수 ≥ 20 | 관찰 필요 |
| 🟢 초록 | 점수 < 20 | 정상 |
| ⬜ 회색 | 미조회 | 데이터 없음 |

`reset_evidence`: 로그오류≥1 OR VAP지연≥1 OR ELK Stuck≥500

---

## 7. AP Reset EFG Relay 구조

AP Dropbear SSH는 paramiko 패스워드 인증 직접 연결이 불안정합니다.  
EFG OpenSSH를 경유해 `sshpass`로 접속하는 relay 방식을 사용합니다.

```
PC (paramiko)
  └─ SSH connect to EFG (OpenSSH, password auth)
       └─ exec_command: "sshpass -p <pw> ssh -o StrictHostKeyChecking=no ubnt@<AP_IP> reboot"
```

**구현:** `ssh_client.py` → `run_ssh_via_relay()`

**전제 조건:**
- EFG에 `sshpass` 패키지 설치 (`apt install sshpass`)
- EFG와 AP 사이 내부망 라우팅 정상

---

## 8. GitHub 연동 설계

### 인증 방식

PAT를 Windows Keychain(`keyring`)에 저장. 코드에 하드코딩 없음.

```python
# github_auth.py
KEYCHAIN_SERVICE = "nvc_auth_github"
keyring.set_password(KEYCHAIN_SERVICE, org, token)   # 저장
keyring.get_password(KEYCHAIN_SERVICE, org)           # 조회
```

### 파일 R/W 방식

GitHub Contents API (PUT)를 사용합니다. 파일 수정 시 현재 SHA 조회 후 갱신.

```
GET  /repos/{org}/{repo}/contents/{path}  → base64 decode → JSON parse
PUT  /repos/{org}/{repo}/contents/{path}  → JSON → base64 encode → SHA 포함
```

### 중복 예약 방지 (_overlaps)

새 예약 생성 시 기존 pending/running 예약에서 같은 AP가 ±60분 내 겹치면 차단합니다.

### 데이터 구조

#### ap_reset_schedule.json

```json
{
  "schedules": [
    {
      "id": "YYYYMMDD-HHMMSS-<username>",
      "created_by": "string",
      "created_at": "ISO8601",
      "target_time": "ISO8601",
      "ap_ids": ["AP01", ...],
      "interval_min": 4,
      "order": "yellow_first | red_first",
      "status": "pending | running | done | cancelled"
    }
  ],
  "history": [
    {
      "schedule_id": "string",
      "ap_id": "AP01",
      "reset_at": "ISO8601",
      "result": "success | failure"
    }
  ]
}
```

#### feedback.json

```json
{
  "items": [
    {
      "id": "YYYYMMDD-HHMMSS-<username>",
      "type": "bug | improvement | notice",
      "title": "string",
      "body": "string",
      "author": "string",
      "created_at": "ISO8601",
      "version": "0.3",
      "status": "open | closed",
      "comments": [
        {
          "author": "string",
          "body": "string",
          "created_at": "ISO8601"
        }
      ]
    }
  ]
}
```

---

## 9. 인증 설계

### 로그인 흐름

```
앱 시작
  ↓
VPN 연결 확인 (vpn_check.py)
  ↓
개발자 컴퓨터 여부 확인 (MachineGuid)
  ├─ YES → 자동 로그인 (_show_main)
  └─ NO  → 로그인 화면
              ↓
            bcrypt 검증
              ↓
            TOTP 등록 여부 확인
              ├─ 미등록 → TOTP 등록 화면 (QR코드)
              └─ 등록됨 → 기기 신뢰 여부 확인
                          ├─ 신뢰 기기 → 자동 통과
                          └─ 미신뢰   → TOTP 입력
                                         ↓
                                       _show_main()
```

### 인증 백엔드 (Authenticator)

| 방식 | 설명 |
|------|------|
| GitHub | nvc-auth/users.json 에서 로드 (HTTPS API) |
| Local | gui/config/users.json 에서 로드 |

설정에서 방식을 변경 가능. Fallback 지원(GitHub 실패 시 Local 시도).

### 자격증명 보관

| 대상 | 보관 방식 |
|------|----------|
| 사용자 비밀번호 | bcrypt 해시 (users.json) |
| TOTP secret | users.json (암호화 없음, 파일 접근 제어) |
| AP SSH 비밀번호 | Windows Keychain (keyring) |
| EFG SSH 비밀번호 | Windows Keychain |
| NAS SSH 키/비밀번호 | Windows Keychain |
| GitHub PAT | Windows Keychain |
| UniFi API Key | Windows Keychain |

---

## 10. 모듈 구조

| 모듈 | 역할 |
|------|------|
| `auth.py` | bcrypt 로그인 + TOTP 2FA + 개발자 모드 (MachineGuid) |
| `vpn_check.py` | TCP 소켓으로 VPN 연결 확인 |
| `nas_sync.py` | paramiko SFTP — NAS JSONL 로컬 캐시 동기화 (차분) |
| `ap_count.py` | 날짜 범위별 AP Stuck 이벤트 집계 |
| `ap_detail.py` | 문제 AP 상세 분석, Stuck 유형 분류 |
| `ap_reset.py` | SSH 원격 reboot — EFG relay 경유 |
| `ap_remote.py` | AP SSH 스캔 — mca-dump 파싱 + logread 진단 + ResetScore |
| `efg_remote.py` | EFG 대시보드 — 시스템/네트워크/트래픽/ARP 조회 |
| `unifi_client.py` | UniFi Local API 클라이언트 (New API Key / Legacy fallback) |
| `log_export.py` | 조건별 로그 필터링 후 파일 저장 |
| `ap_inventory.py` | AP 목록 관리 (ap_inventory.json 기반) |
| `local_storage.py` | 런타임 폴더 생성·관리, app_config.json 로드 |
| `report_writer.py` | operations.log에 사용자명·작업 기록 |
| `keychain.py` | Windows Credential Manager (keyring 래퍼) |
| `ssh_client.py` | paramiko 공통 — 패스워드/키 인증, PTY, relay |
| `font_utils.py` | OS 폰트 감지 및 UI 전체 폰트 적용 |
| `models.py` | 핵심 데이터 모델 |
| `github_auth.py` | GitHub API HTTPS + PAT 인증, 파일 R/W |
| `github_schedule.py` | AP Reset 예약 CRUD (ap_reset_schedule.json) |
| `github_feedback.py` | 팀원 게시판 CRUD (feedback.json) |
| `credentials.py` | NAS 자격증명 Fernet 암호화 저장 |

---

## 11. 주요 데이터 모델

### APRemoteInfo (ap_remote.py)

```python
@dataclass
class APRemoteInfo:
    ap_id: str
    name: str
    ip: str
    model: str
    ssh_ok: bool
    log_error_count: int          # logread grep 결과
    dev_reset_count: int          # WAL_DBGID_DEV_RESET 패턴
    ast_ath_reset: list[int]      # [2G, 5G] 드라이버 리셋
    timeout_vap_cnt: list[int]    # [2G, 5G] VAP 타임아웃
    cu_total: list[float]         # [2G, 5G] 채널이용률
    num_sta: list[int]            # [2G, 5G] 연결 클라이언트
    cpu_usage: float
    mem_usage: float
    uptime: str
    firmware: str
    response_ms: int
    # ... ResetScore 결과
```

### User (models.py)

```python
@dataclass
class User:
    username: str
    display_name: str
    role: str                     # admin | operator | viewer
    permissions: dict[str, bool]
    active: bool

    def can(self, permission: str) -> bool: ...
```

---

## 12. 설정 파일

### app_config.json (git 포함)

| 키 | 설명 |
|----|------|
| `ap_registered_count` | 인벤토리 AP 등록 수 |
| `elk_default_days` | ELK 날짜 범위 기본값 (일) |
| `stuck_thresholds` | ⚠ 표시 임계값 (컬럼별) |
| `unifi_api.base_url` | UniFi Controller 주소 |
| `logs.default_log_root` | 로컬 JSONL 캐시 경로 |
| `col_hidden_default` | 기본 숨김 컬럼 목록 |
| `remote_default_sort` | AP Status 기본 정렬 컬럼 |
| `md_base_path` | 문서 MD 파일 기본 경로 |

### ssh_targets.json (git 제외)

```json
{
  "nas": { "host": "", "port": 22, "username": "", "key_path": "" },
  "efg": { "host": "", "port": 22, "username": "", "password_keychain": true },
  "ap_defaults": { "username": "ubnt", "port": 22 }
}
```

### ap_inventory.json (git 포함, 29대)

```json
{
  "access_points": [
    {
      "id": "AP01",
      "name": "M1F_AP1_OFFICE",
      "ip": "192.168.11.49",
      "site": "Main",
      "floor": "1F",
      "location": "Office",
      "model": "U6 LR",
      "firmware": "unknown",
      "parent_device": "M1F_OFFICE_S8",
      "parent_port": "Port 3",
      "keywords": ["M1F_AP1", "OFFICE"]
    }
  ]
}
```

**모델 현황 (29대):**

| 모델 | 수량 |
|------|------|
| AC Pro | 14대 |
| AC HD | 2대 |
| U6 LR | 6대 |
| U7 Pro | 2대 |
| U7 Pro Wall | 2대 |
| 기타 | 3대 |

---

## 13. 구현 이력

### v0.3 — 2026-05-30

**신규 기능:**
- `github_schedule.py` — AP Reset 예약 CRUD (create/cancel/list/history, ±60분 중복 방지)
- `github_feedback.py` — 팀원 게시판 CRUD (create/comment/close/get_unread_notices)
- `_show_github_schedule_dialog()` — 예약 목록(취소) / AP Reset 이력 2탭 팝업
- `_show_feedback_dialog()` — 목록(필터/댓글) / 작성 2탭 팝업
- `_check_and_show_notices()` — 로그인 후 2.5초 뒤 open 공지 자동 팝업
- 하단 바 [예약 관리] 버튼 (Action.TButton 스타일)
- 상단 타이틀 바 [💬 팀원 게시판] 버튼

**버튼 배치 변경:**
- 하단 바: `[분류 선택] [예약 관리] | 선택N | [AP Reset 실행]`

### v0.2 — 2026-05-29

- NAS 인증 안정성 개선 (Fernet 암호화 자격증명 도입)
- EFG SSH 대시보드 컬럼 정렬 기능 추가
- ResetScore 파라미터 설정 UI 추가 (`_show_score_settings_dialog`)
- AP Reset 포그라운드/백그라운드 모드 분리
- 설정 탭 보안 관리 서브탭 — GitHub PAT 설정 UI 추가
- UniFi API Probe 기능 추가

### v0.1 — 최초 구현

- AP Status 탭 (SSH 조회 + ELK Stuck 집계 + AP Reset)
- EFG Remote 탭 (EFG SSH / EFG API)
- Log Export 탭
- NAS 동기화 탭
- 기본 인증 (bcrypt + TOTP)
- AP 인벤토리 (29대)
