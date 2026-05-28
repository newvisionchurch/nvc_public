# 뉴비전 교회 네트워크 V GUI 개발 구상서

작성일: 2026-04-30  
프로젝트: 뉴비전 교회 네트워크 분석 프로젝트  
문서 목적: EFG / AP / NAS 로그 기반 통합 관리 GUI 개발 방향 정리

---

## 1. 프로젝트 배경

뉴비전 교회 네트워크는 UniFi EFG 장비를 중심으로 약 30대의 AP가 운영되고 있으며, 교회 규모상 약 2,000명 수준의 성도와 다양한 예배/교육/행정 환경을 지원해야 한다.

현재 주요 문제는 일부 AP가 정상적으로 보이지만 실제로는 서비스가 불안정하거나 AP Stuck 상태에 빠지는 현상이다.

이를 분석하기 위해 NAS 장비에 Elastic Stack / ELK 환경을 구축했고, Logstash V27 기준으로 EFG 및 AP 로그를 수집·분류·저장하는 구조를 만들고 있다.

현재 로그는 Kibana로 전달되기 전 또는 별도 출력 단계에서 날짜별, 용량별, AP별, low priority, noise 등으로 분류하여 저장하는 방향으로 발전하고 있다.

이번 GUI 개발의 목적은 Kibana나 수동 SSH 작업에만 의존하지 않고, 운영자가 빠르게 현재 AP 상태를 확인하고, 필요한 로그를 추출하고, 문제 AP를 원격으로 리셋할 수 있는 간단한 통합 상황 관리 도구를 만드는 것이다.

---

## 2. 최종 목표

이 GUI는 대형 NMS(Network Management System)나 복잡한 웹 플랫폼이 아니라, 현재 프로젝트에 필요한 최소 기능을 빠르게 수행하는 로컬 관리 도구를 목표로 한다.

핵심 목표는 다음과 같다.

1. EFG / NAS / AP 로그를 기반으로 현재 문제 AP를 빠르게 찾는다.
2. AP Stuck 관련 로그를 AP별로 카운트한다.
3. 문제가 있는 AP의 상세 정보를 확인한다.
4. 필요 시 AP를 원격 리셋한다.
5. 특정 AP 또는 특정 기간의 로그를 빠르게 export한다.
6. Codex / ChatGPT 분석에 넘길 수 있는 텍스트 로그를 쉽게 생성한다.
7. 향후 기능을 점진적으로 확장할 수 있는 Python 기반 구조를 만든다.

---

## 3. 개발 방향 결정

### 3.1 개발 언어

개발 언어는 Python으로 결정한다.

이유는 다음과 같다.

- SSH 접속 및 명령 실행 라이브러리가 풍부하다.
- 로그 파일 읽기, 필터링, export 처리가 쉽다.
- GUI 라이브러리와 연동이 쉽다.
- VS Code, GitHub, Codex 기반 개발 흐름과 잘 맞는다.
- Windows와 macOS 모두 지원하기 쉽다.
- PyInstaller 등을 사용해 실행 파일로 배포할 수 있다.

### 3.2 GUI 방식

완전한 텍스트 기반 TUI는 배제한다.

초기에는 단순해 보이지만 AP 30대 상태를 한눈에 보여주고 마우스로 클릭하는 운영 방식에는 적합하지 않다.

따라서 GUI는 Tkinter 기반으로 시작한다.

Tkinter 선택 이유:

- Python 기본 GUI 라이브러리이다.
- Windows와 macOS 모두 지원한다.
- 복잡한 그래픽 없이 텍스트 중심 화면을 만들 수 있다.
- 버튼, 표, 리스트, 메뉴 구성 정도는 충분히 가능하다.
- 개발 난이도가 낮다.
- 나중에 필요하면 PyQt, web UI 등으로 확장 가능하다.

### 3.3 개발 환경

기본 개발 환경은 다음으로 정한다.

- VS Code
- Python Extension
- GitHub Repository
- Codex / ChatGPT 기반 코드 작성 및 디버깅
- Markdown 기반 설계 문서 관리
- 기존 ELK / Logstash / Export Script와 같은 Repository 내에서 관리

---

## 4. Repository 구조 방향

기존 GitHub Repository를 그대로 활용하고, 별도 Repository를 새로 만들기보다는 현재 프로젝트 Repository 안에 GUI 폴더를 추가하는 방식이 좋다.

예상 구조:

```text
project-root/
├─ logstash/
│  ├─ pipeline/
│  └─ versions/
├─ scripts/
│  ├─ powershell_export/
│  └─ log_extract/
├─ gui/
│  ├─ main.py
│  ├─ config/
│  ├─ modules/
│  │  ├─ auth.py
│  │  ├─ ap_count.py
│  │  ├─ ap_detail.py
│  │  ├─ ap_reset.py
│  │  ├─ log_export.py
│  │  └─ ssh_client.py
│  ├─ data/
│  ├─ docs/
│  └─ requirements.txt
├─ docs/
│  ├─ design.md
│  └─ operation_guide.md
└─ README.md
```

