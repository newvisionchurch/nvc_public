# NVC NetHub — 설계 참조

NVC NetHub는 뉴비전 교회 UniFi 네트워크 운영을 위한 Python/Tkinter 기반 로컬 관리 도구입니다.
AP 상태 분석, ELK 로그 집계, EFG 원격 조회, AP Reset 예약, 팀 공유 게시판을 하나의 도구에서 제공합니다.

버전은 `source/main.py`의 `GUI_VERSION` 상수로 관리합니다.

## 아키텍처

```text
NetworkGuiApp (source/main.py — 13,400+ 줄, 단일 클래스)
  │
  ├── AP Status 탭
  │   ├── ap_remote.py          # AP SSH 진단, mca-dump, logread, ResetScore
  │   ├── ap_count.py           # raw Log 기반 날짜별 Stuck Count 계산
  │   ├── elk_stuck_record.py   # 날짜별 ELK Stuck Record 저장 및 집계
  │   ├── ap_detail.py          # AP 상세 팝업 분석
  │   └── ap_reset.py           # EFG relay 기반 AP reboot
  │
  ├── EFG Remote 탭
  │   ├── efg_remote.py         # EFG SSH 대시보드 (네트워크/트래픽/ARP/진단)
  │   └── unifi_client.py       # UniFi Controller API 클라이언트
  │
  ├── 동기화 탭
  │   └── nas_sync.py           # NAS JSONL 로그 SFTP 동기화
  │
  ├── Log Export 탭
  │   └── log_export.py         # 조건별 로그 Export
  │
  ├── 설정 탭
  │   ├── font_utils.py         # 폰트 설정
  │   └── diagram_viewer.py     # 구조도 팝업
  │
  └── 공통 인프라
      ├── models.py             # 핵심 dataclass 모델
      ├── constants.py          # 타임아웃, 색상, AP 모델 상수
      ├── auth.py               # 인증, bcrypt, TOTP, 역할/권한 관리
      ├── credentials.py        # 팀 공유 비밀번호 TOTP 기반 암호화/복호화
      ├── keychain.py           # Windows Keychain 래퍼 (NAS/EFG/AP/API Key)
      ├── github_auth.py        # GitHub Contents API 파일 읽기/쓰기
      ├── github_schedule.py    # AP Reset 예약 및 이력 관리
      ├── github_feedback.py    # 팀원/관리자 게시판
      ├── ssh_client.py         # paramiko 공통 SSH 처리 (직접/relay)
      ├── vpn_check.py          # VPN 연결 확인
      ├── ap_inventory.py       # AP 인벤토리 로드
      ├── local_storage.py      # 런타임 폴더 생성 및 설정 로드
      └── report_writer.py      # operations.log 기록
```

## GUI 탭 구성

### 메인 탭 (1단)

| 탭 | 빌드 함수 | 주요 기능 |
|----|----------|----------|
| AP Status | `_build_ap_remote_tab` | AP SSH 진단, ELK Stuck, ResetScore, Reset |
| EFG Remote | `_build_efg_remote_tab` / `_build_unifi_api_tab` | EFG SSH + API |
| 동기화 | `_build_sync_tab` / `_build_sync_ap_tab` / `_build_settings_connection_tab` | NAS/AP 동기화, 연결 현황 |
| Log Export | `_build_export_tab` | 로그 조건 검색 및 저장 |
| 설정 | `_build_settings_all_tabs` | 앱 전체 설정 |

### EFG Remote 서브탭 (2단)

| 서브탭 | 빌드 함수 | 주요 기능 |
|--------|----------|----------|
| EFG SSH | `_build_efg_remote_tab` | 네트워크 상태, 트래픽, ARP, 진단 |
| EFG API | `_build_unifi_api_tab` | UniFi Controller read-only API 탐색 |

### 동기화 서브탭 (2단)

