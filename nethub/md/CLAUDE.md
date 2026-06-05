# NVC NetHub — Claude Code 지침

이 문서는 NVC NetHub 프로젝트에서 Claude Code가 지켜야 할 작업 기준입니다.
세부 설계와 운영 정보는 `docs/` 문서를 우선 확인합니다.

## 커밋 규칙

- 커밋 메시지에 `Co-Authored-By` 줄을 추가하지 않습니다.
- 민감 파일, 런타임 로그, 대용량 원본 로그는 커밋하지 않습니다.
- 문서 변경 후 사용자가 push를 요청하면, 커밋·push 후 `scripts/sync.ps1` 실행 여부를 확인합니다.

## 문서 동기화

`docs/` 아래 MD 파일, 루트 `README.md`, 루트 `CLAUDE.md`를 수정한 경우 공개 문서 저장소 동기화가 필요합니다.

```powershell
.\scripts\sync.ps1
```

동기화 대상은 `nvc_public/nethub/md/`입니다.

코드 변경 이후 문서 반영 대상을 점검하려면 다음 스크립트를 사용합니다.

```powershell
.\scripts\mdreport.ps1
```

이 스크립트는 마지막 MD 기준점 이후의 코드 변경을 확인하고 `runtime/reports/mdreport_change_report.md`에 AI용 변경점 리포트를 생성합니다.

사용자가 AI 작업 창에서 `mdupdate`라고 요청하면, AI는 먼저 `scripts/mdreport.ps1`을 실행해 최신 `runtime/reports/mdreport_change_report.md`를 생성합니다.
그 다음 리포트를 읽고, 리포트에 적힌 변경 파일과 확인 대상 MD를 실제 코드와 대조해 문서를 업데이트합니다.

MD 업데이트 운영 순서:

1. AI 작업 창에서 `mdupdate`라고 요청합니다.
2. AI가 `scripts/mdreport.ps1`을 실행해 AI용 변경점 리포트를 생성합니다.
3. AI가 `runtime/reports/mdreport_change_report.md`를 읽고 실제 MD를 수정합니다.
4. 사용자가 검토한 뒤 `push all`을 요청합니다.
5. push 완료 후 `scripts/sync.ps1`로 public MD를 동기화합니다.
6. 사용자가 build까지 요청한 경우 `scripts/build.ps1`로 배포 패키지를 생성합니다.
7. sync까지 완료된 뒤 다음 기준점이 필요하면 `scripts/mdreport.ps1 -MarkBaseline`을 실행합니다.

`scripts/build.ps1`은 배포 패키지 생성용입니다. 일반 문서 최신화 작업에서는 build를 실행하지 않고, 배포가 필요한 경우에만 `docs/관리자_안내.md` 절차에 따라 별도 실행합니다.
사용자가 `mdupdate push all sync build`처럼 연속 작업을 요청하면 위 순서대로 끝까지 진행합니다.

## `all` 명령 — 전체 작업 순서

사용자가 `all`이라고 요청하면 아래 순서를 끝까지 자동으로 진행합니다.
각 단계는 이전 단계가 성공한 경우에만 계속합니다.
비밀번호가 필요한 단계(backup)는 사용자에게 확인 후 진행합니다.

| 단계 | 명령 | 설명 |
|------|------|------|
| 1 | `.\scripts\mdreport.ps1` | 코드 변경 이후 MD 업데이트 대상 리포트 생성 |
| 2 | mdupdate | 리포트를 읽고 실제 MD 파일 업데이트 |
| 3 | push all | 4개 저장소 변경사항 커밋 및 push |
| 4 | `.\scripts\sync.ps1` | public MD 동기화 (nvc_public) |
| 5 | `.\scripts\build.ps1` | Windows exe 빌드 + SSH로 Mac 빌드 + nvc_release 통합 push |
| 6 | `git tag v{버전} && git push origin v{버전}` | 현재 버전으로 git tag 생성 및 push (버전은 GUI_VERSION에서 읽음) |
| 7 | `.\scripts\mdreport.ps1 -MarkBaseline` | 현재 HEAD를 다음 MD 기준점으로 저장 |
| 8 | `.\scripts\backup.ps1` | nvc_security + nvc_nethub 암호화 백업 (비밀번호 자동) |

