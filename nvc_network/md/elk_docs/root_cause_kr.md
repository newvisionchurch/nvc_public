# Root Cause 작업 노트

업데이트: 2026-04-28 07:50:47 PDT (-07:00)

이 문서는 반복적으로 발생하는 UniFi AP stuck 문제에 대한 현재 Root Cause
가설과 증거를 정리한 문서입니다. 기준 자료는 이전 V26 Logstash AP/low sample
output, UniFi Network UI 화면, 그리고 `E2FAP3VISION`에 대한 live SSH 확인
결과입니다. V27은 다음 관찰 구간을 위해 RRM/AP-stuck timeline field를
추가합니다.

## 현재 상태

이 문제는 단순히 AP를 reboot하면 끝나는 문제가 아닙니다. 이미 여러 AP에서
reboot 또는 reset 후에는 정상으로 돌아오는 것이 확인되었습니다. 따라서 현재
RCA의 목적은 "어떻게 복구할 것인가"가 아니라, **왜 AP가 online 상태로 보이면서
service가 죽는 stuck 상태에 들어가는가**를 찾는 것입니다.

현재 가장 좋은 분석 대상은 `E2FAP3VISION`입니다.

- 장비명: `E2FAP3VISION`
- UniFi 이름: `E2F_AP3_VISION`
- IP: `192.168.11.125`
- 모델: `UAP-AC-Pro-Gen2`
- 펌웨어: `6.8.2.15592`
- MAC: `80:2a:a8:53:95:3d`
- 위치: Education / 2F / Vision
- 분석 시점 uptime: 약 38일
- 증상: AP는 controller에 connected 상태로 보이지만, client 수가 거의 0명이고
  5 GHz VAP가 실질적으로 stuck 상태입니다.

## 확인된 증상

이전 V26 AP sample output 기준으로, 샘플 시간 구간에서 `E2FAP3VISION`과
`E2FAP6PCROOM`이 AP stuck 이벤트 대부분을 차지합니다.

이전 V26 sample file
`samples/outputs/ap/unifi-network-v26-ap-2026.04.28.jsonl` 기준 주요 집계:

| AP | 전체 AP 이벤트 | AP Stuck | Radio Reset | VAP Timeout | Channel Invalid |
| --- | ---: | ---: | ---: | ---: | ---: |
| `E2FAP3VISION` | 487 | 368 | 27 | 185 | 156 |
| `E2FAP6PCROOM` | 454 | 359 | 21 | 186 | 152 |

주요 stuck signature는 다음과 같습니다.

- `osif_vap_init: Timeout waiting for DA vap 0 to stop`
- `wlan_get_active_vport_chan: ch=0`
- `WAL_DBGID_DEV_RESET`
- `vap-0(wifi1ap3):TX OVERFLOW`

`vap_timeout`은 약 9-10초 간격으로 반복되고, 많은 경우 그 직후 `ch=0`
메시지가 쌍으로 따라옵니다.

## Live SSH 증거

`E2FAP3VISION`에 SSH로 접속하여 AP가 실제로 bad state에 머물러 있음을
확인했습니다.

기본 정보:

```text
Model:       UAP-AC-Pro-Gen2
Version:     6.8.2.15592
IP Address:  192.168.11.125
Hostname:    E2FAP3VISION
Status:      Connected
```

테스트 중 5 GHz 채널을 non-DFS channel 36으로 변경한 뒤의 설정:

```text
radio.2.phyname=wifi1
radio.2.channel=36
radio.2.ieee_mode=11naht40
radio.2.acsdfs=enabled
radio.2.devname=wifi1ap3
radio.2.status=enabled
```

문제의 중심 VAP는 `wifi1ap3`입니다. 이 VAP는 5 GHz의 주요 SSID/VAP로
보이며, stuck loop와 직접 연결되어 있습니다.

```text
wireless.4.devname=wifi1ap3
wireless.4.parent=wifi1
qos.vap.4.devname=wifi1ap3
bridge.2.port.1.devname=wifi1ap3
```

interface 자체는 존재하지만 counter가 심각한 실패 상태를 보여줍니다.

```text
wifi1ap3 TX errors: 435444
wifi1ap3 TX dropped: 107185583
```

5 GHz를 DFS channel 112에서 non-DFS channel 36으로 변경한 뒤에도 kernel
로그는 계속 같은 stuck pattern을 출력했습니다.

```text
UBNT-AP-UPDATE wifi1: band(1), channel(36)
osif_vap_init: Timeout waiting for DA vap 0 to stop
wlan_get_active_vport_chan: ch=0
vap-0(wifi1ap3):TX OVERFLOW
```

따라서 DFS channel 112는 trigger 또는 contributing condition일 수는 있지만,
이미 stuck에 빠진 AP는 channel 변경만으로 회복되지 않습니다. runtime wireless
driver/VAP state가 reboot 전까지 계속 hung 상태로 남아 있습니다.

## Process State 증거

Live process 확인에서 `stamgr`가 kernel ioctl 경로에서 stuck된 것이
확인되었습니다.

