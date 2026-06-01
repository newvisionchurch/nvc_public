# NVC NetHub — AI 협업 구조

이 문서는 NVC NetHub 프로젝트의 저장소 역할, AI 작업 범위, 문서 동기화 기준을 정리합니다.

## 저장소 구조

| 저장소 | 접근 | 용도 |
|--------|------|------|
| `newvisionchurch/nvc_nethub` | Private | 메인 코드, ELK 설정, 운영 문서 |
| `newvisionchurch/nvc_security` | Private | 인증 데이터, AP Reset 예약, 게시판 데이터 |
| `newvisionchurch-it/nvc_release` | Private | 배포 패키지 |
| `newvisionchurch/nvc_public` | Public | 공개 MD 문서 사본 |

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
| `nethub/users.json` | 사용자 계정, bcrypt hash, TOTP secret | `auth.py` |
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
        ├── windows/
        └── source/
```

정확한 배포 버전은 Git tag와 배포 폴더명을 기준으로 확인합니다.

## 공개 문서 저장소

`scripts/sync.ps1`은 메인 저장소의 MD 문서를 `nvc_public/nethub/md/`로 복사하고 push합니다.

NVC NetHub의 문서 버튼은 `source/config/app_config.json`의 `md_base_path`를 기준으로 문서를 읽습니다.

```json
"md_base_path": "C:\\Projects\\nvc_public\\nethub\\md"
```

## 문서 자동 로드

AI 작업 시 우선 확인할 문서 구조입니다.

```text
CLAUDE.md
├── docs/nethub.md
├── docs/elk.md
├── docs/efg.md
├── docs/manual.md
├── docs/release.md
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
| 배포 | `docs/release.md` |
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

## Push 워크플로우

사용자가 `push` 또는 `push all`을 요청하면 다음 순서로 진행합니다.

1. 4개 저장소의 `git status` 확인
2. 변경 있는 저장소만 커밋
3. 메인 저장소 문서 변경이 있으면 `scripts/sync.ps1` 필요 여부 확인
4. 각 저장소 push
5. 최종 상태 확인

## MD 리포트와 업데이트 워크플로우

코드 변경 이후 어떤 문서를 확인해야 하는지 보려면 `scripts/mdreport.ps1`을 사용합니다.

```powershell
.\scripts\mdreport.ps1
```

운영 순서:

1. `mdreport.ps1`이 마지막 MD 기준점 이후의 변경을 보고 AI용 리포트를 생성합니다.
2. AI 작업 창에서 `mdupdate`라고 요청합니다.
3. AI는 `runtime/reports/mdreport_change_report.md`를 읽고 실제 MD를 수정합니다.
4. 사용자가 변경 내용을 검토한 뒤 `push all`을 요청합니다.
5. push 완료 후 `scripts/sync.ps1`로 public MD를 동기화합니다.

현재 HEAD를 새 기준점으로 저장하려면 다음을 실행합니다.

```powershell
.\scripts\mdreport.ps1 -MarkBaseline
```
