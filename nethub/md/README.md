# NVC Network 관리 프로젝트

GUI Version 0.3 — 05/30/2026  
뉴비전 교회 UniFi 네트워크(AP 29대, NAS, EFG)의 로그 수집·분석·제어를 위한 통합 관리 프로젝트.

---

## 구성 개요

```
EFG / AP 장비
    │  syslog UDP 51415
    ▼
NAS ELK Stack (Logstash V27)  ← 로그 수집·분류·저장 (JSONL)
    │
    ▼
NVC NetHub (Python/Tkinter v0.3)  ← 로그 분석, 문제 AP 식별
    │
    ▼
EFG / AP 원격 제어 (SSH)  ← AP Reset, EFG 조회, UniFi API
```

| 폴더 | 내용 |
|------|------|
| `source/` | NVC NetHub SW |
| `elk/` | NAS Docker ELK Stack (compose.yaml + logstash.conf) |
| `docs/` | 프로젝트 문서 (AI context 자동 로드) |
| `scripts/` | PowerShell 유틸리티 스크립트 |
| `runtime/` | 런타임 데이터 — git 제외 (로컬 캐시, 작업 파일) |

---

## GitHub 계정 구조

| 저장소 | 접근 | 용도 |
|--------|------|------|
| `newvisionchurch/nvc_nethub` | Private | 메인 코드 (이 저장소) |
| `newvisionchurch/nvc-auth` | Private | 인증 데이터 (users.json 등) |
| `newvisionchurch-it/nvc_release` | Private | 배포 패키지 |
| `newvisionchurch/nvc_public` | Public | MD 문서 공개 사본 |

---

## ELK Stack (V27)

NAS Docker Compose ELK 7.17.10:

- 입력: UDP `51415` (EFG/AP syslog)
- 인덱스: `unifi-network-v27-{ap,low,noise}-*`
- JSONL 출력: `logstash/outputs/{ap,low,noise}/YYYY/M/D/partNNNN`

RCA 경로:
```
system_issue → qos_error → ap_stuck / ap_no_service
rrm_scan_trigger → radio_driver_retry → tx_overflow
  → radio_reset → vap_timeout → channel_invalid → service_impact
```

---

## GUI (v0.3)

로컬 Windows PC에서 실행되는 Python/Tkinter 관리 도구.

### 탭 구성

| 탭 | 기능 | 필요 권한 |
|---|---|---|
| AP Status | mca-dump/logread 진단 + ELK Stuck 집계 + ResetScore 분류 + AP Reset | 모든 사용자 |
| EFG Remote > EFG SSH | EFG 시스템 대시보드 | `can_view_efg_tab` |
| EFG Remote > EFG API | UniFi Controller API 탐색기 (read-only) | `can_view_efg_tab` |
| Log Export | 날짜·AP·유형·키워드 조건 로그 파일 저장 | `can_export` |
| 동기화 | NAS JSONL → PC 캐시 동기화 (SFTP) | `can_sync_nas` |
| 설정 | 일반/자동화/보안/사용자 관리 | (사용자: admin) |

### 인증 흐름

```
VPN 확인 (192.168.11.1:22)
  → 3단계 users.json 로드 (GitHub nvc-auth → NAS → Local)
  → bcrypt 비밀번호 검증
  → TOTP 2FA (Google Authenticator / Authy)
  → 역할(admin/operator/viewer) 기반 권한 적용
```

### 빠른 시작

```powershell
# VPN 연결 후 실행 (venv 자동 생성)
cd C:\Projects\nvc_nethub
.\scripts\run.ps1
```

---

## 런타임 데이터 (git 제외)

`C:\Projects\nvc_nethub\runtime\`:

```
runtime/
├── logs/raw/     ← NAS 동기화 JSONL 캐시
├── exports/      ← Log Export 결과
├── mca_dumps/    ← AP Remote mca-dump 원본
└── operations.log
```

---

## 주의사항

- 비밀번호, API 키, `.env` 파일 — 절대 커밋 금지
- `source/config/users.json`, `source/config/ssh_targets.json` — `.gitignore` 처리
- `runtime/` — `.gitignore` 처리
- 대용량 원본 로그 커밋 금지