---

## 5. 프로그램 실행 방식

초기 버전은 로컬 PC에서 실행되는 GUI 프로그램으로 만든다.

웹 기반 또는 Chrome Extension 방식은 현재 단계에서는 제외한다.

이유:

- 개발 및 디버깅 시간이 크게 늘어난다.
- 인증, 서버, 브라우저 배포 등 고려할 요소가 많다.
- 현재 필요한 것은 빠르게 로그를 보고 AP를 제어하는 로컬 도구이다.
- 장기적으로 필요할 때 웹 기반으로 확장하면 된다.

초기 실행 흐름:

1. 사용자가 로컬 PC에서 GUI 실행
2. 로그인 화면 표시
3. 인증 성공 시 메인 메뉴 진입
4. AP 관리 / 로그 익스포트 기능 사용
5. 필요한 결과는 로컬 저장소에 저장

---

## 6. 로그인 및 보안 방향

### 6.1 초기 방향

초기에는 복잡한 사용자별 권한 시스템을 만들지 않는다.

운영팀 규모가 약 6명 정도로 작고, 프로그램을 사용하는 사람도 제한적이므로 단순하고 강력한 인증 방식을 우선한다.

### 6.2 검토한 인증 방식

검토한 방식:

1. 공통 ID / Password
2. 공통 Password + Google Authenticator OTP
3. Google Login OAuth
4. Google Workspace 계정 기반 로그인

### 6.3 현재 권장 방향

장기적으로는 Google Workspace 계정 기반 Google Login이 가장 좋다.

구조:

- 교회 Google Workspace Admin 계정에서 OAuth Client 설정
- 허용된 교회 이메일 계정만 로그인 가능
- 코드 내부 또는 설정 파일에 허용 이메일 목록 관리
- 초기 버전에서는 로그인한 사용자는 모두 동일 권한 부여
- 향후 필요 시 이메일별 권한 분리 가능

예시 허용 방식:

```text
allowed_users:
  - admin1@thevisionchurch.org
  - admin2@thevisionchurch.org
  - network1@thevisionchurch.org
```

### 6.4 현실적인 1차 구현

그러나 V1에서는 개발 속도를 위해 다음과 같이 단순화할 수 있다.

- 로컬 관리자 패스워드 방식으로 시작
- 이후 Google OAuth를 V2에서 추가
- 또는 Google OAuth는 설계만 먼저 해두고 메뉴/기능 개발 우선

---

## 7. 로컬 저장소 구조

GUI는 클라우드에서 동작하는 프로그램이 아니라 로컬 PC에서 동작하는 프로그램이다.

따라서 로그 export 결과, 분석 결과, 설정 파일, 임시 파일을 저장할 로컬 작업 폴더가 필요하다.

추가 원칙:

- NAS에 저장된 Logstash JSONL 원본은 장기 보관소로 둔다.
- GUI 시작 시 NAS 원본 폴더와 PC 로컬 캐시를 비교한다.
- PC에 없는 파일 또는 크기가 다른 파일만 다운로드한다.
- GUI의 AP 카운트, 상세 분석, export 작업은 PC 로컬 캐시 기준으로 수행한다.
- PC 캐시는 GUI에서 삭제할 수 있으나 NAS 원본은 삭제하지 않는다.
- 삭제 기능은 로컬 작업 폴더 내부 로그 캐시에만 제한한다.

Windows 기준 기본 경로 예시:

```text
C:\NVC_Network_GUI\
```

macOS 기준 기본 경로 예시:

```text
~/NVC_Network_GUI/
```

권장 폴더 구조:

```text
NVC_Network_GUI/
├─ logs/
│  └─ raw/
├─ exports/
│  ├─ ap_stuck/
│  └─ ap_detail/
└─ reports/
   └─ manual/
```

### 7.1 폴더 권한

GUI는 지정된 로컬 작업 폴더 안에서 다음 권한을 가져야 한다.

- 폴더 생성
- 파일 생성
- 파일 읽기
- 파일 쓰기
- export 파일 저장
- 임시 파일 삭제
- 설정 파일 업데이트

운영체제 전체 관리자 권한을 항상 요구하기보다는, GUI 전용 작업 폴더에 대해 충분한 권한을 확보하는 방식이 바람직하다.

---

## 8. 장비 접속 구조

GUI가 향후 접근해야 할 장비는 다음 세 그룹이다.

1. EFG
2. NAS / Synology
3. 약 30대 AP

접속 방식:

- SSH 기반
- Python SSH 라이브러리 사용
- 초기 후보: paramiko
- 명령 실행, 로그 읽기, 스크립트 실행, AP reboot 명령 수행

주의 사항:

- SSH 계정 정보는 코드에 직접 하드코딩하지 않는다.
- 설정 파일 또는 OS Credential Store 사용을 검토한다.
- AP 리셋 같은 위험 명령은 반드시 확인 창을 둔다.
- 명령 실행 로그를 남긴다.

---

## 9. 1차 메뉴 구조

