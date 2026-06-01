# NVC NetHub — 배포 안내

버전 0.4 · 2026-05-31
대상: 배포 담당자 (관리자)

---

## 배포 repo

**https://github.com/newvisionchurch-it/nvc_release** (Private)

팀원은 이 repo에서 최신 버전을 다운로드합니다.

---

## 권한 구조

팀원 1명당 **두 곳에 등록** — 두 곳 모두 해지하면 완전 차단됩니다.

```
팀원 GitHub 개인 계정
  ├── newvisionchurch-it 조직 멤버
  │     → nvc_release repo 접근 (배포 파일 다운로드)
  │
  └── newvisionchurch/nvc_security Collaborator (Read)
        → nethub/users.json 접근 (GUI 로그인)
```

| 상황 | 조치 | 효과 |
|------|------|------|
| 신규 팀원 등록 | -it 조직 초대 + nvc_security Collaborator 추가 | 배포 다운로드 + GUI 로그인 가능 |
| 배포만 차단 | -it 조직에서 제거 | 새 버전 다운로드 불가 |
| GUI 즉시 차단 | nvc_security Collaborator 제거 | GUI 로그인 즉시 불가 |
| 완전 차단 | 두 곳 모두 제거 | 완전 차단 |

---

## 1. 신규 팀원 등록 절차

### 1-1. GitHub 계정 확인
팀원에게 GitHub 개인 계정 username을 먼저 받습니다.
없으면 `https://github.com/signup` 에서 생성.

### 1-2. newvisionchurch-it 조직 초대
`https://github.com/orgs/newvisionchurch-it/people`
- **Invite member** → 팀원 username 입력 → 초대
- 팀원이 초대 수락 후 `nvc_release` 접근 가능

### 1-3. nvc_security Collaborator 추가
`https://github.com/newvisionchurch/nvc_security/settings/access`
- **Add people** → 팀원 username 입력 → **Read** 권한 → 추가
- 팀원이 본인 PAT로 `nethub/users.json` 읽기 가능

### 1-4. GUI 계정 생성
1. GUI 실행 → 관리자 로그인
2. 설정 → 사용자 → [새 사용자 추가]
3. 아이디 / 표시 이름 / 이메일 / 비밀번호 / 역할 설정
4. [저장] — GitHub nvc_security/nethub/users.json 에 자동 저장

### 1-5. PAT 발급 안내 (팀원 본인이 직접)
1. GitHub 로그인 → Settings → Developer settings
2. Personal access tokens → Tokens (classic) → Generate new token
3. 권한: `repo` / 만료: 1년 권장
4. 발급된 PAT를 GUI 최초 로그인 시 입력

> PAT는 최초 1회만 표시 — 즉시 안전한 곳에 보관

---

## 2. 역할 가이드

| 역할 | 대상 | 주요 권한 |
|------|------|----------|
| `admin` | 관리자 | 전체 기능 + 사용자 관리 |
| `manager` | 네트워크 담당자 | AP 조회·Reset, 동기화, EFG |
| `member` | 일반 팀원 | AP 조회, Log Export, EFG 조회 |
| `guest` | 방문자 | 조회만 가능 |

---

## 3. 관리자 등록 PC 설정

1. GUI 로그인 (GitHub 연결 상태)
2. 설정 → 사용자 → [이 컴퓨터 개발자 등록]
3. 타이틀 바에 `⚙ 개발자 모드` 배지 확인

> 등록된 PC에는 admin 비밀번호 해시가 로컬 저장 → 오프라인 로그인 가능

---

## 4. 빌드 방법

```powershell
# source/main.py 에서 GUI_VERSION 버전 올리기
# "Version 0.4  05/31/2026" → "Version 0.5  날짜"

cd C:\Projects\nvc_nethub
.\scripts\build.ps1
```

빌드 완료 후 `nvc_release/nethub/vX.X/` 에 자동 복사 + push.

---

## 5. 문제 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| 빌드 실패 (ImportError) | 패키지 미설치 | `.venv` 활성화 후 `pip install -r source\requirements.txt` |
| 로그인 실패 (모든 PC) | GitHub PAT 만료 또는 nvc_security 권한 없음 | PAT 재발급 또는 Collaborator 추가 확인 |
| OTP 오류 | 시간 동기화 문제 | 스마트폰 시간 자동 동기화 확인 |
| SSH Fail 전체 | VPN 미연결 | VPN 연결 후 재시도 |
| GitHub 연결 실패 | PAT 만료 | 설정 → 보안 → GitHub PAT 재입력 |