| 서브탭 | 빌드 함수 | 주요 기능 |
|--------|----------|----------|
| NAS ELK Sync | `_build_sync_tab` | NAS JSONL 로그 동기화, ELK Record 생성/조회 |
| AP Sync | `_build_sync_ap_tab` | AP mca-dump 파일 관리 |
| 연결 현황 | `_build_settings_connection_tab` | NAS/UniFi API 연결 테스트 |

### 설정 서브탭 (2단)

| 서브탭 | 빌드 함수 | 접근 권한 | 주요 내용 |
|--------|----------|----------|----------|
| 일반 | `_build_settings_general_tab` | 모든 사용자 | 폰트, AP 설정, 점수 임계값, 색상 |
| 자동화 | `_build_settings_automation_tab` | 모든 사용자 | 시작 시 자동 실행 항목 |
| 보안 | `_build_data_paths_tab` | 모든 사용자 | ELK 로그 캐시 경로 |
| 관리자 | `_build_admin_tab` | `can_manage_users` | 사용자 관리 + 보안 설정 서브탭 |

### 관리자 탭 서브탭 (3단, `can_manage_users` 전용)

| 서브탭 | 빌드 함수 | 주요 내용 |
|--------|----------|----------|
| 사용자 관리 | `_build_admin_tab` 내 | 계정/권한/TOTP/개발자 PC 관리 |
| 보안 설정 | `_build_credentials_manager_section` | 팀 공유 비밀번호 (NAS/EFG/AP/EFG API) |

### AP 상세 팝업 탭 (별도 팝업)

| 탭 | 빌드 함수 | 내용 |
|----|----------|------|
| 요약 테이블 | `_build_detail_tab_summary` | AP 지표 요약 |
| VAP / 라디오 | `_build_detail_tab_vap` | 무선 라디오 상세 |
| 로그오류 | `_build_detail_tab_log` | logread 오류 패턴 |
| DevReset | `_build_detail_tab_fwbug` | WAL_DBGID_DEV_RESET 이벤트 |
| Raw mca-dump | `_build_detail_tab_raw` | 원본 mca-dump 텍스트 |

## 역할 및 권한

| 역할 | 설명 | 기본 권한 |
|------|------|----------|
| `admin` | 관리자 | 모든 기능 |
| `manager` | 네트워크 담당자 | AP Reset, 동기화, Export, EFG 조회 |
| `member` | 일반 팀원 | AP 조회, Export, EFG 조회 |
| `guest` | 방문자 | 제한된 조회 |

권한은 사용자별 `permissions`에 저장하며, 역할 템플릿은 관리자 탭 → 역할 권한 설정에서 조정합니다.

## 주요 모듈

| 모듈 | 역할 |
|------|------|
| `models.py` | `AccessPoint`, `Workspace`, `User`, `SyncSettings`, `StuckSummary` 등 핵심 dataclass |
| `constants.py` | 타임아웃, 색상, AP 모델 상수 |
| `auth.py` | GitHub/NAS/Local 인증, bcrypt, TOTP, 역할 권한, dev_machines 관리 |
| `credentials.py` | 팀 공유 비밀번호 TOTP secret 기반 암호화/복호화, 사용자별 암호화 저장 |
| `keychain.py` | Windows Keychain 래퍼 (NAS/EFG/AP SSH 비밀번호, EFG API Key) |
| `vpn_check.py` | VPN 연결 확인 |
| `nas_sync.py` | NAS JSONL 로그 SFTP 동기화, 로컬 캐시 관리 |
| `ap_count.py` | raw Log에서 날짜별 AP Stuck Count 계산 |
| `elk_stuck_record.py` | 날짜별 Record 저장, Log signature 확인, Record 기반 집계 |
| `ap_detail.py` | AP 상세 분석 |
| `ap_remote.py` | AP SSH 진단과 ResetScore 계산 (mca-dump + logread) |
| `ap_reset.py` | AP reboot 실행 (EFG relay 경유) |
| `efg_remote.py` | EFG SSH 대시보드 (인터페이스/트래픽/ARP/진단) |
| `unifi_client.py` | UniFi API 클라이언트 |
| `log_export.py` | 조건별 로그 Export |
| `ap_inventory.py` | AP 인벤토리 로드 |
| `local_storage.py` | 런타임 폴더 생성 및 설정 로드 |
| `report_writer.py` | `operations.log` 기록 |
| `ssh_client.py` | paramiko 공통 SSH 처리 (직접 연결 + EFG relay) |
| `font_utils.py` | 폰트 설정 |
| `github_auth.py` | GitHub Contents API 파일 읽기/쓰기 (PAT 기반) |
| `github_schedule.py` | AP Reset 예약 및 이력 |
| `github_feedback.py` | 팀원/관리자 게시판 |
| `diagram_viewer.py` | 구조도 팝업 |

