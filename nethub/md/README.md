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
| AP Status | AP SSH 진단, ELK Stuck 집계, ResetScore 분류, AP Reset 예약 |
| EFG Remote | EFG SSH 대시보드, UniFi Controller API 탐색 |
| Log Export | 날짜, AP, 유형, 키워드 조건으로 로그 저장 |
| 동기화 | NAS JSONL 로그를 PC 로컬 캐시로 동기화 |
| 설정 | 자동화, 보안, 사용자, 연결 상태 관리 |
| 게시판 | 팀원 게시판, 관리자 게시판 |

## 실행

VPN 연결 후 프로젝트 루트에서 실행합니다.

```powershell
cd C:\Projects\nvc_nethub
.\scripts\run.ps1
```

## 런타임 데이터

`runtime/`은 Git에 포함하지 않습니다.

```text
runtime/
├── logs/raw/      # NAS 동기화 JSONL 캐시
├── exports/       # Log Export 결과
├── mca_dumps/     # AP mca-dump 원본
├── api_probe/     # UniFi API probe 캐시
└── operations.log # 운영 기록
```

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
| `docs/release.md` | 배포 절차 |
| `docs/aiguide.md` | AI 협업 및 저장소 운영 기준 |

## MD 리포트와 업데이트

코드 변경 이후 MD를 최신화할 때는 아래 순서로 진행합니다.

1. AI용 변경점 리포트를 만듭니다.

```powershell
.\scripts\mdreport.ps1
```

2. AI 작업 창에서 `mdupdate`라고 요청합니다.
   AI는 `runtime/reports/mdreport_change_report.md`를 먼저 읽고 필요한 MD를 수정합니다.
3. 변경 내용을 검토한 뒤 `push all`을 요청합니다.
4. push 완료 후 `scripts/sync.ps1`로 public MD를 동기화합니다.
