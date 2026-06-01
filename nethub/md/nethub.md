# NVC Network Hub (nethub) — SW 설계 참조

GUI Version 0.4 — 05/31/2026  
Python 3.10+ / Tkinter / Windows 로컬 관리 도구

---

## 아키텍처 개요

```
NetworkGuiApp (source/main.py)
  │
  ├── AP Status 탭
  │     ap_remote.py (SSH mca-dump + logread + ResetScore)
  │     ap_count.py  (ELK JSONL 캐시 Stuck 집계)
  │     ap_reset.py  (EFG relay → AP reboot)
  │
  ├── EFG Remote 탭
  │     efg_remote.py  (SSH 대시보드)
  │     unifi_client.py (API 탐색기)
  │
  ├── Log Export 탭 → log_export.py
  ├── 동기화 탭     → nas_sync.py (SFTP)
  └── 설정 탭       → auth.py, keychain.py, github_*.py

GitHub API (HTTPS + PAT)
  └── newvisionchurch/nvc-auth
       ├── users.json              (인증 원본)
       ├── ap_reset_schedule.json  (예약)
       └── feedback.json           (게시판)
```

---

## 탭 구성

| 탭 | 서브탭 | 빌드 메서드 | 권한 |
|---|---|---|---|
| AP Status | — | `_build_ap_remote_tab()` | 모든 사용자 |
| EFG Remote | EFG SSH | `_build_efg_remote_tab()` | `can_view_efg_tab` |
| EFG Remote | EFG API | `_build_unifi_api_tab()` | `can_view_efg_tab` |
| Log Export | — | `_build_export_tab()` | `can_export` |
| 동기화 | NAS ELK Sync | `_build_sync_tab()` | `can_sync_nas` |
| 동기화 | AP Sync | (서브탭) | — |
| 동기화 | 연결 현황 | `_build_settings_connection_tab()` | — |
| 설정 | 일반 | `_build_settings_general_tab()` | 모든 사용자 |
| 설정 | 자동화 | `_build_settings_automation_tab()` | 모든 사용자 |
| 설정 | 보안 | `_build_data_paths_tab()` | 모든 사용자 |
| 설정 | 사용자 | `_build_admin_tab()` | `can_manage_users` |

---

## 모듈 구조 (source/modules/)

| 모듈 | 역할 |
|------|------|
| `models.py` | AccessPoint, StuckSummary, Workspace, SyncSettings, SyncSummary, User |
| `constants.py` | SSH 타임아웃, UI 색상, AP 모델 분류, RELAY_MAX_CONCURRENT=4 |
| `auth.py` | bcrypt + TOTP + 3단계 인증(GitHub→NAS→Local) + admin 해시 격리 |
| `vpn_check.py` | TCP 소켓 VPN 확인 (192.168.11.1:22) |
| `nas_sync.py` | paramiko SFTP — NAS JSONL → PC 캐시 (차분) |
| `ap_count.py` | 날짜 범위별 AP Stuck 이벤트 집계 |
| `ap_detail.py` | AP 상세 분석, Stuck 유형 분류 |
| `ap_reset.py` | SSH 원격 reboot (EFG relay) |
| `ap_remote.py` | AP SSH 스캔: mca-dump + logread + ResetScore (`calculate_reset_score`) |
| `efg_remote.py` | EFG SSH 대시보드 (`fetch_efg_dashboard`, `fetch_efg_diagnostics`) |
| `unifi_client.py` | UniFi API (New API Key + Legacy) |
| `log_export.py` | 조건별 로그 필터 후 파일 저장 |
| `ap_inventory.py` | AP 목록 관리 (ap_inventory.json) |
| `local_storage.py` | 런타임 폴더 생성, app_config.json 로드/저장 |
| `report_writer.py` | operations.log 기록 |
| `keychain.py` | Windows Credential Manager (keyring 래퍼) |
| `ssh_client.py` | paramiko 공통 — 패스워드/키 인증, PTY, `run_ssh_via_relay()` |
| `font_utils.py` | OS 폰트 감지, UI 전체 폰트 적용 |
| `github_auth.py` | GitHub API HTTPS + PAT, 파일 R/W (Contents API) |
| `github_schedule.py` | AP Reset 예약 CRUD (±60분 중복 방지) |
| `github_feedback.py` | 팀원 게시판 CRUD |
| `diagram_viewer.py` | Block Diagram 11장 슬라이드 팝업 |
| `credentials.py` | NAS 자격증명 Fernet 암호화 |

---

## 핵심 데이터 모델 (source/modules/models.py)

### AccessPoint (frozen dataclass)

