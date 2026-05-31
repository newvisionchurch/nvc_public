# ============================================================
# V25.1 잠정 확정 및 Code Update 우선순위 추가 (복붙용)
# ============================================================
#
# 현재 상태 (2026-05-25) ─────────────────────────────────────
# 작성 시점: V25 (2026-04)  |  현재 버전: V27
#
# 이 문서에서 계획된 우선순위 달성 현황:
#
#   1순위 (system → AP 상관분석): ✅ 완료
#     V25.2에서 Kibana 상관분석 설계 및 helper 필드 추가
#     EFG Disk Full → system_issue → AP Stuck 인과관계 확인됨
#
#   2순위 (auth_issue): ⏸ 보류
#     V25.2 이후 필요성 재검토 중, 현재 미구현
#
#   3순위 (roaming_issue): ⏸ 보류
#     auth_issue 이후 예정이었으나 현재 미구현
#
#   4순위 (service refinement): 부분 완료
#     QoS error 분리는 V24.3에서 완료
#     DNS/DHCP 세분화는 미구현
#
#   V26/V27 추가 달성 (이 문서 범위 외):
#     V26 — event.storage_class ap/low/noise 라우팅
#     V27 — event.rca_step(1~7), rrm_scan_trigger 분석
# ─────────────────────────────────────────────────────────────

# ------------------------------------------------------------
# 0. 현재 결론
# ------------------------------------------------------------
V25.1은 잠정 확정한다.

현재까지 확인된 내용:
- V24 안정화 완료
- CEF / zcef 복구 완료
- QoS error 분리 완료
- V25.1에서 system_issue 탐지 성공
- 실제 원인은 EFG gateway 측 system resource 문제로 확인됨
- 특히 /boot/firmware full 상태가 system_issue와 직접 연결됨

즉,
V25.1은 단순 실험이 아니라
실제 Root Cause를 탐지한 유효한 버전으로 본다.

# ------------------------------------------------------------
# 1. V25.1 코드 상태
# ------------------------------------------------------------
현재 baseline:
- V24-final 안정 구조 유지
- V25.1 = system_issue split 추가

V25.1 역할:
- system_error
- system_issue
- tier=critical
- outcome=failure

대상 로그:
- No space left on device
- I/O error
- syslog write error
- Suspended write operation because of an I/O error
- Error suspend timeout has elapsed
- buffer allocation failure
- failed to allocate reassemble cont.
- memory allocation failure

판단:
V25.1은 잠정 확정하고,
이후 수정은 “추가”만 허용한다.
기존 V25.1 system_issue 블록 수정은 최소화한다.

# ------------------------------------------------------------
# 2. Code Update 우선순위 (V25 기준)
# ------------------------------------------------------------

## 1순위: system → AP 영향 상관분석 준비
1-1. Kibana에서 system_issue와 ap_stuck 시간 상관관계 확인
1-2. system_issue 이후 5분 내 AP fault 증가 여부 확인
1-3. system_issue 이후 ap_no_service / qos_error 증가 여부 확인

핵심 질문:
- system_issue 직후 AP 문제가 증가하는가?
- AP 문제처럼 보이는 현상의 실제 상위 원인이 EFG/system인가?

중요:
이 단계는 Logstash 코드 추가보다
Kibana / Lens / 시간축 분석이 우선이다.

## 2순위: auth_issue 후보 정리
2-1. EAPOL timeout
2-2. auth failure
2-3. RADIUS reject / retry

원칙:
- V25.2에서는 auth_issue를 별도 분리하되
- 정상 auth lifecycle과 혼동하지 않도록 매우 보수적으로 진행

## 3순위: roaming_issue 후보 정리
3-1. FT decrypt fail
3-2. roaming failure
3-3. handoff 관련 실패 로그

원칙:
- 정밀 분석 영역이므로 auth_issue 이후 진행
- 오탐 방지 최우선

## 4순위: service refinement
4-1. DNS fail refinement
4-2. DHCP fail refinement
4-3. QoS refinement
4-4. policy / firewall 관련 실패 정리

원칙:
- V24.3의 qos_error 큰 분류는 유지
- V25에서는 필요할 때만 세분화

