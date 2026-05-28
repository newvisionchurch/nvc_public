<!-- 현재 상태 (2026-05-25) ─────────────────────────────────────────
작성 시점: V25.2 (2026-04)  |  현재 버전: V27

이 보고서의 핵심 발견 (EFG Disk Full → system_issue → AP Stuck)은
V27 현재까지 유효한 RCA 모델의 기반입니다.

보고서 이후 진행 상황:
  ✅ 단기 조치 완료 — /boot/firmware 불필요 파일 삭제 (rootfs.bkp 등)
  ✅ V26/V27 — event.storage_class, event.rca_step으로 이 흐름을 파이프라인에서
                자동 분류하는 구조 구현
  🔄 장기 조치 — Disk Full 사전 Alert (ELK Kibana Rule) 설계 예정

현재 RCA 체계 (elk/README.md 참조):
  system_issue → qos_error → ap_stuck / ap_no_service
  rrm_scan_trigger → radio_driver_retry → tx_overflow
    → radio_reset → vap_timeout → channel_invalid → service_impact
──────────────────────────────────────────────────────────────── -->

📌 EFG 장애 원인 분석 보고서 (V25.2 기반)
1. 개요
본 분석은 ELK 기반 로그 분석(V25.1 → V25.2)을 통해
AP 장애의 근본 원인을 규명하기 위해 수행되었다.

2. 결론 (Summary)
EFG 장비의 /boot/firmware 디스크 Full 상태로 인해
시스템 오류(system_issue)가 발생하였고,
이로 인해 AP에 영향이 전파되어 AP Stuck 및 No Service 장애가 발생한 것으로 확인된다.

3. 분석 근거
3.1 시스템 로그 (EFG)
다음과 같은 오류 반복 확인:
No space left on device
I/O error occurred while writing
Error suspend timeout
👉 의미:
EFG 내부 저장소 부족
로그 및 시스템 쓰기 실패
정상적인 시스템 동작 불가 상태

3.2 상관관계 분석 (V25.2)
Correlation 분석 결과:
단계
이벤트
Cause
system_issue
Impact
qos_error
Symptom
ap_stuck / ap_no_service

👉 흐름:
EFG Disk Full
→ system_issue 발생
→ qos 처리 오류 발생
→ AP 상태 불안정
→ AP Stuck / No Service 발생


3.3 시간 기반 패턴
system_issue 발생 시점과
AP 장애(ap_stuck, no_service) 발생 시점이
👉 동일 시간대 또는 직후에 반복적으로 발생
→ 단순 상관이 아닌 인과 관계로 판단 가능

4. 원인 분석 (Root Cause)
직접 원인
/boot/firmware 파티션 Full
2차 영향
시스템 로그/프로세스 쓰기 실패
QoS 처리 실패
최종 영향
AP 상태 이상
서비스 중단

5. 조치 방안
5.1 단기 조치
불필요 파일 삭제 (rootfs.bkp 등)
디스크 사용량 확보
5.2 중기 조치
Firmware 업데이트 구조 점검
로그/백업 파일 관리 정책 적용
5.3 장기 조치
디스크 사용량 모니터링 자동화
Disk Full 사전 Alert 설정 (ELK 기반)

6. 향후 분석 방향
동일 패턴 재발 여부 모니터링
system_issue 발생 시 AP 영향 자동 감지
Correlation 기반 Root Cause 자동화 검토

7. 결론 요약
EFG Disk Full
→ System Issue
→ QoS Error
→ AP Stuck / No Service

👉 본 장애는 AP 자체 문제가 아닌 EFG 시스템 자원 문제에서 시작된 연쇄 장애로 판단된다.
# ============================================================
# V25.1 다음 단계: Kibana 상관분석 설계 (복붙용)
# ============================================================

현재 상태:
- V25.1은 잠정 확정된 baseline으로 본다.
- 코드 수정이 1순위가 아니다.
- 먼저 Kibana에서 system_issue와 AP 문제의 시간 상관관계를 분석한다.

핵심 목적:
1. system_issue 발생 직후 AP fault가 실제로 증가하는지 확인
2. AP 문제처럼 보이는 현상의 실제 상위 원인이 EFG / system인지 확인
3. 증상(ap_stuck, ap_no_service, qos_error)과 실제 Root Cause(system_issue)를 분리해서 본다