```python
ap_id: str          # "AP01"
name: str           # "M1F_AP1_OFFICE"
ip: str             # "192.168.11.49"
site: str           # "Main" | "Education"
floor: str          # "1F" | "2F"
location: str       # "Office"
model: str          # "U6 LR" | "AC Pro" | "AC HD" | "U7 Pro" | "U7 Pro Wall"
firmware: str       # "unknown" (인벤토리 기본값)
parent_device: str
parent_port: str
keywords: tuple[str, ...]
```

### APRemoteInfo (ap_remote.py)

```python
ap_id, name, ip, model: str
ssh_ok: bool
log_error_count: int        # logread grep 오류 수
dev_reset_count: int        # WAL_DBGID_DEV_RESET 패턴 수
ast_ath_reset: list[int]    # [2G, 5G] 드라이버 리셋 누적
timeout_vap_cnt: list[int]  # [2G, 5G] VAP 타임아웃 누적
cu_total: list[float]       # [2G, 5G] 채널이용률
num_sta: list[int]          # [2G, 5G] 연결 클라이언트
cpu_usage: float
mem_usage: float
uptime: str
firmware: str
response_ms: int
# ResetScore 결과 포함
```

### Workspace (frozen dataclass)

```python
root: Path                  # C:\Projects\nvc_nethub\runtime
logs_raw: Path              # root/logs/raw
exports_ap_stuck: Path      # root/exports/ap_stuck
exports_ap_detail: Path     # root/exports/ap_detail
reports_manual: Path        # root/reports/manual
mca_dumps: Path             # root/mca_dumps
```

### User (dataclass)

```python
username: str
display_name: str
role: str           # "admin" | "operator" | "viewer"
permissions: dict   # {can_reset_ap, can_sync_nas, can_export, can_view_efg_tab, can_manage_users}

def can(self, permission: str) -> bool: ...
```

---

## AP Status 탭 — 핵심 흐름

### SSH 조회 흐름

```
_start_remote_all() / _start_remote_selected()
  ↓ 스레드 풀 (RELAY_MAX_CONCURRENT=4 동시 세션)
  ↓ fetch_all_ap_info() → fetch_ap_info()
  ↓   mca-dump 파싱 (ast_ath_reset, timeout_vap_cnt, cu_total, num_sta, cpu, mem)
  ↓   logread grep (오류 패턴 수, WAL_DBGID_DEV_RESET 수)
  ↓ 결과 → APRemoteInfo
  ↓ 테이블 업데이트
```

### ResetScore 분류

점수 계산 (`ap_remote.py` — `calculate_reset_score`, `SCORE_PARAM_DEFAULTS`):

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

분류: 점수 ≥60 + reset_evidence → 빨강(Reset 권장) / 점수 ≥20 → 노랑 / <20 → 초록

### AP Reset 흐름

```
_remote_reset_selected()
  ↓ 권한 확인 (can_reset_ap)
  ↓ _ensure_ap_password() → keychain
  ↓ _get_efg_relay() → ssh_targets.json
  ↓
_show_reset_schedule_dialog()  ← 팝업: 시작시각/순서/AP목록/간격/실행모드
  ↓ [실행 예약]
  ↓
_execute_reset_scheduled()
  ↓ 지정 시각까지 대기
  ↓ reset_ap() 순차 실행 (간격 적용)
  ↓ operations.log 기록
  ↓ GitHub ap_reset_schedule.json history 기록
```

### EFG SSH 대시보드

`fetch_efg_dashboard()` — 단일 SSH 세션, `##MARKER##` 구분 수집:
- 시스템 카드: hostname, uptime, kernel, loadavg, free -m
- 네트워크 경로: ip route show
- 인터페이스: ip addr show
- 트래픽: /proc/net/dev
- ARP: arp -n
- 모든 테이블 컬럼 클릭 정렬

### EFG API 탭

- 레이아웃: 좌(30%) / 우(70%)
- Preset 20개 + GET
- 캐시: `runtime/api_probe/YYYYMMDD_HHMMSS_<preset>.json`
- Controller: UniFi Network Application 10.3.58 (Dream Machine Enterprise)
- New API: `/proxy/network/integration/v1/` (API Key)
- Legacy: `/proxy/network/api/s/default/` (일부 endpoint 미지원)

---

## 인증 설계

### 로그인 흐름

```
앱 시작
  ↓ VPN 확인 (192.168.11.1:22)
  ↓ MachineGuid 확인
  ├─ 개발자 등록 PC → 서버 연결 화면(카운트다운) → 자동 로그인
  └─ 일반 PC → 로그인 화면
                ↓ 3단계 users.json 로드 (GitHub → NAS → Local)
                ↓ bcrypt 검증
                ↓ TOTP 등록 여부
                ├─ 미등록 → QR코드 등록 화면
                └─ 등록됨 → TOTP 입력 → _show_main()
```

### admin 해시 격리 정책

```python
# auth.py — _strip_admin_hashes()
if not _is_admin_registered_machine(users_path):
    # 일반 PC: admin password_hash, totp_secret 제거 후 저장
```

