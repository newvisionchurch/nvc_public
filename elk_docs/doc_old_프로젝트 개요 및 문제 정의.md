<!-- 현재 상태 (2026-05-25) ─────────────────────────────────────────
작성 시점: V24 (2026-04)  |  현재 버전: V27  |  AI 도구: Claude Code

V24에서 설계한 "원인 분리" 체계는 V27까지 그대로 구현되었습니다.

V24 이후 달성 내용:
  V25.1 — system_issue 탐지 구현 완료 (EFG Disk Full → AP Stuck 인과관계 확인)
  V25.2 — Kibana 상관분석 설계 + 상관 helper 필드 추가
  V26   — event.storage_class 기반 ap/low/noise 인덱스 분리 완료
  V27   — RRM/채널스캔 분석 추가, event.rca_step(1~7) AP-stuck 타임라인 구현
  GUI V1— AP Stuck 카운트·분석·리셋, NAS 동기화, TOTP 2FA 로컬 관리 도구 완성

이 문서의 Root Cause 분류 체계(system_issue → ap_stuck → ap_no_service → rf_issue)는
현재도 유효하며 V27 운영의 기반이 됩니다.
──────────────────────────────────────────────────────────────── -->

📌 뉴비전 교회 네트워크 분석 프로젝트
V24 기준 Baseline / Issue Definition 통합 문서 (업데이트본)

0. 개요
본 문서는 뉴비전 교회 네트워크 분석 프로젝트의 기준 문서를
현재 상황(V23 중단 이후, V24 재설계 단계) 기준으로 다시 정리한 통합 문서이다.
이 문서의 목적은 과거의 방향을 그대로 반복하는 것이 아니라,
현재까지 확인된 실제 로그 패턴
V23 runtime error 경험
EFG 분석 결과
AP / RF / 정책 / 시스템 문제의 재분류 필요성
을 반영하여,
앞으로의 Source / Logstash / Kibana 분석 기준을 현재 상황에 맞게 재정의하는 것이다.
즉, 본 문서는 단순한 이전 문서의 요약이 아니라,
V24 설계 개념에 맞추어 Baseline, 문제 정의, 분석 목표를 다시 고정하는 기준 문서이다.

1. 프로젝트 최종 목표 (V24 기준 재정의)
이 프로젝트의 최종 목표는 단순히 AP 로그를 수집하거나,
장애를 증상 수준에서 분류하는 것이 아니다.
목표는 다음과 같다.
AP Stuck / AP No Service / 성능 저하 / 인증 실패 / RF 불안정
→ 실제 Root Cause를 로그 기반으로 분리
→ EFG / AP / RF / DHCP / DNS / Policy / QoS / System 중
실제 원인 축을 증명 가능한 형태로 식별
즉,
증상 관찰이 아니라
실제 원인 분리
그리고 증명 가능한 Root Cause 분석 체계 구축
이 최종 목표이다.

2. 핵심 전환: V24는 “증상 분류 단계”가 아니라 “원인 분리 단계”이다
기존 문서에서는 A형 / B형 문제를 중심으로
AP 장애와 성능 문제를 분석하는 방향이 맞았다.
그러나 현재까지의 로그 분석 결과,
다음과 같은 중요한 한계가 확인되었다.
기존 한계
1) AP Stuck 안에 서로 다른 원인이 섞여 있었음
예:
진짜 AP 자체 문제
EFG logging / system 문제
kernel / resource 문제
2) AP No Service 안에도 서로 다른 문제가 섞여 있었음
예:
RF 품질 문제
인증 문제
DHCP / DNS 문제
roaming 문제
단순 lifecycle 로그
3) 따라서 “A형 / B형”만으로는 Root Cause를 충분히 분리할 수 없음

V24의 핵심 전환
V24에서는 다음과 같이 해석한다.
AP Stuck / AP No Service는 최종 원인이 아니라 증상 클래스이다.
실제 Root Cause는 AP 자체, RF, 인증, DHCP/DNS, Policy/QoS,
혹은 EFG/System 문제일 수 있다.
즉,
V24는 증상 중심 분류에서 Root Cause 중심 분류로 전환하는 단계이다.