> tag 단계에서 이미 같은 버전 태그가 존재하면 건너뜁니다.
> build.ps1은 SSH로 교회 Mac(192.168.15.171)에 자동 접속해 Mac 빌드를 실행하고 SCP로 결과물을 수집합니다.

## Push 규칙

사용자가 `push` 또는 `push all`을 요청하면 아래 4개 저장소를 확인합니다.
변경사항이 있는 저장소만 커밋 후 push하고, 변경이 없으면 건너뜁니다.

| 저장소 | 경로 | Remote |
|--------|------|--------|
| `nvc_nethub` | `C:\Projects\nvc_nethub` | `origin/main` |
| `nvc_public` | `C:\Projects\nvc_public` | `origin/main` |
| `nvc_security` | `C:\Projects\nvc_security` | `origin/main` |
| `nvc_release` | `C:\Projects\nvc_release` | `origin/main` |

`nvc_nethub` push 시 `.claude/` 하위 변경도 확인하되, 로컬 민감 경로나 백업 폴더 경로가 포함되면 커밋하지 않습니다.

### 로컬 전용 경로 (Git 저장소 아님 — 절대 커밋/push 대상 아님)

| 경로 | 용도 |
|------|------|
| `C:\Projects\nvc_backup\` | 로컬 암호화 백업 전용 폴더 — git 저장소 아님 |
| `C:\Projects\nvc_backup\nvc_nethub\` | `scripts/backup.ps1` 결과물 저장 위치 |

`nvc_backup`은 git init이 되어 있지 않은 순수 로컬 폴더입니다. push 대상에 포함하지 않습니다.

## Mac 빌드 워크플로

Mac 빌드는 `nvc_mac` 저장소를 통해 별도로 관리합니다.

| 저장소 | 경로 | Remote |
|--------|------|--------|
| `nvc_mac` | `C:\Projects\nvc_mac` | `newvisionchurch/nvc_mac` (private) |

### Win → Mac 소스 동기화

소스 변경 후 Mac에 반영할 때 실행합니다.

```powershell
.\scripts\macsync.ps1
```

동기화 대상: `source/` 전체, `scripts/run.sh`
동기화 후 `nvc_mac`에서 커밋/push → Mac에서 `git pull`

### Mac 빌드 순서

1. Win에서 `.\scripts\macsync.ps1` 실행
2. `nvc_mac` 커밋/push
3. Mac에서 `git pull`
4. Mac VS Code에서 `bash scripts/run.sh` 로 동작 검증
5. Mac에서 `bash scripts/build_mac.sh` 로 PyInstaller 빌드
6. Mac에서 `dist/NVC_NetHub_X.X_mac` 실행 파일 검증
7. Mac에서 빌드 결과물 `nvc_mac` push
8. Win에서 `nvc_release`에 Windows + Mac 바이너리 통합 릴리즈

### macsync 후 nvc_mac push

`macsync.ps1` 실행 후 `nvc_mac` 변경사항을 push할 때는 `push all` 대상에 포함되지 않으므로 별도로 처리합니다.

```powershell
cd C:\Projects\nvc_mac
git add .
git commit -m "sync: v{버전} 소스 반영"
git push origin main
```

## 프로젝트 구조

```text
nvc_nethub/
├── README.md
├── CLAUDE.md
├── source/               # NVC NetHub 소스
│   ├── main.py
│   ├── modules/
│   ├── config/
│   ├── assets/
│   └── tools/
├── elk/                  # NAS ELK Stack 설정
├── docs/                 # 운영 및 개발 문서
├── scripts/              # 실행, 빌드, 동기화 스크립트
├── runtime/              # 로컬 런타임 데이터, Git 제외
└── backup/               # 로컬 보관본, Git 제외
```

## 작업 전 확인

### NVC NetHub 작업

1. `source/main.py`
2. `source/modules/models.py`
3. `source/config/app_config.json`
4. `docs/nethub.md`
5. 관련 모듈 파일

### ELK 작업

1. `docs/elk.md`
2. `elk/logstash/pipeline/logstash.conf`

### EFG/AP 기준 정보 작업

1. `docs/efg.md`
2. `source/config/ap_inventory.json`

## 핵심 제약

### NVC NetHub

- Python/Tkinter 기반 로컬 관리 도구입니다.
- 기능 변경은 안정적인 작은 단위로 진행합니다.
- 사용자 계정과 SSH 접속 정보는 Git에 포함하지 않습니다.
- 실제 인증 원본은 `newvisionchurch/nvc_security`의 `nethub/auth/users.json` 기준입니다.
- 역할명은 `admin`, `manager`, `member`, `guest` 기준입니다.

### 인증 원칙

- **인증 단계는 admin 포함 모든 사용자 100% 동일**: VPN → 팀원 인증(PAT) → 로그인(ID/PW+OTP) → SSH 인증
- **개발자 모드만 예외**: `dev_machines.json` 등록 PC는 인증 전체 skip
- admin도 인증 단계에서 특권 없음 — 로그인 후 `can_manage_users` 권한으로만 구분
- **로그인 후 관리자 전용 기능**: 설정 → 보안 탭, 사용자 탭 (nvc_security 관리)
- **인증 데이터는 메모리에만**: users/credentials/ssh_targets는 세션 중 메모리 전용, 종료 시 삭제
- dev_machines.json만 로컬 파일로 유지 (자동 로그인 설정, 민감 데이터 아님)

### ELK

- `input`, 기본 syslog grok, observer/dataset, output 구조는 보호 구간입니다.
- Logstash 변경은 기존 블록을 크게 바꾸지 않고 작은 delta로 작성합니다.
- 새 분류 규칙은 기존 동작을 깨지 않도록 `unclassified` guard를 유지합니다.

### 보안

- `.env`, 비밀번호, PAT, API Key, private key는 절대 커밋하지 않습니다.
- `source/config/users.json`, `source/config/ssh_targets.json`, `source/config/credentials.enc`는 Git 제외 대상입니다.
- `runtime/`, `backup/`, `nvc_nethub_secure_backup/`는 Git 제외 대상입니다.

## 문서 작성 기준

- 사용자-facing 명칭은 `NVC NetHub`로 통일합니다.
- 고정 애플리케이션 버전은 일반 문서 본문에 반복 기재하지 않습니다.
- 실제 버전 확인이 필요하면 `source/main.py`의 버전 상수 또는 Git tag를 확인합니다.
- 경로, 설정 키, 파일명은 백틱으로 감쌉니다.
- 중복 설명은 줄이고 문서 역할을 분리합니다.

## 주요 문서 역할

| 문서 | 역할 |
|------|------|
| `README.md` | 프로젝트 개요와 빠른 시작 |
| `CLAUDE.md` | Claude Code 작업 기준 |
| `docs/nethub.md` | NVC NetHub 설계 참조 |
| `docs/manual.md` | 사용자 매뉴얼 |
| `docs/elk.md` | ELK 운영 및 Logstash 작업 지침 |
| `docs/efg.md` | EFG/AP 기준 정보 |
| `docs/관리자_안내.md` | 관리자 배포 및 팀원 등록 안내 |
| `docs/사용자_안내.md` | 사용자 설치 및 첫 로그인 안내 |
| `docs/aiguide.md` | 저장소 및 AI 협업 구조 |

## Codex 문서 확인 순서

Codex는 매 턴 모든 MD를 자동으로 읽는다고 가정하지 않습니다.
작업을 시작할 때는 요청 성격에 따라 아래 문서를 우선 확인합니다.

| 작업 유형 | 우선 확인 문서 |
|-----------|----------------|
| 전체 프로젝트 파악 | `README.md`, `CLAUDE.md` |
| NVC NetHub 코드 작업 | `CLAUDE.md`, `docs/nethub.md`, 관련 `source/` 파일 |
| ELK / Logstash 작업 | `CLAUDE.md`, `docs/elk.md`, `elk/logstash/pipeline/logstash.conf` |
| EFG/AP 기준 정보 작업 | `docs/efg.md`, `source/config/ap_inventory.json` |
| 사용자 매뉴얼 작업 | `docs/manual.md`, 실제 화면 코드 |
| 배포 작업 | `docs/관리자_안내.md`, `docs/사용자_안내.md`, `scripts/build.ps1`, `scripts/sync.ps1` |
| 저장소 / push / sync 작업 | `CLAUDE.md`, `docs/aiguide.md` |
| `mdupdate` 요청 | `scripts/mdreport.ps1`, `runtime/reports/mdreport_change_report.md` |
| `all` 요청 | `CLAUDE.md` — `all` 명령 순서 참조 |

반드시 지켜야 하는 작업 규칙은 `CLAUDE.md`에 기록합니다.