초기 GUI는 너무 많은 기능을 넣지 않는다.

메뉴는 크게 두 개로 시작한다.

1. AP 관리
2. 로그 익스포트

---

# 10. AP 관리 메뉴

AP 관리 메뉴의 목적은 문제 AP를 빠르게 찾고, 상세를 확인하고, 필요 시 리셋하는 것이다.

AP 관리 메뉴는 3개 기능으로 시작한다.

```text
AP 관리
├─ 1. AP 스턱 카운트
├─ 2. AP 스턱 상세 분석
└─ 3. AP 리셋
```

---

## 10.1 AP 스턱 카운트

### 목적

30대 AP 전체를 빠르게 스캔하여 AP Stuck 관련 로그 발생 횟수를 AP별로 보여준다.

### 입력

- 검색 기간
- 로그 위치
- AP 목록
- AP별 키워드
- AP Stuck 관련 키워드

### 처리 방식

1. 사전 등록된 AP 1번~30번 목록을 불러온다.
2. 각 AP의 이름/IP/키워드를 기준으로 로그 파일을 검색한다.
3. AP Stuck 관련 키워드를 카운트한다.
4. AP별 발생 횟수를 계산한다.
5. 결과를 GUI에 표시한다.

### 표시 방식

AP 30개를 10개씩 3줄 또는 표 형태로 표시한다.

예시:

```text
AP01  0  Healthy
AP02  3  Warning
AP03  0  Healthy
AP04  15 Critical
...
```

색상 기준 예시:

```text
0      = 정상
1~2    = 주의
3~9    = 문제 의심
10 이상 = 심각
```

### 기대 효과

- 어떤 AP가 현재 가장 문제가 많은지 즉시 파악 가능
- 30개 AP 중 반복 장애 AP를 빠르게 식별
- 수동 grep 작업을 줄임

---

## 10.2 AP 스턱 상세 분석

### 목적

AP 스턱 카운트에서 문제가 발견된 AP를 선택하면 해당 AP의 상세 정보를 텍스트 기반으로 보여준다.

### 표시 정보

- AP 번호
- AP 이름
- IP 주소
- 위치
- 모델명
- 펌웨어 버전
- 최근 Stuck 발생 시각
- Stuck 발생 횟수
- Stuck 유형
  - channel_invalid
  - vap_timeout
  - radio_reset
  - tx_overflow
  - 기타
- 관련 로그 샘플
- 동일 유형 반복 여부
- 동일 모델/펌웨어에서 반복되는지 여부

### 사용 목적

- 특정 AP만 반복 문제인지 확인
- 같은 모델/펌웨어에서 공통 문제가 있는지 확인
- 같은 위치 또는 같은 층에서 반복되는지 확인
- AP 자체 문제인지 EFG/system 영향인지 추가 분석 준비

---

## 10.3 AP 리셋

### 목적

Stuck 상태로 판단되는 AP를 원격으로 리셋한다.

### 방식

- AP SSH 접속
- reboot 또는 UniFi AP 재시작 명령 실행
- 실행 전 확인 창 표시
- 실행 결과 로그 저장

### 안전 장치

AP 리셋은 실제 서비스에 영향을 줄 수 있으므로 반드시 다음 안전 장치를 둔다.

1. AP 선택 확인
2. AP 이름/IP 재확인
3. “정말 리셋하시겠습니까?” 확인
4. 명령 실행 로그 저장
5. 실패 시 오류 메시지 표시
6. 여러 AP 동시 리셋은 V1에서 금지

### V1 원칙

초기 버전에서는 한 번에 하나의 AP만 리셋한다.

---

# 11. 로그 익스포트 메뉴

로그 익스포트 메뉴의 목적은 특정 AP, 특정 기간, 특정 문제 유형에 해당하는 로그를 빠르게 추출하여 분석 가능한 텍스트 파일로 만드는 것이다.

메뉴 구조:

```text
로그 익스포트
├─ 1. 현재 로그 보기
├─ 2. 상세 로그 조건 선택
└─ 3. 로그 익스포트 실행
```

---

## 11.1 현재 로그 보기

### 목적

최근 로그 또는 지정된 로그 폴더의 최신 파일을 빠르게 확인한다.

### 기능

- 최신 로그 파일 목록 표시
- 파일 크기 표시
- 생성/수정 시간 표시
- AP 관련 로그 포함 여부 표시
- 간단한 미리보기 제공

---

## 11.2 상세 로그 조건 선택

### 목적

사용자가 원하는 조건으로 로그를 필터링할 수 있게 한다.

### 조건 예시

- AP 선택
- 기간 선택
- 로그 유형 선택
  - AP Stuck
  - AP No Service
  - RF Issue
  - DHCP/DNS
  - QoS
  - system_issue
- 키워드 직접 입력
- 원본 로그 포함 여부
- Codex 분석용 요약 파일 생성 여부

---

## 11.3 로그 익스포트 실행

### 목적

선택 조건에 맞는 로그를 파일로 저장한다.

### 출력 예시

