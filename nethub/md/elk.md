# ELK Stack — 운영 및 작업 지침

NAS Docker ELK 7.17.10. UniFi AP/EFG syslog 수집·분류·저장. AP Stuck RCA 지원.

---

## 현재 베이스라인 (V27)

| 항목 | 값 |
|------|----|
| ELK 버전 | 7.17.10 (Docker Compose, NAS) |
| 입력 | UDP `51415` (EFG/AP syslog) |
| 활성 파이프라인 | `elk/logstash/pipeline/logstash.conf` |
| 인덱스 | `unifi-network-v27-{ap,low,noise}-*` |
| JSONL 출력 | NAS `logstash/outputs/{ap,low,noise}/YYYY/M/D/partNNNN` |
| NAS Compose 경로 | `/volume1/docker/elk-logstash/elk` |
| Kibana | `192.168.11.7:5601` |
| ES | `192.168.11.7:9200` |

### JSONL 파일 경로 패턴

```
outputs/ap/YYYY/M/D/unifi-network-v27-ap-YYYY.MM.dd-part0001.jsonl
outputs/low/YYYY/M/D/unifi-network-v27-low-YYYY.MM.dd-part0001.jsonl
outputs/noise/YYYY/M/D/unifi-network-v27-noise-YYYY.MM.dd-part0001.jsonl
```

NVC NetHub는 NAS의 `outputs/ap/...` JSONL을 PC `runtime/logs/raw/`로 동기화한 뒤, 동기화 탭의 `ELK Stuck Record`에서 날짜별 AP Count를 저장합니다.
AP Status의 `ELK Stuck Count`는 저장된 Record를 읽어 `누적`, `최고`, `평균` 방식으로 집계합니다.

---

## 파이프라인 섹션 맵

| 섹션 | 역할 | 보호 |
|------|------|:----:|
| 1 | `[meta][raw_message]` 백업, `drop_low_priority` 초기화 | |
| 2 | **Base syslog grok** — 8개 패턴으로 timestamp/host/process/log_message 추출 | ✅ |
| 3 | `event.module` 정규화 (process.name 우선) | |
| 3A | EFG 방화벽 사전분류 (CEF 없는 [RULE] 형태) | |
| 4 | **observer/dataset base** — `ap`/`unifi.ap` 기본, EFG 조건 시 `gateway`/`unifi.efg` | ✅ |
| 5 | event.module 보완 (dnsmasq, hostapd, kernel, stahtd 등) | |
| 6 | AP 이름 추출 — site code, floor, location, priority | |
| 7 | 공통 필드 추출 — client MAC/IP, radio channel/RSSI, VLAN, WLAN, kernel uptime | |
| 7A | EFG 방화벽 rule metadata (DESCR 추출) | |
| 7B | V27 RRM scan metadata — scan_id, interface, channel, method | |
| 8 | CEF 파싱 — `message` 전체에서 `zcef_*` 생성 | |
| 9 | **기본 event 필드 초기화** — action=unclassified, problem_class=low_priority | |
| 10 | RF/AP core 매핑 — low_rssi, retry_high, tx_overflow, channel_invalid, vap_timeout, radio_reset | |
| 10A | V24.1 DHCP — dhcp_request/ack/other/fail | |
| 10B | V24.3 QoS error 분리 | |
| 10C | V25.1 system_issue 분리 | |
| 10C2 | V27 RRM scan trigger 승격 → rrm_scan_risk | |
| 10D | V26 known-noise 분리 → noise index | |
| V25.2 | 상관분석 helper 필드 — correlation_stage/domain/blast_scope | |
| V27 | AP-stuck 타임라인 — rca_step 1~7, rca_stage, loop_family | |
| V27 라우팅 | storage_class 최종 결정 | |
| 11 | severity 정규화 | |
| 12 | 임시 필드 정리 | |
| 13 | timestamp 정규화 (America/Los_Angeles) | |
| 14 | V27 Ruby: 50 MiB 파트 파일 라우팅 | |
| output | ES + JSONL file + stdout (AP only) | ✅ |

