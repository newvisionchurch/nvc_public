# NVC Network 관리 프로젝트

GUI Version 0.2 — 05/29/2026  
대상: 뉴비전 교회 네트워크 운영팀

뉴비전 교회 네트워크(UniFi EFG / AP 29대 / NAS)의 로그 수집·분석·제어를 위한 통합 관리 프로젝트입니다.

---

## 문서 동기화 — nvcpush.ps1

이 프로젝트의 MD 문서는 **`nvcpush.ps1`** 을 통해 `nvc_public` 저장소로 복제됩니다.  
GUI 상단 오른쪽의 **프로젝트 개요 / GUI 개요 / GUI 매뉴얼** 버튼은 복제된 파일을 직접 읽으므로,  
문서를 수정한 뒤 반드시 아래 명령을 실행해야 GUI에 반영됩니다.

```powershell
# nvc_network 루트에서 실행
.\nvcpush.ps1
```

**동작 방식:**
- `nvc_network` 내 모든 `.md` 파일 → `nvc_public/nvc_network/md/` 에 자동 복사
- 복사 후 `nvc_public` git commit + push 자동 실행
- GUI는 `C:\Projects\nvc_public\nvc_network\md\` 경로를 직접 읽으므로 push 후 즉시 반영

---

## 프로젝트 구조

```
EFG / AP 장비
    │  syslog UDP 51415
    ▼
NAS ELK Stack (Logstash V27)
    │  로그 수집·분류·저장 (JSONL)
    ▼
PC 관리 GUI (Python / Tkinter)  ← Version 0.1
    │  로그 분석 → 문제 AP 식별
    ▼
EFG / AP 장비 원격 제어 (SSH)
    └─ AP Reset · EFG 설정 조회 · 로그 Export · UniFi API
```

| 폴더 | 내용 |
|------|------|
| `elk/` | NAS Docker ELK Stack — `compose.yaml` + Logstash 파이프라인 |
| `gui/` | PC 관리 GUI — Python/Tkinter 로컬 관리 도구 |
| `efg/` | EFG 장비 참조 문서 (GUI 코드가 직접 참조, 경로 변경 금지) |

---

## ELK Stack 현재 상태 (V27)

NAS Docker Compose로 운영 중인 ELK 7.17.10 환경입니다.

- **수신:** UDP `51415` — EFG/AP syslog
- **인덱스:** `unifi-network-v27-{ap,low,noise}-*`
- **JSONL 출력:** `logstash/outputs/{ap/low/noise}/YYYY/M/D/partNNNN`

RCA 분석 모델:
```
system_issue → qos_error → ap_stuck / ap_no_service
rrm_scan_trigger → radio_driver_retry → tx_overflow
  → radio_reset → vap_timeout → channel_invalid → service_impact
```

자세한 내용: [`elk/README.md`](elk/README.md)

---

## GUI 현재 상태 (Version 0.1)

로컬 Windows PC에서 실행되는 Python/Tkinter 관리 도구입니다.

### 탭 구성

| 탭 | 기능 | 필요 권한 |
|---|---|---|
| **AP Status** | AP SSH 스캔(mca-dump/logread) + ELK Stuck 집계 + AP Reset | 모든 사용자 |
| **EFG Remote > EFG SSH** | EFG 시스템 대시보드 (시스템·네트워크·트래픽·ARP) | `can_view_efg_tab` |
| **EFG Remote > EFG API** | UniFi Controller API 탐색기 (read-only GET) | `can_view_efg_tab` |
| **Log Export** | 조건별 로그 필터·파일 저장 | `can_export` |
| **동기화** | NAS JSONL → PC 캐시 동기화 | `can_sync_nas` |
| **설정** | API 설정, Probe, AP 등록수, 정렬, mca 덤프 관리 | 모든 사용자 |
| **사용자 관리** | 계정·권한 관리 | `can_manage_users` (admin) |

### 역할 및 권한

| 역할 | 기본 권한 |
|------|----------|
| `admin` | 모든 기능 + 사용자 관리 |
| `operator` | 동기화, AP Status, EFG Remote, Log Export, AP Reset |
| `viewer` | AP Status(조회), EFG Remote, Log Export (Reset·동기화 불가) |

### 인증 흐름

```
VPN 연결 확인 (192.168.11.1:22) → 실패 시 종료
    ↓
개인 ID + bcrypt 비밀번호
    ↓
TOTP 6자리 (Google Authenticator / Authy, 30초 갱신)
    ↓
역할(Role) 로딩 → 권한에 따라 탭/버튼 활성화
```

---

## 빠른 시작

```powershell
# 의존성 설치 (처음 한 번)
pip install -r gui\requirements.txt

# VPN 연결 후 실행
python gui\main.py
```

---

## 런타임 데이터

`C:\Projects\nvc_network\runtime\` (git 제외):

```
runtime/
├── logs/raw/          ← NAS 동기화 JSONL 캐시
├── exports/           ← Log Export 결과
├── mca_dumps/         ← AP Remote mca-dump 원본
├── api_probe/         ← EFG API GET 응답 JSON
└── operations.log     ← 작업 기록
```

---

## 주의사항

- 비밀번호, API 키, `.env` 파일은 절대 커밋 금지
- `gui/config/users.json`, `gui/config/ssh_targets.json` — `.gitignore` 처리
- `efg/EFG.md` 경로는 GUI 코드가 직접 참조하므로 변경 금지
- 대용량 원본 로그 커밋 금지