분석 대상:
- event.problem_class:system_issue
- event.problem_class:ap_stuck
- event.action:(channel_invalid OR vap_timeout OR radio_reset)
- event.problem_class:ap_no_service
- event.action:qos_error

분석 우선순위:
1. Kibana Discover에서 원본 로그 시간축 검증
2. Kibana Lens에서 time series 비교
3. KQL 조합 정리
4. 1분 / 5분 / 10분 window 비교
5. system_issue 이후 AP 문제 증가 여부 확인

------------------------------------------------------------
1. Discover에서 먼저 확인할 것
------------------------------------------------------------

목표:
- system_issue 로그가 실제로 어떤 시간대에 몰리는지 확인
- 그 직전/직후에 ap_stuck / ap_no_service / qos_error가 붙는지 확인
- observer.type, device.name, event.module, event.action까지 같이 본다

Discover 추천 컬럼:
- @timestamp
- observer.type
- device.name
- event.module
- event.action
- event.problem_class
- event.tier
- event.outcome
- log_message

Discover 기본 KQL:
event.problem_class:(system_issue OR ap_stuck OR ap_no_service)
OR event.action:(channel_invalid OR vap_timeout OR radio_reset OR qos_error)

Discover 확인 포인트:
- system_issue가 gateway(observer.type:gateway)에서 먼저 뜨는가
- 직후 AP fault가 늘어나는가
- 같은 시간대 특정 device.name 또는 특정 module에 집중되는가
- qos_error가 system_issue 이후 따라오는가
- ap_stuck이 독립적으로 발생하는지, system_issue cluster 뒤에 붙는지 확인

------------------------------------------------------------
2. Lens Time Series 1차 설계
------------------------------------------------------------

Lens 목적:
- 같은 시간축에 system_issue와 AP fault를 겹쳐서 본다
- 먼저 “겹쳐 보이는가”를 확인하고
- 그 다음에 “얼마나 늦게 따라오는가”를 본다

시각화 1:
[System vs AP Fault Count Over Time]

차트:
- Line chart 또는 Stacked area 말고 우선 Line chart 권장
- X축: @timestamp
- Interval:
  - 1차: 1 minute
  - 2차: 5 minute
  - 3차: 10 minute

Series A:
KQL = event.problem_class:system_issue

Series B:
KQL = event.problem_class:ap_stuck

Series C:
KQL = event.problem_class:ap_no_service

Series D:
KQL = event.action:qos_error

해석:
- system_issue spike 직후 B/C/D가 따라 올라오는지 본다
- 완전 동시인지, 1~5분 후행인지 확인
- 10분에서는 큰 흐름, 1분에서는 직접 후행 여부를 본다

------------------------------------------------------------
3. Lens Time Series 2차 설계
------------------------------------------------------------

시각화 2:
[system_issue 이후 AP fault 상세 분해]

Series:
- event.action:channel_invalid
- event.action:vap_timeout
- event.action:radio_reset
- event.action:qos_error

목적:
- ap_stuck 내부에서도 어떤 action이 system_issue와 가장 강하게 붙는지 확인
- radio_reset형인지, vap_timeout형인지, channel_invalid형인지 분리

중요:
- ap_stuck 전체만 보면 뭉개질 수 있으므로
- action breakdown chart를 별도로 만든다

------------------------------------------------------------
4. Breakdown 기준
------------------------------------------------------------

우선 breakdown은 4개만 본다:

1) observer.type
- gateway
- ap

의미:
- system_issue가 gateway 쪽에서 먼저 시작되는지 확인

2) device.name
의미:
- 특정 EFG / 특정 AP 집중 여부 확인

3) event.module
예:
- syslog-ng
- kernel
- dnsmasq
- hostapd
- ubios-udapi-server

의미:
- system_issue의 실제 발생 계층 확인

4) network.vlan 또는 network.name (있으면)
의미:
- 특정 서비스/VLAN 영향 집중 여부 확인

------------------------------------------------------------
5. Time Window 비교 원칙
------------------------------------------------------------

1분 window:
- 직접적인 직후 반응 확인
- 순간 spike / 동시 발생 탐지
- 가장 민감하지만 노이즈 많음

5분 window:
- 실제 운영 해석용 주력
- “system_issue 이후 AP 문제가 증가하는가” 보기 가장 좋음