```text
exports/
└─ ap_stuck/
   └─ AP04_2026-04-30_stuck_export.txt
```

### 기존 PowerShell 스크립트 활용

현재 Codex/GitHub 환경에서 이미 20MB 수준의 텍스트 로그 파일에서 AP 및 유형별로 필요한 로그를 추출하는 PowerShell 스크립트가 존재한다.

이 스크립트는 다음 방식으로 활용한다.

1. 기존 PowerShell 로직 분석
2. Python으로 재구현하거나 wrapper 방식으로 호출
3. GUI에서 버튼으로 실행
4. 결과 파일을 GUI 폴더 구조 아래 저장
5. 이후 V2에서 완전 Python 모듈로 통합

---

# 12. Python 코드 모듈 구조

하나의 Python 파일에 모든 기능을 넣지 않는다.

기능별 모듈로 나누어 관리한다.

예상 모듈:

```text
gui/
├─ main.py
├─ modules/
│  ├─ auth.py
│  ├─ local_storage.py
│  ├─ ap_inventory.py
│  ├─ ap_count.py
│  ├─ ap_detail.py
│  ├─ ap_reset.py
│  ├─ log_export.py
│  ├─ ssh_client.py
│  └─ report_writer.py
└─ config/
   ├─ ap_inventory.json
   ├─ app_config.json
   └─ ssh_targets.json
```

---

## 12.1 main.py

역할:

- 프로그램 시작점
- 로그인 화면 호출
- 메인 메뉴 구성
- 각 메뉴 모듈 연결

---

## 12.2 auth.py

역할:

- 로그인 처리
- 초기에는 로컬 패스워드 또는 간단 인증
- 향후 Google OAuth 연동

---

## 12.3 local_storage.py

역할:

- 로컬 작업 폴더 확인
- 폴더 생성
- 파일 권한 확인
- export 경로 관리

---

## 12.4 ap_inventory.py

역할:

- 30대 AP 목록 관리
- AP 번호, 이름, IP, 위치, 모델, 펌웨어 정보 관리

예시 구조:

```json
{
  "AP01": {
    "name": "M1F_AP1_EXAMPLE",
    "ip": "192.168.1.101",
    "location": "Main 1F",
    "model": "U6-Pro",
    "firmware": "unknown"
  }
}
```

---

## 12.5 ap_count.py

역할:

- AP별 Stuck 로그 카운트
- 기간별 스캔
- 결과 요약 생성

---

## 12.6 ap_detail.py

역할:

- 문제 AP 상세 분석
- AP별 로그 샘플 추출
- Stuck 유형 분류
- 모델/펌웨어/위치 기반 비교

---

## 12.7 ap_reset.py

역할:

- AP SSH 접속
- 리셋 명령 실행
- 실행 결과 저장
- 사용자 확인 처리

---

## 12.8 log_export.py

역할:

- 조건별 로그 추출
- Codex 분석용 파일 생성
- 기존 PowerShell 스크립트 로직 이식

---

## 12.9 ssh_client.py

역할:

- EFG / NAS / AP SSH 접속 공통 처리
- 명령 실행
- 오류 처리
- 접속 timeout 관리

---

# 13. AP Stuck 키워드 후보

초기 AP Stuck 판단 키워드는 기존 Logstash / Kibana 분석 기준을 활용한다.

후보:

```text
channel_invalid
vap_timeout
radio_reset
TX OVERFLOW
wlan_get_active_vport_chan: ch=0
osif_vap_init: Timeout waiting for DA vap 0 to stop
WAL_DBGID_DEV_RESET
DEV_RESET
```

Logstash 기준 주요 action:

```text
event.action:channel_invalid
event.action:vap_timeout
event.action:radio_reset
event.problem_class:ap_stuck
```

V1 GUI에서는 위 키워드를 중심으로 AP별 Stuck count를 계산한다.

---

# 14. V1 개발 범위

V1은 아래 범위까지만 구현한다.

## 포함

- Tkinter GUI 기본 창
- 로그인 화면
- 로컬 작업 폴더 생성
- AP Inventory 설정 파일
- AP 스턱 카운트
- AP 스턱 상세 분석
- AP 리셋 명령 구조
- 로그 익스포트 기본 구조
- 기존 로그 파일 기반 분석
- 결과 파일 저장

## 제외

- 완전한 웹 기반 UI
- Chrome Extension
- 복잡한 그래프
- Google Drive 연동
- 다중 사용자 권한 세분화
- 자동 알림 시스템
- 실시간 대시보드
- 여러 AP 동시 리셋
- 장기 통계 분석

---

# 15. V2 이후 확장 후보

V2 이후에는 다음 기능을 검토할 수 있다.

1. Google OAuth 로그인
2. Google Workspace 허용 이메일 기반 접근 제어
3. Google Shared Drive 연동
4. Kibana / Elasticsearch API 직접 조회
5. AP 상태 실시간 모니터링
6. EFG system_issue 자동 감지
7. NAS 로그 저장소 상태 확인
8. AP 리셋 후 복구 여부 자동 확인
9. Daily Report 자동 생성
10. 팀원별 작업 로그 관리
11. PowerShell 스크립트 완전 Python 이식
12. Codex 분석용 Markdown 자동 생성

