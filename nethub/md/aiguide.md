# NVC NetHub — AI 협업 구조

이 문서는 NVC NetHub 프로젝트의 저장소 역할, AI 작업 범위, 문서 동기화 기준을 정리합니다.

## 저장소 구조

| 저장소 | 접근 | 용도 |
|--------|------|------|
| `newvisionchurch/nvc_nethub` | Private | 메인 코드, ELK 설정, 운영 문서 |
| `newvisionchurch/nvc_security` | Private | 인증 데이터, AP Reset 예약, 게시판 데이터 |
| `newvisionchurch-it/nvc_release` | Private | 배포 패키지 (Windows + macOS) |
| `newvisionchurch/nvc_public` | Public | 공개 MD 문서 사본 |
| `newvisionchurch/nvc_mac` | Private | macOS 빌드 워크스페이스 (임시 검증용) |

## 작업 범위

Claude Code는 이 저장소에서 다음 작업을 담당합니다.

- `source/` NVC NetHub 코드 작성 및 수정
- `elk/logstash/pipeline/logstash.conf` Logstash 파이프라인 수정
- `docs/`, `README.md`, `CLAUDE.md` 문서 정리
- `source/config/ap_inventory.json` AP 인벤토리 관리
- `scripts/` PowerShell 스크립트 작성
- 요청 시 4개 저장소 상태 확인, 커밋, push

## 메인 저장소

```text
nvc_nethub/
├── source/       # NVC NetHub 소스
├── elk/          # ELK 설정
├── docs/         # 운영 및 개발 문서
├── scripts/      # 실행, 빌드, 문서 동기화
├── README.md
└── CLAUDE.md
```

Git 제외 대상:

- `source/config/users.json`
- `source/config/ssh_targets.json`
- `source/config/credentials.enc`
- `runtime/`
- `backup/`
- `nvc_nethub_secure_backup/`

## 인증 및 팀 공유 데이터

NVC NetHub는 GitHub API와 PAT를 사용해 `newvisionchurch/nvc_security`의 데이터를 읽고 씁니다.

| 파일 | 내용 | 사용 모듈 |
|------|------|----------|
| `nethub/auth/users.json` | 사용자 계정, bcrypt hash, TOTP secret | `auth.py` |
| `nethub/auth/ssh_targets.json` | NAS/EFG/AP SSH 접속 정보 | `github_auth.py` |
| `nethub/auth/credentials.json` | 팀 공유 비밀번호 (사용자별 TOTP 암호화) | `credentials.py` |
| `ap_reset_schedule.json` | AP Reset 예약 및 이력 | `github_schedule.py` |
| `nethub/feedback.json` | 팀원 게시판 | `github_feedback.py` |
| `nethub/admin_notes.json` | 관리자 게시판 | `github_feedback.py` |

PAT는 Windows Keychain에 저장합니다.

## 배포 저장소

`newvisionchurch-it/nvc_release`는 팀원 배포 패키지를 보관합니다.

```text
nvc_release/
└── nethub/
    └── vX.X/
        ├── windows/   ← Windows exe
        ├── mac/        ← macOS 실행 파일
        └── source/     ← 소스 패키지
```

정확한 배포 버전은 Git tag와 배포 폴더명을 기준으로 확인합니다.

## 공개 문서 저장소

`scripts/sync.ps1`은 메인 저장소의 MD 문서를 `nvc_public/nethub/md/`로 복사하고 push합니다.

NVC NetHub의 문서 버튼(NetHub 매뉴얼, NetHub 개요 등)은 `nvc_public` GitHub raw URL에서 직접 문서를 읽습니다.

```
https://raw.githubusercontent.com/newvisionchurch/nvc_public/main/nethub/md/{filename}
```

로컬 경로(`md_base_path`)는 사용하지 않습니다. 팀원 PC나 Mac에 `nvc_public` 클론이 없어도 문서가 정상 표시됩니다.

## 문서 자동 로드

AI 작업 시 우선 확인할 문서 구조입니다.

```text
CLAUDE.md
├── docs/nethub.md
├── docs/elk.md
├── docs/efg.md
├── docs/manual.md
├── docs/관리자_안내.md
├── docs/사용자_안내.md
└── docs/aiguide.md
```

Codex는 모든 MD를 매 턴 자동으로 읽는다고 가정하지 않습니다.
작업 성격에 맞게 `CLAUDE.md`와 관련 `docs/*.md`를 먼저 확인한 뒤, 실제 코드와 설정 파일을 대조합니다.

| 작업 유형 | 우선 확인 문서 |
|-----------|----------------|
| NVC NetHub 코드 | `CLAUDE.md`, `docs/nethub.md` |
| ELK | `CLAUDE.md`, `docs/elk.md` |
| EFG/AP 기준 | `docs/efg.md` |
| 사용자 안내 | `docs/manual.md` |
| 배포 | `docs/관리자_안내.md`, `docs/사용자_안내.md` |
| 저장소 운영 | `CLAUDE.md`, `docs/aiguide.md` |
| MD 리포트 반영 | `runtime/reports/mdreport_change_report.md` |