# ------------------------------------------------------------
# 3. 절대 원칙
# ------------------------------------------------------------
- baseline 유지
- small delta only
- input 수정 금지
- output 수정 금지
- base grok 수정 금지
- 기존 event.action 제거 금지
- 정상 lifecycle drop 금지
- Root Cause는 problem_class 중심 해석

# ------------------------------------------------------------
# 4. 다음 Chat에서 할 일
# ------------------------------------------------------------
다음 Chat의 목표는
“V25.1 코드 수정”이 아니라
“Kibana에서 system_issue와 AP 문제의 상관관계를 분석하는 방법 정리”이다.

즉 다음 Chat에서는:
- 어떤 Kibana 기능을 쓸지
- 어떤 Lens를 만들지
- 어떤 KQL을 조합할지
- 어떤 시간창(window)으로 볼지
를 먼저 정리한다.

# ------------------------------------------------------------
# 5. 다음 Chat용 Kibana 분석 목표
# ------------------------------------------------------------
주제:
system_issue와 AP 문제의 관계 분석

분석 대상:
- event.problem_class:system_issue
- event.problem_class:ap_stuck
- event.action:(channel_invalid OR vap_timeout OR radio_reset)
- event.problem_class:ap_no_service
- event.action:qos_error

핵심 목표:
- system_issue 발생 직후 AP 문제 증가 여부 확인
- EFG 문제와 AP 증상의 인과관계 확인
- AP 문제처럼 보이는 현상의 실제 상위 원인 증명

# ------------------------------------------------------------
# 6. 한 줄 정리
# ------------------------------------------------------------
V25.1은 잠정 확정한다.
다음 단계의 1순위는 코드 추가가 아니라
Kibana에서 system_issue → AP 영향 상관분석 구조를 정리하는 것이다.

# ============================================================

# ============================================================
# V25 MASTER GUIDE
# UniFi Logstash Root Cause Analysis Framework
# ============================================================

# ------------------------------------------------------------
# 0. 목표
# ------------------------------------------------------------
V24 = 안정성 / 기본 분류 완료
V25 = Root Cause 정밀 분석 단계

목표:
증상 → 원인 분리
AP / RF / AUTH / ROAMING / SERVICE / SYSTEM

# ------------------------------------------------------------
# 1. 절대 원칙 (변경 금지)
# ------------------------------------------------------------
- V24-final baseline 유지
- input 수정 금지
- output 수정 금지
- base grok 수정 금지
- 기존 event.action 수정 금지
- small delta only (한 번에 1개 기능)

# ------------------------------------------------------------
# 2. Root Cause 구조
# ------------------------------------------------------------
system_issue        (최상위)
network_service     (DHCP / DNS / QoS / Policy)
auth_issue          (인증 실패)
roaming_issue       (이동 실패)
rf_issue            (무선 품질)
ap_stuck            (AP 장애)
low_priority        (정상 흐름)

# ------------------------------------------------------------
# 3. V25 구현 순서 (중요)
# ------------------------------------------------------------

## 3-1. V25.1 system_issue (최우선)
대상:
- No space left
- I/O error
- disk full
- buffer error
- memory error

결과:
event.action        = system_error
event.problem_class = system_issue
event.tier          = critical

이유:
모든 문제의 최상위 원인

------------------------------------------------------------

## 3-2. V25.2 auth_issue
대상:
- EAPOL timeout
- auth fail
- RADIUS reject

결과:
event.action        = auth_fail
event.problem_class = auth_issue
event.tier          = high

------------------------------------------------------------

## 3-3. V25.3 roaming_issue
대상:
- FT decrypt fail
- roaming failure

결과:
event.action        = roaming_fail
event.problem_class = roaming_issue

------------------------------------------------------------

## 3-4. V25.4 service refinement
대상:
- dns fail
- dhcp fail
- qos refine
- firewall drop

결과:
network_service 내부 세분화

# ------------------------------------------------------------
# 4. 분석 방법 (핵심)
# ------------------------------------------------------------

질문 기반 분석:

1. system 문제인가?
2. service 문제인가?
3. auth 문제인가?
4. roaming 문제인가?
5. rf 문제인가?
6. ap 문제인가?

# ------------------------------------------------------------
# 5. KQL 핵심 쿼리
# ------------------------------------------------------------