3. 현재까지 확정된 핵심 사실
현재까지의 분석을 통해 다음 사항이 사실상 확정되었다.
3.1 EFG는 보조 장비가 아니라 핵심 Root Cause 축이다
EFG는 단순 gateway 장비가 아니다.
EFG는 네트워크의 중심 제어 지점이며 다음 기능을 담당한다.
DHCP
DNS
Firewall / Policy Engine
QoS / Traffic Management
IDS/IPS
Logging / System resource
즉,
사용자가 체감하는 “WiFi 문제”, “AP 문제”, “인터넷 안됨” 현상은
실제로는 EFG 쪽 문제에서 시작되었을 수 있다.

3.2 AP 문제처럼 보이는 현상이 실제로는 system_issue일 수 있다
다음과 같은 로그는 AP 문제가 아니라
EFG / kernel / system resource 문제로 보아야 한다.
예:
No space left on device
syslog-ng I/O error
Suspended write operation because of an I/O error
Error suspend timeout has elapsed
failed to allocate reassemble cont.
이런 이벤트는
단순 보조 정보가 아니라 최상위 Root Cause 후보이다.

3.3 hostapd / disconnect / lifecycle 로그는 대부분 “원인”이 아니라 “흐름”이다
다음과 같은 로그는 많이 보이더라도
기본적으로는 정상 lifecycle 또는 흐름 이벤트일 가능성이 높다.
예:
authentication OK
association
disassociated
MLME-AUTHENTICATE
GTK rekeying
binding station to interface
따라서 V24에서는
정상 lifecycle은 위로 올리지 않고 낮은 우선순위로 유지한다.

3.4 RF 문제와 서비스 실패 문제는 반드시 분리해야 한다
기존에는 low_rssi, retry_high 같은 RF 품질 이벤트가
AP No Service에 섞여 해석되기도 했다.
하지만 현재 기준에서는 분리해야 한다.
RF 품질 문제 = rf_issue
인증 / DHCP / DNS 실패 = 서비스 실패 계열
roaming failure = roaming_issue
즉,
무선 품질 문제와 실제 서비스 실패 문제는 같은 것으로 보면 안 된다.

4. 프로젝트 목적 (현재 기준 유지 + 보정)
기존 목적은 유효하다.
즉, UniFi 기반 교회 네트워크에서 발생하는 문제를
로그 기반으로 분석하고 Root Cause를 증명한다는 방향은 유지한다.
다만 V24에서는 목적을 다음과 같이 더 정확히 정의한다.
V24 기준 목적
AP 증상과 실제 원인을 분리한다
EFG를 Root Cause 분석의 핵심 축으로 포함한다
RF / 인증 / DHCP / DNS / Policy / QoS / System 문제를 분리한다
단순한 장애 탐지가 아니라 원인 증명 가능한 구조를 만든다
Logstash는 최소 변경 원칙으로 안정성을 우선한다

5. 장애 개념 정의 (V24 기준 수정)
기존의 A형 / B형 정의는 “증상 분류”로서는 여전히 유효하다.
다만 그것을 Root Cause와 동일시하면 안 된다.
5.1 증상 클래스
A형: AP Stuck
특징:
사용자 0명 또는 연결 불가
AP reset 후 복구
AP는 online처럼 보일 수 있음
대표 로그 패턴:
vap_timeout
channel_invalid (ch=0)
radio_reset / driver_reset
tx_overflow
interface down 계열
의미:
AP 자체 문제일 수 있음
그러나 EFG / system 문제의 결과일 가능성도 배제하면 안 됨

B형: AP No Service / 성능 붕괴형
특징:
연결은 되어 보임
실제 체감은 unusable
인터넷 안됨 / 속도 거의 없음
대표 로그 패턴:
dhcp_fail
dns_fail
auth_fail
policy_drop
qos_limit
low_rssi
retry_high
low_phy_rate
의미:
EFG + RF + 서비스 계층의 복합 문제 가능성이 높음

