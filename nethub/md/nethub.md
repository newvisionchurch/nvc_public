# NVC NetHub — 설계 참조

NVC NetHub는 뉴비전 교회 UniFi 네트워크 운영을 위한 Python/Tkinter 기반 로컬 관리 도구입니다.
AP 상태 분석, ELK 로그 집계, EFG 원격 조회, AP Reset 예약, 팀 공유 게시판을 하나의 도구에서 제공합니다.

## 아키텍처

```text
NetworkGuiApp (source/main.py)
  ├── AP Status
  │   ├── ap_remote.py      # AP SSH 진단, mca-dump, logread, ResetScore
  │   ├── ap_count.py       # raw Log 기반 날짜별 Stuck Count 계산
  │   ├── elk_stuck_record.py
  │   │                     # 날짜별 ELK Stuck Record 저장 및 집계
  │   ├── ap_detail.py      # AP 상세 분석
  │   └── ap_reset.py       # EFG relay 기반 AP reboot
  ├── EFG Remote
  │   ├── efg_remote.py     # EFG SSH 대시보드
  │   └── unifi_client.py   # UniFi API 탐색
  ├── Log Export
  │   └── log_export.py
  ├── 동기화
  │   └── nas_sync.py
  └── 설정 / 사용자 / 게시판
      ├── auth.py
      ├── keychain.py
      ├── github_auth.py
      ├── github_schedule.py
      └── github_feedback.py
```

## 탭 구성

| 탭 | 서브탭 | 주요 기능 | 권한 |
|----|--------|----------|------|
| AP Status | - | AP SSH 진단, ELK Stuck Record 집계, ResetScore, AP Reset | 모든 사용자 |
| EFG Remote | EFG SSH | EFG 시스템, 네트워크, 트래픽, ARP 조회 | `can_view_efg_tab` |
| EFG Remote | EFG API | UniFi Controller read-only API 탐색 | `can_view_efg_tab` |
| 동기화 | NAS ELK Sync | NAS JSONL 로그 동기화, ELK Stuck Record 생성/조회/삭제 | `can_sync_nas` |
| 동기화 | AP Sync | AP mca-dump 파일 관리 | 모든 사용자 |
| 동기화 | 연결 현황 | NAS, UniFi API 연결 테스트 | 모든 사용자 |
| Log Export | - | 로컬 캐시 로그 조건 검색 및 저장 | `can_export` |
| 설정 | 일반 | 폰트, 정렬, 임계값, 색상 | 모든 사용자 |
| 설정 | 자동화 | 시작 시 자동 실행 항목 | 모든 사용자 |
| 설정 | 보안 | 경로, Keychain, GitHub PAT, UniFi API Key | 모든 사용자 |
| 설정 | 사용자 | 계정, 권한, TOTP, 개발자 PC | `can_manage_users` |

## 역할 및 권한

| 역할 | 설명 | 기본 권한 |
|------|------|----------|
| `admin` | 관리자 | 모든 기능 |
| `manager` | 네트워크 담당자 | AP Reset, 동기화, Export, EFG 조회 |
| `member` | 일반 팀원 | AP 조회, Export, EFG 조회 |
| `guest` | 방문자 | 제한된 조회 |

권한은 사용자별로 `permissions`에 저장하며, 역할 템플릿은 사용자 관리 화면에서 조정할 수 있습니다.

## 주요 모듈

| 모듈 | 역할 |
|------|------|
| `models.py` | 핵심 dataclass 모델 |
| `constants.py` | 타임아웃, 색상, AP 모델 상수 |
| `auth.py` | GitHub/NAS/Local 인증, bcrypt, TOTP, 역할 권한 |
| `vpn_check.py` | VPN 연결 확인 |
| `nas_sync.py` | NAS JSONL 로그 SFTP 동기화 |
| `ap_count.py` | raw Log에서 날짜별 AP Stuck Count 계산 |
| `elk_stuck_record.py` | 날짜별 Record 저장, Log signature 확인, Record 기반 집계 |
| `ap_detail.py` | AP 상세 분석 |
| `ap_remote.py` | AP SSH 진단과 ResetScore 계산 |
| `ap_reset.py` | AP reboot 실행 |
| `efg_remote.py` | EFG SSH 대시보드 |
| `unifi_client.py` | UniFi API 클라이언트 |
| `log_export.py` | 조건별 로그 Export |
| `ap_inventory.py` | AP 인벤토리 로드 |
| `local_storage.py` | 런타임 폴더 생성 및 설정 로드 |
| `report_writer.py` | `operations.log` 기록 |
| `keychain.py` | Windows Keychain 래퍼 |
| `ssh_client.py` | paramiko 공통 SSH 처리 |
| `font_utils.py` | 폰트 설정 |
| `github_auth.py` | GitHub Contents API 파일 읽기/쓰기 |
| `github_schedule.py` | AP Reset 예약 및 이력 |
| `github_feedback.py` | 팀원/관리자 게시판 |
| `diagram_viewer.py` | 구조도 팝업 |
| `credentials.py` | NAS 저장 자격증명 암호화 |

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
```

## AP Status 흐름

```text
전체 조회 / 선택 조회
  → fetch_all_ap_info()
  → fetch_ap_info()
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
각 행을 선택하면 대상 AP 총수, 실행 시각, 결과 메시지 등 상세 내용이 하단에 표시됩니다.

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