---

## 보호된 baseline — 절대 수정 금지

```
input { udp { port => 51415 } }    ← 포트 변경 금지

섹션 2: Base syslog grok (8개 패턴)
  → 깨지면 log_message, device.name, process.name 모두 실패

섹션 4: observer/dataset base 로직
  → AP vs EFG 분리 기반

output { elasticsearch { ... } }   ← ES 인덱스 출력 구조
output { file { ... } }            ← JSONL 파일 출력
output { stdout { ... } }          ← AP-only 디버그
```

---

## RCA 모델

### 경로 1 (V25.2) — EFG 자원 장애

```
EFG system_issue (disk full / I/O error / memory failure)
  → QoS error (qos_control.sh 실패)
  → ap_stuck or ap_no_service
```

실증 근거: EFG `/boot/firmware` 디스크 full → syslog write error → AP QoS 실패 연쇄.

### 경로 2 (V27) — RRM 스캔 유발 AP Stuck

```
rrm_scan_trigger (iwpriv wifi1ap3 acsrrm)
  → radio_driver_retry (STA_VDEV_XRETRY)
  → tx_overflow (VAP TX queue overflow)
  → radio_reset (WAL_DBGID_DEV_RESET)
  → vap_timeout (osif_vap_init: Timeout waiting for DA vap 0 to stop)
  → channel_invalid (wlan_get_active_vport_chan: ch=0)
  → service_impact (qos_error or ap_no_service)
```

실증: `E2FAP3VISION` (UAP-AC-Pro-Gen2, fw 6.8.2.15592) — wifi1ap3 TX dropped: 107,185,583. 동일 패턴: `E2FAP6PCROOM`.

### V27 rca_step 1~7

| `rca_step` | `rca_stage` | event.action |
|:---:|---|---|
| 1 | `01_rrm_scan_trigger` | `rrm_scan_trigger` |
| 2 | `02_radio_driver_retry` | `retry_high` + STA_VDEV_XRETRY |
| 3 | `03_tx_overflow` | `tx_overflow` |
| 4 | `04_radio_reset` | `radio_reset` |
| 5 | `05_vap_stop_timeout` | `vap_timeout` |
| 6 | `06_channel_zero` | `channel_invalid` |
| 7 | `07_service_impact` | `qos_error` / `ap_no_service` |

---

## Event 분류 체계

### event.problem_class

| `problem_class` | tier | storage_class | 대표 `event.action` |
|---|:---:|:---:|---|
| `system_issue` | critical | ap | `system_error` |
| `rrm_scan_risk` | cause_candidate | ap | `rrm_scan_trigger` |
| `ap_stuck` | high | ap | `channel_invalid`, `vap_timeout`, `radio_reset` |
| `ap_no_service` | high | ap | `dhcp_fail`, `qos_error` |
| `rf_issue` | normal | ap | `low_rssi`, `retry_high`, `tx_overflow`, `anomaly_detected` |
| `low_priority` | low | low | `dhcp_request`, `dhcp_ack` 등 |
| `low_priority` (noise) | noise | noise | 고빈도 known-noise |

분석 순서: `system_issue` → `rrm_scan_risk` → `ap_stuck` → `ap_no_service`

### event.correlation_stage (V25.2)

| problem_class | correlation_stage | blast_scope |
|---|---|---|
| `system_issue` | cause | global |
| `rrm_scan_trigger` | possible_cause | local |
| retry_high (STA_VDEV_XRETRY) | precursor | local |
| tx_overflow | precursor | local |
| `ap_stuck` | symptom | local |
| `ap_no_service` / `qos_error` | impact | local |

---

## 감지 키워드

**system_issue (critical)**
```
No space left on device
I/O error
Suspended write operation because of an I/O error
Error suspend timeout has elapsed
syslog write error
buffer allocation failure
memory allocation failure
```