5.2 Root Cause 클래스 (V24 핵심)
V24부터는 다음 Root Cause 클래스를 중심으로 해석한다.
1. system_issue
의미:
EFG 또는 kernel / resource / logging / system 문제
예:
No space left on device
syslog-ng I/O error
suspend timeout
failed to allocate reassemble cont.
설명:
AP 문제처럼 보이는 현상의 실제 원인일 수 있음
V24에서 최상위 우선순위로 취급

2. ap_stuck
의미:
진짜 AP 자체 장애 / hang / reset / radio freeze
예:
vap_timeout
radio_reset
ap_freeze
interface down
watchdog timeout

3. ap_no_service
의미:
사용자는 연결되거나 연결 시도하나 실제 서비스 실패
예:
dhcp_fail
dns_fail
일부 auth_fail
주의:
low_rssi, retry_high 같은 RF 품질 이벤트는 여기에서 분리해야 함

4. rf_issue
의미:
무선 품질 / 간섭 / retry / low signal 문제
예:
low_rssi
retry_high
low_phy_rate
anomaly_detected

5. auth_issue
의미:
인증 계층 문제
예:
auth_fail
eapol_timeout
radius_retry

6. roaming_issue
의미:
roaming / FT 관련 문제
예:
FT decrypt fail
roaming failure

7. low_priority
의미:
정상 lifecycle / 참고용 흐름 / 낮은 분석 가치 로그
예:
authentication OK
association
GTK rekey
dnsmasq reload
systemd normal lifecycle
일반 hostapd normal flow

6. 운영 원칙 (유지 + V24 보강)
기존의 운영 원칙 중 옳은 것은 그대로 유지한다.
유지할 원칙
Baseline 유지
Moving Target 방지
로그 + 설정 병행 분석
구조 우선
단일 원인 가정 금지
event.action 기반 분석
AP + EFG + RF 통합 분석
V24에서 더 강화할 원칙
1) EFG를 반드시 포함해서 본다
AP만 보고 결론 내리지 않는다.
2) 증상과 원인을 분리해서 본다
A형 / B형은 증상이고, Root Cause는 별도 클래스이다.
3) 정상 이벤트는 아래로 내린다
정상 lifecycle과 noise는 가능한 한 root cause 판단에서 분리한다.
4) 작은 수정만 허용한다
V23 실패 경험상 big rewrite는 금지한다.
5) 설정 변경보다 로그 증거를 우선한다
원인 증명이 먼저이고 설정 변경은 마지막 단계이다.

7. 분석 대상 구조 (현재 기준 업데이트)
7.1 EFG 영역
분석 핵심:
DHCP
DNS
Policy Engine
QoS / Traffic shaping
IDS/IPS
logging / disk / kernel / resource 상태
EFG는 단순 배경 장비가 아니라
Root Cause 후보의 핵심 축이다.

7.2 AP 영역
분석 핵심:
RF / RSSI
Retry / PHY rate
Roaming
Radio reset / ch=0 / vap timeout
AP naming / 위치 / 중요도

7.3 Network / VLAN 영역
분석 핵심:
VLAN별 문제 분리
DHCP per network
DNS per network
특정 network만의 장애 여부

7.4 시간축 분석
장애는 단일 이벤트가 아니라
시간 기준 상관관계로 봐야 한다.
예:
AP Stuck 직전 system_issue 존재 여부
low_rssi 증가 직후 dhcp_fail 또는 dns_fail 발생 여부
특정 시간대 EFG error 증가 여부

8. 분석 축 (유지)
분석은 반드시 아래 5축으로 수행한다.
AP
위치
모델
시간
Network(VLAN)
이 다섯 축은 현재도 유효하며 유지한다.

9. Logstash / Source 설계 원칙 (V24 기준 재정의)
V24는 Logstash 기능 확장 프로젝트가 아니다.
안정 재설계 프로젝트이다.
핵심 원칙
1) Baseline 유지
기존 안정판을 기준으로 삼는다.
2) Small delta only
한 번에 큰 구조 변경 금지
3) Syntax 안정성 최우선
복잡한 regex, 무리한 OR 확장, 전체 재작성 금지
4) 허용 변경 범위 최소화
실질적으로는 filter 내부의 mapping 조정만 허용
5) input / output / base grok는 최대한 고정
검증된 부분은 건드리지 않는다