```text
Name:   stamgr
State:  D (disk sleep)
wchan: dev_ioctl
cmd:    /bin/stamgr -i 1 -c 30 -b ng -r 15 -n 0 -K
```

이것은 AP가 단순히 busy한 user-space 상태가 아니라, UniFi wireless 관리
프로세스가 kernel wireless driver ioctl 응답을 기다리며 빠져나오지 못하고
있다는 강한 증거입니다.

조사 중 `scan-rrm-check` 프로세스들도 보였지만, `/proc/<pid>/cmdline`을 읽기
전에 종료되었습니다. 다만 아래 low-log 증거와 흐름이 일치합니다.

## Low Log 상관관계

이전 V26 low output에는 AP-focus index에 보이지 않던 RRM scan trigger들이
많이 남아 있습니다. 당시 이 이벤트들은 low-priority context로 분류되어
있었기 때문에 RCA 화면에서는 묻혀 있었습니다.

관련 예:

```text
E2FAP3VISION syswrapper: Trigger rrm scan(...): ... iwpriv wifi1ap3 acsrrm ...
E2FAP6PCROOM syswrapper: Trigger rrm scan(...): ... iwpriv wifi1ap3 acsrrm ...
```

`E2FAP3VISION`에서 관찰된 `wifi1ap3` 대상 RRM scan channel 예:

- `161`
- `48`
- `149`
- `108`
- `64`

같은 mechanism이 `E2FAP6PCROOM` 등 다른 `UAP-AC-Pro-Gen2` AP에서도
나타납니다. 이 AP들도 동일한 AP-stuck pattern을 보입니다.

## 현재 Root Cause 가설

현재 가장 강한 가설은 다음과 같습니다.

`UAP-AC-Pro-Gen2` 펌웨어 `6.8.2.15592`에서 UniFi RRM/channel scan 로직이
5 GHz VAP, 특히 `wifi1ap3`에 대해 `iwpriv ... acsrrm`를 반복 실행할 때,
wireless driver ioctl 경로가 hang될 수 있습니다. 이때 `stamgr`가 `dev_ioctl`
에서 block되고, AP는 `osif_vap_init` / `ch=0` / `TX OVERFLOW` loop에
들어갑니다. AP는 controller에는 계속 connected로 보이지만, client가 거의
0명으로 떨어지고 실제 WiFi service는 회복되지 않습니다. reboot는 runtime
driver state를 초기화하기 때문에 임시 복구가 됩니다.

요약 흐름:

```text
RRM scan / acsrrm
-> wifi1ap3 driver ioctl hang
-> stamgr D state in dev_ioctl
-> VAP stop/init loop
-> ch=0 + TX overflow
-> AP online, but no effective 5 GHz service
-> reboot temporarily clears runtime driver state
```

## Reboot가 도움이 되지만 Root Cause가 아닌 이유

Reboot는 AP의 runtime radio/driver state를 초기화합니다. 그래서 AP가 다시
정상처럼 보입니다. 하지만 이것은 root cause를 제거한 것이 아니라, stuck된
상태를 지운 것입니다.

여러 AP에서 같은 현상이 반복된다는 점은 단일 AP hardware 고장보다 공통 trigger
가능성을 더 강하게 보여줍니다.

- 동일하거나 유사한 모델: `UAP-AC-Pro-Gen2`
- 동일 펌웨어 계열: `6.8.2`
- 동일 5 GHz scan mechanism: `iwpriv ... acsrrm`
- 동일 stuck symptom: `vap_timeout`, `ch=0`, `TX OVERFLOW`
- 동일 운영 결과: AP는 online, client는 거의 0명

## 현재로서는 Primary Cause 가능성이 낮은 것

다음 항목들은 보조 요인일 수는 있지만, 현재 evidence 기준으로 primary root
cause로 보기는 어렵습니다.

- EFG/firewall drop: V26 noise split에서 high-volume noise로 확인되었지만,
  AP 내부 `stamgr`가 wireless driver ioctl에서 stuck되는 현상을 설명하지는
  못합니다.
- 특정 client 1대 문제: `E1FAP2WISDOM`에는 특정 client의 `low_rssi` 반복이
  있지만, 이는 별도 RF/client 문제이며 이번 AP-stuck failure mode와 다릅니다.
- DFS channel 112 단독 원인: channel 36으로 변경해도 이미 stuck된 AP는
  회복되지 않았습니다. DFS는 trigger일 수 있으나 전체 설명은 아닙니다.
- 일반 RF retry: 다른 AP들의 retry/high anomaly는 context로 중요하지만,
  `osif_vap_init`과 `dev_ioctl` hang을 직접 설명하기에는 부족합니다.

## 다음 검증 단계

서비스 복구가 급하지 않다면, 충분한 증거를 수집하기 전까지 affected AP를 바로
reboot하지 않는 것이 좋습니다.

권장 테스트:

1. UniFi에서 RRM/channel scan을 유발하는 기능을 찾고, 작은 test group에서
   비활성화합니다.
   - Radio AI
   - WiFi AI
   - Nightly Channel Optimization
   - Auto Channel Optimization
   - Channel Optimization
   - RRM
   - Auto Optimize Network
