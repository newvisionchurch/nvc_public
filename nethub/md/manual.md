# NVC NetHub — 사용자 매뉴얼

이 문서는 뉴비전 교회 네트워크 운영팀이 NVC NetHub를 실행하고 주요 기능을 사용하는 방법을 설명합니다.

## 목차

- [시작하기](#시작하기)
- [AP Status](#ap-status)
- [EFG Remote](#efg-remote)
- [Log Export](#log-export)
- [동기화](#동기화)
- [설정](#설정)
- [게시판](#게시판)
- [자주 묻는 질문](#자주-묻는-질문)

## 시작하기

### 요구사항

| 항목 | 요구 사항 |
|------|-----------|
| 운영체제 | Windows 10/11 64-bit |
| Python | Source 패키지 실행 시 Python 3.10 이상 |
| VPN | 교회 내부망 VPN 연결 필요 |
| OTP 앱 | Google Authenticator 또는 Authy |
| GitHub PAT | `nvc_security` 접근 권한이 있는 개인 토큰 |

### 실행

Windows 배포 패키지는 포함된 실행 파일을 실행합니다.

Source 패키지는 프로젝트 루트에서 실행합니다.

```powershell
.\scripts\run.ps1
```

### 로그인

1. VPN을 먼저 연결합니다.
2. 시작 인증 화면에서 `1단계 VPN 접속 인증`을 통과합니다.
3. `2단계 팀원 인증 (GitHub ID & PAT)`에서 본인 GitHub ID와 PAT를 입력합니다.
4. 앱이 PAT의 GitHub ID, `nvc_security` 접근 권한, NetHub 사용자 정보의 GitHub ID 등록 여부를 확인합니다.
5. 팀원 인증이 끝나면 NetHub 계정 ID와 비밀번호를 입력합니다.
6. 로그인한 계정의 등록 GitHub ID와 현재 PAT의 GitHub ID가 같아야 개인 인증이 통과됩니다.
7. 최초 로그인 시 OTP QR 코드를 등록하고, 이후 로그인 시 OTP 앱의 6자리 코드를 입력합니다.

PAT는 로그인 화면이 아니라 시작 인증 화면에서만 입력합니다.
GitHub ID가 등록되어 있지 않거나 PAT의 GitHub ID와 로그인 사용자의 GitHub ID가 다르면 로그인이 실패합니다.

### 화면 구성

```text
상단: NVC NetHub / 문서 버튼 / 구조도 / 팀원 게시판
탭: AP Status | EFG Remote | 동기화 | Log Export | 설정
```

## AP Status

AP 29대의 상태를 조회, 분석, 제어하는 메인 탭입니다.

### 주요 버튼

| 버튼 | 동작 |
|------|------|
| 전체 조회 | 등록 AP 전체 SSH 진단 |
| 선택 조회 | 선택한 AP만 SSH 진단 |
| 중지 | 진행 중 조회 취소 |
| 동작 설명 | 컬럼 의미 확인 |
| 경고 설정 | 경고 표시 임계값 설정 |
| 컬럼 설정 | 표시 컬럼 선택 |
| ELK Stuck Count | 날짜 범위별 ELK Stuck Record 집계 |
| AP Reset 후보 분류 | ResetScore 기준으로 대상 분류 |
| AP Reset 실행 | 선택 AP Reset 예약 |
| 예약 관리 | Reset 예약 및 이력 확인 |
| 상세 보기 | AP 상세 로그와 Stuck 유형 확인 |

### 주요 컬럼

| 컬럼 | 의미 |
|------|------|
| AP / Name / IP / Model | AP 인벤토리 기본 정보 |
| 분류 / 점수 | ResetScore 결과 |
| ELK Stuck | 저장된 Record 기반 Stuck 이벤트 수 |
| 로그오류 | 현재 부팅 세션의 `logread` 오류 수 |
| DevReset | `WAL_DBGID_DEV_RESET` 감지 수 |
| 재시작2G/5G | 드라이버 리셋 누적 |
| VAP지연2G/5G | VAP 초기화 지연 |
| CU2G/CU5G | 채널 이용률 |
| Client2G/5G | 연결 클라이언트 수 |
| Ch/BW | UniFi API 채널/채널 폭 |
| CPU/Mem/Uptime | AP 시스템 상태 |
| 응답(ms) | SSH 진단 응답 시간 |

### Reset 분류

| 색상 | 기준 | 의미 |
|------|------|------|
| 빨강 | 높은 점수와 Reset 근거 | Reset 권장 |
| 노랑 | 관찰 필요 점수 | 관찰 필요 |
| 초록 | 낮은 점수 | 정상 |
| 회색 | 미조회 | 데이터 없음 |

### AP Reset

1. AP를 선택합니다.
2. `AP Reset 실행`을 누릅니다.
3. AP 비밀번호를 확인합니다.
4. 시작 시각, 실행 순서, 대상 AP, 간격을 설정합니다.
5. 예약은 GitHub에 저장되고, 실행 결과는 `runtime/operations.log`와 GitHub 이력에 AP별로 기록됩니다.

### AP Reset 예약 관리

`예약 관리`에서 예약과 이력을 확인합니다.

| 탭 | 내용 |
|----|------|
| 예약 목록 | 아직 실행 전이거나 실행 중인 예약만 표시하고 선택 예약을 취소 |
| AP Reset 이력 | 완료된 AP별 Reset 결과를 최신순으로 표시 |

목록에서 행을 선택하면 하단 상세 영역에 대상 AP 총수, 실행 시각, 예약 ID, 결과 메시지가 표시됩니다.
Reset 진행 팝업과 예약 관리 팝업은 동시에 열어 둘 수 있습니다.

## EFG Remote

EFG 시스템 상태와 UniFi Controller API를 조회합니다.

### EFG SSH

| 섹션 | 내용 |
|------|------|
| 시스템 정보 | hostname, uptime, loadavg, memory, disk |
| 프로세스 상태 | unifi, mongod, java, dnsmasq, dhcp |
| 네트워크 경로 | `ip route show` |
| 인터페이스 | 이름, IP, MAC, 상태 |
| 트래픽 통계 | Rx/Tx, 패킷, 오류 |
| ARP 테이블 | IP, MAC, 인터페이스, 상태 |

### EFG API

UniFi Controller API를 read-only GET으로 탐색합니다.

- 좌측: API 연결 상태, Site 정보, 장비 현황, WLAN, 장비 목록
- 우측: Preset 선택, GET 실행, JSON 응답 확인
- 전체 Preset GET 결과는 `runtime/api_probe/`에 저장됩니다.
- API Key는 설정 탭의 보안 영역에서 등록합니다.

## Log Export

로컬 캐시에서 조건별 로그를 필터링해 파일로 저장합니다.

| 필터 | 설명 |
|------|------|
| 날짜 범위 | From ~ To |
| AP | 전체 또는 특정 AP |
| 유형 | `ap_stuck`, `ap_no_service`, `rf_issue`, `dhcp_dns`, `qos`, `system_issue` |
| 키워드 | 추가 텍스트 조건 |

Export 전 동기화 탭에서 최신 로그를 먼저 동기화하는 것을 권장합니다.

## 동기화

### NAS ELK Sync

화면은 왼쪽 `NAS ELK Sync`, 오른쪽 `ELK Stuck Record`로 나뉩니다.
왼쪽은 NAS Log 동기화와 PC Log 상태 확인, 오른쪽은 날짜별 Record 생성과 조회에 사용합니다.

| 버튼 | 동작 |
|------|------|
| Log 동기화 | NAS JSONL 로그를 로컬 캐시로 동기화 |
| Log 삭제 | 지정 날짜 범위 로컬 캐시 삭제 |
| 전체 Log 삭제 | 로컬 캐시 전체 삭제 |
| Log 상태 새로고침 | 로컬 캐시 용량과 날짜 현황 갱신 |

### ELK Stuck Record

`ELK Stuck Record`는 날짜별 AP Stuck Count를 저장해 두는 데이터입니다.
AP Status의 `ELK Stuck Count`는 이 Record를 읽어 빠르게 `ELK Stuck` 컬럼을 갱신합니다.

| 버튼 | 동작 |
|------|------|
| Record 목록 | 저장된 전체 Record 날짜 범위와 요약 확인 |
| ELK Stuck Record | 선택 날짜 범위의 Record 생성 |
| 오늘만 Record | 오늘 날짜만 다시 Count하고 저장 |
| 빈 Count만 | 선택 범위에서 Record가 없는 날짜만 Count |
| 다시 Count | 선택 범위의 Record를 강제로 다시 계산 |
| 전체 Record 삭제 | 저장된 ELK Stuck Record 전체 삭제 |
| 그래프 | 날짜별 AP ELK Stuck 추이 라인 그래프 팝업 |

Record 목록에는 AP별 `누적`, `평균`, `최고` 요약 행이 먼저 표시되고, 그 아래에 최근 날짜부터 날짜별 Count가 표시됩니다.
값의 크기에 따라 행 색상이 적용됩니다 — 회색(0), 파랑(낮음), 노랑(중간), 빨강(높음).

#### ELK Stuck 그래프

`그래프` 버튼을 누르면 날짜별 AP ELK Stuck 추이 팝업이 열립니다.

- X축: 날짜, Y축: ELK Stuck Count
- AP별 고유 색상 라인으로 추이 표시
- 오른쪽 패널에서 AP 클릭으로 표시/숨김 선택, 전체/해제 버튼 제공
- Y축 최소/최대 값을 직접 지정하거나 자동 범위로 설정 가능
- 라인 위에 마우스를 올리면 AP ID, 이름, 값이 툴팁으로 표시

### ELK Stuck Count 집계

AP Status의 `ELK Stuck Count` 옆에서 집계 방식을 선택할 수 있습니다.

| 방식 | 의미 |
|------|------|
| 누적 | 선택 날짜 범위에서 AP별 Count 합계 |
| 최고 | 선택 날짜 범위에서 AP별 일별 Count 최대값 |
| 평균 | 선택 날짜 범위에서 AP별 일별 Count 평균값, 소수점 버림 |

선택한 날짜에 Record가 없으면 AP Status에서 모두 0처럼 보일 수 있습니다.
먼저 동기화 탭에서 Log를 동기화하고 `ELK Stuck Record`를 생성한 뒤 집계합니다.

### AP Sync

AP mca-dump 파일을 관리합니다. 기본 저장 경로는 `runtime/mca_dumps/`입니다.

### 연결 현황

NAS Sync 설정과 UniFi API 연결 상태를 확인합니다.

## 설정

| 서브탭 | 내용 | 접근 |
|--------|------|------|
| 일반 | 폰트, AP 등록 수, 정렬, 임계값, 색상 | 모든 사용자 |
| 자동화 | 시작 시 자동 조회, ELK Log 동기화/Record/집계, 점수분류 Refresh | 모든 사용자 |
| 보안 | ELK 로그 캐시 경로 | 모든 사용자 |
| 관리자 | 사용자 관리, 팀 공유 비밀번호 | `can_manage_users` |

관리자 탭은 `can_manage_users` 권한이 있는 계정에만 표시됩니다.

관리자 탭 서브탭:

| 서브탭 | 내용 |
|--------|------|
| 사용자 관리 | 계정 추가/편집/삭제, 역할, 권한, TOTP, 개발자 PC |
| 보안 설정 | 팀 공유 비밀번호 (NAS/EFG/AP SSH, EFG API Key) nvc_security 암호화 저장 |

### 자동화 설정

시작 시 자동 실행에서 ELK 관련 흐름을 연결할 수 있습니다.

| 옵션 | 동작 |
|------|------|
| AP 전체 조회 | 시작 후 AP SSH mca-dump 전체 조회 |
| ELK Stuck Count 집계 | 시작 후 AP Status의 `ELK Stuck` 값을 Record 기준으로 반영 |
| 동기화 후 집계 | ELK Count 전에 NAS 오늘 Log 동기화를 먼저 수행 |
| 시작 시 NAS 1일 자동 동기화 후 오늘 Record | 시작 동기화 후 오늘 날짜 Record 생성 |
| Record 확정 확인 횟수 | 같은 Log 상태가 몇 번 확인되면 과거 날짜 Record를 안정 상태로 볼지 설정 |
| 동기화 탭 왼쪽 비율 | `NAS ELK Sync`와 `ELK Stuck Record` 패널 분할 비율 (3~35%) |
| EFG SSH 진단 | 시작 후 EFG SSH 진단 자동 실행 |
| EFG API 전체 Preset 조회 | 시작 후 UniFi API 전체 Preset 자동 조회 |
| 시작 시 메시지 창 열기 | 시작 자동 실행 로그를 좌(AP 조회) / 우(NAS·EFG·ELK) 분할 메시지 창에 표시 |
| 시작 작업 완료 후 자동 닫기 | 자동 실행이 모두 끝난 뒤 지정 초 후 메시지 창 닫기 |

`ELK Stuck Count 집계`와 `동기화 후 집계`가 모두 켜져 있으면 실행 순서는 오늘 Log 동기화, 오늘 Record 생성, AP Status 집계입니다.
AP 전체 조회도 켜져 있으면 AP별 진행 메시지는 `AP 조회 중... AP_NAME (n/29)` 형식으로 표시됩니다.

#### 시작 메시지 창

시작 자동 실행 진행 상황을 실시간으로 확인합니다.

- 왼쪽 패널: AP 조회 진행 현황
- 오른쪽 패널: NAS 동기화, EFG 진단, EFG API, ELK Count 결과
- 각 항목별 색상 구분: AP(파랑), NAS(주황), EFG(청록), API(자홍), ELK(초록), 시작(보라)
- 상단 `메시지` 버튼으로 언제든 다시 열 수 있습니다.

## 게시판

상단 `팀원 게시판` 버튼에서 팀 공유 글을 작성하고 확인합니다.

| 유형 | 작성 가능자 |
|------|------------|
| 버그 | 모든 사용자 |
| 개선 | 모든 사용자 |
| 공지 | 관리자 |
| 기록 | 관리자 게시판 |

새 공지가 있으면 로그인 후 자동으로 표시됩니다.
공지 팝업은 큰 읽기 영역과 스크롤을 제공하며, 같은 공지를 다시 보지 않으려면 `다음부터 이 공지 보지 않음`을 체크한 뒤 닫습니다.

## 자주 묻는 질문

| 증상 | 원인 | 해결 |
|------|------|------|
| VPN 없이 실행 | 내부망 접근 불가 | VPN 연결 후 재시도 |
| OTP 코드 오류 | 스마트폰 시간 오차 | 시간 자동 동기화 확인 |
| SSH Fail 표시 | AP 전원 또는 네트워크 문제 | VPN과 AP 상태 확인 |
| ELK Stuck이 모두 0 | 해당 날짜 Record 없음 | 동기화 탭에서 NAS ELK Sync 후 ELK Stuck Record 생성 |
| AP Reset 버튼 비활성 | 권한 없음 | 관리자에게 권한 요청 |
| 게시판 오류 | GitHub PAT 만료 또는 권한 부족 | PAT 재입력 또는 권한 확인 |
| GitHub 연결 실패 | PAT 없음, 만료, 권한 부족, 네트워크 문제 | 시작 화면의 `GitHub PAT 입력`으로 팀원 인증 재시도 |
| 개인 인증 실패 | 로그인 사용자 GitHub ID와 PAT GitHub ID 불일치 | 본인 GitHub 계정의 PAT 입력 또는 관리자에게 GitHub ID 등록 요청 |
