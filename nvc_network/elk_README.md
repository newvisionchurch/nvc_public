# ELK Stack — NAS Docker 구성

뉴비전 교회 NAS에서 운영 중인 ELK Stack Docker 설정입니다.
UniFi AP / EFG syslog를 수집·분류·저장하고 AP Stuck RCA를 지원합니다.

---

## 현재 베이스라인: V27

| 항목 | 값 |
|------|----|
| ELK 버전 | 7.17.10 (Docker Compose) |
| Logstash 입력 | UDP `51415` |
| 활성 파이프라인 | `logstash/pipeline/logstash.conf` |
| 활성 Compose | `compose.yaml` |
| 타임존 | `America/Los_Angeles` |

### 인덱스 패턴

```
unifi-network-v27-ap-*     ← AP/RCA 중심 이벤트
unifi-network-v27-low-*    ← 저우선순위 컨텍스트 이벤트
unifi-network-v27-noise-*  ← 고빈도 known-noise 이벤트
```

### JSONL 파일 출력

NAS 볼륨의 `logstash/outputs/` 아래 `ap/`, `low/`, `noise/` 로 분류.  
날짜별 `YYYY/M/D/` 폴더에 약 50 MiB 단위 `partNNNN` 파일로 저장.

```
outputs/ap/YYYY/M/D/unifi-network-v27-ap-YYYY.MM.dd-part0001.jsonl
outputs/low/YYYY/M/D/unifi-network-v27-low-YYYY.MM.dd-part0001.jsonl
outputs/noise/YYYY/M/D/unifi-network-v27-noise-YYYY.MM.dd-part0001.jsonl
```

---

## 버전 이력 요약

| 버전 | 주요 내용 |
|------|-----------|
| V23.3 | 기본 파싱 baseline — grok, observer/dataset 분리, AP 이름 추출 |
| V24.1 | DHCP action 매핑 (dhcp_request / dhcp_ack / dhcp_fail) |
| V24.2 | CEF 파싱 수정 — `message` 전체에서 CEF 추출, `zcef_*` 복구 |
| V24.3 | QoS error 분리 (`qos_error`, `ap_no_service`, tier=high) |
| V25.1 | `system_issue` 탐지 추가 (disk full, I/O error, memory failure) |
| V25.2 | 상관분석 helper 필드 추가 (`correlation_stage`, `blast_scope` 등) |
| V26 | storage_class 라우팅 — ap/low/noise 인덱스 분리 |
| V27 | RRM scan trigger 승격, AP-stuck 타임라인 필드 (rca_step 1~7) |

---

## 파이프라인 구조 (필터 섹션 맵)

Logstash 필터는 섹션 번호 순으로 실행된다.  
**보호 구간(ABSOLUTE DO NOT CHANGE)** 은 절대 수정 금지.

| 섹션 | 역할 | 보호 여부 |
|------|------|:---------:|
| 1 | `[meta][raw_message]` 백업, `drop_low_priority` 초기화 | |
| 2 | **Base syslog grok** — 4가지 패턴으로 timestamp/host/process/log_message 추출 | ✅ 보호 |
| 3 | `event.module` 정규화 (process.name 우선) | |
| 3A | EFG 방화벽 사전분류 (CEF 없는 [RULE] 형태) | |
| 4 | **observer/dataset base** — 기본값 `ap`/`unifi.ap`, EFG 조건 시 `gateway`/`unifi.efg` | ✅ 보호 |
| 5 | event.module 보완 (dnsmasq, hostapd, kernel, stahtd 등) | |
| 6 | AP 이름 추출 — site code, floor, location, priority | |
| 7 | 공통 필드 추출 — client MAC/IP, radio channel/RSSI, VLAN, WLAN, kernel uptime | |
| 7A | EFG 방화벽 rule metadata (DESCR 추출) | |
| 7B | V27 RRM scan metadata 추출 — scan_id, interface, channel, method | |
| 8 | CEF 파싱 수정 — `message` 전체에서 `meta.cef_raw` 추출 후 `zcef_*` 생성 | |
| 9 | **기본 event 필드 초기화** — action=unclassified, problem_class=low_priority | |
| 10 | RF / AP core 매핑 (V23.3 보존) — low_rssi, retry_high, tx_overflow, channel_invalid, vap_timeout, radio_reset, anomaly | |
| 10A | V24.1 DHCP 확장 — dhcp_request/ack/other/fail | |
| 10B | V24.3 QoS error 분리 | |
| 10C | V25.1 system_issue 분리 | |
| 10C2 | V27 RRM scan trigger 승격 → rrm_scan_risk | |
| 10D | V26 known-noise 분리 → noise index | |
| V25.2 | 상관분석 helper 필드 — correlation_stage/domain/blast_scope | |
| V27 | AP-stuck 타임라인 필드 — rca_step 1~7, rca_stage, loop_family | |
| V27 라우팅 | storage_class 최종 결정 — correlation_target=true → ap | |
| 11 | severity 정규화 | |
| 12 | 임시 필드 정리 (bridge_name, syslog_pri 제거) | |
| 13 | timestamp 정규화 (America/Los_Angeles) | |
| 14 | V27 Ruby: 50 MiB 파트 파일 라우팅 | |