---

## 6. 팀원 배포 이메일 템플릿

> `[ ]` 항목은 실제 값으로 교체 후 카카오톡/문자로 전달하세요.
> **비밀번호·PAT는 이메일 금지 — 카카오톡/문자로만 전달**

---

제목: 뉴비전 네트워크 관리 GUI v0.4 배포 안내

안녕하세요,

네트워크 관리 GUI v0.4가 배포되었습니다.
아래 순서대로 설치해 주세요.

■ 사전 준비 (관리자가 처리)
- GitHub 계정 username을 알려주세요
- 두 곳에 등록해 드립니다 (배포 접근 + GUI 로그인)
- 아이디/비밀번호/QR코드를 별도 카카오톡으로 전달드립니다

■ 설치 및 실행 순서

1. OTP 앱 사전 설치 (스마트폰)
   · iOS    : App Store → "Google Authenticator" 또는 "Authy"
   · Android: Play Store → "Google Authenticator" 또는 "Authy"

2. GitHub PAT 발급 (본인 GitHub 계정에서 직접)
   · GitHub → Settings → Developer settings
   · Personal access tokens → Tokens (classic) → Generate new token
   · 권한: repo / 만료: 1년
   · 발급된 토큰 안전하게 보관 (최초 1회만 표시)

3. GitHub 로그인 후 아래 repo에서 zip 다운로드
   https://github.com/newvisionchurch-it/nvc_release
   · Windows          : nethub/v0.4/windows/NVC_NetHub_v0.4.zip
   · macOS / 개발환경 : nethub/v0.4/source/NVC_NetHub_v0.4_source.zip

4. 압축 해제 → 설치_안내.txt 참고

5. VPN 연결 후 실행
   · Windows: NVC_Network_GUI.exe 더블클릭
   · macOS / 개발환경: 터미널에서 ./run.sh

6. 로그인 화면에서
   · GitHub PAT 입력 (최초 1회 — 이후 자동)
   · 아이디: [아이디] (카카오톡으로 전달)
   · 비밀번호: [비밀번호] (카카오톡으로 전달)

7. OTP 등록 (최초 1회)
   · GUI 화면에 QR 코드 자동 표시
   · OTP 앱으로 스캔 → 앱에 "NVC Network GUI" 항목 생성
   · 앱의 6자리 코드 입력 → 로그인 완료
   · 이후 로그인 시마다 앱의 6자리 코드 사용

■ 사용자가 직접 만들어야 하는 것
- GitHub 개인 계정 (없으면 https://github.com/signup)
- GitHub PAT (본인 계정에서 발급 — 2번 참고)
- OTP 앱 설치 (스마트폰 — 1번 참고)

■ 사용자가 관리자에게 주어야 하는 것
- GitHub 개인 계정 username

■ 관리자로부터 받는 것 (별도 카카오톡)
- GUI 아이디
- GUI 초기 비밀번호
- OTP QR 코드 이미지

■ 로그인 전체 절차 요약 (최초 1회)

  [사전 준비]
  1. OTP 앱 설치 — Google Authenticator 또는 Authy
  2. GitHub 계정 → username 관리자에게 전달
  3. GitHub PAT 발급 (repo 권한, 1년) → 안전하게 보관
  4. 관리자로부터 아이디/비밀번호/QR코드 수령

  [설치]
  5. nvc_release repo → zip 다운로드 → 압축 해제

  [실행 및 로그인]
  6. VPN 연결
  7. NVC_Network_GUI.exe 실행 (macOS: ./run.sh)
  8. GitHub PAT 입력 (최초 1회, 이후 자동)
  9. 아이디 / 비밀번호 입력
  10. QR 코드 화면 → OTP 앱으로 스캔
      → 앱에 "NVC Network GUI" 항목 생성
  11. 앱의 6자리 코드 입력 → 로그인 완료 ✓

  [이후 로그인]
  VPN 연결 → 실행 → 아이디 / 비밀번호 / OTP 6자리

■ 주의사항
- 반드시 VPN 연결 후 실행하세요
- PAT·비밀번호는 안전하게 보관하세요

문의: ai@newvisionchurch.org

---

*NVC NetHub v0.4 — 관리자 배포 안내*