관리자 등록 PC만 오프라인 admin 로그인 가능.

---

## 자격증명 보관

| 대상 | 보관 방식 |
|------|----------|
| 사용자 비밀번호 | bcrypt 해시 (users.json) |
| TOTP secret | users.json (admin은 관리자 등록 PC에만 로컬 보관) |
| AP SSH 비밀번호 | Windows Keychain (keyring) |
| EFG SSH 비밀번호 | Windows Keychain |
| NAS SSH 키/비밀번호 | Windows Keychain |
| GitHub PAT | Windows Keychain — `KEYCHAIN_SERVICE = "nvc_auth_github"` |
| UniFi API Key | Windows Keychain |

---

## 설정 파일

### app_config.json (주요 키)

| 키 | 값/설명 |
|----|---------|
| `workspace_root_windows` | `C:\Projects\nvc_nethub\runtime` |
| `auth.method` | `"github"` |
| `auth.github_org` | `"newvisionchurch"` |
| `auth.github_repo` | `"nvc-auth"` |
| `nas_sync.remote_output_root` | `/docker/elk/logstash/outputs` |
| `nas_sync.storage_classes` | `["ap"]` |
| `nas_sync.days_to_sync` | `2` |
| `vpn_check.host` | `192.168.11.1` |
| `vpn_check.port` | `22` |
| `ap_stuck.status_thresholds` | `{healthy:0, warning:2, suspect:9}` |
| `stuck_thresholds` | 컬럼별 ⚠ 임계값 |
| `unifi_api.base_url` | `https://192.168.11.1` |
| `unifi_api.site_id` | `88f7af54-98f8-306a-a1c7-c9349722b1f6` |
| `startup_actions` | 시작 시 자동 실행 항목 |
| `col_hidden_default` | 기본 숨김 컬럼 목록 |
| `md_base_path` | `C:\Projects\nvc_public\nvc_network\md` |

### ssh_targets.json (git 제외, 예제: ssh_targets.example.json)

```json
{
  "nas": { "host": "", "port": 22, "username": "", "key_path": "" },
  "efg": { "host": "192.168.11.1", "port": 22, "username": "", "password_keychain": true },
  "ap_defaults": { "username": "ubnt", "port": 22 }
}
```

---

## GitHub 연동 설계

```
GET  /repos/newvisionchurch/nvc-auth/contents/{path}  → base64 decode → JSON
PUT  /repos/newvisionchurch/nvc-auth/contents/{path}  → JSON → base64 → SHA 포함
```

### ap_reset_schedule.json 구조

```json
{
  "schedules": [{
    "id": "YYYYMMDD-HHMMSS-<username>",
    "target_time": "ISO8601",
    "ap_ids": ["AP13", "AP14"],
    "interval_min": 4,
    "order": "yellow_first | red_first",
    "status": "pending | running | done | cancelled"
  }],
  "history": [{ "ap_id": "AP13", "reset_at": "ISO8601", "result": "success | failure" }]
}
```

중복 방지: 같은 AP에 ±60분 내 겹치는 pending/running 예약 차단.

---

## AP 인벤토리 (ap_inventory.json — 29대)

| 모델 | 수량 |
|------|------|
| AC Pro | 14 |
| U6 LR | 9 |
| AC HD | 2 |
| U7 Pro | 2 |
| U7 Pro Wall | 2 |

`AP_MODELS_WITH_5G_RESET = frozenset({"AC Pro", "AC HD"})` — 5G reset 컬럼 지원 모델

---

## 색상 상수 (source/modules/constants.py)

```python
COLOR_HEALTHY  = "#176b3a"    # Healthy (Green)
COLOR_WARNING  = "#8a5a00"    # Warning (Yellow)
COLOR_SUSPECT  = "#a64600"    # Suspect (Orange)
COLOR_CRITICAL = "#b3261e"    # Critical (Red)
COLOR_MUTED    = "#5b677a"    # 미조회/비활성
COLOR_OUTPUT_BG = "#fafafa"   # 텍스트 출력 영역 배경
COLOR_BORDER    = "#cfd8dc"   # 카드/입력 영역 테두리
```

---

## 변경 이력 요약

| 버전 | 주요 내용 |
|------|----------|
| v0.1 | AP Status + EFG Remote + Log Export + NAS 동기화 + bcrypt/TOTP 인증 |
| v0.2 | ResetScore UI, EFG SSH 정렬, AP Reset 포그라운드/백그라운드, GitHub PAT |
| v0.4 (05/31) | AP Reset 예약(GitHub), 팀원 게시판, Block Diagram, 3단계 인증, 자동화 설정 |
| v0.3 (05/31) | 색상 설정 팝업 분리, 설정 탭 재구성(일반/자동화/보안/사용자), admin 해시 격리 |
