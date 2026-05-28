<!-- 현재 상태 (2026-05-25) ─────────────────────────────────────────
작성 시점: V24 (2026-04)  |  현재 버전: V27

이 문서는 EFG 장비의 UniFi 설정 화면(PDF 이미지 기반)을 분석·정리한 자료입니다.
GUI 코드에서 참조하는 정규화된 EFG 기준 정보는 efg/EFG.md를 사용합니다.

현재 운영 환경:
  - EFG Gateway IP: 192.168.11.1
  - AP 수: 약 29대 (Main / Education 건물)
  - UniFi Network: 10.2.105 / UniFi OS: 5.0.16
  - WAN: Comcast Business 500Mbps (Static IPv4, Failover Only)

이 문서의 분석 내용(A형/B형 장애, VLAN 구조, Dashboard 지표)은
V27 RCA 분석 및 GUI AP 인벤토리 구성의 기초 자료로 활용되었습니다.
──────────────────────────────────────────────────────────────── -->

EFG Configuration Summary
1. Overview
●	UniFi EFG (Enterprise Gateway Firewall) 중심 네트워크 구조
●	Main / Education 두 건물 구조
●	EFG가 Core Gateway 역할 수행
●	Internet, AP, Client, Traffic 상태를 통합 관제
________________________________________
2. Dashboard (운영 관측 지표)
●	Internet Activity / Throughput
●	Average Latency / Packet Loss
●	Top APs / Top Clients / Top Applications
●	WiFi Connectivity (Association / Authentication / DHCP / DNS)
●	AP Radio TX Retries
▶ 의미
●	장애 시점 기준 비교용 핵심 지표
●	B형 문제(속도/체감 장애) 1차 확인 포인트
________________________________________
3. Device Inventory
●	Gateway / Switch / AP 전체 목록
●	각 장비 상태, IP, 연결 상태
▶ 의미
●	네트워크 전체 구성 파악
●	장비 장애 vs 네트워크 장애 분리 가능
________________________________________
4. Client Inventory
●	사용자 단말 목록
●	연결 AP / 트래픽 사용량
▶ 의미
●	특정 사용자 / 특정 AP 문제 추적
●	"연결은 되는데 인터넷 안됨" 상황 분석 가능
________________________________________
5. AP Inventory
●	약 29~30대 AP 구성
●	모델 혼합 (AC Pro / AC HD / U6 / U7)
●	AP Naming 규칙 적용 (M/E + Floor + Location)
▶ 의미
●	위치 기반 분석 가능 (Pastor / Office / Sanctuary)
●	특정 AP 반복 장애 추적 가능
________________________________________
6. Network / VLAN 구조
●	다수 Network 존재
○	Core Network
○	NVFinance
○	KidsRegistration
○	NVCampus
○	Chapel 계열
●	각 Network는 별도 DHCP / Gateway 구성
▶ 의미
●	네트워크 단위 문제 분리 가능
●	특정 VLAN만 느린 문제 가능
________________________________________
7. DHCP 설정
●	Network 단위 DHCP Server
●	DHCP Range / Lease 설정
▶ 의미
●	DHCP 지연 시 인터넷 장애처럼 보일 수 있음
●	DHCPACK 흐름 분석 필요
________________________________________
8. DNS 구조
1) Network 내부 DNS
●	DHCP 통해 클라이언트에 전달
2) WAN DNS
●	EFG에서 사용하는 DNS
▶ 의미
●	DNS 문제 시 WiFi 정상 + 인터넷 불가 현상 발생
________________________________________
9. Internet / WAN 설정
●	Primary WAN 구성
●	Internet Health Monitoring
▶ 의미
●	외부 인터넷 문제 vs 내부 문제 구분
________________________________________
10. Policy Engine (Firewall)
●	Traffic Rule 기반 정책 구조
●	Allow / Block / Reject 정책
▶ 의미
●	Migration 정책 잔존 가능성
●	특정 트래픽 차단 가능
________________________________________
11. Traffic Management / QoS
●	Smart Queue / Bandwidth Control
●	Traffic shaping 가능
▶ 의미
●	속도 저하의 주요 원인 후보
●	B형 문제와 직접 연결
________________________________________
12. CyberSecurity (IDS/IPS)
●	Intrusion Prevention = ON
●	Detection Mode = Notify and Block
▶ 의미
●	트래픽 차단 / 지연 가능
●	False Positive 발생 가능
________________________________________
13. WiFi / RF 설정
●	SSID 구성
●	AP Radio 상태
▶ 의미
●	Low RSSI / Retry / Roaming 문제 분석 가능
________________________________________
14. Topology / Uplink
●	AP → Switch → EFG 구조
▶ 의미
●	uplink / PoE 문제 분석 가능 (A형)
________________________________________
15. Monitoring / Statistics
●	시간 기반 그래프
●	네트워크 품질 변화 추적
▶ 의미
●	주일 vs 평일 / 시간 패턴 분석 가능
________________________________________
16. System / Event Log
●	UniFi 내부 이벤트 로그 존재
▶ 의미
●	ELK 로그와 비교 분석 가능
________________________________________
🔥 핵심 분석 포인트
A형 (AP 먹통)
●	AP 자체 문제
●	uplink / PoE
●	radio freeze
B형 (성능 붕괴)
●	Firewall / Policy
●	QoS / Traffic shaping
●	DHCP / DNS
●	RF 문제
________________________________________
🎯 결론
B1 문서는 다음을 정의함:
1.	현재 네트워크 구조 (Baseline)
2.	분석 가능한 모든 구성 요소
3.	Root Cause 후보 영역
👉 다음 단계: 로그 기반 검증 (ELK)
________________________________________
🚀 다음 단계 준비
필요 로그:
EFG:
●	Firewall rule hit
●	DHCP flow
●	DNS query
●	QoS shaping
●	IDS/IPS detection
AP:
●	connect / disconnect
●	RSSI / retry
●	vap timeout
________________________________________
한 줄 요약
👉 B1 = "네트워크 구조 + 원인 후보 정의 완료 문서"