---

# 16. 개발 우선순위

## 1단계: 골격

- Repository 내 gui 폴더 생성
- main.py 생성
- Tkinter 기본 화면
- 로컬 작업 폴더 생성
- AP Inventory JSON 생성

## 2단계: AP 스턱 카운트

- 로그 폴더 선택
- AP별 키워드 검색
- AP별 stuck count 표시
- 색상 상태 표시

## 3단계: AP 상세 분석

- 문제 AP 선택
- 상세 로그 추출
- stuck 유형 분류
- 관련 로그 샘플 표시

## 4단계: 로그 익스포트

- AP 선택
- 기간 선택
- 유형 선택
- export 파일 생성
- Codex 분석용 텍스트 생성

## 5단계: AP 리셋

- SSH 접속 모듈
- 단일 AP reboot
- 확인 창
- 실행 로그 저장

## 6단계: 보안 개선

- 로그인 강화
- Google OAuth 검토
- SSH credential 관리 개선

---

# 17. 운영 원칙

1. 처음부터 너무 크게 만들지 않는다.
2. AP Stuck 문제 해결에 직접 필요한 기능부터 만든다.
3. GUI는 단순하고 직관적으로 만든다.
4. 모든 명령 실행은 로그로 남긴다.
5. AP 리셋은 반드시 확인 절차를 둔다.
6. 기존 Logstash / ELK 구조를 흔들지 않는다.
7. 기존 PowerShell export script는 버리지 않고 재사용한다.
8. Codex가 이해하기 쉬운 Markdown 문서를 계속 유지한다.
9. V1은 로컬 실행 도구로 제한한다.
10. 웹/클라우드/Google Drive 연동은 V2 이후로 미룬다.

---

# 18. 한 줄 결론

뉴비전 네트워크 V GUI는 Python + Tkinter + VS Code + GitHub + Codex 기반으로 개발하며, V1에서는 로컬 PC에서 실행되는 단순 GUI로 AP Stuck 카운트, 상세 분석, AP 리셋, 로그 익스포트 기능에 집중한다.

목표는 복잡한 통합 NMS가 아니라, 현재 운영팀이 AP 문제를 빠르게 발견하고 로그를 추출하며 필요한 조치를 실행할 수 있는 실용적인 로컬 상황 관리 도구를 만드는 것이다.

---

# 19. 팀 공개 대비 인증·권한 구조 개선 계획 (2026-05 확정)

## 19.1 배경 및 목표

초기 V1은 단일 공유 비밀번호 구조였다. 팀원 약 8명이 VPN을 통해 각자의 집 PC에서
교회 내부 NAS/EFG/AP에 접속하는 구조로 발전함에 따라 아래 목표를 확정했다.

- 팀원 개인별 계정 (공유 비밀번호 제거)
- 2차 인증 (TOTP) 적용
- 역할(Role) 기반 권한 제어 — admin이 개인별 설정
- 내부 장비 자격증명 보안 (팀원이 비밀번호를 직접 보지 않음)
- VPN 연결 여부 시작 시 자동 확인

개발 원칙은 기존과 동일하다: 안정적 변경, 완벽한 검증, 하나씩 순차 적용.

---

## 19.2 접속 흐름

```
팀원 PC (집)
    │
    ▼
[1] VPN 연결 확인 ──실패──▶ "VPN을 먼저 연결하세요" 안내 후 종료
    │ 성공
    ▼
[2] 개인 ID + 비밀번호 입력
    │
    ▼
[3] TOTP 6자리 코드 입력 (Google Authenticator / Authy)
    │ 성공
    ▼
[4] 역할(Role) 로딩 → 권한에 따라 버튼/탭 활성화
    │
    ▼
[5] 내부 장비 접속 (NAS/EFG/AP) → SSH 키 또는 OS 키체인 자격증명 사용
```

---

## 19.3 인증 방식: TOTP 채택

| 방식 | 평가 |
| --- | --- |
| TOTP (Google Authenticator) | 채택. 외부 서비스 불필요, 오프라인 작동, 무료, 업계 표준 |
| 이메일 코드 | 차선. SMTP 서버 필요, 인터넷 필요 |
| SMS | 비권장. 유료 외부 서비스 필요 |

TOTP 초기 설정: admin이 팀원 계정 생성 시 QR코드를 생성하면 팀원이 앱으로
한 번 스캔한다. 이후 로그인마다 앱의 6자리 코드를 입력한다 (30초마다 갱신).

---

## 19.4 역할(Role) 구조

```
admin    — 모든 기능 + 사용자 관리 화면
operator — Log 동기화, AP 카운트, AP 상세, Log Export, AP Reset (admin 승인 시)
viewer   — AP 카운트, AP 상세, Log Export만 (Log 동기화/AP Reset 불가)
```

