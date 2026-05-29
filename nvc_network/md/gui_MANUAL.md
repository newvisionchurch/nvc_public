# NVC Network GUI — 사용자 매뉴얼

GUI Version 0.1 — 05/29/2026  
대상: 뉴비전 교회 네트워크 운영팀

---

## 목차

- [공통 사전 준비](#공통-사전-준비)
- [Part 1 — 관리자(Admin) 매뉴얼](#part-1--관리자admin-매뉴얼)
- [Part 2 — 팀원(User) 매뉴얼](#part-2--팀원user-매뉴얼)
- [Part 3 — 탭별 기능 상세](#part-3--탭별-기능-상세)
- [권한 구조 참조표](#권한-구조-참조표)
- [자주 묻는 질문](#자주-묻는-질문)

---

## 공통 사전 준비

### 시스템 요구사항

| 항목 | 요구 사양 |
|------|-----------|
| 운영체제 | Windows 10 / 11 (64-bit) |
| Python | 3.10 이상 (3.13 권장) |
| 스마트폰 앱 | Google Authenticator 또는 Authy |
| VPN | 교회 내부망 VPN (GUI 실행 전 반드시 연결) |

### Python 설치 확인

```powershell
python --version
```

`Python 3.10.x` 이상이 출력되면 정상입니다.  
설치 시 **"Add Python to PATH"** 옵션을 반드시 체크하세요.

---

## Part 1 — 관리자(Admin) 매뉴얼

> 이 섹션은 **관리자 1명만** 진행합니다.

---

### 1-1. 최초 설치

```powershell
cd C:\Projects\nvc_network
pip install -r gui\requirements.txt
```

---

### 1-2. Admin 계정 초기 생성

> **최초 1회만** 실행합니다. `gui/config/users.json`이 이미 있다면 건너뜁니다.

```powershell
python gui\init_users.py
```

> `gui/config/users.json`은 git에서 제외됩니다. 외부 유출 주의.

---

### 1-3. SSH 접속 설정

```powershell
copy gui\config\ssh_targets.example.json gui\config\ssh_targets.json
```

```json
{
  "nas":  { "host": "NAS_IP", "username": "nas_user", "port": 22 },
  "efg":  { "host": "EFG_IP", "username": "root",     "port": 22 },
  "ap_defaults": {
    "username": "ubnt",
    "port": 22,
    "reboot_command": "reboot"
  }
}
```

> 비밀번호는 파일에 저장하지 않습니다. 첫 실행 시 입력하면 Windows Credential Manager에 저장할지 묻습니다.

---

### 1-4. UniFi API 설정

Settings 탭 → **UniFi API 설정**:

1. Base URL: `https://192.168.11.1` (EFG IP)
2. API Key: UniFi Network → Settings → Integrations → API Keys에서 발급
3. **API Key 저장 (keyring)** 버튼 클릭
4. **API 연결 테스트 (Probe)** 버튼으로 연결 확인

---

### 1-5. GUI 실행

```powershell
# VPN을 먼저 연결한 뒤 실행
python gui\main.py
```

GUI는 항상 **최대화 상태**로 시작됩니다.

---

### 1-6. 팀원 계정 추가

설정 탭 → **사용자 관리** 서브탭 → **새 사용자 추가** 버튼

| 항목 | 설명 |
|------|------|
| 계정 ID | 팀원이 로그인할 ID |
| 표시 이름 | GUI에 표시될 이름 |
| 비밀번호 | 초기 비밀번호 8자 이상 |
| 역할 | `admin` / `operator` / `viewer` |

---

### 1-7. 팀원 계정 편집 / TOTP 초기화 / 삭제

사용자 관리 서브탭 → 해당 팀원 선택 → **편집** / **TOTP 초기화** / **삭제** 버튼

---

### 1-8. 개발자 모드 등록

등록된 PC에서 비밀번호·TOTP 없이 자동 로그인됩니다.

1. 정상 로그인 후 사용자 관리 서브탭 → **이 컴퓨터 개발자 등록**
2. 다음 실행부터 자동 로그인, 상단에 `⚙ 개발자 모드` 배지 표시

---

## Part 2 — 팀원(User) 매뉴얼

---

### 2-1. 사전 확인

- [ ] 관리자에게 계정 ID와 초기 비밀번호를 받았다.
- [ ] 스마트폰에 Google Authenticator 또는 Authy가 설치되어 있다.
- [ ] VPN이 설치되어 있고 연결 방법을 알고 있다.
- [ ] Python 3.10 이상이 설치되어 있다.

---

### 2-2. 최초 설치 및 실행

```powershell
cd C:\Projects\nvc_network
pip install -r gui\requirements.txt

# VPN 연결 후
python gui\main.py
```

---

### 2-3. 첫 로그인 — TOTP 초기 설정

1. ID / 비밀번호 입력 → 로그인
2. QR 코드 화면이 표시되면 Google Authenticator → **+** → **QR 코드 스캔**
3. 앱에 표시된 6자리 코드 입력 → **설정 완료**

> 코드는 30초마다 바뀝니다. 이후 매번 로그인 시 앱의 현재 코드를 입력합니다.

---

### 2-4. 이후 매번 로그인

```
1. VPN 연결
2. python gui\main.py 실행
3. ID + 비밀번호 → 로그인
4. 스마트폰 6자리 코드 입력 → 확인
5. 메인 화면 진입
```

---

### 2-5. 인증 앱 기기 변경 시

스마트폰을 교체하거나 앱을 재설치한 경우 **관리자에게 TOTP 초기화를 요청**하세요.  
초기화 후 다음 로그인 시 QR 코드를 다시 스캔하면 됩니다.

---

## Part 3 — 탭별 기능 상세

권한에 따라 일부 탭이나 버튼이 비활성화될 수 있습니다.

---

### AP Status 탭

AP와 직접 SSH 접속(EFG relay 경유)하여 무선 하드웨어 상태를 진단합니다.

#### 조회 방법

| 버튼 | 동작 |
|------|------|
| **전체 조회** | 인벤토리의 모든 AP(29대) 동시 조회 |
| **선택 조회** | 선택한 AP만 조회 (Shift로 다중 선택) |
| **중지** | 진행 중인 조회 중단 |

#### 진단 컬럼

| 컬럼 | 의미 |
|------|------|
| **ELK Stuck** | ELK Log 기반 누적 Stuck 이벤트 수 |
| **로그오류** | 현재 부팅 세션 중 오류 패턴 수 |
| **재시작2G/5G** | 드라이버 무선 리셋 횟수 |
| **VAP지연2G/5G** | VAP 초기화 타임아웃 횟수 |
| **CU2G%/CU5G%** | 채널 이용률 (높을수록 혼잡) |
| **2.4G/5G Cli** | 현재 연결 클라이언트 수 |
| **Ch 2G/5G/6G** | 현재 채널 (UniFi API 연동) |
| **BW 2G/5G/6G** | 채널 폭 MHz (UniFi API 연동) |
| **CPU%/Mem%** | AP CPU·메모리 사용률 |
| **Uptime** | 마지막 재부팅 이후 경과 시간 |
| **응답(ms)** | SSH 응답 왕복 시간 |

임계값 초과 셀에 `⚠` 표시. **⚠ 설정** 버튼으로 임계값 변경 가능.

#### ELK Stuck Count (Stuck 집계)

날짜 범위를 선택하고 **ELK Stuck Count** 실행:

| 횟수 | 상태 색상 |
|------|----------|
| 0 | Healthy (초록) |
| 1~2 | Warning (노랑) |
| 3~9 | Suspect (주황) |
| 10+ | Critical (빨강) |

#### AP Reset — 다중 선택 일괄 실행

> `can_reset_ap` 권한 필요

1. 테이블에서 Reset할 AP 클릭 (Shift 다중 선택)
2. **AP Reset 선택** → 선택 행이 빨간색으로 변함
3. **AP Reset 실행** → 대상 목록 확인 후 **예**
4. 결과 표시 + `runtime/operations.log` 기록

---

### EFG Remote 탭

#### EFG SSH 서브탭

**진단 조회** 버튼 클릭 → EFG SSH 접속 후 대시보드 표시:

| 섹션 | 표시 정보 |
|------|----------|
| 시스템 정보 | Hostname, Uptime, Kernel, Load, Memory |
| 네트워크 경로 | ip route show |
| 인터페이스 | 이름, IP/prefix, MAC, 상태 |
| 트래픽 통계 | 수신/송신 MB, 패킷, 오류 |
| ARP 테이블 | IP, MAC, 인터페이스, 상태 |

모든 테이블 헤더 클릭 시 정렬.

#### EFG API 서브탭

UniFi Controller API를 read-only GET으로 탐색합니다.

**시작 방법:**
1. 설정 탭 → **API 연결 테스트 (Probe)** → 연결 성공 확인
2. EFG API 탭 → **전체 Preset GET** 버튼 → 20개 preset 순서대로 실행

**좌측 패널:**
- API 연결 상태 / Site 정보 / 장비 현황 카드
- WLAN: SSID 드롭다운에서 선택 → 상세 정보 표시
- 장비 목록: **AP** / **SW** / **EFG** 중 선택하여 해당 장비만 표시

**우측 탐색기:**
- Preset 선택 → 저장된 캐시 자동 표시
- **GET** 버튼으로 최신 데이터 재조회
- AP 정보/통계 preset: AP 드롭다운에서 선택 시 즉시 조회

**주요 Preset:**

| 분류 | Preset | 용도 |
|------|--------|------|
| Official | sites | Site ID 확인 / API 연결 테스트 |
| Official | devices | AP/Switch 목록, online/offline, firmware |
| Official | clients | 현재 접속 클라이언트 수 |
| Official | networks | VLAN ID / 네트워크 목록 |
| Official | AP 정보 | 특정 AP 상세 (드롭다운 선택) |
| Official | AP 통계 | 특정 AP 최신 통계 |
| Legacy | wlanconf | SSID 설정, 밴드, 인증, VLAN |
| Legacy | networkconf | VLAN/네트워크 설정 |
| Legacy | health | Gateway/LAN/WAN health |
| Legacy | sysinfo | 컨트롤러 버전 정보 |
| Legacy | dashboard | 사이트 트래픽 통계 |
| Legacy | rogueap | 주변 AP 스캔 (채널 간섭 분석) |

---

### Log Export 탭

날짜 범위, AP, 유형, 키워드 조건에 맞는 로그를 파일로 저장합니다.

| 항목 | 설명 |
|------|------|
| 날짜 범위 | 분석 시작·종료 날짜 |
| AP | 특정 AP 또는 ALL |
| Type | ap_stuck / ap_no_service / rf_issue / dhcp_dns / qos / system_issue |
| Keyword | 추가 키워드 필터 (선택사항) |

**Export 실행** → 결과 하단 표시 + `runtime/exports/` 저장

---

### 동기화 탭

> `can_sync_nas` 권한 필요

NAS에 저장된 Logstash JSONL 로그를 PC 로컬 캐시로 동기화합니다 (차분 동기화).

1. **NAS Log 동기화** 클릭
2. 비밀번호 입력 (저장된 경우 생략)
3. 동기화 완료 → `runtime/logs/raw/` 저장

---

### 설정 탭

스크롤로 모든 설정에 접근 가능합니다.

#### UniFi API 설정
- Base URL / API Key 입력 및 keyring 저장
- **API 연결 테스트 (Probe)**: 연결 상태·스타일·Endpoint·응답시간 확인

#### AP 등록 수
현재 인벤토리 AP 수와 별도로 등록 수를 지정합니다 (Summary 카드 기준).

#### AP Remote 기본 정렬
GUI 시작 시 AP 테이블 초기 정렬 기준 선택 후 저장.

#### mca 덤프 관리
AP Remote 조회 시 저장된 mca-dump 파일을 날짜별로 관리.

---

### 사용자 관리 서브탭 (Admin 전용)

> `can_manage_users` 권한 필요

| 기능 | 설명 |
|------|------|
| 새 사용자 추가 | 계정 ID, 이름, 비밀번호, 역할, 권한 설정 |
| 편집 | 이름·역할·권한 변경, 비밀번호 초기화 |
| TOTP 초기화 | 인증 앱 분실/기기 변경 시 QR 재발급 |
| 삭제 | 계정 삭제 (본인 계정 삭제 불가) |
| 이 컴퓨터 개발자 등록 | 현재 PC 자동 로그인 등록 |

---

## 권한 구조 참조표

| 권한 플래그 | 설명 | viewer | operator | admin |
|-------------|------|:---:|:---:|:---:|
| `can_reset_ap` | AP 원격 재부팅 | ✗ | 관리자 설정 | ✓ |
| `can_sync_nas` | NAS 로그 동기화 | ✗ | ✓ | ✓ |
| `can_export` | Log Export 실행 | ✓ | ✓ | ✓ |
| `can_view_efg_tab` | EFG Remote 탭 접근 | ✓ | ✓ | ✓ |
| `can_manage_users` | 사용자 관리 탭 | ✗ | ✗ | ✓ |

---

## 자주 묻는 질문

**Q. VPN 연결했는데 "교회 내부 네트워크에 접속할 수 없습니다"가 나옵니다.**  
A. VPN이 완전히 연결된 뒤 GUI를 실행하세요. VPN 연결 직후 몇 초 대기 후 재시작하면 해결됩니다.

---

**Q. 6자리 코드가 올바르지 않습니다.**  
A. 코드는 30초마다 바뀝니다. 앱에서 새 코드로 바뀐 직후 입력하거나, PC 시간 동기화를 확인하세요.  
Windows: 설정 → 시간 및 언어 → 지금 동기화.

---

**Q. 스마트폰을 교체했습니다.**  
A. 관리자에게 TOTP 초기화를 요청하세요. 초기화 후 새 기기에서 QR 코드를 다시 스캔하면 됩니다.

---

**Q. Log 동기화 / AP Reset 버튼이 비활성화되어 있습니다.**  
A. 해당 권한이 없는 계정입니다. 관리자에게 권한 설정을 요청하세요.

---

**Q. AP Remote 조회 시 "Auth failed" 오류가 납니다.**  
A. AP SSH 비밀번호가 틀렸거나 변경되었습니다.  
설정 탭 → **AP SSH 비밀번호 삭제** → 조회 재시도 시 새 비밀번호 입력.

---

**Q. EFG API Probe는 성공했는데 데이터가 비어 있습니다.**  
A. **전체 Preset GET** 버튼을 실행하세요. Probe 후 자동으로 데이터를 가져오지 않습니다.

---

**Q. EFG API에서 특정 preset이 404가 납니다.**  
A. 이 컨트롤러(UniFi OS / Dream Machine Enterprise)에서 지원하지 않는 Legacy endpoint입니다.  
`stat/event`, `stat/device`, `rest/device`, `stat/sta` 등은 해당 컨트롤러에서 미지원입니다.

---

**Q. AP 정보/통계 preset에서 "deviceId 매핑이 없습니다"가 나옵니다.**  
A. 먼저 `[Official] devices` preset을 GET하여 AP 목록을 가져온 뒤 다시 시도하세요.  
전체 Preset GET 실행 후에는 자동으로 매핑됩니다.

---

**Q. 개발자 모드가 동작하지 않습니다.**  
A. 사용자 관리 서브탭 → **이 컴퓨터 개발자 등록**을 다시 실행하세요.  
Machine ID는 Windows MachineGuid 기반이므로 VPN 어댑터 변경의 영향을 받지 않습니다.

---

*문의: 네트워크 관리자 (ai@newvisionchurch.org)*