---

## 보호된 baseline 구간

아래 구간은 수정 금지. 변경 시 전체 파이프라인 파괴 위험.

```
input { udp { port => 51415 } }          ← 입력 포트
```

```logstash
# 섹션 2: Base syslog grok — 8개 패턴
# 이 grok이 깨지면 log_message, device.name, process.name 모두 실패
```

```logstash
# 섹션 4: observer/dataset base
# AP vs EFG 분리 로직 — CEF, EFG device.name, dnsmasq, ubios-udapi-server
```

```logstash
output {
  elasticsearch { hosts, user, password, index ... }   ← ES 출력
  file { ... }                                          ← JSONL 출력
  stdout { ... }                                        ← AP only 디버그
}
```

---

## RCA 모델

AP Stuck / AP No-Service 는 증상이며 근본 원인이 아니다.

### 확정된 RCA 경로 1 (V25.2)

```
EFG system_issue (disk full / I/O error / memory failure)
  → QoS error (qos_control.sh 실패)
  → ap_stuck or ap_no_service (서비스 영향)
```

**실증 근거:** EFG `/boot/firmware` 디스크 full → syslog write error → AP QoS 실패 연쇄.

### 확정된 RCA 경로 2 (V27)

```
rrm_scan_trigger (iwpriv wifi1ap3 acsrrm / ApScanChannel)
  → radio_driver_retry (STA_VDEV_XRETRY — firmware retry burst)
  → tx_overflow (VAP TX queue overflow)
  → radio_reset (WAL_DBGID_DEV_RESET)
  → vap_timeout (osif_vap_init: Timeout waiting for DA vap 0 to stop)
  → channel_invalid (wlan_get_active_vport_chan: ch=0)
  → service_impact (qos_error or ap_no_service)
```

**실증 근거 — E2FAP3VISION (UAP-AC-Pro-Gen2, fw 6.8.2.15592):**
- `wifi1ap3` TX errors: 435,444 / TX dropped: 107,185,583
- `stamgr` process가 kernel `dev_ioctl`에서 `D state` (disk sleep) → wireless driver hang 확인
- 채널 DFS 112 → non-DFS 36 변경 후에도 stuck 지속 → DFS는 trigger일 뿐, stuck 상태는 reboot 전까지 유지
- 동일 패턴: `E2FAP6PCROOM` (같은 모델, 동일 firmware)

### V27 AP-Stuck 타임라인 (rca_step 1~7)

| `rca_step` | `rca_stage` | event.action | 원인 |
|:---:|---|---|---|
| 1 | `01_rrm_scan_trigger` | `rrm_scan_trigger` | RRM/채널스캔 명령 실행 |
| 2 | `02_radio_driver_retry` | `retry_high` + STA_VDEV_XRETRY | 라디오 드라이버 retry 폭증 |
| 3 | `03_tx_overflow` | `tx_overflow` | VAP TX 큐 오버플로우 |
| 4 | `04_radio_reset` | `radio_reset` | 라디오 펌웨어 reset |
| 5 | `05_vap_stop_timeout` | `vap_timeout` | VAP stop/init timeout |
| 6 | `06_channel_zero` | `channel_invalid` | 활성 VAP 채널 = ch=0 |
| 7 | `07_service_impact` | `qos_error` / `ap_no_service` | 서비스 영향 |

