# NVC NetHub — 사용자/배포 매뉴얼

GUI Version 0.4 — 05/31/2026  
대상: 뉴비전 교회 네트워크 운영팀

---

## 목차

- [1. 시작하기](#1-시작하기)
- [2. AP Status 탭](#2-ap-status-탭)
- [3. EFG Remote 탭](#3-efg-remote-탭)
- [4. Log Export 탭](#4-log-export-탭)
- [5. 동기화 탭](#5-동기화-탭)
- [6. 설정 탭](#6-설정-탭)
- [7. 팀원 게시판](#7-팀원-게시판)
- [8. 권한 구조](#8-권한-구조)
- [9. 관리자 초기 설정](#9-관리자-초기-설정)
- [10. 배포 절차](#10-배포-절차)

---

## 1. 시작하기

### 1-1. 시스템 요구사항

| 항목 | 요구 사양 |
|------|-----------|
| 운영체제 | Windows 10/11 (64-bit) |
| Python | 3.10 이상 (3.13 권장) |
| VPN | 교회 내부망 VPN — GUI 실행 **전** 반드시 연결 |
| OTP 앱 | Google Authenticator 또는 Authy (TOTP 2FA) |
| 네트워크 | 192.168.11.x 대역 접근 가능 |

### 1-2. 설치 및 실행

```powershell
# 방법 1 — scripts/run.ps1 (권장: venv 자동 생성)
cd C:\Projects\nvc_nethub
.\scripts\run.ps1

# 방법 2 — 직접 실행
pip install -r source\requirements.txt
python source\main.py
```

`python --version` 실행 시 Microsoft Store 메시지가 나오면:  
Windows 설정 → 앱 → 앱 실행 별칭에서 `python.exe`/`python3.exe` 별칭을 끄세요.

### 1-3. 로그인

1. ID / 비밀번호 입력 → [로그인]
2. 최초 로그인: TOTP 2FA 등록 화면 표시
   - QR 코드를 Google Authenticator / Authy로 스캔
   - 앱에 표시된 6자리 코드 입력
3. 이후 로그인 시마다 TOTP 코드 입력 (기기 신뢰 등록 후 생략 가능)

개발자 등록 PC: 비밀번호·TOTP 없이 자동 로그인.

### 1-4. 화면 구성

```
[타이틀 바] 뉴비전 네트워크 관리 v0.3  [GUI 매뉴얼] [GUI 개요] [프로젝트 개요] [💬 팀원 게시판]
[메인 탭]   AP Status | EFG Remote | Log Export | 동기화 | 설정
```

---

## 2. AP Status 탭

AP 29대의 상태를 조회·분석·제어하는 메인 탭.

### 2-1. 컨트롤 바

**섹션 1 — 상태 카드**: 등록 / 온라인 / 오프라인 AP 수

**섹션 2 — SSH 조회**

| 버튼 | 동작 |
|------|------|
| 전체 조회 | 등록 AP 전체 SSH 접속, mca-dump + logread |
| 선택 조회 | 선택한 AP만 조회 |
| 중지 | 진행 중 조회 취소 |
| 동작 설명 | 컬럼 의미 팝업 |
| 경고 설정 | ⚠ 임계값 설정 팝업 |
| 컬럼 설정 | 표시 컬럼 선택 팝업 |

**섹션 3 — ELK 날짜 범위**: 전체/범위 선택, 오늘 단축 버튼, 날짜 목록 팝업

**섹션 4 — ELK Stuck Count**: 버튼 클릭 시 지정 날짜 범위 Stuck 집계

### 2-2. AP 목록 테이블

| 컬럼 | 소스 | 의미 |
|------|------|------|
| AP | 인벤토리 | AP01 ~ AP29 |
| Name | 인벤토리 | 위치명 |
| IP | 인벤토리 | 관리 IP |
| Model | 인벤토리 | 모델명 |
| 분류 | ResetScore | 색상 아이콘 (🔴🟡🟢⬜) |
| 점수 | ResetScore | 합산 위험 점수 |
| ELK Stuck | ELK 캐시 | Stuck 이벤트 수 |
| 로그오류 | logread grep | 부팅 세션 내 오류 수 |
| DevReset | logread | WAL_DBGID_DEV_RESET 수 |
| 재시작2G/5G | mca-dump | 드라이버 리셋 누적 |
| VAP지연2G/5G | mca-dump | VAP 타임아웃 누적 |
| CU2G%/CU5G% | mca-dump | 채널이용률 |
| Client2G/5G | mca-dump | 연결 클라이언트 수 |
| Ch/BW 2G/5G/6G | UniFi API | 채널/채널폭 |
| CPU% / Mem% | mca-dump | 사용률 |
| Uptime | mca-dump | 마지막 재부팅 경과 시간 |
| 응답(ms) | SSH | mca-dump 왕복 시간 |

**Stuck 색상**: 0=흰색 / 1~2=노랑 / 3~9=주황 / 10+=빨강

### 2-3. AP 분류 및 Reset 후보

SSH 조회 후 **[AP Reset 후보 분류]** → ResetScore 계산.

| 색상 | 기준 | 의미 |
|------|------|------|
| 빨강 | 점수 ≥60 + Reset 근거 | Reset 권장 |
| 노랑 | 점수 ≥20 | 관찰 필요 |
| 초록 | 점수 <20 | 정상 |
| 회색 | 미조회 | 데이터 없음 |

### 2-4. AP Reset 실행

**권한**: `can_reset_ap`

```
GUI (PC) → EFG (OpenSSH) → sshpass → AP (Dropbear)
```

1. Reset 대상 AP 선택
2. [AP Reset 실행] 클릭
3. AP 비밀번호 입력 (처음 한 번)
4. Reset 스케줄 팝업: 시작시각 / 실행순서 / 대상 AP / 간격(분) / 실행모드 설정
5. [실행 예약] → 지정 시각에 순차 실행

결과: `runtime/operations.log` + GitHub `ap_reset_schedule.json` history 기록

### 2-5. 예약 관리

하단 [예약 관리] → GitHub `nvc-auth/ap_reset_schedule.json` 기반 팀 공유.

**탭 1 — 예약 목록**: 상태별 색상 (노랑=pending / 파랑=running / 초록=done / 회색=cancelled)  
**탭 2 — AP Reset 이력**: 전체 이력, AP별/결과별 필터

### 2-6. 상세 보기

AP 선택 후 [상세 보기] → Stuck 유형 분류 (channel_invalid/vap_timeout/radio_reset) + mca-dump 전체 + 오류 목록

---

## 3. EFG Remote 탭

**권한**: `can_view_efg_tab`

### 3-1. EFG SSH 대시보드

[진단 조회] → EFG SSH 접속하여 수집:

| 섹션 | 내용 |
|------|------|
| 시스템 정보 | hostname, uptime, loadavg, memory, disk |
| 프로세스 상태 | unifi/mongod/java/dnsmasq/dhcp 실행 상태 |
| 네트워크 경로 | ip route show |
| 인터페이스 | 이름/IP/MAC/상태 |
| 트래픽 통계 | Rx/Tx MB, 패킷, 오류 |
| ARP 테이블 | IP/MAC/인터페이스/상태 |

### 3-2. EFG API 탐색기

UniFi Controller API read-only GET 탐색. 레이아웃: 좌(30%) / 우(70%).

**좌측**: API 연결 상태 / Site 정보 / 장비 현황 / WLAN 드롭다운 / 전체 장비 목록 (AP/SW/EFG 필터)  
**우측**: Preset 20개 드롭다운 + [GET] / JSON 뷰어

**전체 Preset GET**: 20개 순서대로 실행 → `runtime/api_probe/YYYYMMDD_HHMMSS_<preset>.json` 저장

API Key: 설정 → 보안 → UniFi API Key 등록 필요.

---

## 4. Log Export 탭

**권한**: `can_export`

로컬 캐시에서 조건별 로그 필터 후 파일 저장.

| 필터 | 설명 |
|------|------|
| 날짜 범위 | From ~ To (YYYY-MM-DD) |
| AP | ALL 또는 특정 AP (AP01~AP29) |
| 유형 | ap_stuck / ap_no_service / rf_issue / dhcp_dns / qos / system_issue |
| 키워드 | 추가 텍스트 필터 (선택사항) |

[Export 실행] → `runtime/exports/` 저장.  
Export 전 동기화 탭에서 최신 로그 동기화 권장.

---

## 5. 동기화 탭

**NAS ELK Sync / AP Sync / 연결 현황** 3개 서브탭.

### 5-1. NAS ELK Sync

**권한**: `can_sync_nas`

| 버튼 | 동작 |
|------|------|
| Log 동기화 | NAS JSONL → PC 로컬 캐시 (차분) |
| Log 삭제 | 지정 날짜 범위 로컬 캐시 삭제 |
| 전체 Log 삭제 | 로컬 캐시 전체 삭제 |
| Log 상태 새로고침 | 용량 및 날짜 현황 갱신 |

NAS SSH 접속 정보: `source/config/ssh_targets.json` 필요.

### 5-2. AP Sync

AP mca-dump 파일 관리. 저장 경로: `runtime/mca_dumps/` (날짜별 폴더)

### 5-3. 연결 현황

경로 정보 + NAS Sync 설정 + UniFi EFG API 연결 테스트 ([TEST] 버튼 → 상태/응답시간/HTTP 코드).

---

## 6. 설정 탭

**일반 / 자동화 / 보안 / 사용자** 4개 서브탭.

### 6-1. 일반

UI 폰트, AP 등록 수, 기본 정렬, ⚠ 임계값, ELK Stuck 기본 날짜 범위, logread 최대 줄 수.

**색상 설정**: [주요 메뉴 색상] / [기능 메뉴 색상] 팝업으로 UI 색상 변경.

### 6-2. 자동화

GUI 시작 시 자동 실행 항목 설정:
- AP 전체 조회 (SSH) / AP 분류 색상 적용 / 정렬 적용
- ELK Stuck Count 집계 (날짜 범위 + 동기화 선행 여부)
- EFG SSH 진단 / EFG API 전체 Preset

### 6-3. 보안

- ELK 로그 캐시 경로 변경
- Windows Keychain 비밀번호 삭제 (NAS/AP/EFG SSH)
- UniFi EFG API 설정 (Base URL, API Key)
- 인증 방식 설정 (Admin 전용): GitHub/Local, 자동이동 대기, GitHub PAT

### 6-4. 사용자 (Admin)

**권한**: `can_manage_users`

테이블: username / display_name / role / active / 마지막 로그인 / TOTP 등록  
버튼: 새 사용자 추가 / 편집 / TOTP 초기화 / 역할 권한 설정 / 삭제

**개발자 컴퓨터 등록**: [이 컴퓨터 개발자 등록] → 자동 로그인 PC 등록

---

## 7. 팀원 게시판

상단 타이틀 바 [💬 팀원 게시판] → GitHub `nvc-auth/feedback.json` 기반.

| 유형 | 작성 가능자 |
|------|------------|
| 버그 | 모든 사용자 |
| 개선 | 모든 사용자 |
| 공지 | Admin 전용 |

로그인 후 2.5초 뒤 새 공지(status=open)가 있으면 자동 팝업 표시.

---

## 8. 권한 구조

| 기능 | viewer | operator | admin |
|------|:---:|:---:|:---:|
| AP Status 조회 | ✓ | ✓ | ✓ |
| AP Reset 실행 | ✗ | 설정에 따름 | ✓ |
| EFG Remote | ✓ | ✓ | ✓ |
| Log Export | ✓ | ✓ | ✓ |
| NAS 동기화 | ✗ | ✓ | ✓ |
| 예약 관리 | ✓ | ✓ | ✓ |
| 팀원 게시판 | ✓ | ✓ | ✓ |
| 공지 작성 | ✗ | ✗ | ✓ |
| 사용자 관리 | ✗ | ✗ | ✓ |
| GitHub PAT 설정 | ✗ | ✗ | ✓ |

---

## 9. 관리자 초기 설정

### 9-1. SSH 설정 파일 생성

`source/config/ssh_targets.example.json` 복사 → `ssh_targets.json` 작성:

```json
{
  "nas": { "host": "192.168.x.x", "port": 22, "username": "admin", "key_path": "~/.ssh/id_rsa" },
  "efg": { "host": "192.168.11.1", "port": 22, "username": "admin" },
  "ap_defaults": { "username": "ubnt", "port": 22 }
}
```

### 9-2. GitHub PAT 등록

GitHub → Settings → Developer settings → Personal access tokens → Generate new token  
권한: `repo` (nvc-auth read/write). GUI → 설정 → 보안 → GitHub PAT → [저장].

### 9-3. UniFi API Key 등록

UniFi Controller → Settings → System → API → Generate API Key.  
GUI → 설정 → 보안 → API Key 입력 → [API Key 저장 (keyring)].

### 9-4. 관리자 컴퓨터 등록

GUI 로그인 (GitHub 연결 상태) → 설정 → 사용자 → [이 컴퓨터 개발자 등록].  
등록된 PC: admin 비밀번호 해시 로컬 저장 → 오프라인 admin 로그인 가능.

### 9-5. 팀원 등록 절차

1. `github.com/orgs/newvisionchurch-it/people` → Invite member (nvc_release 접근)
2. `github.com/newvisionchurch/nvc-auth/settings/access` → Add people (Read 권한)
3. GUI → 설정 → 사용자 → [새 사용자 추가] → GitHub nvc-auth/users.json 에 자동 저장

---

## 10. 배포 절차

배포 repo: `https://github.com/newvisionchurch-it/nvc_release` (Private)

### 10-1. 빌드

```powershell
cd C:\Projects\nvc_nethub
python -m venv .venv
.venv\Scripts\activate
pip install -r source\requirements.txt
pip install pyinstaller

# Windows exe 빌드
.\source\tools\build_dist.ps1

# Source 패키지 빌드
.\source\tools\build_source.ps1
```

빌드 완료 → `C:\Projects\nvc_release\releases\v0.3\` 자동 복사.

### 10-2. 배포 push

```powershell
cd C:\Projects\nvc_release
git add .
git commit -m "release: v0.3 배포 패키지 추가"
git push origin main
```

### 10-3. 팀원 설치 안내

팀원: `github.com/newvisionchurch-it/nvc_release` → `releases/v0.3/win/nvc_nethub_GUI_v0.3.zip` 다운로드 → 압축 해제 → `config/ssh_targets.json` 기존 파일 복사 → VPN 연결 → 실행.

### 10-4. 팀원 배포 이메일 템플릿

```
제목: 뉴비전 네트워크 NVC NetHub v0.3 배포 안내

■ 사전 준비 (관리자가 처리)
  - GitHub 계정 username을 알려주세요
  - newvisionchurch-it 조직 초대 + nvc-auth Collaborator 등록 처리

■ 설치 순서
  1. OTP 앱 설치 (스마트폰)
     iOS/Android: Google Authenticator 또는 Authy

  2. GitHub PAT 발급 (본인 계정)
     GitHub → Settings → Developer settings → Tokens (classic)
     권한: repo / 만료: 1년

  3. nvc_release repo에서 zip 다운로드
     https://github.com/newvisionchurch-it/nvc_release
     → releases/v0.3/win/nvc_nethub_GUI_v0.3.zip

  4. 압축 해제

  5. VPN 연결 후 실행
     Windows: nvc_nethub_GUI.exe 더블클릭

  6. 로그인
     - GitHub PAT 입력 (최초 1회, 이후 자동)
     - 아이디: [아이디] / 비밀번호: [비밀번호] (별도 전달)

  7. OTP 등록 (최초 1회)
     - QR 코드를 OTP 앱으로 스캔
     - 앱에 "NVC NetHub" 항목 생성
     - 6자리 코드 입력 → 로그인 완료

■ 이후 로그인: VPN → 실행 → 아이디/비밀번호/OTP 6자리

■ 주의: 비밀번호·PAT는 카카오톡/문자로만 전달 (이메일 금지)

문의: ai@newvisionchurch.org
```

---

## 자주 묻는 질문

| 증상 | 원인 | 해결 |
|------|------|------|
| VPN 없이 실행 | 내부망 접근 불가 | VPN 연결 후 재시도 |
| TOTP 코드 틀림 | 스마트폰 시간 오차 | 시간 자동 동기화 확인 |
| SSH Fail 표시 | AP 전원/네트워크 문제 | VPN 정상 확인 후 AP 점검 |
| ELK Stuck 전부 0 | 동기화 미실행 | 동기화 탭 → NAS ELK Sync 실행 |
| AP Reset 버튼 비활성 | `can_reset_ap` 권한 없음 | Admin에게 권한 요청 |
| 예약 관리 오류 | GitHub PAT 미등록 | 설정 → 보안에서 PAT 확인 |
| 팀원 게시판 제출 실패 | PAT 만료 또는 인터넷 | PAT 재발급 확인 |
| 빌드 실패 (ImportError) | 패키지 미설치 | `pip install -r source\requirements.txt` |
| 로그인 실패 (모든 PC) | GitHub PAT 만료 | PAT 재발급 + nvc-auth Collaborator 확인 |
