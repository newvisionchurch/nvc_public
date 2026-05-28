<!-- 현재 상태 (2026-05-25) ─────────────────────────────────────────
작성 시점: V24~V25 (2026-04)  |  현재 버전: V27

이 문서에서 설계한 4단계 프레임워크 달성 현황:

  1단계 (Monitoring & Alert): 🔄 진행 중
    - Kibana 인덱스/Dashboard 구성 완료
    - Kibana Rule 기반 자동 Alert는 미완성 (V2 예정)

  2단계 (Correlation): ✅ 기반 완료
    - V25.2에서 system_issue ↔ AP fault 상관분석 설계 완료
    - V27 event.rca_step 필드로 AP-stuck 시퀀스 자동 분류

  3단계 (Root Cause 자동화): 🔄 부분 완료
    - GUI에서 AP Stuck 카운트·분석 기능 구현 (로컬 자동화)
    - ELK 기반 완전 자동 판단은 미완성

  4단계 (Reporting & 운영): 🔄 부분 완료
    - GUI 로그 Export 기능으로 수동 보고 가능
    - Daily Report 자동 생성은 V2 예정

현재 실제 운영 도구:
  - ELK Kibana: 인덱스 시각화 및 수동 분석
  - GUI (gui/): AP Stuck 카운트, 분석, 리셋, 로그 Export
──────────────────────────────────────────────────────────────── -->

📊 Elastic Stack RCA Framework 구축

1. 개요
본 프로젝트는 교회 네트워크 운영을 위해
**Elastic Stack (ELK 기반)**을 활용하여 다음을 목표로 합니다:
✔ 장애 자동 감지
✔ 원인 분석 (RCA: Root Cause Analysis)
✔ 운영 효율화 및 체계화

2. 시스템 정의
본 시스템은 다음 두 가지 역할을 동시에 수행합니다:
✔ NOC (Network Operation Center)
→ 실시간 네트워크 상태 감시
✔ Observability Platform
→ 로그 기반 분석 및 원인 추적

3. 전체 단계 구성
프로젝트는 총 4단계로 구성됩니다:
🔹 1단계: Monitoring & Alert
문제 자동 감지
이상 상태 즉시 인지
🔹 2단계: Correlation (상관 분석)
시스템 문제와 AP 문제의 관계 분석
🔹 3단계: Root Cause 분석
문제의 근본 원인 자동 추정
🔹 4단계: Reporting & 운영
보고서 자동화
팀 운영 체계 구축

🚀 4. 1단계 상세 (현재 진행 단계)
🎯 목표
“문제 자동 감지 + 어디가 문제인지 즉시 확인”

📌 구성 요소
1) Alert (Rule)
Kibana Rule 기반 자동 감지
특정 조건 만족 시 Alert 발생
예:
system_issue 발생
AP stuck 발생
특정 이벤트 급증

2) Dashboard
현재 네트워크 상태 시각화
문제 발생 여부 즉시 확인
구성:
전체 이벤트 수
system_issue 추이
AP 상태 분포

3) Top 문제 AP 분석
GROUP BY: ap.name.keyword
문제 발생 AP 자동 식별

✅ 기대 효과
✔ 장애 발생 즉시 인지
✔ 문제 AP 빠르게 식별
✔ 운영자 대응 시간 최소화

🔍 5. 2단계 계획 (Correlation)
🎯 목표
“왜 문제가 발생했는지 보이게 만들기”

📌 주요 분석
system_issue ↔ AP stuck 상관관계 분석
시간축 기반 이벤트 흐름 분석
VLAN / 위치 기반 문제 분포

기대 효과
✔ 단순 알람 → 원인 추정 가능
✔ 문제 패턴 파악 가능

🧠 6. 3단계 계획 (Root Cause 자동화)
🎯 목표
“시스템이 원인을 자동으로 좁혀주기”

📌 예시 로직
system_issue 발생
→ 5분 내 AP stuck 증가
→ 자동 판단:
"EFG 문제 가능성 높음"

기대 효과
✔ 운영자 판단 부담 감소
✔ 빠른 장애 대응

📄 7. 4단계 계획 (운영 및 보고)
🎯 목표
“지속 가능한 운영 체계 구축”

📌 구성
주간 리포트 자동 생성
장애 패턴 분석
팀 교육 자료 구축

기대 효과
✔ 운영 표준화
✔ 지식 축적
✔ 팀원 참여 확대

👥 8. 운영 방향
✔ 평상시: 시스템 자동 감시
✔ 문제 발생: Alert 수신
✔ 관리자: Kibana 분석
✔ 결과: 조치 및 기록

🏁 9. 결론
본 프로젝트는 단순한 모니터링 도구 도입이 아니라
“교회 네트워크 운영 체계 구축”
을 목표로 하며,
Elastic Stack 기반으로
자동 감지 → 분석 → 원인 파악 → 보고
까지 연결되는 통합 운영 플랫폼으로 발전시킬 계획입니다.