---

## Event 분류 체계

### event.problem_class 전체 목록

| `event.problem_class` | tier | storage_class | 의미 | 대표 event.action |
|---|:---:|:---:|---|---|
| `system_issue` | critical | ap | EFG/커널/자원 장애 — 최상위 RCA 후보 | `system_error` |
| `rrm_scan_risk` | cause_candidate | ap | RRM/채널스캔 — AP stuck 선행 지표 | `rrm_scan_trigger` |
| `ap_stuck` | high | ap | AP/라디오 증상 | `channel_invalid`, `vap_timeout`, `radio_reset` |
| `ap_no_service` | high | ap | 서비스 영향 증상 | `dhcp_fail`, `qos_error` |
| `rf_issue` | normal | ap | 무선 품질 문제 | `low_rssi`, `retry_high`, `tx_overflow`, `anomaly_detected` |
| `low_priority` | low | low | 정상 lifecycle | `dhcp_request`, `dhcp_ack`, `auth OK` 등 |
| `low_priority` (noise) | noise | noise | 고빈도 known-noise | `firewall_no_rule_description`, `wan_in_default_drop` |

> **분석 순서:** `system_issue` → `rrm_scan_risk` → `ap_stuck` → `ap_no_service`  
> AP 문제처럼 보이는 현상이 실제로는 `system_issue`(EFG 자원 부족)의 결과일 수 있다.

### event.correlation_stage (V25.2)

| problem_class | correlation_stage | blast_scope |
|---|---|---|
| `system_issue` | cause | global |
| `rrm_scan_trigger` | possible_cause | local |
| retry_high (STA_VDEV_XRETRY) | precursor | local |
| tx_overflow | precursor | local |
| `ap_stuck` | symptom | local |
| `ap_no_service` / `qos_error` | impact | local |

### 감지 키워드 — 원본 로그 패턴

**system_issue (tier=critical)**
```
No space left on device
I/O error
Suspended write operation because of an I/O error
Error suspend timeout has elapsed
syslog write error
buffer allocation failure
failed to allocate reassemble cont.
memory allocation failure
```

**rrm_scan_risk (V27 승격)**
```
Trigger rrm scan(N): sleep N;iwpriv wifi1apN acsrrm CHANNEL; sleep N;
Trigger rrm scan(N): sleep N;iwpriv wifi1apN set ApScanChannel=active:CHANNEL:DURATION; sleep N;
scan-rrm-check
iwpriv wifiXapY acsrrm
iwpriv wifiXapY ApScanChannel
```
→ 추출 필드: `rrm.target_interface` (예: `wifi1ap3`), `rrm.target_channel` (예: `161`), `rrm.method`, `rrm.scan_id`

**ap_stuck (tier=high)**
```
osif_vap_init: Timeout waiting for DA vap 0 to stop   → vap_timeout
wlan_get_active_vport_chan: ch=0                        → channel_invalid
WAL_DBGID_DEV_RESET / DEV_RESET                        → radio_reset
vap-0(wifi1ap3):TX OVERFLOW                            → tx_overflow (rca_step 3)
STA_VDEV_XRETRY / WAL_DBGID_STA_VDEV_XRETRY           → retry_high (rca_step 2)
```

**ap_no_service / QoS (tier=high)**
```
qos_control.sh / tc class / qos_sta_control / RTNETLINK / cmd failed → qos_error
DHCPNAK(                                                               → dhcp_fail
```

---

## AP 이름 / 위치 / 우선순위 체계

AP 이름 패턴 예:
- `M2F_AP3_PASTOR_A2` → site=Main, floor=2F, seq=3, location=PASTOR_A2
- `E2F_AP3_VISION` → site=Education, floor=2F, seq=3, location=VISION

| site.code | site.name |
|---|---|
| M | Main |
| E | Education |