## 핵심 데이터 모델

### `AccessPoint`

```python
ap_id: str
name: str
ip: str
site: str
floor: str
location: str
model: str
firmware: str
parent_device: str
parent_port: str
keywords: tuple[str, ...]
```

### `Workspace`

```python
root: Path
logs_raw: Path
exports_ap_stuck: Path
exports_ap_detail: Path
reports_manual: Path
mca_dumps: Path
```

### `User`

```python
username: str
display_name: str
role: str
permissions: dict
email: str
github_username: str
```

## AP Status 흐름

```text
전체 조회 / 선택 조회
  → _ensure_ap_password()     # Keychain → credentials 복호화 → 세션 캐시
  → _get_efg_relay()          # EFG SSH 비밀번호 확인
  → fetch_all_ap_info()
    → fetch_ap_info() × N     # SSH relay 또는 직접 연결
    → mca-dump 파싱
    → logread 오류 패턴 집계
    → ResetScore 계산
  → 테이블 표시
```

주요 진단 지표:

| 지표 | 소스 | 의미 |
|------|------|------|
| ELK Stuck | `ELK Stuck Record` | 날짜 범위 내 Record 기반 Stuck 이벤트 수 |
| 로그오류 | `logread` | 현재 부팅 세션 오류 패턴 수 |
| DevReset | `logread` | `WAL_DBGID_DEV_RESET` 감지 수 |
| 재시작2G/5G | `athstats.ast_ath_reset` | 드라이버 리셋 누적 |
| VAP지연2G/5G | `timeout_waiting_for_vap_cnt` | VAP 초기화 지연 |
| CU2G/CU5G | `athstats.cu_total` | 채널 이용률 |
| Client2G/5G | `radio_table[*].num_sta` | 연결 클라이언트 수 |
| Ch/BW | UniFi API | 채널과 채널 폭 |
| CPU/Mem/Uptime | `mca-dump` | AP 시스템 상태 |

## AP Reset

AP Reset은 EFG relay를 통해 실행합니다.

```text
NVC NetHub → EFG SSH → sshpass + ssh → AP reboot
```

AP 직접 paramiko password 인증은 Dropbear 호환성 문제로 실패할 수 있으므로 relay 경로를 기본으로 사용합니다.

예약 실행 흐름:

```text
대상 AP 선택
  → 권한 확인
  → AP/EFG 자격증명 확인
  → 예약 팝업
  → GitHub 예약 저장
  → 순차 reboot
  → operations.log 기록
  → GitHub history 기록
```

예약 관리는 `newvisionchurch/nvc_security:ap_reset_schedule.json`을 기준으로 표시합니다.
`예약 목록`은 `pending`/`running` 상태만 보여 주고 취소할 수 있으며, `AP Reset 이력`은 완료된 AP별 실행 결과를 최신순으로 보여 줍니다.

## ELK Stuck Record 설계

`ELK Stuck Count`는 AP Status에서 raw Log를 매번 직접 스캔하지 않습니다.
동기화 탭에서 날짜별 `ELK Stuck Record`를 생성하고, AP Status는 저장된 Record를 읽어 `ELK Stuck` 컬럼을 계산합니다.

