<!-- 현재 상태 (2026-05-25) ─────────────────────────────────────────
작성 시점: V24 (2026-04)  |  현재 버전: V27

V24 체크리스트 이후 V27에서 추가로 검증해야 할 항목:
  ✔ event.storage_class — ap/low/noise 값 정상 라우팅 여부
  ✔ event.rca_step       — 1~7 단계 매핑 정상 여부 (V27 핵심)
  ✔ event.rca_stage      — trigger/symptom/service_impact 분류
  ✔ rrm_scan_trigger     — ap 인덱스로 라우팅되는지 확인

V24 항목(event.problem_class, system_issue 탐지)은 모두 구현 완료 상태입니다.
활성 파이프라인: elk/logstash/pipeline/logstash.conf
──────────────────────────────────────────────────────────────── -->

📌 Logstash Conf Validation Checklist (V24 기준)

1. 목적 (V24 업데이트)
본 문서는 Logstash Config의 정상 동작뿐 아니라
Root Cause 분석 가능 상태까지 검증하는 체크리스트이다.
목표:
로그 수집 정상 여부 확인
필드 파싱 정확성 검증
event.action 매핑 검증
event.problem_class 검증 (V24 핵심)
Kibana Root Cause 분석 가능 상태 확인

2. Logstash 문법 검증
명령:
logstash -t -f logstash.conf
확인:
Configuration OK
Error 없음

3. 입력 포트 확인
UDP 51415 listen 여부
EFG syslog 설정 일치
방화벽 허용

4. Raw 로그 수신 확인
확인 필드:
message
meta.raw_message
성공 기준:
로그 유실 없음
원본 메시지 유지

5. 기본 파싱 확인
확인 필드:
@timestamp
device.name
event.module
log_message

6. AP 이름 파싱
테스트:
M2F_AP3_PASTOR_A2
M1F_AP5_BONDANG
확인:
site.code / site.name
floor
location.type
ap.priority

7. 필드 추출 검증
확인:
client.mac / ip / hostname
network.vlan / name
radio.channel / rssi

8. event.action 검증 (기존 유지)
RF
Low RSSI → low_rssi
retry → retry_high
AP 장애
ch=0 → channel_invalid
vap timeout → vap_timeout
인증
auth failure → auth_fail
DHCP/DNS
DHCPREQUEST → dhcp_request
DHCPACK → dhcp_ack
DNS fail → dns_fail
정책
firewall drop → policy_drop
QoS
shaping → shaping_applied

9. 🔥 NEW: event.problem_class 검증 (V24 핵심)
다음 매핑이 반드시 확인되어야 한다:
event.action
problem_class
low_rssi
rf_issue
retry_high
rf_issue
dhcp_fail
ap_no_service
dns_fail
ap_no_service
policy_drop
ap_no_service
vap_timeout
ap_stuck
channel_invalid
ap_stuck
radio_reset
ap_stuck


🔴 system_issue 검증 (최우선)
다음 로그 발생 시 반드시 확인:
No space left on device
I/O error
syslog write error
buffer allocation failure
확인:
event.problem_class = system_issue
event.tier = critical
👉 이게 없으면 V24 실패

10. event.outcome 검증
failure 조건:
auth_fail
dhcp_fail
dns_fail
policy_drop
vap_timeout
channel_invalid
radio_reset

11. event.category 검증
다음 분류 확인:
rf
ap_fault
network_service
security
qos
wifi_session

12. event.dataset 검증
unifi.efg
unifi.ap
👉 EFG vs AP 구분 가능해야 함

13. CEF 파싱 검증
확인:
client.mac
network.vlan
radio.rssi

14. 🔥 Kibana 검증 (V24 업데이트)
A형 증상 필터
event.action:(vap_timeout OR channel_invalid OR radio_reset)
B형 증상 필터
event.action:(low_rssi OR retry_high OR qos_limit OR policy_drop OR dhcp_fail OR dns_fail)

🔥 Root Cause 필터 (V24 핵심 추가)
system_issue
event.problem_class:system_issue
RF 문제
event.problem_class:rf_issue
서비스 실패
event.problem_class:ap_no_service

🔥 핵심 확인 질문
AP 문제인가?
EFG 문제인가?
system_issue 존재하는가?
RF 문제인가?
DHCP / DNS 문제인가?
Policy / QoS 영향인가?

15. 오탐 확인
Low RSSI 오분류 여부
DROP 과다 매칭 여부
ch=0 오탐 여부
system_issue 누락 여부 (중요)

16. 성능 확인
CPU 사용량
ingest 지연
stdout 비활성 여부

17. 최종 기준 (V24)
다음이 모두 만족되어야 한다:
로그 수신 정상
event.action 정상 매핑
event.problem_class 정상 분류
system_issue 검출 가능
Kibana에서 Root Cause 분석 가능

🔥 한 줄 요약
V23: event.action 검증
V24: problem_class까지 검증해야 진짜 완료
 