과거 호환을 위해 `runtime/reports/manual/elk_stuck_records.json`도 읽을 수 있습니다.

### 날짜별 Record

Record는 날짜 단위로 저장되며, 각 날짜는 AP별 Count를 가집니다.
AP 순서는 `source/config/ap_inventory.json`의 `AP01`부터 `AP29` 순서를 따릅니다.

Record 목록 화면은 상단에 `누적`, `평균`, `최고` 요약 행을 먼저 표시하고, 그 아래에는 최근 날짜부터 날짜별 Count를 표시합니다.

### 집계 방식

AP Status의 `ELK Stuck Count`는 선택한 날짜 범위의 Record를 AP별로 독립 집계합니다.

| 방식 | 계산 |
|------|------|
| 누적 | `sum(ap_daily_counts)` |
| 최고 | `max(ap_daily_counts)` |
| 평균 | `int(sum(ap_daily_counts) / day_count)` |

평균은 반올림하지 않고 소수점 이하를 버립니다.
Record가 없는 날짜는 Count가 0처럼 반영될 수 있으므로, 사용자는 먼저 동기화 탭에서 Record를 생성해야 합니다.

### Log 완전성 판단

`elk_stuck_record.local_log_signature()`는 날짜별 Log 파일 상태를 요약합니다.

| 항목 | 의미 |
|------|------|
| `file_count` | 해당 날짜 Log 파일 수 |
| `total_bytes` | 전체 파일 크기 |
| `latest_mtime` | 가장 최근 수정 시각 |

같은 날짜의 signature가 여러 번 동일하면 Record를 안정 상태로 봅니다.
확인 횟수는 `nas_sync.record_stable_confirm_count`이며 기본값은 `3`입니다.
오늘 날짜는 Log가 계속 증가할 수 있으므로 다시 Count 대상이 될 수 있습니다.

PC에 해당 날짜 Log가 없으면 해당 날짜 Record도 없어야 합니다.
`purge_records_without_logs()`는 Log가 없는 날짜의 Record를 정리합니다.

## 인증 설계

```text
NVC NetHub 시작
  → VPN 확인
  → 개발자 등록 PC 여부 확인
  ├── 등록 PC: 자동 로그인
  └── 일반 PC:
      → GitHub / NAS / Local 순서로 users.json 로드
      → bcrypt 검증
      → TOTP 등록 또는 인증
      → 메인 화면 표시
```

인증 원본 기준:

| 순서 | 위치 |
|------|------|
| 1 | `newvisionchurch/nvc_security:nethub/users.json` |
| 2 | NAS SFTP 경로 |
| 3 | `source/config/users.json` |

관리자 계정의 `password_hash`와 `totp_secret`은 관리자 등록 PC가 아닌 로컬 백업에서 제거합니다.

## GitHub 연동

| 데이터 | 저장소/파일 |
|--------|-------------|
| 사용자 | `newvisionchurch/nvc_security:nethub/users.json` |
| AP Reset 예약 | `newvisionchurch/nvc_security:ap_reset_schedule.json` |
| 팀원 게시판 | `newvisionchurch/nvc_security:nethub/feedback.json` |
| 관리자 게시판 | `newvisionchurch/nvc_security:nethub/admin_notes.json` |

GitHub PAT는 Windows Keychain에 저장합니다.

## 주요 설정

| 키 | 설명 |
|----|------|
| `workspace_root_windows` | 로컬 런타임 루트 |
| `md_base_path` | NVC NetHub 문서 버튼이 읽는 공개 MD 경로 |
| `auth.github_repo` | 인증 데이터 저장소 |
| `auth.github_file` | 사용자 파일 경로 |
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
| `source/config/users.json` | 사용자 계정 로컬 백업 | 제외 |
| `source/config/ssh_targets.json` | 실제 SSH 접속 정보 | 제외 |
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
