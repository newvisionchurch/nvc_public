# NVC NetHub — 설계 참조

NVC NetHub는 뉴비전 교회 UniFi 네트워크 운영을 위한 Python/Tkinter 기반 로컬 관리 도구입니다.
AP 상태 분석, ELK 로그 집계, EFG 원격 조회, AP Reset 예약, 팀 공유 게시판을 하나의 도구에서 제공합니다.

## 아키텍처

```text
NetworkGuiApp (source/main.py)
  ├── AP Status
  │   ├── ap_remote.py      # AP SSH 진단, mca-dump, logread, ResetScore
  │   ├── ap_count.py       # ELK JSONL 캐시 Stuck 집계
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
| AP Status | - | AP SSH 진단, ELK Stuck 집계, ResetScore, AP Reset | 모든 사용자 |
| EFG Remote | EFG SSH | EFG 시스템, 네트워크, 트래픽, ARP 조회 | `can_view_efg_tab` |
| EFG Remote | EFG API | UniFi Controller read-only API 탐색 | `can_view_efg_tab` |
| Log Export | - | 로컬 캐시 로그 조건 검색 및 저장 | `can_export` |
| 동기화 | NAS ELK Sync | NAS JSONL 로그를 PC 캐시로 동기화 | `can_sync_nas` |
| 동기화 | AP Sync | AP mca-dump 파일 관리 | 모든 사용자 |
| 동기화 | 연결 현황 | NAS, UniFi API 연결 테스트 | 모든 사용자 |
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
| `ap_count.py` | 날짜 범위별 AP Stuck 집계 |
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
| ELK Stuck | 로컬 JSONL 캐시 | 날짜 범위 내 Stuck 이벤트 수 |
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
  → 순차 reboot
  → operations.log 기록
  → GitHub history 기록
```

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
| `vpn_check.host` / `vpn_check.port` | VPN 확인 대상 |
| `unifi_api.base_url` | UniFi Controller URL |
| `startup_actions` | 시작 시 자동 실행 항목 |
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
| `runtime/exports/` | Log Export 결과 |
| `runtime/mca_dumps/` | AP mca-dump 원본 |
| `runtime/api_probe/` | UniFi API 응답 캐시 |