각 팀원별로 역할 외에 기능 단위 on/off를 admin 화면에서 설정한다.

```json
{
  "username": "hong",
  "role": "operator",
  "permissions": {
    "can_reset_ap": false,
    "can_sync_nas": true,
    "can_export": true,
    "can_view_efg_tab": true,
    "can_manage_users": false
  }
}
```

---

## 19.5 사용자 데이터 저장

파일 위치: `gui/config/users.json` (`.gitignore` 등록, git 제외)

비밀번호는 bcrypt 해시로 저장하며, TOTP 시크릿은 암호화 저장한다.
평문 비밀번호는 파일에 저장하지 않는다.

---

## 19.6 내부 장비 자격증명 보안

팀원이 NAS/EFG/AP 비밀번호를 직접 보지 않아도 접속되는 구조:

- 장비 자격증명을 OS 키체인(Windows Credential Manager)에 저장
- 앱이 OS에서 자격증명을 꺼내 사용, 팀원 화면에 노출 안 됨
- Python 라이브러리: `keyring` (Step 8에서 적용)

---

## 19.7 구현 로드맵

| 단계 | 작업 | 변경 범위 | 상태 |
| --- | --- | --- | --- |
| Step 1 | `allow_blank_dev_login` → `false` | 설정 1줄 | 완료 |
| Step 2 | VPN 연결 확인 (시작 시 체크) | 신규 모듈 소규모 | 완료 |
| Step 3 | 개인 계정 + bcrypt 비밀번호 해시 | `auth.py` 교체, `users.json` 신규 | 완료 |
| Step 4 | TOTP 2차 인증 | 로그인 화면 단계 추가 | 완료 |
| Step 5 | 역할별 버튼/탭 비활성화 UI | `main.py` 권한 체크 | 완료 |
| Step 6 | Admin 사용자 관리 화면 | 신규 탭 | 완료 |
| Step 7 | NAS sync `AutoAddPolicy` → `RejectPolicy` | 설정 변경 | 완료 |
| Step 8 | 장비 자격증명 OS 키체인 저장 | `keyring` 라이브러리 적용 | 완료 |
| Step 9 | operations.log에 사용자 이름 기록 | `report_writer.py` 소수정 | 완료 |

---

## 19.8 추가 라이브러리 계획

```
paramiko>=3.4,<4   # 기존
bcrypt>=4.0,<5     # Step 3: 비밀번호 해시
pyotp>=2.9,<3      # Step 4: TOTP 2FA
qrcode>=7.4,<8     # Step 6: Admin QR코드 생성
keyring>=25,<26    # Step 8: OS 키체인
```

---

# 20. 현재 구현 상태 (2026-05-26 기준)

초기 설계(섹션 1~14)와 보안 개선(섹션 19)이 완료된 후 추가로 구현된 기능입니다.

---

## 20.1 탭 구성 (최종)

| 탭 | 설명 | 비고 |
| --- | --- | --- |
| Log 동기화 | NAS JSONL → PC 캐시 동기화 | `can_sync_nas` |
| AP ELK Log | ELK 로그 기반 Stuck 집계·상세 분석 | 구 "AP 관리" 탭 (펌웨어 컬럼 제거) |
| AP Remote | AP SSH 직접 스캔 — mca-dump 진단 | 신규 |
| EFG Remote | EFG 시스템 대시보드 | 기존 AP Relay Test → 대시보드로 전면 교체 |
| Log Export | 조건별 로그 필터·저장 | |
| 사용자 관리 | 계정·권한 관리 (admin 전용) | |
| EFG 참조 | EFG.md 읽기 전용 | |

---

## 20.2 AP Remote 탭

AP와 직접 SSH 연결(EFG relay 경유)하여 `mca-dump` + `logread` 를 단일 세션에서 실행.  
결과를 파싱하여 무선 하드웨어 상태 진단 컬럼을 표시.

### SSH 경로

```
GUI (paramiko) → EFG (OpenSSH, PTY 불필요)
                    └─ sshpass + ssh -T → AP (Dropbear, PTY 불필요)
```

AP에 직접 paramiko 패스워드 인증 시 PTY 필요 (Dropbear 요구).  
EFG 경유 시 OpenSSH가 중간에서 처리하므로 PTY 불필요. 이 구조가 채택됨.

### 진단 컬럼

| 컬럼 | mca-dump 경로 | 의미 |
| --- | --- | --- |
| ELK Stuck | ELK Log 캐시 (in-memory) | ELK 기반 Stuck 이벤트 수 (ground truth) |
| 로그오류 | `logread \| grep -cE "stuck\|ath_reset\|vap.*timeout"` | 현재 부팅 세션 오류 패턴 수 |
| 재시작2G/5G | `radio_table[ng/na].athstats.ast_ath_reset` | 드라이버 레벨 채널 리셋 누적 |
| VAP지연2G/5G | `radio_table[ng/na].athstats.timeout_waiting_for_vap_cnt` | VAP 초기화 타임아웃 누적 |
| CU2G%/CU5G% | `radio_table[ng/na].athstats.cu_total` | 채널 이용률 |

