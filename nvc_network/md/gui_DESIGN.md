# NVC Network GUI — 설계 구상서

GUI Version 0.2 — 05/29/2026

---

## 1. 목적

뉴비전 교회 네트워크(UniFi EFG / AP 29대)에서 반복 발생하는 AP Stuck 문제를 운영팀이 Kibana나 SSH 수동 작업 없이 GUI 하나로 빠르게 찾고, 분석하고, 원격 제어할 수 있는 로컬 관리 도구.

---

## 2. 기술 스택

| 항목 | 선택 | 이유 |
|------|------|------|
| 언어 | Python 3.10+ | SSH/SFTP 라이브러리 풍부, 로그 처리 용이 |
| GUI | Tkinter | Python 기본 내장, Windows 지원 |
| SSH | paramiko | AP/EFG/NAS 접속 통합 처리 |
| 인증 | bcrypt + TOTP | 비밀번호 해시 + 2FA |
| 자격증명 | keyring (Windows Credential Manager) | OS 레벨 보안 보관 |
| API | UniFi Local Network API | New API Key 방식 |
| 이미지 | Pillow (PIL) | 교회 로고 리사이즈 표시 |

---

## 3. 탭 구성 (Version 0.1)

| 탭 | 서브탭 | 핵심 기능 |
|---|---|---|
| AP Status | — | mca-dump/logread 진단 + ELK Stuck 집계 + AP Reset |
| EFG Remote | EFG SSH | EFG 시스템 대시보드 |
| EFG Remote | EFG API | UniFi Controller API 탐색기 |
| Log Export | — | 조건별 로그 파일 저장 |
| 동기화 | NAS ELK Sync | NAS JSONL → PC 캐시 동기화 |
| 설정 | 여러 서브탭 | API 설정/Probe, 정렬, mca 덤프, 폰트 등 |
| 설정 > 사용자 관리 | — | 계정·권한 관리 (admin) |

---

## 4. 주요 모듈

| 모듈 | 역할 |
|------|------|
| `auth.py` | bcrypt 로그인 + TOTP 2FA + 개발자 모드 (MachineGuid) |
| `vpn_check.py` | TCP 소켓으로 VPN 연결 확인 |
| `nas_sync.py` | paramiko SFTP — NAS JSONL 로컬 캐시 동기화 |
| `ap_count.py` | 날짜 범위별 AP Stuck 이벤트 집계 |
| `ap_detail.py` | 문제 AP 상세 분석, Stuck 유형 분류 |
| `ap_reset.py` | SSH 원격 reboot — EFG relay 경유 |
| `ap_remote.py` | AP SSH 스캔 — mca-dump 파싱 + logread 진단 |
| `efg_remote.py` | EFG 대시보드 — system/네트워크/트래픽/ARP 조회 |
| `unifi_client.py` | UniFi Local API 클라이언트 (New API Key / Legacy) |
| `log_export.py` | 조건별 로그 필터링 후 파일 저장 |
| `font_utils.py` | OS 폰트 감지 및 UI 전체 적용 |
| `keychain.py` | Windows Credential Manager (keyring 래퍼) |
| `ssh_client.py` | paramiko 공통 — 패스워드/키 인증, PTY, relay |
| `models.py` | 핵심 데이터 모델 |

---

## 5. AP Status 탭 상세

- AP 29대 인벤토리 기반 테이블 (Treeview)
- 전체/선택 조회 (최대 4개 병렬)
- 진단 컬럼: ELK Stuck, 로그오류, 재시작2G/5G, VAP지연2G/5G, CU%, 클라이언트수, 채널, 채널폭, CPU/Mem, Uptime, 응답ms
- UniFi API 연동: 채널(Ch), 채널폭(BW), WLAN standard, model, firmware
- 임계값 경고 (`⚠`), `⚠ 설정` 팝업으로 변경 → `app_config.json` 저장
- AP Reset: 다중 선택(빨간색) 후 일괄 실행

---

## 6. EFG SSH 탭 상세

