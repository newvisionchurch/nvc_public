# Logstash Rules

This document describes the active Logstash parsing and classification behavior
for the V27 UniFi / EFG RCA baseline.

Active file:

- `elk/logstash/pipeline/logstash.conf`

## Design Goals

- Preserve the raw message in `[meta][raw_message]`.
- Keep AP and EFG logs in one searchable index family.
- Separate AP symptoms from real root-cause candidates.
- Keep normal lifecycle events for evidence; filter them in Kibana.
- Split AP-focus, low-priority, and known-noise events into separate indices.
- Promote RRM/channel-scan trigger events into AP-focus analysis when they can
  explain AP stuck.
- Add AP-stuck sequence fields so time-ordered analysis can follow trigger,
  driver stress, reset, VAP timeout, invalid channel, and service impact.
- Use small, safe deltas instead of broad rewrites.

## Protected Baseline

Do not rewrite these areas without a specific reason and validation plan:

- UDP input on port `51415`.
- Base syslog grok parsing.
- AP vs gateway observer/dataset separation.
- CEF extraction into `zcef_*` fields.
- Output index behavior.

## Core Fields

Important fields used by the current pipeline:

- `message`
- `log_message`
- `[meta][raw_message]`
- `[meta][source_type]`
- `[meta][cef_raw]`
- `@timestamp`
- `[observer][type]`
- `[event][dataset]`
- `[event][module]`
- `[event][action]`
- `[event][category]`
- `[event][severity]`
- `[event][outcome]`
- `[event][problem_class]`
- `[event][problem_name]`
- `[event][tier]`
- `[event][reason]`
- `[event][correlation_stage]`
- `[event][correlation_domain]`
- `[event][blast_scope]`
- `[event][correlation_target]`
- `[event][storage_class]`
- `[event][visibility]`
- `[event][noise_class]`
- `[event][rca_sequence]`
- `[event][rca_step]`
- `[event][rca_stage]`
- `[event][rca_stage_name]`
- `[event][loop_family]`
- `[event][periodic_signal]`
- `[device][name]`
- `[device][info]`
- `[process][name]`
- `[process][pid]`
- `[ap][name]`
- `[ap][sequence]`
- `[ap][priority]`
- `[site][code]`
- `[site][name]`
- `floor`
- `[location][name]`
- `[location][type]`
- `[client][mac]`
- `[client][ip]`
- `[client][hostname]`
- `[network][name]`
- `[network][ssid]`
- `[network][subnet]`
- `[network][vlan]`
- `[network][bridge]`
- `[radio][band]`
- `[radio][channel]`
- `[radio][channel_width]`
- `[radio][rssi]`
- `[radio][index]`
- `[radio][fwlog_id]`
- `[radio][driver_event]`
- `[wlan][interface]`
- `[kernel][uptime_seconds]`
- `[rrm][scan_id]`
- `[rrm][method]`
- `[rrm][target_interface]`
- `[rrm][target_channel]`
- `[rrm][scan_mode]`
- `[rrm][scan_duration_ms]`
- `[rrm][pre_sleep_seconds]`
- `[rrm][post_sleep_seconds]`

## Observer and Dataset

Default classification is AP:

- `[observer][type] = ap`
- `[event][dataset] = unifi.ap`

Gateway/EFG classification is used when the message indicates CEF, EFG,
`dnsmasq`, or `ubios-udapi-server`:

- `[observer][type] = gateway`
- `[event][dataset] = unifi.efg`

## AP Naming

The pipeline extracts AP names with these patterns:

- `M2F_AP3_PASTOR_A2`
- `M1F_AP5_BONDANG`
- `E1F_AP6_NEWSONG`
- `E2F_AP6_PC_ROOM`
- legacy compact forms such as `M2FAP3...`

Site mapping:

- `M` -> `Main`
- `E` -> `Education`

Location and AP priority are derived from names. Pastor, office, and Room 200
areas are treated as higher priority analysis targets.

## Current Event Mapping

### RRM / Channel Scan Risk

V27 promotes these events from low-priority context into the AP-focus index:

- `Trigger rrm scan`
- `scan-rrm-check`
- `iwpriv ... acsrrm`
- `iwpriv ... ApScanChannel`

Mapped to:

- `[event][action] = rrm_scan_trigger`
- `[event][problem_class] = rrm_scan_risk`
- `[event][category] = rf`
- `[event][tier] = cause_candidate`
- `[event][storage_class] = ap`
- `[event][visibility] = ap_focus`
- `[event][reason] = rrm_or_channel_scan_trigger`

The parser extracts RRM metadata when present:

- `[rrm][target_interface]`, for example `wifi1ap3`
- `[rrm][target_channel]`, for example `161`
- `[rrm][method]`, such as `acsrrm` or `ApScanChannel`
- `[rrm][scan_id]`, `[rrm][scan_mode]`, and scan sleep/duration fields

This change is based on live RCA evidence where repeated `iwpriv wifi1ap3
acsrrm ...` scans preceded or coexisted with `osif_vap_init`, `ch=0`, and
`TX OVERFLOW` loops on `UAP-AC-Pro-Gen2` firmware `6.8.2`.

### RF Issue

Mapped to `[event][problem_class] = rf_issue`:

- `low_rssi`
- `retry_high`
- `low_phy_rate`
- `tx_overflow`
- `anomaly_detected`

These are wireless quality problems. Do not mix them with service failure unless
the time series also shows DHCP, DNS, QoS, policy, or system evidence.

For AP-stuck sequencing, V27 marks firmware-level `STA_VDEV_XRETRY` events as
step 2 in the AP-stuck timeline. Client-level retry/anomaly events remain RF
context unless their time window aligns with AP-stuck symptoms.

### AP Stuck

Mapped to `[event][problem_class] = ap_stuck`:

- `channel_invalid`
- `vap_timeout`
- `radio_reset`

These are AP/radio symptoms. In V25.2 they must be compared against preceding
EFG/system events before concluding that the AP itself is the root cause.

V27 adds timeline fields to these symptoms so they can be viewed in order
with preceding `rrm_scan_trigger` events.

### DHCP and Service

Supporting DHCP events:

- `dhcp_request`
- `dhcp_ack`
- `dhcp_other`

Failure event:

- `dhcp_fail` -> `[event][problem_class] = ap_no_service`

### QoS Error

Mapped to:

- `[event][action] = qos_error`
- `[event][problem_class] = ap_no_service`
- `[event][category] = network_service`
- `[event][tier] = high`
- `[event][outcome] = failure`
- `[event][reason] = qos_control_failure`

Detection patterns include `qos_control.sh`, `tc class`, `qos_sta_control`,
`cmd failed`, and `RTNETLINK`.

### System Issue

Mapped to:

- `[event][action] = system_error`
- `[event][problem_class] = system_issue`
- `[event][category] = system`
- `[event][tier] = critical`
- `[event][outcome] = failure`
- `[event][reason] = system_resource_failure`

Detection patterns include:

- `No space left on device`
- `I/O error`
- `syslog write error`
- `Suspended write operation because of an I/O error`
- `Error suspend timeout has elapsed`
- `buffer allocation failure`
- `failed to allocate reassemble cont.`
- `memory allocation failure`

This is the top-priority RCA class. V25.2 analysis found that EFG system resource
failure can appear upstream of QoS errors and AP symptoms.

### Known Noise

V26/V27 keeps these events but routes them to the `noise` index:

- `firewall_no_rule_description`
- `wan_in_default_drop`
- `ubnt_neighbor_update_ess_ap`

These patterns were very high-volume in `samples/raw` and were previously
filtered out by debug export scripts. V26 moved that decision into Logstash
classification and index routing instead of deleting the events.

Known-noise events are normalized to `event.severity:info` and
`event.outcome:success` after severity parsing, because they are expected
background traffic for this RCA workflow.

## V25.2 / V27 Correlation Fields

The pipeline adds helper fields for events that should participate in Kibana
correlation analysis.

For `system_issue`:

- `[event][correlation_stage] = cause`
- `[event][correlation_domain] = system`
- `[event][blast_scope] = global`
- `[event][correlation_target] = true`