| location 패턴 | location.type | ap.priority |
|---|---|---|
| PASTOR, OFFICE, R200 | pastor / office | critical |
| BONDANG, SANCTUARY, FELLOWSHIP, VIPROOM, PCROOM, ROOM | sanctuary / meeting | high |
| 그 외 | general | normal |

---

## 분석 원칙

### 5축 분석

모든 장애 분석은 반드시 아래 5개 축에서 수행한다.

1. **AP** — 어떤 AP인가
2. **위치 (Site/Floor/Location)** — 건물·층·구역 기준 패턴
3. **모델** — 동일 모델(예: UAP-AC-Pro-Gen2)에서 반복 여부
4. **시간** — 동일 시간대 반복 여부, system_issue 직후 AP fault 증가 여부
5. **Network/VLAN** — 특정 VLAN에만 나타나는 패턴

### 금지 사항

- AP만 보고 결론 내지 않는다 — EFG를 반드시 포함한다
- 단일 로그로 결론 내지 않는다
- `system_issue`를 부차적 로그로 취급하지 않는다
- `rf_issue`와 `ap_no_service`(서비스 실패)를 혼동하지 않는다
- 정상 lifecycle 로그를 root cause로 승격하지 않는다
- Logstash 변경 시 **반드시 small delta** — big rewrite 금지

---

## Kibana 분석 가이드

### 5단계 분석 흐름 (1/5/10분 window)

**Step 1 — system_issue 선행 확인**
```kql
event.problem_class:system_issue AND observer.type:gateway
```
→ AP fault 이전 EFG에서 system_issue가 먼저 발생했는지 확인

**Step 2 — 통합 상관관계 조회**
```kql
event.problem_class:(system_issue OR ap_stuck OR ap_no_service)
OR event.action:(qos_error OR rrm_scan_trigger)
```

**Step 3 — AP-stuck 타임라인 (V27)**
```kql
event.rca_sequence:rrm_to_ap_stuck AND ap.name:"E2F_AP3_VISION"
```
→ `event.rca_step` 1→7 순서로 정렬하여 trigger → service_impact 흐름 확인

**Step 4 — Lens 시계열 비교**
- Series A: `event.problem_class:system_issue`
- Series B: `event.problem_class:ap_stuck`
- Series C: `event.problem_class:ap_no_service`
- Series D: `event.action:qos_error`
- 인터벌: 1분(즉시성) → **5분(실전 판단)** → 10분(전체 파형)

**Step 5 — gateway/AP breakdown**
```kql
observer.type:gateway AND event.problem_class:system_issue
observer.type:ap AND event.problem_class:ap_stuck
```

### 핵심 판단 질문

- `system_issue` 직후 1~5분 내 AP fault가 증가하는가?
- AP fault보다 먼저 gateway/system 로그가 증가하는가?
- 특정 `event.module`(syslog-ng, kernel)이 선행하는가?
- 특정 AP 모델/펌웨어에서만 반복되는가?
- AP fault가 독립 사건인가, `system_issue` 후행 증상인가?

### 해석 기준

| 패턴 | 해석 |
|------|------|
| `system_issue` spike 직후 1~5분 내 AP fault 반복 증가 | EFG/system이 상위 원인 |
| 일부 구간만 같이 움직임 | system_issue는 일부 구간의 원인 |
| system_issue 없이도 AP fault 다수 발생 | AP 자체 / RF / 서비스 계층 별도 분석 |
| `rrm_scan_trigger` 이후 수분 내 vap_timeout / ch=0 반복 | RRM trigger가 AP stuck 원인 |

---

## Logstash 변경 절차

새 규칙을 추가할 때 반드시 따르는 순서:

