# NVC Network 관리 프로젝트

뉴비전 교회 네트워크(UniFi EFG / AP 29대 / NAS)의 로그 수집·분석·제어를 위한 통합 관리 프로젝트입니다.

---

## 프로젝트 목표

교회 네트워크에서 반복 발생하는 **AP Stuck / AP No-Service** 문제를 해결하기 위해 세 가지 축으로 구성됩니다.

```
EFG / AP 장비
    │  syslog UDP 51415
    ▼
NAS ELK Stack (Logstash V27)
    │  로그 수집·분류·저장 (JSONL)
    ▼
PC 관리 GUI (Python / Tkinter)
    │  로그 분석 → 문제 AP 식별
    ▼
EFG / AP 장비 원격 제어 (SSH)
    └─ AP Reset · EFG 설정 조회 · 로그 Export
```

**궁극적 목표:** 운영팀이 Kibana나 SSH 수동 작업 없이 GUI 단 하나로 문제 AP를 빠르게 찾고, 분석하고, 원격으로 제어할 수 있는 실용적인 로컬 관리 도구를 완성하는 것입니다.

---

## 프로젝트 구성

| 폴더 | 내용 |
|------|------|
| `elk/` | NAS Docker ELK Stack — `compose.yaml` + Logstash 파이프라인 |
| `gui/` | PC 관리 GUI — Python/Tkinter 로컬 관리 도구 |
| `efg/` | EFG 장비 참조 문서 (GUI 코드가 직접 참조, 경로 변경 금지) |
| `codex/` | Codex 시대 작업물 백업 (추후 삭제 예정) |

---

## ELK Stack 현재 상태 (V27)

NAS Docker Compose로 운영 중인 ELK 7.17.10 환경입니다.

- **수신:** UDP `51415` — EFG/AP syslog
- **인덱스:** `unifi-network-v27-{ap,low,noise}-*`
- **JSONL 출력:** `logstash/outputs/{ap/low/noise}/YYYY/M/D/partNNNN`

### RCA 분석 모델

AP Stuck은 증상이며 근본 원인이 아닙니다.

```
system_issue → qos_error → ap_stuck / ap_no_service

rrm_scan_trigger → radio_driver_retry → tx_overflow
  → radio_reset → vap_timeout → channel_invalid → service_impact
```

자세한 내용: [`elk/README.md`](elk/README.md)

---

## GUI 현재 상태 (V1)

로컬 Windows PC에서 실행되는 Python/Tkinter 관리 도구입니다.  
시작 시 자동으로 최대화 상태로 열립니다.

### 구현 완료 기능

| 기능 | 설명 |
|------|------|
| VPN 연결 확인 | 시작 시 `192.168.11.1:22` 자동 체크 |
| 개인 계정 로그인 | bcrypt 비밀번호 해시 |
| TOTP 2FA | Google Authenticator / Authy |
| 역할 기반 권한 | admin / operator / viewer |
| Admin 사용자 관리 | 계정 생성, 권한 per-user 설정 |
| 개발자 모드 | 등록된 PC에서 자동 로그인 (Windows MachineGuid 기반) |
| Log 동기화 | SFTP로 최신 JSONL 로그 로컬 캐시 |
| AP Stuck 카운트 | 날짜 범위별 AP별 Stuck 이벤트 집계 |
| AP 상세 분석 | Stuck 유형 분류, 로그 샘플 추출 |
| AP Remote 스캔 | mca-dump + logread 진단 (29대 전체 또는 선택 조회) |
| AP 다중 Reset | Reset 대상을 복수 지정(빨강 표시)하여 일괄 reboot |
| EFG 대시보드 | EFG 시스템·네트워크·트래픽·ARP 조회 |
| 로그 Export | 조건별 필터링 후 텍스트 파일 저장 |
| OS 키체인 | Windows Credential Manager로 장비 자격증명 보호 |

### AP 네트워크 현황

- **AP 29대 전체** `192.168.11.x` 서브넷으로 통합 완료
- EFG `192.168.8.0/21` 라우트가 전 AP에 도달 가능

### 접속 흐름

```
팀원 PC (집)
    │
    ▼  VPN 연결 확인 ──실패──▶ 안내 후 종료
    ▼  개인 ID + 비밀번호
    ▼  TOTP 6자리 코드
    ▼  역할(Role) 로딩 → 권한에 따라 UI 활성화
    ▼  NAS / EFG / AP SSH 접속 (OS 키체인 자격증명)
```

자세한 내용: [`gui/README.md`](gui/README.md) · [`gui/MANUAL.md`](gui/MANUAL.md)

---

## 빠른 시작

**1. 패키지 설치 (처음 한 번)**
```powershell
cd C:\Projects\nvc_network\gui
pip install -r requirements.txt
```

**2. GUI 실행**
```powershell
python C:\Projects\nvc_network\gui\main.py
```

> VPN을 먼저 연결한 후 실행하세요.

---

## V2 이후 확장 계획

- EFG system_issue 자동 감지 및 알림
- AP 상태 실시간 모니터링
- Kibana / Elasticsearch API 직접 조회
- AP 리셋 후 복구 여부 자동 확인
- Google OAuth 로그인 (Workspace 이메일 기반)
- Daily Report 자동 생성

---

## 런타임 데이터

GUI 실행 시 생성되는 로그 캐시·Export 파일은 `C:\Projects\nvc_network\runtime\` 에 저장됩니다.
이 폴더는 git 관리 대상이 아닙니다.

---

## 주의사항

- 비밀번호, API 키, `.env` 파일은 절대 커밋하지 않습니다.
- `gui/config/users.json`, `gui/config/ssh_targets.json` 은 `.gitignore` 처리됩니다.
- `efg/EFG.md` 경로는 GUI 코드가 직접 참조하므로 변경하지 않습니다.
- 대용량 원본 로그는 커밋하지 않습니다.