event.problem_class:system_issue
event.problem_class:auth_issue
event.problem_class:roaming_issue
event.problem_class:rf_issue
event.problem_class:ap_stuck

------------------------------------------------------------

# ------------------------------------------------------------
# 6. 구현 원칙
# ------------------------------------------------------------
- 기존 mapping 절대 수정 금지
- unclassified 줄이기 위해 무리한 추가 금지
- lifecycle 로그 유지 (drop 금지)
- Kibana에서 필터링

# ------------------------------------------------------------
# 7. 성공 기준
# ------------------------------------------------------------

✔ 문제 발생 시 바로 Root Cause 보임
✔ AP 문제 vs 인증 문제 vs RF 문제 구분 가능
✔ QoS / DHCP / DNS 문제 분리됨
✔ system 문제가 최상위에서 보임

# ------------------------------------------------------------
# 8. 최종 개념
# ------------------------------------------------------------

V24 = "보이게 만드는 단계"
V25 = "정확히 구분하는 단계"

# ============================================================

Code Update 우선순위 (V24 기준)
V24 = 안정성 / 내구성 / 기본 분류
V25 = 정밀 Root Cause 분석

1-1 V24.2 장시간 안정성 검증
1-2 QoS Error 분리 (V24.3 핵심)
1-3 정상 lifecycle 유지 (Drop 금지)

2-1 problem_class 4축 고정
2-2 unclassified 패턴 정리
2-3 CEF 안정성 재검증

3-1 auth_issue 후보 정리
3-2 roaming_issue 후보 정리
3-3 system_issue 후보 정리


📌 V24 Code Update 우선순위 (복붙용)
🔥 핵심 방향
V24 = 안정성 / 내구성 / 기본 분류
V25 = 정밀 Root Cause 분석

🥇 1순위 (안정화 핵심)
1-1. V24.2 장시간 안정성 검증
CEF / zcef 지속 생성 여부 확인
index field 수 유지 여부 확인
observer.type / dataset 정상 유지
RF / AP / DHCP action 정상 유지
unclassified 증가 원인이 설명 가능한지 확인

1-2. QoS Error 분리 (V24.3 핵심)
대상:
qos_control.sh
tc class
qos_sta_control
cmd failed
결과:
event.action = qos_error
event.category = network_service
event.problem_class = ap_no_service
event.tier = high
event.outcome = failure

1-3. 정상 lifecycle 유지 (Drop 금지)
authentication OK / associated / binding 등
유지:
event.problem_class = low_priority
목적:
분석 증거 보존

🥈 2순위 (기본 분류 정리)
2-1. problem_class 4축 고정
rf_issue
ap_stuck
ap_no_service
low_priority

2-2. unclassified 정기 점검
Top log_message 패턴 확인
“큰 문제 그룹”만 선별하여 추가

2-3. CEF 안정성 재검증
gateway + CEF 존재 확인
zcef 누락 여부 확인

🥉 3순위 (V25 준비)
3-1. auth_issue 후보 정리
EAPOL timeout
auth failure

3-2. roaming_issue 후보 정리
FT decrypt fail
roaming 실패

3-3. system_issue 후보 정리
disk / memory / I/O error
system resource 문제

📊 실행 순서
1. V24.2 장시간 관찰 (가장 중요)
2. QoS 반복 발생 확인
3. V24.3 (QoS 최소 추가)
4. 기본 분류 안정화
5. V25 준비

📌 상세 설명
1순위 의미
지금은 기능 추가 단계가 아님
“파싱이 깨지지 않는가” 확인 단계
CEF 복구 이후 안정성 검증이 핵심
QoS가 중요한 이유
현재 unclassified 안에 숨겨진 유일한 “실제 문제 로그”
사용자 체감 성능 문제와 직접 연결 가능
복잡한 분석 없이도 가치 있음
lifecycle 로그 처리
절대 Drop 금지
Root Cause 분석의 흐름 데이터
Kibana에서 필터로 제어
2순위 의미
구조를 흔들지 않고 “큰 분류만 안정화”
세부 분류는 하지 않음 (V25로 넘김)
3순위 의미
지금은 설계만
실제 구현은 V25에서
