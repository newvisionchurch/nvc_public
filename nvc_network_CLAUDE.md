# NVC Network 프로젝트 — Claude Code 지침

## 프로젝트 구조

```
NVC_Network/
├── elk/                  ← NAS ELK Stack (compose.yaml + logstash.conf)
│   ├── README.md         ← ELK 상세 문서 (V27 베이스라인, RCA 모델, Kibana 가이드)
│   ├── MEMO.md           ← NAS 운영 명령어
│   ├── compose.yaml
│   ├── logstash/pipeline/logstash.conf
│   └── docs/             ← ELK 세부 문서
├── gui/                  ← Python/Tkinter 관리 GUI
│   ├── DESIGN.md         ← GUI 설계 구상서 (초기 설계부터 현재 구현까지 이력)
│   ├── MANUAL.md         ← 사용자 매뉴얼
│   ├── README.md         ← 기술 퀵스타트
│   ├── main.py           ← GUI 진입점 (NetworkGuiApp 클래스)
│   ├── modules/          ← GUI 모듈
│   └── config/           ← 런타임 설정 (일부 gitignore)
├── efg/                  ← EFG 참조 문서 (GUI 코드가 직접 참조)
│   └── EFG.md            ← EFG 참조 (경로 변경 금지)
└── codex/                ← Codex 시대 백업 (추후 삭제 예정)
```

## 작업 전 필수 확인

### ELK 작업 시
1. `elk/README.md` — 현재 베이스라인(V27), RCA 모델, Kibana 분석 가이드
2. `elk/docs/logstash-rules.md` — 파이프라인 규칙, 감지 키워드, 검증 체크리스트
3. `elk/docs/index.md` — 운영 가이드
4. `elk/logstash/pipeline/logstash.conf` — 활성 파이프라인

### GUI 작업 시
1. `gui/modules/models.py` — 핵심 데이터 모델
2. `gui/main.py` — 메인 GUI 클래스 (NetworkGuiApp)
3. `gui/config/app_config.json` — 앱 설정 (stuck_thresholds 포함)
4. 상세 설계 이력 필요 시: `gui/DESIGN.md`

## 핵심 제약사항

### ELK
- `input`, `output`, 기본 syslog grok 섹션은 **보호된 baseline** — 변경 금지
- 변경은 반드시 소규모 델타로 (큰 변경 금지)
- Logstash 동작 변경 시 `elk/docs/logstash-rules.md` 함께 업데이트

### GUI
- Python + Tkinter (로컬 PC 도구, 웹 아님)
- 안정적인 변경만, 한 번에 하나씩
- `gui/config/users.json` — git 제외 (실제 계정 정보, bcrypt 해시)
- `gui/config/ssh_targets.json` — git 제외 (실제 SSH 정보)
- `efg/EFG.md` — GUI 코드가 `APP_DIR.parent / "efg" / "EFG.md"` 로 참조 — **경로 변경 금지**
- 런타임 데이터: `C:\Projects\nvc_network\runtime\` (app_config.json에 설정)

### 공통
- 비밀번호, API 키, `.env` 절대 커밋 금지
- 대용량 로그 파일 커밋 금지

## ELK 현재 상태 (V27)

- ELK 7.17.10, NAS Docker Compose
- Logstash UDP 51415 수신
- 인덱스: `unifi-network-v27-{ap,low,noise}-*`
- JSONL 출력: NAS `logstash/outputs/{ap,low,noise}/YYYY/M/D/partNNNN`

## GUI 현재 상태 (V1 — 기능 완성)

### 인증 흐름
```
VPN 연결 확인 (192.168.11.1:22) → 실패 시 종료
    ↓
개인 ID + bcrypt 비밀번호
    ↓
TOTP 6자리 (Google Authenticator / Authy, 30초 갱신)
    ↓
역할(Role) 로딩 → 권한에 따라 탭/버튼 활성화
    ↓