**rrm_scan_risk (V27)**
```
Trigger rrm scan(N): sleep N;iwpriv wifi1apN acsrrm CHANNEL
Trigger rrm scan(N): sleep N;iwpriv wifi1apN set ApScanChannel=active:CHANNEL:DURATION
iwpriv wifiXapY acsrrm
iwpriv wifiXapY ApScanChannel
```
추출 필드: `rrm.target_interface`, `rrm.target_channel`, `rrm.method`, `rrm.scan_id`

**ap_stuck (high)**
```
osif_vap_init: Timeout waiting for DA vap 0 to stop   → vap_timeout
wlan_get_active_vport_chan: ch=0                        → channel_invalid
WAL_DBGID_DEV_RESET / DEV_RESET                        → radio_reset
vap-0(wifi1ap3):TX OVERFLOW                            → tx_overflow
STA_VDEV_XRETRY / WAL_DBGID_STA_VDEV_XRETRY           → retry_high
```

**ap_no_service / QoS (high)**
```
qos_control.sh / tc class / qos_sta_control / RTNETLINK / cmd failed → qos_error
DHCPNAK(                                                               → dhcp_fail
```

---

## 핵심 필드 목록

| 필드 | 설명 |
|------|------|
| `message`, `log_message`, `[meta][raw_message]` | 원본 로그 보존 |
| `[observer][type]` | `ap` / `gateway` |
| `[event][dataset]` | `unifi.ap` / `unifi.efg` |
| `[event][action]` | 이벤트 분류 액션 |
| `[event][problem_class]` | RCA 분류 클래스 |
| `[event][tier]` | critical / cause_candidate / high / normal / low |
| `[event][storage_class]` | ap / low / noise |
| `[event][correlation_stage]` | cause / possible_cause / precursor / symptom / impact |
| `[event][rca_sequence]` | `rrm_to_ap_stuck` |
| `[event][rca_step]` | 1~7 |
| `[event][rca_stage]` | `01_rrm_scan_trigger` ~ `07_service_impact` |
| `[rrm][target_interface]` | 예: `wifi1ap3` |
| `[rrm][target_channel]` | 예: `161` |
| `[ap][name]`, `[ap][priority]` | AP 이름 및 우선순위 |
| `[site][code]`, `[site][name]` | M=Main, E=Education |

---

## AP 이름 체계

패턴: `{site}{floor}_AP{seq}_{location}` — 예: `M2F_AP3_PASTOR_A2`, `E2F_AP3_VISION`

| location 패턴 | ap.priority |
|---|---|
| PASTOR, OFFICE, R200 | critical |
| BONDANG, SANCTUARY, FELLOWSHIP, VIPROOM, PCROOM, ROOM | high |
| 그 외 | normal |

---

## 변경 원칙

1. **Small delta only** — new `if` 블록 추가, 기존 블록 수정 금지
2. **한 번에 1개 기능** — 복수 problem_class 동시 추가 금지
3. **unclassified guard 유지** — 새 rule은 `[event][action] == "unclassified"` 조건 필수
4. **기존 event.action 수정 금지** — 새 action 추가만 가능
5. **정상 lifecycle drop 금지** — low_priority 유지, 삭제 금지

### 새 problem_class 추가 패턴

```logstash
# 섹션 번호. 설명 (버전)
if [event][action] == "unclassified" and (
  [message] =~ /패턴1/ or
  [message] =~ /패턴2/
) {
  mutate {
    replace => {
      "[event][action]"        => "action_name"
      "[event][problem_class]" => "problem_class"
      "[event][tier]"          => "high"
      "[event][outcome]"       => "failure"
    }
  }
}
```

---

## Kibana 분석 가이드

### 5단계 분석 흐름

**Step 1 — system_issue 선행 확인**
```kql
event.problem_class:system_issue AND observer.type:gateway
```

**Step 2 — 통합 상관관계**
```kql
event.problem_class:(system_issue OR ap_stuck OR ap_no_service)
OR event.action:(qos_error OR rrm_scan_trigger)
```

**Step 3 — AP-stuck 타임라인**
```kql
event.rca_sequence:rrm_to_ap_stuck AND ap.name:"E2F_AP3_VISION"
```

