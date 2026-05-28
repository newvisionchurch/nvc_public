<!-- 현재 상태 (2026-05-25) ─────────────────────────────────────────
작성 시점: V24 (2026-04)  |  현재 버전: V27

V24에서 설계한 필드 체계가 V27까지 다음과 같이 구현되었습니다:

  구현 완료:
    event.problem_class  — system_issue/ap_stuck/ap_no_service/rf_issue 등 분류
    event.storage_class  — ap/low/noise 인덱스 라우팅 (V26 추가)
    event.rca_step       — AP-stuck 타임라인 단계 1~7 (V27 추가)
    event.rca_stage      — trigger/symptom/service_impact (V27 추가)

  설계 당시 미구현 → 현재 상태:
    event.problem_name   — 미구현 (event.action으로 충분히 세분화)
    event.tier           — 부분 구현 (critical/high/normal/low)
    auth_issue/roaming_issue — 미구현 (V25 로드맵에서 보류)

활성 파이프라인: elk/logstash/pipeline/logstash.conf
──────────────────────────────────────────────────────────────── -->

📌 Logstash Field & Source Design (V24 업데이트)

본 문서는 V24 목표 설계 문서이며, 현재 conf는 RF/AP core 우선 반영 상태이다.
DHCP/DNS/Policy/System issue 확장은 다음 단계에서 순차 적용한다.

1. 목적 (V24 기준 재정의)
EFG 및 UniFi AP 로그를 구조화하여
단순 장애 분류가 아니라 다음을 가능하게 한다:
증상 (AP Stuck / No Service)
→ Root Cause 분리
→ EFG / AP / RF / DHCP / DNS / Policy / QoS / System
→ 실제 원인을 증명 가능한 구조로 식별

👉 V23: 구조화
👉 V24: Root Cause 분리 및 증명

2. 핵심 변경 (V24에서 추가된 개념)
기존 (유지)
event.action 중심 분석
AP + EFG 통합
raw message 유지
V24 추가 (핵심)
event.action  → "무슨 일이 일어났는가"
event.problem_class → "문제의 종류 (Root Cause 계층)"
event.problem_name → "구체적인 원인"
event.tier → "중요도"

👉 기존 설계는 유지
👉 Root Cause 해석 계층을 위에 추가

3. 설계 원칙 (V24 강화)
AP + EFG 통합 구조 유지
raw message 유지
event.action 필수
Root Cause 분리 가능 구조
RF / DHCP / DNS / Policy / QoS 동시 분석 가능
정상 lifecycle은 아래로 분리

4. 분석 대상 (변경 없음, 중요)
4.1 EFG
DHCP / DNS
Policy Engine
QoS / Traffic
IDS/IPS
system / logging / resource
4.2 AP
RF / RSSI / Retry
Roaming
Radio 상태
AP 장애 패턴

5. 데이터 구조 (V24 확장)
기존 유지
event.*
ap.*
client.*
network.*
radio.*
site.*
security.*
qos.*
dns.*
dhcp.*
meta.*

V24 추가 필드
event.problem_class
event.problem_name
event.tier


6. event.problem_class 정의 (핵심)
🔴 system_issue (최상위)
EFG / kernel / resource 문제
예:
disk full
syslog I/O error
memory / buffer failure
👉 AP 문제처럼 보여도 실제 원인

🔵 ap_stuck
vap_timeout
radio_reset
channel_invalid
ap_freeze

🟡 ap_no_service
dhcp_fail
dns_fail
일부 auth_fail

🟢 rf_issue
low_rssi
retry_high
low_phy_rate

🟣 auth_issue
auth_fail
eapol_timeout

🟤 roaming_issue
FT decrypt fail
roaming 실패

⚪ low_priority
auth OK
association
lifecycle
debug / noise

7. event.action (유지 + 의미 명확화)
👉 event.action은 변경하지 않는다
역할:
event.action = "로그 이벤트 정의"
event.problem_class = "Root Cause 분류"

예:
event.action
problem_class
low_rssi
rf_issue
dhcp_fail
ap_no_service
dns_fail
ap_no_service
policy_drop
ap_no_service
vap_timeout
ap_stuck
radio_reset
ap_stuck


8. NEW: system_issue 매핑 (V24 핵심 추가)
다음 로그는 반드시 별도로 분류:
No space left on device
I/O error
syslog write failure
buffer allocation failure
reassemble cont fail

👉 매핑:
event.problem_class = system_issue
event.tier = critical

👉 가장 먼저 확인해야 하는 Root Cause

9. event.tier 정의
critical → system_issue
high → ap_stuck / dhcp_fail / dns_fail
normal → rf_issue / auth_issue
low → lifecycle / noise


10. 필터링 전략 (수정)
유지
장애 로그
DHCP / DNS
RF / retry
policy / QoS
V24 추가
👉 lifecycle / noise는 유지하되 low_priority로 분류

11. 장애 해석 방식 (중요 변경)
기존 ❌
A형 / B형 중심
V24 ✅
1. system_issue 존재 여부 먼저 확인
2. AP 문제인지 EFG 문제인지 분리
3. RF / DHCP / DNS / Policy로 세분화


12. Kibana 분석 기준 (업데이트)
다음 질문에 답해야 한다:
AP 문제인가?
EFG 문제인가?
system_issue 존재하는가?
RF 문제인가?
DHCP / DNS 문제인가?
Policy / QoS 영향인가?
특정 VLAN 문제인가?
장애 직전 공통 패턴은?

13. 운영 원칙 (강화)
event.action 필수 유지
Root Cause 분리 우선
AP만 보고 결론 금지
EFG 반드시 포함
system_issue 최우선 확인
small delta only (Logstash 수정 최소화)

14. 금지
event.action 제거
raw 제거
AP 단독 분석
system_issue 무시
RF / 서비스 문제 혼합
big rewrite

15. 최종 목표 (V24)
EFG + AP 로그를 하나의 체계로 통합하여,
증상과 실제 원인을 분리하고,
Root Cause를 증명 가능한 형태로 식별하는 것


🔥 한 줄 요약
event.action은 유지한다
하지만 V24부터는 problem_class로 Root Cause를 해석한다