NAS / EFG / AP SSH 접속 (Windows Credential Manager)
```

### 탭 구성 및 권한
| 탭 | 설명 | 필요 권한 |
|---|---|---|
| Log 동기화 | NAS JSONL → PC 캐시 동기화 | `can_sync_nas` |
| AP ELK Log | ELK 로그 기반 Stuck 집계·상세 분석 | 모든 사용자 |
| AP Remote | AP SSH 직접 스캔 — mca-dump 진단 | 모든 사용자 |
| EFG Remote | EFG 시스템 대시보드 | `can_view_efg_tab` |
| Log Export | 조건별 로그 필터·파일 저장 | `can_export` |
| 사용자 관리 | 계정·권한 관리 | `can_manage_users` (admin) |
| EFG 참조 | EFG.md 읽기 전용 표시 | `can_view_efg_tab` |

### 역할 및 권한
| 역할 | 기본 권한 |
|------|----------|
| `admin` | 모든 기능 + 사용자 관리 |
| `operator` | Log 동기화, AP ELK Log, AP Remote, EFG Remote, Export, AP Reset |
| `viewer` | AP ELK Log, AP Remote (조회), EFG Remote, Export (동기화·Reset 불가) |

권한은 역할 기본값 외에 per-user 단위로 admin 화면에서 on/off 가능.

### 모듈 구조
| 모듈 | 역할 |
|------|------|
| `auth.py` | bcrypt 로그인 + TOTP 2FA |
| `vpn_check.py` | TCP 소켓으로 VPN 연결 확인 |
| `nas_sync.py` | paramiko SFTP — NAS JSONL 로컬 캐시 동기화 |
| `ap_count.py` | 날짜 범위별 AP Stuck 이벤트 집계 |
| `ap_detail.py` | 문제 AP 상세 분석, Stuck 유형 분류 |
| `ap_reset.py` | SSH 원격 reboot — EFG relay 경유 (run_ssh_via_relay) |
| `ap_remote.py` | AP SSH 스캔 — mca-dump 파싱 + logread 진단 |
| `efg_remote.py` | EFG 대시보드 — system/네트워크/트래픽/ARP 조회 |
| `log_export.py` | 조건별 로그 필터링 후 파일 저장 |
| `ap_inventory.py` | AP 목록 관리 (ap_inventory.json 기반) |
| `local_storage.py` | 런타임 폴더 생성·관리 |
| `report_writer.py` | operations.log에 사용자명·작업 기록 |
| `keychain.py` | Windows Credential Manager (keyring 래퍼) |
| `ssh_client.py` | paramiko 공통 — 패스워드/키 인증, PTY, relay |
| `models.py` | 핵심 데이터 모델 |

### AP ELK Log 탭 — Stuck 판단 기준
```
event.problem_class: ap_stuck (channel_invalid, vap_timeout, radio_reset)
```
| Stuck 카운트 | 상태 색상 |
|-------------|----------|
| 0 | Healthy (Green) |
| 1~2 | Warning (Yellow) |
| 3~9 | Suspect (Orange) |
| 10+ | Critical (Red) |

### AP Remote 탭 — 진단 컬럼 (mca-dump + logread)
| 컬럼 | 소스 | 설명 |
|------|------|------|
| ELK Stuck | ELK Log 캐시 | ELK 기반 Stuck 이벤트 수 (ELK Log Stuck Count 실행 후 동기화) |
| 로그오류 | `logread \| grep -cE` | 현재 부팅 세션 내 stuck/ath_reset/vap timeout 패턴 수 |
| 재시작2G/5G | `athstats.ast_ath_reset` | 드라이버 레벨 무선 리셋 누적 |
| VAP지연2G/5G | `athstats.timeout_waiting_for_vap_cnt` | VAP 초기화 타임아웃 누적 |
| CU2G%/CU5G% | `athstats.cu_total` | 채널 이용률 (Channel Utilization) |
| 2.4G Cli / 5G Cli | `radio_table[*].num_sta` | 연결 클라이언트 수 |
| 응답(ms) | SSH 응답 시간 | mca-dump 전체 왕복 시간 |

임계값 초과 시 셀 앞에 `⚠` 표시. `⚠ 설정` 버튼으로 임계값 변경 → `app_config.json`의 `stuck_thresholds`에 저장.

### AP Reset — EFG Relay 구조
```
GUI (paramiko) → EFG (OpenSSH) → sshpass → AP (Dropbear)
```
- `run_ssh_via_relay()` 사용
- EFG에 sshpass 설치 필요
- AP에 직접 paramiko 패스워드 인증 불가 (Dropbear 호환성 문제) → relay 필수

### EFG Remote 탭 — 대시보드
`fetch_efg_dashboard()` 단일 SSH 세션으로 `##MARKER##` 구분 수집:
- 시스템 정보 카드: hostname, uptime, kernel, loadavg, free -m
- 네트워크 경로 카드: ip route show
- 인터페이스 테이블: ip addr show (이름, IP/prefix, MAC, 상태)
- 트래픽 통계 테이블: /proc/net/dev (수신/송신 MB, 패킷, 오류)
- ARP 테이블: arp -n (IP, MAC, 인터페이스, 상태)
- 모든 테이블 컬럼 클릭 정렬 지원 (↑/↓ 표시)

### SSH 접속 구조
| 대상 | 방식 | PTY |
|------|------|-----|
| NAS (Synology) | paramiko SSHClient, 키 인증 | 불필요 |
| EFG (OpenSSH) | paramiko Transport, 패스워드 인증 | 불필요 (use_pty=False) |
| AP (Dropbear) 직접 | paramiko Transport, 패스워드 인증 | **필요** (Dropbear 요구) |
| AP via EFG relay | EFG에서 sshpass+ssh, ssh -T | 불필요 (OpenSSH 경유) |

### 주요 설정 파일
| 파일 | 내용 | git 포함 |
|------|------|---------|
| `config/app_config.json` | 앱 경로, NAS 설정, VPN 체크, `stuck_thresholds` | ✅ |
| `config/ap_inventory.json` | AP 29대 목록 (이름, IP, 모델, 위치) | ✅ |
| `config/ssh_targets.example.json` | SSH 설정 예제 | ✅ |
| `config/users.json` | 사용자 계정 (bcrypt 해시, TOTP secret) | ❌ |
| `config/ssh_targets.json` | NAS/EFG/AP SSH 접속 정보 | ❌ |

### V2 이후 확장 계획
- Google OAuth 로그인 (Workspace 이메일 기반)
- Kibana / Elasticsearch API 직접 조회
- AP 상태 실시간 모니터링
- EFG system_issue 자동 감지 및 알림
- AP 리셋 후 복구 여부 자동 확인
- Daily Report 자동 생성
