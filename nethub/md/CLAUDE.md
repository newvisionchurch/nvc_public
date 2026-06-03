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
6. sync까지 완료된 뒤 다음 기준점이 필요하면 `scripts/mdreport.ps1 -MarkBaseline`을 실행합니다.

`scripts/build.ps1`은 배포 패키지 생성용입니다. 일반 문서 최신화 작업에서는 build를 실행하지 않고, 배포가 필요한 경우에만 `docs/release.md` 절차에 따라 별도 실행합니다.

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
- 실제 인증 원본은 `newvisionchurch/nvc_security`의 `nethub/users.json` 기준입니다.
- 역할명은 `admin`, `manager`, `member`, `guest` 기준입니다.

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
| `docs/release.md` | 배포 절차 |
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
| 배포 작업 | `docs/release.md`, `scripts/build.ps1`, `scripts/sync.ps1` |
| 저장소 / push / sync 작업 | `CLAUDE.md`, `docs/aiguide.md` |
| `mdupdate` 요청 | `scripts/mdreport.ps1`, `runtime/reports/mdreport_change_report.md` |

반드시 지켜야 하는 작업 규칙은 `CLAUDE.md`에 기록합니다.