## 보안 원칙

| 항목 | Private | Public |
|------|---------|--------|
| 코드 | `nvc_nethub` | 제외 |
| 공개용 MD | `nvc_nethub` 원본 | `nvc_public` 사본 |
| 인증 데이터 | `nvc_security` | 제외 |
| 비밀번호/API Key/PAT | 제외 | 제외 |
| 런타임 로그 | 제외 | 제외 |

## 로컬 전용 경로 (Git 저장소 아님)

| 경로 | 용도 |
|------|------|
| `C:\Projects\nvc_backup\` | 로컬 암호화 백업 전용 — git init 없음 |
| `C:\Projects\nvc_backup\nvc_nethub\v{버전}\` | `scripts/backup.ps1` 결과물 위치 |

`nvc_backup`은 push 대상이 아닙니다. `scripts/backup.ps1`로만 관리합니다.

## `all` 명령 전체 워크플로우

사용자가 `all`이라고 요청하면 아래 순서를 순차적으로 진행합니다.

| 단계 | 작업 |
|------|------|
| 1 | `.\scripts\mdreport.ps1` — 변경 리포트 생성 |
| 2 | mdupdate — MD 파일 업데이트 |
| 3 | push all — 4개 저장소 push |
| 4 | `.\scripts\sync.ps1` — public MD 동기화 |
| 5 | `.\scripts\build.ps1` — Windows + macOS 빌드 및 nvc_release push |
| 6 | git tag 생성 및 push |
| 7 | `.\scripts\mdreport.ps1 -MarkBaseline` — 기준점 저장 |
| 8 | `.\scripts\backup.ps1` — 암호화 백업 (비밀번호 자동) |

`build.ps1`은 Windows exe 빌드 후 SSH로 교회 Mac에 자동 빌드를 실행하고, SCP로 결과물을 수집해 `nvc_release`에 통합합니다.

## Push 워크플로우

사용자가 `push` 또는 `push all`을 요청하면 다음 순서로 진행합니다.

1. 4개 저장소의 `git status` 확인
2. 변경 있는 저장소만 커밋
3. 메인 저장소 문서 변경이 있으면 `scripts/sync.ps1` 필요 여부 확인
4. 각 저장소 push
5. 최종 상태 확인

## MD 리포트와 업데이트 워크플로우

코드 변경 이후 문서를 최신화하려면 AI 작업 창에서 `mdupdate`라고 요청합니다.
AI는 먼저 `scripts/mdreport.ps1`을 실행해 어떤 문서를 확인해야 하는지 리포트를 갱신합니다.

운영 순서:

1. AI 작업 창에서 `mdupdate`라고 요청합니다.
2. AI가 `scripts/mdreport.ps1`을 실행해 마지막 MD 기준점 이후의 변경 리포트를 생성합니다.
3. AI는 `runtime/reports/mdreport_change_report.md`를 읽고 실제 MD를 수정합니다.
4. 사용자가 변경 내용을 검토한 뒤 `push all`을 요청합니다.
5. push 완료 후 `scripts/sync.ps1`로 public MD를 동기화합니다.
6. 사용자가 build까지 요청한 경우 `scripts/build.ps1`로 배포 패키지를 생성합니다.
7. sync까지 완료된 뒤 다음 기준점이 필요하면 `scripts/mdreport.ps1 -MarkBaseline`을 실행합니다.

`scripts/build.ps1`은 배포 패키지를 만들고 `nvc_release`까지 반영하는 release 단계입니다.
일반 MD 최신화 작업에서는 build를 실행하지 않고, 문서 변경 검토와 public MD 동기화까지만 진행합니다.

현재 HEAD를 새 기준점으로 저장하려면 다음을 실행합니다.

```powershell
.\scripts\mdreport.ps1 -MarkBaseline
```

## ELK Stuck 문서 반영 기준

ELK Stuck 관련 코드가 변경되면 `README.md`, `docs/nethub.md`, `docs/manual.md`를 우선 대조합니다.

| 코드/설정 | 문서 반영 포인트 |
|-----------|------------------|
| `source/modules/elk_stuck_record.py` | Record 저장 경로, 집계 방식, 안정 확인 로직 |
| `source/main.py` AP Status | `ELK Stuck Count` 버튼, 집계 방식 선택, 메시지 표시 |
| `source/main.py` NAS ELK Sync | Log 동기화 패널과 `ELK Stuck Record` 패널 |
| `source/config/app_config.json` | `startup_actions`, `nas_sync`, `elk_stuck_record`, `score_refresh` |

사용자-facing 설명에서는 `ELK Stuck Count`, `ELK Stuck Record`, `Record`, `Log 동기화`의 역할을 분리해서 씁니다.