```text
NAS ELK Sync
  → runtime/logs/raw/ 로 JSONL Log 동기화
  → ELK Stuck Record 생성
  → runtime/reports/elk_stuck_record/elk_stuck_records.json 저장
  → AP Status ELK Stuck Count에서 Record 집계
```

### 집계 방식

| 방식 | 계산 |
|------|------|
| 누적 | `sum(ap_daily_counts)` |
| 최고 | `max(ap_daily_counts)` |
| 평균 | `int(sum(ap_daily_counts) / day_count)` |

## 인증 설계

모든 사용자(admin 포함)가 동일한 7단계 인증을 거칩니다. 개발자 등록 PC만 예외로 전체 skip합니다.

```text
NVC NetHub 시작
  → VPN 확인
  → dev_machines.json 등록 PC 여부 확인
  ├── 등록 PC: 자동 로그인 (인증 전체 skip)
  └── 일반 PC:
      → 1단계 VPN 접속 인증
      → 2단계 팀원 인증: GitHub ID + PAT 소유 확인
      → 3단계 로그인: ID / 비밀번호
      → 4단계 NAS SSH 인증
      → 5단계 EFG SSH 인증
      → 6단계 EFG API 인증
      → 7단계 AP SSH 인증
      → 메인 화면 표시
```

로그인 성공 시 자동 처리:

```text
_auto_sync_credentials()
  → Keychain에서 NAS/EFG/AP 비밀번호 조회
  → 없으면 credentials.json 복호화 → Keychain 복원
  → 세션 캐시 (_pw_session_cache) 및 _ap_ssh_password 저장
  → TOTP 등록 팀원 전체에 대해 credentials.json 갱신 (관리자만)
```

TOTP 등록 완료 시 자동 처리:

```text
_add_user_to_credentials()
  → 세션 캐시의 NAS/EFG/AP/EFG API 비밀번호를
    신규 사용자의 TOTP secret으로 암호화
  → 기존 credentials.json에 해당 사용자 항목 추가
```

비밀번호 해결 흐름 (`_resolve_password`):

```text
1. 세션 캐시 (_pw_session_cache)
2. Windows Keychain
3. nvc_security/credentials.json 복호화 → Keychain 복원
4. 없으면 None (팝업 없음)
```

인증 원본 기준:

| 순서 | 위치 |
|------|------|
| 1 | `newvisionchurch/nvc_security:nethub/auth/users.json` |
| 2 | NAS SFTP 경로 |
| 3 | `source/config/users.json` (로컬 백업) |

## GitHub 연동

| 데이터 | 저장소/파일 |
|--------|-------------|
| 사용자 | `newvisionchurch/nvc_security:nethub/auth/users.json` |
| SSH 접속 정보 | `newvisionchurch/nvc_security:nethub/auth/ssh_targets.json` |
| 팀 공유 비밀번호 | `newvisionchurch/nvc_security:nethub/auth/credentials.json` |
| AP Reset 예약 | `newvisionchurch/nvc_security:ap_reset_schedule.json` |
| 팀원 게시판 | `newvisionchurch/nvc_security:nethub/feedback.json` |
| 관리자 게시판 | `newvisionchurch/nvc_security:nethub/admin_notes.json` |

GitHub PAT는 Windows Keychain에 저장합니다.

## credentials.json 구조

팀 공유 접속 비밀번호(NAS/EFG/AP SSH, EFG API Key)를 팀원별로 TOTP secret으로 개별 암호화하여 저장합니다.
팀원은 본인의 TOTP secret으로만 복호화할 수 있습니다.

```json
{
  "nas":     { "<username>": { "salt": "...", "token": "..." }, ... },
  "efg":     { "<username>": { "salt": "...", "token": "..." }, ... },
  "ap":      { "<username>": { "salt": "...", "token": "..." }, ... },
  "efg_api": { "<username>": { "salt": "...", "token": "..." }, ... }
}
```