### ⚠ 임계값 시스템

- 임계값 초과 셀에 `⚠ {값}` 텍스트 접두어 표시 (Tkinter Treeview는 셀 단위 색상 미지원)
- `⚠ 설정` 버튼으로 임계값 변경 → `app_config.json`의 `stuck_thresholds` 키에 저장
- 정렬 시 `⚠ ` 접두어는 제거하고 숫자로 비교

### 주요 기술 결정

- `mca-dump` + `logread` 를 `;` 구분자로 연결, `##MARKER##` 구분자로 파싱
- `exit_status` 검사 제거 — `;` 로 연결된 복합 명령은 항상 마지막 명령의 exit code 반환
- AC Pro mca-dump 65KB 이상 출력 잘림 문제: `ssh -tt`(PTY) → `ssh -T`(no PTY) 로 해결
- ELK Stuck 컬럼은 AP ELK Log 탭의 Stuck Count 실행 후 `_elk_stuck_counts` dict에 캐시

---

## 20.3 AP Reset

기존: AP 직접 SSH (paramiko 패스워드 인증 → Dropbear 호환성 문제)  
변경: EFG relay 경유 (`run_ssh_via_relay()`)

```python
run_ssh_via_relay(
    relay_host=efg_host, relay_username=..., relay_password=...,
    target_host=ap.ip,   target_username=..., target_password=...,
    command="reboot",
)
```

내부적으로 EFG에서 `sshpass -p '...' ssh -T -o StrictHostKeyChecking=no ... AP_IP 'reboot'` 실행.

---

## 20.4 EFG Remote 탭 — 대시보드

기존: AP relay test (전체/선택 AP 연결 테스트, sshpass 경로 확인)  
변경: EFG 시스템 대시보드

### 수집 명령 구조

단일 SSH 세션에서 `##MARKER##` 구분자로 10개 섹션 수집:

```bash
printf '##HN##\n'; hostname;
printf '##UP##\n'; uptime;
printf '##KN##\n'; uname -r;
printf '##LD##\n'; cat /proc/loadavg;
printf '##MEM##\n'; free -m;
printf '##IFACE##\n'; ip addr show;
printf '##NETDEV##\n'; cat /proc/net/dev;
printf '##ARP##\n'; arp -n;
printf '##ROUTE##\n'; ip route show;
printf '##END##\n'
```

### 대시보드 구성

| 섹션 | 표시 정보 |
| --- | --- |
| 시스템 정보 카드 | Hostname, Uptime, Kernel, CPU Load, Memory |
| 네트워크 경로 카드 | ip route show 결과 (읽기 전용 Text 위젯) |
| 네트워크 인터페이스 테이블 | 인터페이스명, IP/prefix, MAC, 상태 |
| 트래픽 통계 테이블 | /proc/net/dev 기반 수신/송신 MB·패킷·오류 |
| ARP 테이블 | arp -n 기반 연결 장치 IP·MAC·인터페이스·상태 |

모든 테이블: 헤더 클릭으로 컬럼 정렬 (IP는 옥텟 숫자 정렬, 나머지 알파벳·숫자 정렬).

---

## 20.5 모듈 추가 현황

| 모듈 | 추가 시점 | 역할 |
| --- | --- | --- |
| `ap_remote.py` | AP Remote 탭 구현 | AP SSH 스캔, mca-dump 파싱, 진단 데이터 |
| `efg_remote.py` | EFG Remote 탭 구현 | EFG 대시보드 데이터 수집·파싱 |
| `keychain.py` | 보안 Step 8 | Windows Credential Manager keyring 래퍼 |
| `ssh_client.py` | SSH 공통화 | `run_ssh_command()`, `run_ssh_via_relay()`, PTY 관리 |

### ssh_client.py 핵심 결정

- `run_ssh_command()`: 패스워드 없으면 SSHClient (키 인증), 있으면 raw Transport (auth_none probe 회피)
- 패스워드 인증 순서: `auth_password` → `auth_interactive_dumb` (fallback)
- `run_ssh_via_relay()`: EFG에서 `sshpass+ssh -T` 실행으로 Dropbear 호환성 우회
- `_exec_on_transport()`: `chan.makefile("rb").read()` 로 블로킹 읽기 — poll 루프 race condition 제거

---

## 20.6 알려진 설계 한계

| 항목 | 한계 | 비고 |
| --- | --- | --- |
| Treeview 셀 색상 | Tkinter ttk.Treeview는 셀 단위 색상 미지원 | `⚠ {값}` 텍스트 접두어로 대체. Reset 대상 행은 tag로 배경색 적용 가능 |
| 클라이언트 수 0 | Stuck인지 사용자 없음인지 구분 불가 | ELK Stuck 컬럼으로 cross-reference |
| EFG sshpass | EFG에 sshpass 설치 필요 | 없으면 AP Remote/Reset 불가 |
| 다중 인스턴스 | Windows Named Mutex로 중복 실행 방지 | |