For `ap_stuck`:

- `[event][correlation_stage] = symptom`
- `[event][correlation_domain] = ap`
- `[event][blast_scope] = local`
- `[event][correlation_target] = true`

For `ap_no_service` and `qos_error`:

- `[event][correlation_stage] = impact`
- `[event][correlation_domain] = service`
- `[event][blast_scope] = local`
- `[event][correlation_target] = true`

For `rrm_scan_trigger`:

- `[event][correlation_stage] = possible_cause`
- `[event][correlation_domain] = ap`
- `[event][blast_scope] = local`
- `[event][correlation_target] = true`

For AP-stuck RF precursors such as firmware `STA_VDEV_XRETRY` and
`tx_overflow`:

- `[event][correlation_stage] = precursor`
- `[event][correlation_domain] = ap`
- `[event][blast_scope] = local`
- `[event][correlation_target] = true`

## V27 AP-Stuck Timeline

V27 adds an ordered RCA sequence named `rrm_to_ap_stuck`.

| `event.rca_step` | `event.rca_stage` | Event action | Meaning |
| ---: | --- | --- | --- |
| 1 | `01_rrm_scan_trigger` | `rrm_scan_trigger` | RRM/channel scan trigger candidate |
| 2 | `02_radio_driver_retry` | `retry_high` with `STA_VDEV_XRETRY` | Radio firmware retry burst |
| 3 | `03_tx_overflow` | `tx_overflow` | VAP TX queue overflow |
| 4 | `04_radio_reset` | `radio_reset` | Radio firmware reset |
| 5 | `05_vap_stop_timeout` | `vap_timeout` | VAP stop/init timeout |
| 6 | `06_channel_zero` | `channel_invalid` | Active VAP channel becomes `ch=0` |
| 7 | `07_service_impact` | `qos_error` or `ap_no_service` | Service impact symptom |

Use this sequence in Discover by sorting by `@timestamp` and inspecting
`event.rca_step`, `event.rca_stage`, `ap.name`, `rrm.target_interface`,
`rrm.target_channel`, `wlan.interface`, `radio.driver_event`, and
`kernel.uptime_seconds`.

## V27 Storage Classes

V27 routes all events to one of three index groups:

| `event.storage_class` | Index pattern | Purpose |
| --- | --- | --- |
| `ap` | `unifi-network-v27-ap-*` | AP/RCA focus events |
| `low` | `unifi-network-v27-low-*` | Supporting low-priority context |
| `noise` | `unifi-network-v27-noise-*` | Known high-volume noise |

The Logstash `stdout` debug output is limited to `event.storage_class:ap`.
All events are still written to Elasticsearch.

V27 also writes local JSONL files through the Logstash `file` output:

| `event.storage_class` | Container path |
| --- | --- |
| `ap` | `/usr/share/logstash/outputs/ap/YYYY/M/D/unifi-network-v27-ap-YYYY.MM.dd-part0001.jsonl` |
| `low` | `/usr/share/logstash/outputs/low/YYYY/M/D/unifi-network-v27-low-YYYY.MM.dd-part0001.jsonl` |
| `noise` | `/usr/share/logstash/outputs/noise/YYYY/M/D/unifi-network-v27-noise-YYYY.MM.dd-part0001.jsonl` |

The host path is the `outputs` volume mounted in `elk/compose.yaml`.
V27 creates approximate 50 MiB part files for all three storage classes. Part
numbers increase as each file approaches the size limit. The `YYYY/M/D` folders
keep long-running NAS output directories from becoming too large.

## CEF / zcef

The pipeline extracts CEF payloads from the full original `message`, not only
from `log_message`, because EFG samples may keep the full CEF data in the raw
message while the parsed message only contains the trailing description.

CEF key-value data is prefixed as `zcef_*`, then copied into common fields such
as `client.mac`, `client.ip`, `network.vlan`, `radio.channel`, and `radio.rssi`
when present.

## Kibana Queries

Use these KQL fragments as starting points:

```text
event.problem_class:system_issue
```