**Step 4 — Lens 시계열**: 1분(즉시성) → 5분(실전 판단) → 10분(전체 파형)

**Step 5 — gateway/AP breakdown**
```kql
observer.type:gateway AND event.problem_class:system_issue
observer.type:ap AND event.problem_class:ap_stuck
```

### 해석 기준

| 패턴 | 해석 |
|------|------|
| system_issue spike 직후 1~5분 내 AP fault 증가 | EFG/system이 상위 원인 |
| 일부 구간만 같이 움직임 | system_issue는 일부 구간의 원인 |
| system_issue 없이도 AP fault 다수 | AP 자체 / RF / 서비스 계층 별도 분석 |
| rrm_scan_trigger 후 수분 내 vap_timeout/ch=0 | RRM이 AP stuck 원인 |

## NVC NetHub Record 연동

운영자가 AP Status에서 보는 `ELK Stuck` 값은 Logstash raw Log를 매번 직접 스캔한 값이 아니라, `ELK Stuck Record`에 저장된 날짜별 Count를 집계한 값입니다.

| 단계 | 위치 | 설명 |
|------|------|------|
| Log 생성 | NAS ELK | Logstash가 AP 관련 JSONL을 `outputs/ap/YYYY/M/D/`에 저장 |
| Log 동기화 | NVC NetHub 동기화 탭 | NAS JSONL을 PC `runtime/logs/raw/`로 복사 |
| Record 생성 | NVC NetHub 동기화 탭 | 날짜별 AP Count를 `runtime/reports/elk_stuck_record/elk_stuck_records.json`에 저장 |
| AP Status 반영 | NVC NetHub AP Status | 선택 날짜 범위와 집계 방식으로 `ELK Stuck` 컬럼 갱신 |

Logstash 분류가 바뀌면 `event.problem_class`, `event.action`, AP 이름 추출 결과가 Record Count에 직접 영향을 줄 수 있으므로, 파이프라인 변경 후에는 NVC NetHub에서 해당 날짜 Record를 다시 생성해 확인합니다.

---

## NAS 운영 명령어

```bash
# NAS SSH 후 root
sudo -i
cd /volume1/docker/elk-logstash/elk

# Config test
docker exec -it logstash /usr/share/logstash/bin/logstash -t -f /usr/share/logstash/pipeline/logstash.conf

# 재시작
docker compose restart logstash
# 또는
docker restart -t 30 logstash

# 로그 확인
docker logs --tail 100 -f logstash

# ES 인덱스 확인
curl -u elastic:<password> "http://localhost:9200/_cat/indices/unifi-network-*?v"

# 클러스터 상태
curl -u elastic:<password> http://localhost:9200/_cluster/health?pretty
```

### Kibana Dev Tools

```
# 현재 V27 인덱스 조회
GET /unifi-network-v27-ap-*/_search
{ "track_total_hits": true, "size": 10, "sort": [{ "@timestamp": "desc" }] }

# 구버전 인덱스 삭제 (패턴 확인 필수)
DELETE unifi-network-v26-ap-*
```

---

## 변경 절차

1. Kibana Discover에서 대상 로그 패턴 샘플 수집
2. Small delta 작성 — 기존 섹션 다음에 새 `if` 블록 추가
3. `docker exec -it logstash ... --config.test_and_exit` — config test
4. `docker compose restart logstash` — 적용
5. 검증 체크리스트 전체 통과
6. `elk/logstash/pipeline/logstash.conf` 파일 버전 헤더 업데이트

---

## 검증 체크리스트

변경 후 반드시 확인:

- [ ] Logstash config test 통과
- [ ] UDP `51415` 수신 정상
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

## V28 후보 (우선순위 순)

1. `auth_issue` — EAPOL timeout, auth fail, RADIUS reject (오탐 방지 최우선)
2. `roaming_issue` — FT decrypt fail, roaming failure
3. DNS/DHCP 세분화

V28 추가 시: 기존 블록 수정 금지, 새 `if` 블록을 기존 섹션 다음에 추가, 한 번에 1개, 검증 체크리스트 전체 통과.