10분 window:
- 큰 흐름 / 장애 구간 확인
- 장시간 장애 cluster 확인용

원칙:
- 1분 = 즉시성
- 5분 = 실전 판단 기준
- 10분 = 전체 장애 파형 확인

실무 추천:
- 최종 판단은 5분 window 중심
- 1분 / 10분은 보조 검증용

------------------------------------------------------------
6. 추천 KQL 세트
------------------------------------------------------------

[1] system_issue만 보기
event.problem_class:system_issue

[2] AP stuck만 보기
event.problem_class:ap_stuck

[3] AP no service만 보기
event.problem_class:ap_no_service

[4] system + AP fault 통합 보기
event.problem_class:(system_issue OR ap_stuck OR ap_no_service)
OR event.action:qos_error

[5] AP fault 상세 action
event.action:(channel_invalid OR vap_timeout OR radio_reset OR qos_error)

[6] gateway 중심만 보기
observer.type:gateway AND event.problem_class:system_issue

[7] AP 중심 fault 보기
observer.type:ap AND (
  event.problem_class:(ap_stuck OR ap_no_service)
  OR event.action:qos_error
)

[8] critical/high failure만 보기
event.outcome:failure AND event.tier:(critical OR high)

------------------------------------------------------------
7. 분석 순서
------------------------------------------------------------

Step 1.
Discover에서 system_issue 발생 시각 3~5개 대표 구간 선정

Step 2.
각 구간에서 ±10분 범위로 event.problem_class 흐름 확인

Step 3.
Lens 1분 window로 system_issue 직후 ap_stuck / ap_no_service / qos_error 반응 확인

Step 4.
같은 데이터를 5분 window로 다시 봐서 노이즈 제거

Step 5.
10분 window로 장애 cluster 전체 파형 확인

Step 6.
action breakdown으로
- channel_invalid
- vap_timeout
- radio_reset
- qos_error
중 어떤 것이 가장 강하게 따라오는지 확인

Step 7.
observer.type / event.module / device.name breakdown으로
실제 시작점이 gateway/system인지 확인

------------------------------------------------------------
8. 핵심 판단 질문
------------------------------------------------------------

1. system_issue 직후 ap_stuck이 증가하는가?
2. system_issue 직후 ap_no_service가 증가하는가?
3. system_issue 직후 qos_error가 증가하는가?
4. AP fault보다 먼저 gateway/system 로그가 증가하는가?
5. 특정 event.module(syslog-ng, kernel 등)이 선행하는가?
6. 특정 device.name 또는 특정 VLAN에 집중되는가?
7. AP fault가 독립 사건인지, system_issue 후행 증상인지 설명 가능한가?

------------------------------------------------------------
9. 해석 기준
------------------------------------------------------------

A. 강한 상관관계
- system_issue spike 직후 1~5분 내 AP fault 증가
- 반복 구간에서 동일 패턴 재현
- gateway/system module이 항상 선행

=> 해석:
AP 문제처럼 보이는 현상의 상위 원인은 EFG/system일 가능성이 높다

B. 약한 상관관계
- 일부 구간만 같이 움직임
- system_issue 없이도 AP fault 다수 발생

=> 해석:
system_issue는 일부 장애 구간의 원인일 수 있으나
모든 AP fault의 공통 원인은 아닐 수 있다

C. 무관
- system_issue와 AP fault가 시간적으로 거의 분리됨

=> 해석:
ap_stuck 또는 ap_no_service는 별도 원인 축(RF, AP 자체, 서비스 계층)일 가능성 검토

------------------------------------------------------------
10. 이번 단계의 산출물
------------------------------------------------------------

이번 Chat에서 만들 것:
- Discover 조회 구조
- Lens 2~4개 설계
- KQL 세트
- 1/5/10분 window 해석 기준
- breakdown 기준

이번 Chat에서 하지 않을 것:
- V25.1 코드 수정
- auth_issue / roaming_issue 바로 추가
- 기존 baseline 변경

------------------------------------------------------------
11. 한 줄 결론
------------------------------------------------------------

이번 단계의 목표는
“system_issue가 AP 문제의 상위 원인인지”
Kibana 시간축으로 먼저 증명하는 것이다.

# ============================================================