10. 필드 설계 방향 (V24 추가 개념)
기존의 event.action 중심 구조는 유지한다.
이 방향은 여전히 맞다.
다만 V24에서는 아래 개념을 추가로 도입하는 것이 합리적이다.
event.problem_class
event.problem_name
event.tier
의미
event.action
개별 로그 동작 / 패턴 정의
event.problem_class
문제의 큰 분류
예: system_issue, rf_issue, auth_issue
event.problem_name
보다 구체적인 문제명
예: disk_full, syslog_io_error, dhcp_fail, ft_decrypt_fail
event.tier
우선순위 / 분석 중요도
예: critical, high, normal, low
즉, V24는
event.action은 유지하되, Root Cause 해석 계층을 위에 추가하는 방향이 적절하다.

11. 분석 목표 (현재 기준 수정)
기존 목표 중 일부는 유지 가능하다.
그러나 V24에서는 질문이 더 정밀해야 한다.
V24 핵심 질문
반복되는 장애 AP는 무엇인가
그것이 진짜 AP 문제인가, 아니면 EFG / system_issue의 결과인가
특정 VLAN / 특정 위치 / 특정 모델에서만 발생하는가
RF 문제인가, 인증 문제인가, DHCP/DNS 문제인가
Policy / QoS 영향이 있는가
system_issue가 발생한 직후 AP 문제처럼 보이는 현상이 증가하는가
정상 lifecycle을 제외하고도 동일 패턴이 반복되는가
장애 직전 공통 로그는 무엇인가
즉,
V24의 분석 목표는 “어느 AP가 문제인가”에서 끝나지 않고,
“문제처럼 보이는 현상의 실제 원인이 무엇인가”까지 가야 한다.

12. 현재 단계 (V24 기준 현실 반영)
현재 프로젝트 상태는 다음과 같이 정리한다.
과거
증상 중심 분류
AP 중심 해석
parsing / 구조화 중심
현재
구조 이해 완료
로그 수집 구조 확보
EFG 중요성 확인
AP 증상과 실제 원인이 다를 수 있음을 확인
V23는 runtime error로 중단
V24는 안정성 우선 재설계 단계 진입
즉, 현재는 단순 확장 단계가 아니라,
V24 = 안정 재설계 + Root Cause 분리 단계
이다.

13. 금지 사항 (현재 기준 수정)
다음은 계속 금지한다.
감 기반 판단
단일 로그로 결론
즉흥적 설정 변경
AP만 보고 결론
EFG만 보고 결론
big rewrite
baseline 파괴
검증되지 않은 대규모 regex 확장
추가로, V24에서는 다음도 금지한다.
증상 클래스를 원인 클래스로 오인하는 것
정상 lifecycle 로그를 root cause로 승격하는 것
RF 문제와 서비스 실패 문제를 혼동하는 것
system_issue를 부차적 로그로 취급하는 것

14. 핵심 결론
이 프로젝트는 더 이상
“AP 장애를 찾는 프로젝트”로 보면 안 된다.
현재의 정확한 정의는 다음과 같다.
이 프로젝트는 EFG + AP + RF + DHCP/DNS + Policy/QoS + System 로그를 통합하여,
증상과 실제 원인을 분리하고,
Root Cause를 증명 가능한 형태로 식별하는 분석 프로젝트이다.
즉,
A형 / B형은 유지하되 증상 클래스로 본다
Root Cause는 별도 구조로 분리한다
EFG를 중심 축으로 다시 본다
V24는 안정성 우선, small delta 기반으로 진행한다

15. 한 줄 요약
V23까지는 로그를 구조화하는 단계였다.
V24부터는 AP 증상 뒤에 숨어 있는 실제 Root Cause,
특히 EFG / System / RF / 서비스 계층 문제를 분리해 증명하는 단계이다.