`fetch_efg_dashboard()` — 단일 SSH 세션, `##MARKER##` 구분 수집:
- 시스템 정보 카드 (hostname, uptime, kernel, loadavg, free -m)
- 네트워크 경로 카드 (ip route show)
- 인터페이스 테이블 (ip addr show)
- 트래픽 통계 테이블 (/proc/net/dev)
- ARP 테이블 (arp -n)
- 모든 테이블 컬럼 클릭 정렬 지원

---

## 7. EFG API 탭 상세

**레이아웃**: 좌(30%) / 우(70%) 분할

**좌측 패널**:
- API 연결 상태 카드 (상태, API 스타일, 마지막 조회)
- Site 카드 (Site Name, Site ID)
- 장비 현황 카드 (Device/AP/Client Count)
- WLAN 카드: SSID 드롭다운 → 상세 (networkconf/usergroup 연동)
- 장비 목록: AP / SW / EFG 라디오 버튼 필터 (● ONLINE / ○ OFFLINE)

**우측 패널**:
- Preset 드롭다운 (20개) + GET 버튼
- device detail/stats 선택 시 AP 드롭다운 자동 표시
- JSON 응답 뷰어 (캐시 자동 로드, Preview 8000자)

**컨트롤러**: UniFi Network 10.3.58 (UDMENT)
- Official API: sites, devices, clients, networks, device detail/stats
- Legacy API: wlanconf, networkconf, usergroup, wlangroup, firewallrule, firewallgroup, portforward, routing, health, sysinfo, dashboard, rogueap, report/daily·hourly

---

## 8. SSH 접속 구조

| 대상 | 방식 | PTY |
|------|------|-----|
| NAS (Synology) | paramiko SSHClient, 키 인증 | 불필요 |
| EFG (OpenSSH) | paramiko Transport, 패스워드 | 불필요 |
| AP (Dropbear) 직접 | paramiko Transport, 패스워드 | **필요** |
| AP via EFG relay | EFG에서 sshpass+ssh | 불필요 |

AP Dropbear는 paramiko 직접 패스워드 인증 불가 → EFG relay 필수.

---

## 9. 데이터 흐름

```
NAS JSONL 로그
    │  동기화 탭 (paramiko SFTP)
    ▼
runtime/logs/raw/
    │  AP Status → ELK Stuck Count
    ▼
AP별 Stuck 집계 → 테이블 색상

AP (Dropbear SSH via EFG relay)
    │  AP Status → 전체/선택 조회
    ▼
mca-dump + logread → 진단 컬럼

EFG (OpenSSH)
    │  EFG SSH 탭
    ▼
시스템 대시보드

UniFi Controller API
    │  EFG API 탭 / AP Status API 연동
    ▼
장비/클라이언트/WLAN 정보 → runtime/api_probe/ 저장
```

---

## 10. 런타임 데이터 구조

```
runtime/                          ← app_config.json workspace_root_windows
├── logs/raw/                     ← NAS 동기화 JSONL
│   └── YYYY/M/D/partNNNN.jsonl
├── exports/ap_stuck/             ← Log Export 결과
├── exports/ap_detail/
├── mca_dumps/                    ← AP Remote mca-dump 원본
│   └── YYYY-MM-DD/
├── api_probe/                    ← EFG API GET 응답
│   └── YYYYMMDD_HHMMSS_<preset>.json
└── operations.log                ← 작업 기록
```

---

## 11. 설계 결정 메모

- **AP Reset relay**: AP Dropbear SSH ↔ paramiko 패스워드 인증 불가 → EFG(OpenSSH)에서 sshpass 경유
- **UniFi API use_legacy 우선**: `use_legacy=True` 명시 시 `config.api_style` 우선도 역전
- **Settings 탭 스크롤**: `<<NotebookTabChanged>>` 이벤트로 `bind_all(<MouseWheel>)` 활성화
- **폰트**: `font_utils.py`에서 OS 자동 감지 후 전체 UI 통일 적용