---

# 21. 2026-05-27 업데이트 — AP Remote 개선 및 기타

---

## 21.1 AP 네트워크 서브넷 통합

- AP 29대 전체가 `192.168.11.x` 단일 서브넷으로 마이그레이션 완료
- `gui/config/ap_inventory.json` 전체 IP 업데이트
- EFG `192.168.8.0/21` 라우트가 `192.168.11.x`를 포함하므로 추가 라우팅 불필요

---

## 21.2 개발자 모드 Machine ID 안정화

### 문제
`uuid.getnode()` (MAC 주소 기반) 사용 시 VPN 연결 시 가상 어댑터 MAC이 반환되어  
등록 시점(VPN 없음)과 실행 시점(VPN 있음) ID 불일치 → 개발자 자동 로그인 실패.

### 해결
Windows 레지스트리 `HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid` 사용.  
OS 설치 시 1회 생성, 네트워크 어댑터 상태와 완전 무관.

```python
import winreg
with winreg.OpenKey(winreg.HKEY_LOCAL_MACHINE, r"SOFTWARE\Microsoft\Cryptography") as key:
    guid, _ = winreg.QueryValueEx(key, "MachineGuid")
return hashlib.sha256(guid.encode()).hexdigest()[:16]
```

비-Windows 환경 fallback: hostname hash.

---

## 21.3 AP Remote — relay SSH ConnectTimeout

### 문제
응답 없는 AP에서 sshpass+ssh가 무한 대기 → AP 1대당 최대 20초 hang.

### 해결
`run_ssh_via_relay()`의 relay_command에 `-o ConnectTimeout={max(5, timeout-2)}` 추가.  
내부 ssh가 AP보다 2초 먼저 종료 → paramiko channel timeout 이전에 정리됨.

오류 메시지 개선: `str(exc) or type(exc).__name__` — `socket.timeout()`의 빈 문자열 방지.

relay stdout이 비어 있으면 "AP 미응답" 명시적 에러 반환.

---

## 21.4 AP Remote — 다중 Reset 선택 시스템

### 설계 변경

V1 원칙(단일 AP Reset)에서 다중 선택 Reset으로 확장.

```
_reset_target_ap_ids: set[str]   — Reset 대상 AP ID 집합
remote_table tag "reset_target"  — 배경 #ffcccc, 전경 #cc0000 (빨간색)
```

### 버튼 배치

```
[상세 보기]    [AP Reset 선택] [AP 선택 해제] | [모두 선택] [모두 해제] ·· [AP Reset 실행]
```

### 동작

| 버튼 | 내부 동작 |
| --- | --- |
| AP Reset 선택 | `remote_table.selection()` → `_reset_target_ap_ids.add()` + tag 적용 |
| AP 선택 해제 | `_reset_target_ap_ids.discard()` + tag 제거 |
| 모두 선택 | inventory 전체 순회, tag 적용 |
| 모두 해제 | set 전체 clear, tag 전체 제거 |
| AP Reset 실행 | set 순회 → IP 검증 → 확인 팝업(목록) → 순차 reset → 결과 표시 |

Shift 다중 선택: `remote_table.selection()`이 다중 iid 반환 → 자동 지원.

### 상세 보기 다중

`remote_table.selection()` 기반, 최대 3개 제한.  
초과 시 안내 후 앞 3개만 별도 창으로 표시.

---

## 21.5 AP Remote — 기본 정렬 설정

### 저장 위치

`app_config.json`의 `remote_default_sort` 키 (문자열, 기본값 `"name"`).

### 정렬 방향 규칙

```python
_REMOTE_SORT_NUMERIC = {"elk_stuck", "log_stuck", "rst_2g", "rst_5g",
                         "vto_2g", "vto_5g", "cu_2g", "cu_5g", "ms"}
# 숫자형 → 내림차순 (큰 값 상단)
# 문자형 → 오름차순 (A→Z)
```

### 적용 시점

GUI 시작 시 `_load_remote_inventory_rows()` 직후 **1회만** 적용.  
이후 수동 컬럼 클릭 정렬 유지. `_finish_remote_scan`에서 재정렬 없음.

### 토글 우회 기법

`_sort_remote_table(col)`은 `remote_sort_reverse[col]`을 toggle한다.  
`_apply_remote_sort(col)`은 원하는 방향의 반대값을 미리 저장한 후 호출:

```python
self.remote_sort_reverse[col] = not want_reverse  # 원하는 방향의 반대
self._sort_remote_table(col)                       # toggle → 원하는 방향
```

---

## 21.6 기타 UI 변경

| 변경 | 내용 |
| --- | --- |
| GUI 시작 최대화 | `self.state("zoomed")` — 항상 최대화 상태로 시작 |
| 개발자 모드 팝업 제거 | 시작 시 4초 자동 닫힘 팝업 삭제. 상단 배지 텍스트만 유지 |
| 설정 탭 추가 | "AP Remote 기본 정렬" 섹션 추가 (Combobox + 저장 버튼) |
