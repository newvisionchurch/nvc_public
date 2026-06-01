# NVC Network 프로젝트 — Claude Code 지침

## 커밋 규칙

커밋 메시지에 `Co-Authored-By` 줄을 추가하지 않는다.

## Push 규칙

사용자가 "push all" 또는 "push"를 요청하면 아래 4개 저장소를 모두 push한다.

| 저장소 | 경로 | Remote |
|--------|------|--------|
| nvc_nethub | `c:/Projects/nvc_nethub` | origin/main |
| nvc_public | `c:/Projects/nvc_public` | origin/main |
| nvc_security | `c:/Projects/nvc_security` | origin/main |
| nvc_release | `c:/Projects/nvc_release` | origin/main |

각 저장소에 변경사항이 있으면 커밋 후 push, 없으면 건너뛴다.

`nvc_nethub` push 시 `.claude/settings.json`, `.claude/nvc_nethub.code-workspace` 등 `.claude/` 하위 파일도 함께 스테이징하여 커밋에 포함한다.

---

## 프로젝트 구조

```
nvc_nethub/
├── CLAUDE.md             ← AI 지침 (이 파일)
├── README.md             ← GitHub repo 소개
├── .claude/              ← Claude Code 설정
├── .vscode/
├── .gitignore
│
├── source/               ← nethub GUI (Python/Tkinter)
│   ├── main.py           ← 진입점 (NetworkGuiApp 클래스) — GUI_VERSION = "Version 0.3  05/30/2026"
│   ├── modules/          ← 기능 모듈 (아래 모듈 구조 참조)
│   ├── config/           ← 런타임 설정 (일부 gitignore)
│   │   ├── app_config.json     ← git 포함
│   │   ├── ap_inventory.json   ← git 포함 (AP 29대)
│   │   ├── ssh_targets.example.json ← git 포함
│   │   ├── users.json          ← git 제외
│   │   └── ssh_targets.json    ← git 제외
│   ├── assets/           ← 이미지 리소스
│   ├── tools/            ← 빌드 도구
│   └── requirements.txt
│
├── elk/                  ← NAS ELK Stack
│   ├── compose.yaml      ← Docker Compose (ES + Kibana + Logstash)
│   ├── logstash/pipeline/logstash.conf  ← 활성 파이프라인 (V27)
│   └── source/           ← 구버전 conf 히스토리
│
├── docs/                 ← AI context 문서 (자동 로드)
│   ├── aiguide.md        ← AI 협업 구조
│   ├── elk.md            ← ELK 작업 지침 (V27 베이스라인, RCA, 검증)
│   ├── nethub.md         ← nethub SW 설계
│   ├── efg.md            ← EFG 장비 참조
│   └── manual.md         ← 사용자/배포 매뉴얼
│
├── scripts/              ← PowerShell 스크립트
│   ├── run.ps1           ← GUI 실행 (venv 자동 생성)
│   └── sync.ps1          ← nvc_nethub → nvc_public MD 동기화
│
└── runtime/              ← 런타임 데이터 (git 제외)
    ├── logs/raw/         ← NAS 동기화 JSONL 캐시
    ├── exports/          ← Log Export 결과
    ├── mca_dumps/        ← AP mca-dump 원본
    └── operations.log
```

---

## 작업 전 필수 확인

### ELK 작업 시
1. `docs/elk.md` — V27 베이스라인, 파이프라인 섹션 맵, RCA 모델, 감지 키워드, 보호구간, 검증 체크리스트
2. `elk/logstash/pipeline/logstash.conf` — 활성 파이프라인 코드

### nethub (SW) 작업 시
1. `source/modules/models.py` — 핵심 데이터 모델
2. `source/main.py` — NetworkGuiApp 클래스 (탭 빌드 메서드 포함)
3. `source/config/app_config.json` — 앱 설정
4. 상세 설계/모듈 참조: `docs/nethub.md`

---

## 핵심 제약사항