1. **패턴 확인** — Kibana Discover에서 대상 `log_message` 패턴 샘플 수집
2. **Small delta 작성** — 기존 섹션 다음에 새 `if` 블록 추가, 기존 블록 수정 금지
3. **Config test** — `docker compose exec logstash logstash --config.test_and_exit`
4. **Reload 또는 재시작** — `docker compose restart logstash`
5. **검증** — 아래 [검증 체크리스트](#검증-체크리스트) 전체 통과 확인
6. **문서 업데이트** — `logstash-rules.md`에 새 패턴/필드 추가

### NAS 운영 명령어

```bash
# config test
docker compose exec logstash logstash --config.test_and_exit -f /usr/share/logstash/pipeline/logstash.conf

# reload (설정 변경 후)
docker compose restart logstash

# 로그 확인
docker compose logs -f logstash

# Elasticsearch 상태
curl -u elastic:PASSWORD http://localhost:9200/_cat/indices?v
```

---

## 검증 체크리스트

변경 후 반드시 확인:

- [ ] Logstash config test 통과
- [ ] UDP `51415` 수신 정상 (EFG syslog 유입)
- [ ] `message`와 `[meta][raw_message]` 보존
- [ ] `observer.type` — `gateway` / `ap` 분리 정상
- [ ] `event.dataset` — `unifi.efg` / `unifi.ap` 분리 정상
- [ ] CEF 이벤트 → `zcef_*` 생성 정상
- [ ] `system_issue` → tier=critical, outcome=failure
- [ ] `qos_error` → ap_no_service, tier=high
- [ ] `rrm_scan_trigger` → rrm_scan_risk, storage_class=ap
- [ ] RRM metadata 필드 (`rrm.target_interface`, `rrm.target_channel`) 생성
- [ ] AP stuck timeline `rca_step` 1~7 생성
- [ ] `low_priority` → storage_class=low
- [ ] known-noise → storage_class=noise
- [ ] JSONL 파일 `outputs/{ap,low,noise}/YYYY/M/D/partNNNN` 생성
- [ ] 50 MiB 초과 시 다음 part 파일로 롤오버

---

## V28 이후 로드맵

현재 미구현 — 우선순위 순:

| 우선순위 | 클래스 | 대상 로그 | 비고 |
|:---:|--------|-----------|------|
| 1 | `auth_issue` | EAPOL timeout, auth fail, RADIUS reject | V25 로드맵에서 보류 — 오탐 방지 최우선 |
| 2 | `roaming_issue` | FT decrypt fail, roaming failure | auth_issue 이후 예정 |
| 3 | DNS/DHCP 세분화 | dns_fail, dhcp_fail 패턴 세분화 | V25.4 service refinement |
| 4 | RRM 비활성화 효과 검증 | Radio AI, WiFi AI, Nightly Channel Optimization 끈 뒤 AP stuck 재발 여부 | UAP-AC-Pro-Gen2 fw 6.8.2 검증 |

### V28 추가 시 원칙

- 기존 event.action/problem_class 블록 수정 금지
- 새 `if` 블록을 기존 섹션 다음에 추가
- 한 번에 1개 문제 클래스만 추가
- 추가 후 반드시 검증 체크리스트 전체 통과

---

## 주요 파일

| 파일 | 역할 |
|------|------|
| `compose.yaml` | Docker Compose (ES, Kibana, Logstash) |
| `logstash/pipeline/logstash.conf` | 활성 파이프라인 (V27) |
| `logstash/config/` | Logstash JVM/시스템 설정 |
| `docs/logstash-rules.md` | 파이프라인 규칙 상세 (필드, 패턴, 검증) |
| `docs/root_cause_kr.md` | RCA 분석 상세 (E2FAP3VISION 사례 포함) |
| `docs/nas-setup.md` | NAS Docker 셋업 |
| `docs/efg-settings.md` | EFG 설정 참조 |
| `MEMO.md` | NAS 운영 명령어 모음 |
| `CLAUDE.md` | Logstash 작업 시 AI 지침 |

---

## 주의사항

- `input`, `output`, 기본 syslog grok, observer/dataset 섹션은 **보호된 baseline** — 변경 금지
- 변경은 반드시 소규모 델타(small delta)로 — 큰 리라이트 금지
- Logstash 동작 변경 시 `docs/logstash-rules.md` 함께 업데이트
- 비밀번호/API 키 절대 커밋 금지
- ES 비밀번호(`compose.yaml`)는 NAS 로컬 파일에만 존재