사용자 추가 트리거:
- **비밀번호 변경 시**: 관리자가 보안 설정 탭에서 저장 → 전체 TOTP 등록 사용자 재암호화
- **TOTP 등록 완료 시**: 신규 사용자 항목 자동 추가 (`_add_user_to_credentials`)

## 주요 설정

| 키 | 설명 |
|----|------|
| `workspace_root_windows` | 로컬 런타임 루트 |
| `md_base_path` | NVC NetHub 문서 버튼이 읽는 공개 MD 경로 |
| `auth.github_org` | 인증 데이터 조직 |
| `auth.github_repo` | 인증 데이터 저장소 |
| `auth.github_file` | 사용자 파일 경로 |
| `auth.github_ssh_targets_file` | SSH 접속 정보 파일 경로 |
| `auth.github_credentials_file` | 팀 공유 비밀번호 파일 경로 |
| `nas_sync.remote_output_root` | NAS JSONL 출력 루트 |
| `nas_sync.auto_record_today_after_startup_sync` | 시작 NAS 동기화 후 오늘 Record 생성 여부 |
| `nas_sync.record_stable_confirm_count` | Log signature 안정 확인 횟수 |
| `nas_sync.sync_left_ratio_percent` | 동기화 탭 왼쪽 패널 폭 비율 |
| `vpn_check.host` / `vpn_check.port` | VPN 확인 대상 |
| `unifi_api.base_url` | UniFi Controller URL |
| `startup_actions` | 시작 시 자동 실행 항목 |
| `startup_actions.elk_count` | 시작 시 AP Status ELK Stuck Count 반영 여부 |
| `startup_actions.elk_sync_first` | ELK Count 전에 NAS Log 동기화 수행 여부 |
| `startup_actions.message_window` | 통합 메시지 창 자동 표시 여부 |
| `startup_actions.message_window_auto_close` | 시작 자동 실행 완료 후 메시지 창 자동 닫기 여부 |
| `startup_actions.message_window_auto_close_seconds` | 메시지 창 자동 닫기 대기 시간(초) |
| `elk_stuck_record.aggregate_mode` | `sum`, `max`, `avg` 중 기본 집계 방식 |
| `score_refresh` | 점수분류 Refresh 주기, 정렬, 자동 OFF 조건 |
| `stuck_thresholds` | 테이블 경고 표시 임계값 |

## 설정 파일

| 파일 | 내용 | Git |
|------|------|-----|
| `source/config/app_config.json` | 앱 설정 | 포함 |
| `source/config/ap_inventory.json` | AP 인벤토리 | 포함 |
| `source/config/ssh_targets.example.json` | SSH 예제 | 포함 |
| `source/config/dev_machines.json` | 개발자 PC 등록 | 포함 (민감 데이터 아님) |
| `source/config/users.json` | 사용자 계정 로컬 백업 | 제외 |
| `source/config/ssh_targets.json` | 실제 SSH 접속 정보 (로그인 시 자동 로드) | 제외 |
| `source/config/credentials.enc` | 암호화 자격증명 | 제외 |

## AP 인벤토리

`source/config/ap_inventory.json`은 AP 이름, IP, 모델, 위치, 상위 스위치 정보를 관리합니다.

현재 운영 기준은 `docs/efg.md`의 AP 인벤토리 표와 함께 관리합니다.

## 운영 기록

| 경로 | 내용 |
|------|------|
| `runtime/operations.log` | 사용자명 포함 작업 이력 |
| `runtime/logs/raw/` | NAS 동기화 로그 캐시 |
| `runtime/reports/elk_stuck_record/elk_stuck_records.json` | 날짜별 ELK Stuck Record |
| `runtime/exports/` | Log Export 결과 |
| `runtime/mca_dumps/` | AP mca-dump 원본 |
| `runtime/api_probe/` | UniFi API 응답 캐시 |
