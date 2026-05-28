# ELK 문서 인덱스

이 폴더에는 ELK / Logstash / UniFi / EFG RCA 프로젝트의 현재 운영 문서가 있습니다.

## 시작 지점

| 문서 | 내용 |
|------|------|
| `../README.md` | 프로젝트 개요, V27 베이스라인, 파이프라인 구조 맵, RCA 모델 |
| `../CLAUDE.md` | Logstash 작업 시 AI 지침 (보호 구간, 변경 원칙, 다음 V28 후보) |
| `../RUNBOOK.md` | NAS SSH 명령어, Docker/Logstash 운영 명령 모음 |
| `logstash-rules.md` | 활성 파이프라인 규칙 — 필드 전체 목록, 감지 키워드, 이벤트 매핑 |
| `nas-setup.md` | NAS Docker 배포 및 검증 절차 |
| `efg-settings.md` | EFG 설정 요약 — VLAN, WiFi, 방화벽, 시스로그 포트 등 |
| `root_cause_kr.md` | RCA 분석 상세 — E2FAP3VISION 사례, stamgr hang 증거, 검증 단계 |

## V27 작업 순서

1. `../README.md` — V27 베이스라인, 파이프라인 섹션 맵, 보호 구간 확인
2. `../logstash/pipeline/logstash.conf` — 활성 파이프라인 코드 확인
3. `logstash-rules.md` — 현재 필드/분류 규칙 상세 확인
4. `efg-settings.md` — VLAN, WiFi, 방화벽 컨텍스트 필요 시 참조
5. `root_cause_kr.md` — E2FAP3VISION RCA 상세 및 다음 검증 단계 참조

## 현재 분석 초점

V27은 V25.2 상관분석 모델과 V26 인덱스 분리를 유지하면서,
RRM/채널스캔 트리거를 AP-focus 인덱스로 승격하여 AP-stuck 타임라인을 추적 가능하게 했습니다.

**확정된 두 가지 RCA 경로:**

```
system_issue → qos_error / service impact → ap_stuck or ap_no_service
```
```
rrm_scan_trigger → radio_driver_retry → tx_overflow → radio_reset
  → vap_timeout → channel_invalid → service_impact
```

**현재 인덱스 그룹:**

```
unifi-network-v27-ap-*     ← AP/RCA 중심 이벤트
unifi-network-v27-low-*    ← 저우선순위 컨텍스트
unifi-network-v27-noise-*  ← known-noise
```

**Kibana 권장 분석 흐름:**

- Discover: 원시 시간순 이벤트 확인
- Lens: 1분/5분/10분 window 시계열 비교
- Breakdown: `observer.type`, `device.name`, `event.module`, `ap.name`, `network.vlan`
- AP stuck 분석: `event.rca_step`, `event.rca_stage`, `rrm.target_interface`, `rrm.target_channel`, `kernel.uptime_seconds`

## 과거 참조 (아카이브)

프로젝트 V24~V25.2 설계 이력은 `codex/docs_org/ORG/` 에 보존되어 있습니다.
현재 운영에는 참조할 필요 없으며, 설계 의도나 변경 이력이 궁금할 때만 참조합니다.
