# NVC NetHub — 배포 안내

이 문서는 배포 담당자가 팀원에게 NVC NetHub를 배포하고 접근 권한을 관리하는 절차를 정리합니다.
배포 버전은 Git tag와 `nvc_release` 폴더명을 기준으로 확인합니다.

## 배포 저장소

| 항목 | 값 |
|------|-----|
| 저장소 | `https://github.com/newvisionchurch-it/nvc_release` |
| 접근 | Private |
| 용도 | Windows 패키지와 Source 패키지 배포 |

## 권한 구조

팀원은 두 곳에 등록되어야 합니다.

```text
팀원 GitHub 개인 계정
  ├── newvisionchurch-it 조직 멤버
  │   └── nvc_release 다운로드 권한
  └── newvisionchurch/nvc_security Collaborator
      └── NVC NetHub 로그인 데이터 접근 권한
```

| 상황 | 조치 | 효과 |
|------|------|------|
| 신규 팀원 등록 | `newvisionchurch-it` 초대 + `nvc_security` Collaborator 추가 | 배포 다운로드와 로그인 가능 |
| 배포만 차단 | `newvisionchurch-it`에서 제거 | 새 패키지 다운로드 불가 |
| 로그인 차단 | `nvc_security` Collaborator 제거 | 로그인 데이터 접근 불가 |
| 완전 차단 | 두 곳 모두 제거 | 배포와 로그인 모두 차단 |

## 신규 팀원 등록

### 1. GitHub 계정 확인

팀원에게 GitHub username을 받습니다.
계정이 없으면 `https://github.com/signup`에서 생성하도록 안내합니다.

### 2. 배포 저장소 접근 추가

`https://github.com/orgs/newvisionchurch-it/people`

1. `Invite member` 선택
2. 팀원 username 입력
3. 초대 수락 확인

### 3. 로그인 데이터 접근 추가

`https://github.com/newvisionchurch/nvc_security/settings/access`

1. `Add people` 선택
2. 팀원 username 입력
3. 필요한 권한으로 추가

### 4. NVC NetHub 계정 생성

1. NVC NetHub 실행
2. 관리자 로그인
3. 설정 → 사용자 → 새 사용자 추가
4. ID, 표시 이름, 이메일, 초기 비밀번호, 역할 설정
5. 저장

사용자 데이터는 `newvisionchurch/nvc_security:nethub/users.json`에 저장됩니다.

### 5. PAT 발급 안내

팀원이 본인 GitHub 계정에서 직접 발급합니다.

1. GitHub → Settings → Developer settings
2. Personal access tokens → Tokens classic
3. `repo` 권한 선택
4. 만료 기간 설정
5. 발급된 토큰을 안전하게 보관

PAT는 최초 1회만 표시됩니다.

## 역할 가이드

| 역할 | 대상 | 주요 권한 |
|------|------|----------|
| `admin` | 관리자 | 전체 기능, 사용자 관리 |
| `manager` | 네트워크 담당자 | AP 조회, Reset, 동기화, EFG 조회 |
| `member` | 일반 팀원 | AP 조회, Log Export, EFG 조회 |
| `guest` | 방문자 | 제한된 조회 |

## 관리자 등록 PC

관리자 등록 PC는 오프라인 상황에서도 관리자 로그인을 지원하기 위한 로컬 신뢰 장치입니다.

1. NVC NetHub 로그인
2. 설정 → 사용자
3. `이 컴퓨터 개발자 등록` 실행
4. 상단에 개발자 모드 배지 표시 확인

등록된 PC에는 관리자 계정의 로컬 인증 정보가 보존될 수 있으므로 신뢰할 수 있는 장비에서만 사용합니다.

## 빌드

빌드 전에 실제 배포 버전은 `source/main.py`의 버전 상수와 Git tag를 확인합니다.

```powershell
cd C:\Projects\nvc_nethub
.\scripts\build.ps1
```

빌드 결과는 `nvc_release/nethub/[버전]/` 아래에 보관합니다.
`scripts/build.ps1`은 exe/source 패키지 생성, `nvc_release` 복사, 관련 저장소 push까지 수행하는 배포용 스크립트입니다.
일반 Markdown 최신화 작업은 채팅의 `mdupdate`, `push all`, `sync` 순서로 처리하며 build를 실행하지 않습니다.
릴리즈까지 진행할 때는 채팅에서 `mdupdate push all sync build`처럼 한 번에 요청할 수 있으며, AI는 문서 갱신, 커밋/push, 공개 문서 동기화, 배포 빌드 순서로 처리합니다.
`mdupdate` 요청을 받으면 AI가 먼저 `scripts/mdreport.ps1`을 실행해 최신 리포트를 만든 뒤 문서를 수정합니다.

## 문서 최신화

배포 전 코드 변경이 문서에 반영되었는지 확인하려면 AI 작업 창에서 `mdupdate`를 요청합니다.
AI는 `scripts/mdreport.ps1`을 실행한 뒤 `runtime/reports/mdreport_change_report.md`와 실제 코드를 대조해 `README.md`, `CLAUDE.md`, `docs/*.md`를 업데이트합니다.
문서 검토와 push가 끝난 뒤 public 문서 동기화가 필요하면 다음을 실행합니다.

```powershell
.\scripts\sync.ps1
```

동기화까지 완료된 뒤 다음 MD 기준점을 저장하려면 다음을 실행합니다.

```powershell
.\scripts\mdreport.ps1 -MarkBaseline
```

## 문제 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| 빌드 실패 | 패키지 미설치 | `.venv` 활성화 후 `pip install -r source\requirements.txt` |
| 로그인 실패 | PAT 만료 또는 `nvc_security` 권한 없음 | PAT 재발급 또는 Collaborator 확인 |
| OTP 오류 | 스마트폰 시간 오차 | 시간 자동 동기화 확인 |
| SSH Fail 전체 | VPN 미연결 | VPN 연결 후 재시도 |
| GitHub 연결 실패 | PAT 만료 또는 네트워크 문제 | PAT 재등록 |

## 배포 안내 템플릿

아래 템플릿의 `[항목]`은 배포 시 실제 값으로 교체합니다.
비밀번호와 PAT는 이메일에 쓰지 말고 별도 안전 채널로 전달합니다.

```text
제목: NVC NetHub [버전] 배포 안내

안녕하세요.

NVC NetHub [버전]이 배포되었습니다.
아래 순서대로 설치해 주세요.

사전 준비
- GitHub 개인 계정 username을 관리자에게 전달
- OTP 앱 설치: Google Authenticator 또는 Authy
- GitHub PAT 발급: repo 권한

설치
1. GitHub 로그인
2. nvc_release 저장소 접속
3. [배포 경로]에서 패키지 다운로드
4. 압축 해제
5. VPN 연결
6. NVC NetHub 실행

최초 로그인
1. GitHub PAT 입력
2. 관리자에게 받은 ID / 초기 비밀번호 입력
3. OTP QR 코드 등록
4. OTP 6자리 코드 입력

이후 로그인
- VPN 연결
- NVC NetHub 실행
- ID / 비밀번호 / OTP 입력

주의
- PAT와 비밀번호는 안전하게 보관하세요.
- VPN 연결 후 실행하세요.
```
