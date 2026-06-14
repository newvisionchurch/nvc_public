# NVC NetHub

뉴비전 교회 UniFi 네트워크를 운영하기 위한 통합 관리 프로젝트입니다.
NAS ELK Stack으로 EFG/AP syslog를 수집하고, NVC NetHub에서 AP 상태 분석, 로그 조회, EFG 원격 조회, AP Reset 예약을 수행합니다.

## 구성 개요

```
EFG / AP 장비
    │  syslog UDP 51415
    ▼
NAS ELK Stack
    │  Logstash 분류 + JSONL 저장
    ▼
NVC NetHub
    │  AP 분석, Log Export, EFG 조회, AP Reset
    ▼
운영 기록 / 예약 / 팀 공유 데이터
```

## 폴더 구조

| 경로 | 내용 |
|------|------|
| `source/` | Python/Tkinter 기반 NVC NetHub 소스 |
| `elk/` | NAS Docker ELK Stack 설정 |
| `docs/` | 운영 및 개발 참고 문서 |
| `scripts/` | 실행, 문서 동기화, 빌드용 PowerShell 스크립트 |
| `runtime/` | 로컬 런타임 데이터, Git 제외 |
| `backup/` | 로컬 보관본, Git 제외 |

## 저장소 구조

| 저장소 | 접근 | 용도 |
|--------|------|------|
| `newvisionchurch/nvc_nethub` | Private | 메인 코드, ELK 설정, 문서 |
| `newvisionchurch/nvc_security` | Private | 인증 및 팀 공유 데이터 |
| `newvisionchurch-it/nvc_release` | Private | 배포 패키지 |
| `newvisionchurch/nvc_public` | Public | 공개 MD 문서 사본 |

## 주요 기능

| 영역 | 기능 |
|------|------|
| AP Status | AP SSH 진단, ELK Stuck Record 집계, ResetScore 분류, AP Reset 예약 및 이력 |
| EFG Remote | EFG SSH 대시보드, UniFi Controller API 탐색 |
| 동기화 | NAS JSONL 로그 동기화, 날짜별 ELK Stuck Record 생성 |
| Log Export | 날짜, AP, 유형, 키워드 조건으로 로그 저장 |
| 설정 | 자동화, 보안, 사용자, 연결 상태 관리 |
| 게시판 | 팀원 게시판, 관리자 게시판, 공지 팝업 |

## 실행

VPN 연결 후 프로젝트 루트에서 실행합니다.

```powershell
cd C:\Projects\nvc_nethub
.\scripts\run.ps1
```

처음 실행하거나 GitHub 인증이 실패한 경우 시작 인증 화면에서 `GitHub PAT 입력`으로 본인 GitHub ID와 PAT를 입력합니다.
PAT는 Windows Keychain에 저장되며, `newvisionchurch/nvc_security` 접근 권한과 NetHub 사용자 정보의 GitHub ID 등록이 필요합니다.

## 런타임 데이터

`runtime/`은 Git에 포함하지 않습니다.

```text
runtime/
├── logs/raw/      # NAS 동기화 JSONL 캐시
├── exports/       # Log Export 결과
├── reports/elk_stuck_record/
│                  # 날짜별 ELK Stuck Record
├── mca_dumps/     # AP mca-dump 원본
├── api_probe/     # UniFi API probe 캐시
└── operations.log # 운영 기록
```

## ELK Stuck 운영 흐름

`ELK Stuck Count`는 AP Status에서 raw Log를 매번 직접 스캔하지 않고, 동기화 탭의 `ELK Stuck Record`에 저장된 날짜별 Count를 읽어 집계합니다.

1. 동기화 탭의 `NAS ELK Sync`에서 NAS JSONL Log를 PC로 동기화합니다.
2. 같은 화면의 `ELK Stuck Record`에서 날짜별 Record를 생성합니다.
3. AP Status의 `ELK Stuck Count`에서 선택 날짜 범위와 집계 방식(`누적`, `최고`, `평균`)으로 `ELK Stuck` 컬럼을 갱신합니다.

시작 자동 실행에서 `elk_sync_first`(동기화 후 집계)가 켜져 있으면 오늘 Log 동기화, 오늘 Record 생성, AP Status 집계가 순서대로 실행됩니다. 꺼져 있으면 ELK Count가 NAS 동기화와 무관하게 독립 실행됩니다. AP 조회와 EFG 진단은 NAS 동기화 결과와 무관하게 항상 독립 실행됩니다.

## 보안 원칙

- 비밀번호, API Key, PAT, private key는 커밋하지 않습니다.
- `source/config/users.json`과 `source/config/ssh_targets.json`은 Git 제외 대상입니다.
- `runtime/`, `backup/`, 대용량 원본 로그는 Git에 포함하지 않습니다.
- 공개 문서에는 비밀번호, 토큰, 민감한 원본 로그를 기록하지 않습니다.

## 참고 문서

| 문서 | 목적 |
|------|------|
| `docs/nethub.md` | NVC NetHub 설계 참조 |
| `docs/manual.md` | 사용자 매뉴얼 |
| `docs/elk.md` | ELK 운영 및 Logstash 작업 지침 |
| `docs/efg.md` | EFG/AP 네트워크 기준 정보 |
| `docs/관리자_안내.md` | 관리자 배포 및 팀원 등록 안내 |
| `docs/사용자_안내.md` | 사용자 설치 및 첫 로그인 안내 |
| `docs/aiguide.md` | AI 협업 및 저장소 운영 기준 |

## MD 리포트와 업데이트

코드 변경 이후 MD를 최신화할 때는 아래 순서로 진행합니다.

1. AI 작업 창에서 `mdupdate`라고 요청합니다.
   AI가 `scripts/mdreport.ps1`을 먼저 실행해 `runtime/reports/mdreport_change_report.md`를 갱신한 뒤 필요한 MD를 수정합니다.
2. 변경 내용을 검토한 뒤 `push all`을 요청합니다.
3. push 완료 후 `sync`를 요청하거나 `scripts/sync.ps1`로 public MD를 동기화합니다.
4. 릴리즈 패키지가 필요하면 `build`를 요청하거나 `scripts/build.ps1`을 실행합니다.
5. 동기화까지 끝난 뒤 다음 기준점이 필요하면 `.\scripts\mdreport.ps1 -MarkBaseline`을 실행합니다.

`scripts/build.ps1`은 배포 패키지를 만들 때 사용하는 단계입니다. 일반 MD 최신화만 할 때는 `mdupdate → push all → sync` 흐름을 사용하고, 릴리즈까지 진행할 때는 `mdupdate → push all → sync → build` 순서로 처리합니다.
