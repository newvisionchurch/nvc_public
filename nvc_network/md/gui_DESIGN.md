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

## 3. 탭 구성

| 탭 | 서브탭 | 핵심 기능 |
|---|---|---|
| AP Status | — | mca-dump/logread 진단 + ELK Stuck 집계 + AP Reset + 분류 점수 |
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
| `ap_remote.py` | AP SSH 스캔 — mca-dump 파싱 + logread 진단 + ResetScore 계산 |
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
- 진단 컬럼: 분류, 점수, ELK Stuck, 로그오류, DevReset, 재시작2G/5G, VAP지연2G/5G, CU%, 클라이언트수, 채널, 채널폭, CPU/Mem, Uptime, 응답ms
- UniFi API 연동: 채널(Ch), 채널폭(BW), WLAN standard, model, firmware
- 임계값 경고 (`⚠`), `경고 설정` 팝업으로 변경 → `app_config.json` 저장
- **AC Pro DevReset 강조**: AC Pro + DevReset > 0 + 재시작5G > 0 AND 조건 충족 시 `▶ N` 빨간 Bold 표시
- AP Reset: 다중 선택(빨간색) 후 일괄 실행, AC Pro 전체 선택 버튼 제공

### 컨트롤 바 버튼 구성 (좌→우)

```
[전체 조회] [선택 조회] [중지]  |  [동작 설명] [경고 설정] [컬럼 설정]
  날짜 범위 선택  |  [ELK Stuck Count]  |  Stuck Count 결과
  [분류규칙] [분류설정] [Reset후보/관찰/정상/미조회 범례 4줄] [AP Reset 후보 분류]
```

### 하단 버튼 구성 (좌→우)

```
[상세 보기]  ···  [AP Reset 선택] [AC Pro 선택] [AP 선택 해제] | [모두 해제] | [분류 선택]  선택N  [AP Reset 실행]
```

---

## 6. AP Reset 후보 분류 (ResetScore)

`ap_remote.calculate_reset_score()` 함수로 AP별 점수를 계산하고 색상을 결정.

### 점수 구성 (현행 기본값 — 2026-05-29 실데이터 기반 재조정)

| 항목 | 소스 | 구간 | 점수 |
|------|------|------|------|
| **LogScore** | logread (stuck\|ath_reset\|vap timeout) | 1~2건 | +30 |
| | | 3~9건 | +40 |
| | | ≥10건 | +50 |
| **ElkScore** | NAS JSONL ELK Stuck | 500~1999 | +15 |
| | | 2000~4999 | +30 |
| | | ≥5000 | +50 |
| **Reset5GScore** | ast_ath_reset/day | 100~299/day | +10 |
| (AC HD 전용) | | 300~599/day | +20 |
| | | ≥600/day | +30 |
| **VapScore** | timeout_waiting_for_vap_cnt | 1~20 | +15 |
| | | ≥21 | +30 |
| **CuScore** | cu_total | 70~84% | +10 |
| | | ≥85% | +15 |

### 모델별 예외

| 모델 | DevReset(fw_bug_count) | 재시작5G |
|------|----------------------|---------|
| **AC Pro** | 점수 제외 (fw 6.8.x 구조적 버그) | DevReset>0 AND 재시작5G>0 이면 제외 |
| **AC HD** | log_error에 합산 (실제 이상) | 정상 반영 |

### 최종 판정

| 색상 | 조건 |
|------|------|
| 🔴 빨강 (Reset후보) | 합산 ≥ 60 AND reset_evidence=True |
| 🟡 노랑 (관찰) | 합산 ≥ 20 |
| 🟢 초록 (정상) | 합산 < 20 |
| ⬜ 회색 (미조회) | SSH 미조회 / 오프라인 |

`reset_evidence = True` 조건: `log_error > 0` OR `vap > 0` OR `restart5g/day ≥ 200 (AC Pro/HD)`

### 팝업 3종

| 팝업 | 버튼 | 설명 |
|------|------|------|
| 분류 규칙 | `분류규칙` | 전체 점수 조건·소스·예외를 테이블로 표시 (읽기 전용) |
| 분류 설정 | `분류설정` | 각 구간 임계값·점수 수정 후 저장 |
| 경고 설정 | `경고 설정` | ⚠ 표시 임계값 + AC Pro DevReset 강조 ON/OFF |

---

## 7. EFG SSH 탭 상세

`fetch_efg_dashboard()` — 단일 SSH 세션, `##MARKER##` 구분 수집:
- 시스템 정보 카드 (hostname, uptime, kernel, loadavg, free -m)
- 네트워크 경로 카드 (ip route show)
- 인터페이스 테이블 (ip addr show)
- 트래픽 통계 테이블 (/proc/net/dev)
- ARP 테이블 (arp -n)
- 모든 테이블 컬럼 클릭 정렬 지원

---

## 8. EFG API 탭 상세

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

## 9. SSH 접속 구조

| 대상 | 방식 | PTY |
|------|------|-----|
| NAS (Synology) | paramiko SSHClient, 키 인증 | 불필요 |
| EFG (OpenSSH) | paramiko Transport, 패스워드 | 불필요 |
| AP (Dropbear) 직접 | paramiko Transport, 패스워드 | **필요** |
| AP via EFG relay | EFG에서 sshpass+ssh | 불필요 |

AP Dropbear는 paramiko 직접 패스워드 인증 불가 → EFG relay 필수.

---

## 10. 데이터 흐름

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
mca-dump + logread → 진단 컬럼 → ResetScore 계산 → 분류 색상

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

## 11. 런타임 데이터 구조

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

## 12. 설계 결정 메모

- **AP Reset relay**: AP Dropbear SSH ↔ paramiko 패스워드 인증 불가 → EFG(OpenSSH)에서 sshpass 경유
- **AC Pro DevReset 분리**: WAL_DBGID_DEV_RESET은 fw 6.8.x 칩셋 버그(QCA9563/QCA9882) — Reset 후 재부팅 시 재발. DevReset과 재시작5G 모두 점수 제외, AC Pro 전체 선택 버튼으로 수동 Reset 대응
- **AC HD 재시작**: DevReset 없이 ast_ath_reset 증가 → 실제 하드웨어 이상으로 판단, 점수 정상 반영
- **ElkScore 임계값 재조정 (2026-05-29)**: 실데이터 분포(AC Pro ELK 500~7327) 반영 — elk_thr_s 1000→500, elk_thr_m 3000→2000, elk_thr_l 10000→5000
- **판정 임계값 재조정 (2026-05-29)**: red 70→60, yellow 30→20, evidence_r5g 300→200
- **UniFi API use_legacy 우선**: `use_legacy=True` 명시 시 `config.api_style` 우선도 역전
- **Settings 탭 스크롤**: `<<NotebookTabChanged>>` 이벤트로 `bind_all(<MouseWheel>)` 활성화
- **폰트**: `font_utils.py`에서 OS 자동 감지 후 전체 UI 통일 적용
- **팝업 스타일 통일**: 모든 설정/정보 팝업 — BG=#f5f7fa, HBG=#37474f 다크헤더, CBG=#ffffff 흰 테이블, Canvas 스크롤