2. `E2FAP3VISION`, `E2FAP6PCROOM`은 5 GHz channel을 non-DFS 고정 채널로
   유지합니다. 예: channel 36 또는 44, width 40 MHz.
3. 현재 증거를 충분히 저장한 뒤, test AP 1대만 reboot하여 clean observation
   window를 시작합니다.
4. 이후 24-72시간 동안 새 `Trigger rrm scan`과 새 AP-stuck event가 발생하는지
   관찰합니다.
5. 기존 optimization/RRM 설정을 유지한 control AP와 비교합니다.

## Logstash V27 구현 완료 (✅ 현재 베이스라인)

V26의 AP/low/noise 분리가 효과적이었고, V27에서 다음이 모두 구현되었습니다.

**승격된 RCA 이벤트 (V27 구현 완료):**
- `Trigger rrm scan`, `scan-rrm-check`, `iwpriv ... acsrrm`, `iwpriv ... ApScanChannel`
  → `event.action = rrm_scan_trigger`, `event.problem_class = rrm_scan_risk`

**추출 필드 (V27 구현 완료):**
- `rrm.target_interface` (예: `wifi1ap3`)
- `rrm.target_channel` (예: `161`)
- `rrm.method` (`acsrrm` 또는 `ApScanChannel`)
- `rrm.scan_id`, `rrm.scan_mode`, `rrm.scan_duration_ms`

**AP-stuck 타임라인 필드 (V27 구현 완료):**
```
event.rca_sequence = rrm_to_ap_stuck
event.rca_step = 1..7   (rrm_scan_trigger → service_impact)
event.rca_stage = 01_rrm_scan_trigger .. 07_service_impact
```

현재 Kibana에서 아래 흐름을 추적할 수 있습니다:
```
rrm_scan_trigger (rca_step=1) → vap_timeout (step=5) → channel_invalid (step=6)
→ AP online but no clients
```

**다음 단계 (V28 후보):** `auth_issue` 분리, `roaming_issue` 분리, RRM 비활성화 효과 검증

## Kibana 상관분석 방법론

V25.2에서 설계한 `system_issue ↔ AP fault` 상관분석 절차이다.
V27 JSONL 데이터 및 Kibana에서 동일하게 적용한다.

### 분석 단계

**Step 1 — Discover에서 대표 구간 선정**
```text
event.problem_class:(system_issue OR ap_stuck OR ap_no_service)
OR event.action:(channel_invalid OR vap_timeout OR radio_reset OR qos_error)
```
→ `system_issue` 발생 시각 3~5개 대표 구간 선정, 각 구간 ±10분 범위 확인

**Step 2 — Lens 시계열 (1분 window)**

| Series | KQL |
|--------|-----|
| A | `event.problem_class:system_issue` |
| B | `event.problem_class:ap_stuck` |
| C | `event.problem_class:ap_no_service` |
| D | `event.action:qos_error` |

→ system_issue spike 직후 B/C/D가 따라 올라오는지 확인

**Step 3 — Time Window 비교 원칙**

| Window | 용도 |
|--------|------|
| 1분 | 즉시성 — 순간 spike / 직후 반응 확인 |
| 5분 | **실전 판단 기준** — 노이즈 제거, 실제 후행 여부 |
| 10분 | 전체 장애 cluster 파형 확인 |

→ 최종 판단은 **5분 window** 중심, 1분/10분은 보조 검증

**Step 4 — Action Breakdown**

```text
event.action:(channel_invalid OR vap_timeout OR radio_reset OR qos_error)
```
→ ap_stuck 내부에서 어떤 action이 system_issue와 가장 강하게 연결되는지 확인

**Step 5 — Observer/Module Breakdown**

```text
observer.type:gateway AND event.problem_class:system_issue
```
→ system_issue가 gateway(EFG)에서 먼저 시작되는지 확인
→ `event.module`(syslog-ng, kernel, dnsmasq)이 어떤 계층에서 선행하는지 확인

### 해석 기준

| 패턴 | 해석 |
|------|------|
| system_issue spike 직후 1~5분 내 AP fault 반복 증가, gateway module 선행 | EFG/system이 상위 원인 — 강한 상관 |
| 일부 구간만 같이 움직임, system_issue 없이도 AP fault 발생 | system_issue는 일부 원인 — 약한 상관 |
| system_issue와 AP fault 시간적으로 분리 | 별도 원인 축 (RF, AP 자체, 서비스 계층) 검토 |

## 현재 RCA 신뢰도

현재 confidence:

- failure mechanism: 높음
- trigger hypothesis: 중상

확인된 mechanism:

- `wifi1ap3` VAP stuck
- kernel `ch=0`
- 반복되는 VAP stop/init timeout
- `stamgr`가 `dev_ioctl`에서 block
- reboot 시 runtime state 초기화로 임시 복구

아직 controlled validation이 필요한 항목:

- RRM/channel optimization을 끄면 재발이 멈추는지
- 이 문제가 `UAP-AC-Pro-Gen2` 펌웨어 `6.8.2`에 집중되는지
- `NVCampus`/`wifi1ap3` QoS, band steering, AP group 설정이 hang 가능성을
  높이는지