```text
event.problem_class:ap_stuck
```

```text
event.problem_class:ap_no_service
```

```text
event.action:qos_error
```

```text
event.action:rrm_scan_trigger
```

```text
event.problem_class:rrm_scan_risk
```

```text
event.rca_sequence:rrm_to_ap_stuck
```

```text
event.storage_class:ap AND event.rca_step:*
```

```text
event.storage_class:ap
```

```text
event.storage_class:low
```

```text
event.storage_class:noise
```

```text
event.problem_class:(system_issue OR ap_stuck OR ap_no_service)
OR event.action:(qos_error OR rrm_scan_trigger)
```

```text
event.outcome:failure AND event.tier:(critical OR high)
```

## AP Stuck 감지 키워드 참조

Logstash 파이프라인에서 AP Stuck 관련 이벤트를 탐지하는 원본 로그 패턴이다.

### ap_stuck 직접 패턴
```text
osif_vap_init: Timeout waiting for DA vap 0 to stop   → vap_timeout
wlan_get_active_vport_chan: ch=0                        → channel_invalid
WAL_DBGID_DEV_RESET                                     → radio_reset
DEV_RESET                                               → radio_reset
vap-0(wifiXapY):TX OVERFLOW                            → tx_overflow
STA_VDEV_XRETRY                                         → retry_high (rca_step 2)
```

### rrm_scan_risk 패턴 (V27 AP-focus 승격)
```text
Trigger rrm scan                   → rrm_scan_trigger
scan-rrm-check                     → rrm_scan_trigger
iwpriv wifiXapY acsrrm             → rrm_scan_trigger
iwpriv wifiXapY ApScanChannel      → rrm_scan_trigger
```

### system_issue 패턴 (최상위 RCA)
```text
No space left on device
I/O error occurred while writing
Suspended write operation because of an I/O error
Error suspend timeout has elapsed
failed to allocate reassemble cont.
memory allocation failure
buffer allocation failure
syslog-ng I/O error
```

### ap_no_service / QoS 패턴
```text
qos_control.sh                     → qos_error
tc class                           → qos_error
qos_sta_control                    → qos_error
RTNETLINK answers: No such file    → qos_error
cmd failed                         → qos_error
```

## Event Tier 정의

| tier | 의미 | 해당 problem_class |
|------|------|--------------------|
| `critical` | 최상위 RCA 후보 — 즉시 확인 필수 | `system_issue` |
| `cause_candidate` | AP stuck의 선행 원인 후보 | `rrm_scan_risk` |
| `high` | 서비스 영향 있는 장애 증상 | `ap_stuck`, `ap_no_service`, `qos_error`, `dhcp_fail` |
| `normal` | 무선 품질 문제 (서비스 실패와 구분) | `rf_issue`, `auth_issue` |
| `low` | 정상 lifecycle / 분석 컨텍스트용 | `low_priority` |

## Validation Checklist

After any rule change, verify:

- Logstash config test passes.
- UDP `51415` still receives EFG syslog.
- `message` and `[meta][raw_message]` are preserved.
- `observer.type` separates `gateway` from `ap`.
- `event.dataset` separates `unifi.efg` from `unifi.ap`.
- CEF events still generate `zcef_*`.
- `system_issue` maps to critical failure.
- `qos_error` maps to service impact.
- `rrm_scan_trigger` maps to AP focus with `event.problem_class:rrm_scan_risk`.
- RRM metadata fields such as `rrm.target_interface` and
  `rrm.target_channel` are populated when present.
- AP stuck timeline fields such as `event.rca_step` and `event.rca_stage` are
  populated for stages 1 through 7.
- AP stuck actions still map correctly.
- Known noise routes to `event.storage_class:noise`.
- Low-priority context routes to `event.storage_class:low`.
- RCA focus events route to `event.storage_class:ap`.
- JSONL files are written under `outputs/{ap,low,noise}/YYYY/M/D/`.
- AP, low, and noise JSONL files roll to new `partNNNN` files at approximately
  50 MiB.
- Kibana Discover and Lens can compare 1, 5, and 10 minute windows.