### ELK
- `input`, `output`, 기본 syslog grok(섹션 2), observer/dataset(섹션 4) — **보호된 baseline, 수정 금지**
- 변경은 반드시 소규모 델타 (new `if` 블록 추가, 기존 블록 수정 금지)
- 한 번에 1개 기능만 추가

### nethub SW
- Python + Tkinter (로컬 PC 도구, 웹 아님)
- 안정적인 변경만, 한 번에 하나씩
- `source/config/users.json` — git 제외 (bcrypt 해시, TOTP secret)
- `source/config/ssh_targets.json` — git 제외 (실제 SSH 정보)
- 런타임 데이터: `C:\Projects\nvc_nethub\runtime\` (`workspace_root_windows` in app_config.json)

### 공통
- 비밀번호, API 키, `.env` 절대 커밋 금지
- 대용량 로그 파일 커밋 금지

---

## ELK 현재 상태 (V27)

| 항목 | 값 |
|------|----|
| 버전 | ELK 7.17.10 (Docker Compose, NAS) |
| 입력 | UDP `51415` |
| 인덱스 | `unifi-network-v27-{ap,low,noise}-*` |
| JSONL 출력 | NAS `logstash/outputs/{ap,low,noise}/YYYY/M/D/partNNNN` |
| 파이프라인 | `elk/logstash/pipeline/logstash.conf` |

---

## nethub SW 현재 상태 (v0.3)

버전: `GUI_VERSION = "Version 0.3  05/30/2026"` (source/main.py)

### 탭 구성 및 권한

| 탭 | 서브탭 | 빌드 메서드 | 필요 권한 |
|---|---|---|---|
| AP Status | — | `_build_ap_remote_tab()` | 모든 사용자 |
| EFG Remote | EFG SSH | `_build_efg_remote_tab()` | `can_view_efg_tab` |
| EFG Remote | EFG API | `_build_unifi_api_tab()` | `can_view_efg_tab` |
| Log Export | — | `_build_export_tab()` | `can_export` |
| 동기화 | NAS ELK Sync | `_build_sync_tab()` | `can_sync_nas` |
| 동기화 | 연결 현황 | `_build_settings_connection_tab()` | — |
| 설정 | 일반/자동화/보안/사용자 | `_build_settings_*_tab()` | (사용자: `can_manage_users`) |

### 역할 및 권한

| 권한 | admin | operator | viewer |
|------|:---:|:---:|:---:|
| `can_reset_ap` | ✓ | 설정에 따름 | ✗ |
| `can_sync_nas` | ✓ | ✓ | ✗ |
| `can_export` | ✓ | ✓ | ✓ |
| `can_view_efg_tab` | ✓ | ✓ | ✓ |
| `can_manage_users` | ✓ | ✗ | ✗ |

### 모듈 구조 (source/modules/)

| 모듈 | 역할 |
|------|------|
| `models.py` | 핵심 데이터 모델 (AccessPoint, StuckSummary, Workspace, SyncSettings, User) |
| `constants.py` | 타임아웃, UI 색상, AP 모델 분류, 병렬 조회 상한 |
| `auth.py` | bcrypt 로그인 + TOTP 2FA + 3단계 인증(GitHub→NAS→Local) + admin 해시 격리 |
| `vpn_check.py` | TCP 소켓으로 VPN 연결 확인 (192.168.11.1:22) |
| `nas_sync.py` | paramiko SFTP — NAS JSONL 로컬 캐시 동기화 (차분) |
| `ap_count.py` | 날짜 범위별 AP Stuck 이벤트 집계 |
| `ap_detail.py` | 문제 AP 상세 분석, Stuck 유형 분류 |
| `ap_reset.py` | SSH 원격 reboot — EFG relay 경유 |
| `ap_remote.py` | AP SSH 스캔 — mca-dump 파싱 + logread 진단 + ResetScore |
| `efg_remote.py` | EFG 대시보드 — system/네트워크/트래픽/ARP 조회 |
| `unifi_client.py` | UniFi Local API 클라이언트 (New API Key / Legacy) |
| `log_export.py` | 조건별 로그 필터링 후 파일 저장 |
| `ap_inventory.py` | AP 목록 관리 (ap_inventory.json 기반) |
| `local_storage.py` | 런타임 폴더 생성·관리, app_config.json 로드 |
| `report_writer.py` | operations.log에 사용자명·작업 기록 |
| `keychain.py` | Windows Credential Manager (keyring 래퍼) |
| `ssh_client.py` | paramiko 공통 — 패스워드/키 인증, PTY, relay |
| `font_utils.py` | OS 폰트 감지 및 UI 전체 폰트 적용 |
| `github_auth.py` | GitHub API HTTPS + PAT 인증, 파일 R/W |
| `github_schedule.py` | AP Reset 예약 CRUD (nvc-auth/ap_reset_schedule.json) |
| `github_feedback.py` | 팀원 게시판 CRUD (nvc-auth/feedback.json) |
| `diagram_viewer.py` | Block Diagram 11장 슬라이드 팝업 뷰어 |
| `credentials.py` | NAS 자격증명 Fernet 암호화 저장 |

### AP Status 탭 — Stuck 판단 기준

| Stuck 카운트 | 상태 색상 |
|-------------|----------|
| 0 | Healthy (Green) |
| 1~2 | Warning (Yellow) |
| 3~9 | Suspect (Orange) |
| 10+ | Critical (Red) |

임계값: `app_config.json` → `ap_stuck.status_thresholds`

### AP Status 탭 — 진단 컬럼

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

임계값 초과 시 셀 앞에 `⚠` 표시. `app_config.json` → `stuck_thresholds`에 저장.

### AP Reset — EFG Relay 구조

```
GUI (paramiko) → EFG (OpenSSH) → sshpass → AP (Dropbear)
```

- `ssh_client.py` → `run_ssh_via_relay()` 사용
- EFG에 `sshpass` 설치 필요
- AP에 직접 paramiko 패스워드 인증 불가 (Dropbear 호환성 문제)

### SSH 접속 구조

| 대상 | 방식 | PTY |
|------|------|-----|
| NAS (Synology) | paramiko SSHClient, 키 인증 | 불필요 |
| EFG (OpenSSH) | paramiko Transport, 패스워드 인증 | 불필요 |
| AP (Dropbear) 직접 | paramiko Transport, 패스워드 인증 | **필요** |
| AP via EFG relay | EFG에서 sshpass+ssh | 불필요 |

### 주요 설정 파일

| 파일 | 내용 | git |
|------|------|-----|
| `source/config/app_config.json` | workspace_root, nas_sync, vpn_check, stuck_thresholds, unifi_api, startup_actions | ✅ |
| `source/config/ap_inventory.json` | AP 29대 목록 (id, name, ip, model, site, floor, location, keywords) | ✅ |
| `source/config/ssh_targets.example.json` | SSH 설정 예제 | ✅ |
| `source/config/users.json` | 사용자 계정 (bcrypt 해시, TOTP secret) | ❌ |
| `source/config/ssh_targets.json` | NAS/EFG/AP SSH 접속 정보 | ❌ |

### GitHub 연동

- Org/Repo: `newvisionchurch/nvc-auth`
- PAT: Windows Keychain (`keyring`) — `KEYCHAIN_SERVICE = "nvc_auth_github"`
- 파일: `ap_reset_schedule.json` (예약), `feedback.json` (게시판)
- `users.json` 원본도 nvc-auth에 보관 (3단계 인증 1순위)

### 인증 흐름 (3단계)

```
1순위: GitHub (nvc-auth/users.json)
2순위: NAS (SFTP 경유 users.json)
3순위: Local (source/config/users.json)
```

로드 성공 시 로컬 자동 백업. 관리자 등록 PC가 아니면 admin `password_hash`·`totp_secret` 제거 후 저장.

### 실행 방법

```powershell
# scripts/run.ps1 — venv 자동 생성 후 source/main.py 실행
.\scripts\run.ps1
```
