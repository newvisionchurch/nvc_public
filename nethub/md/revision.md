# NVC NetHub — Revision History

최신 버전이 위에 표시됩니다.

---

## v0.9  (2026-06-14)

### 기능 개선

- 시작 순서 엔진 — NAS 의존 체인(NAS → Record → ELK Count)과 독립 실행(AP 조회, EFG SSH/API, 메시지 창)을 분리하여 v0.8 호환성 복원
- `elk_sync_first` 의미 보존 — `false`: ELK Count 즉시 독립 실행 / `true`: NAS 체인 완료 후 실행
- 8단계 인증 — 기존 7단계에서 4단계 "관리자 인증(접속 비밀번호 등록 여부 확인)" 추가

### 안정성 수정

- NAS 동기화 실패(`Log 동기화 중지됨`, `Log 동기화를 건너뜁니다`) 시에도 시작 체인 정상 중단
- NAS 비밀번호 미설정 조기 종료 시 모든 콜백 해제, 메시지 창 및 score_refresh pending 해제
- `elk_sync_first=false` 환경에서 ELK Count가 NAS 결과에 무관하게 독립 실행되도록 수정
- NAS 실패 시 메시지 창 `elk_count` pending과 score_refresh auto-off pending 모두 해제

### 스크립트 개선

- `scripts/run_mac_dist.sh` 추가 — macOS 배포 패키지 전용 실행 스크립트 (SCRIPT_DIR 기준)
- `scripts/run.sh` 개선 — `.venv` 우선 재사용, 버전·tkinter 재확인 후 신규 생성 분기
- `scripts/build.ps1` — 배포용 `run.sh`를 `run_mac_dist.sh`에서 읽어 생성 (하드코딩 제거)

---

## v0.8  (2026-06-09)

### 버그 수정

- NAS Log 동기화 — `auto_add_host_key` 기본값 `false` → `true` 변경 (`app_config.dist.json`) — known_hosts 미등록 PC에서 동기화 실패 문제 해결
- TOTP 초기화 — Git 미설치 PC에서 Windows 오류 팝업 발생 문제 수정 (`auth.py` — `shutil.which` 사전 확인)
- OTP 첫 등록 완료 시 credentials.json 자동 추가 누락 수정 (`main.py` — `_add_user_to_credentials` 호출 추가)
- `macsync.ps1` / `build.ps1` — Mac용 `run.sh` 생성 시 CRLF → LF 강제 적용

### UI 개선

- 사용자 관리 테이블 — `PAT 인증` 컬럼 제거 (관리자 화면에서 확인 불가 구조)
- 사용자 관리 테이블 — 모든 컬럼 중앙 정렬 적용
- 사용자 관리 테이블 — `2FA` 컬럼명 → `OTP`로 변경
- 사용자 관리 버튼 — `TOTP 초기화` → `OTP 초기화`로 변경
- 로그인 성공 시 `last_login` 자동 저장 — 사용자 관리 테이블에 마지막 로그인 시각 표시

---

## v0.7  (2026-06-05)

### macOS 지원

- 단일 인스턴스 잠금 — Windows(named mutex) / macOS·Linux(flock) 분기 처리
- `ctypes.windll` 직접 참조 제거 — macOS에서 크래시 없이 동작
- `requirements.txt` — `cryptography` 패키지 명시 추가 (Fernet 암호화 의존성)
- `app_config.json` — `ui_font.family` 기본값 제거, OS별 자동 감지 적용
- UI 텍스트 하드코딩 Windows 경로 제거 — 실제 workspace 경로 동적 참조
- `scripts/run.sh` 추가 — macOS/Linux용 실행 스크립트 (venv 자동 생성, tkinter 설치 안내, NAS known_hosts 자동 등록, 교회 내부망 설정 자동 적용)
- `docs/사용자_안내.md` — macOS 설치 및 실행 방법 추가
- `nvc_mac` 저장소 추가 — macOS 빌드 워크스페이스 (임시 검증 및 빌드용)
- `scripts/macsync.ps1` 추가 — Win 소스를 nvc_mac으로 동기화
- `scripts/build_mac.sh` 추가 — macOS PyInstaller 빌드 스크립트
- `build.ps1` — SSH로 Mac 자동 빌드, SCP로 결과물 수집, nvc_release 통합

### UI 개선

- AP Reset 선택 표시 — 선택된 행 텍스트 빨강 굵게 (배경색은 점수 분류 유지)
- AP Reset 선택 카운트 카드 — 0개일 때 흰색, 1개 이상 주황 배경으로 강조
- 메시지 창 하단 연결 상태바 추가 — AP SSH / NAS SSH / EFG SSH / EFG API / ELK 조회 성공/실패 칩 표시, 모두 성공 시 `✓ Setup Done` 표시
- ELK 그래프 팝업 — macOS 호환성 수정 (`-toolwindow` Windows 전용 분기)

### 설정 / 구조 개선

- MD 문서(매뉴얼 등) — 로컬 경로 의존성 제거, `nvc_public` GitHub raw URL로 직접 fetch
- 절대 경로 완전 제거 — `app_config.json`, `main.py` fallback 모두 상대/동적 경로로 변경
- NAS SSH 진단 — 연결 현황 탭에서 관리자 탭 → 보안 설정으로 이동
- 기본 작업 폴더명 변경 — `C:\NVC_Network_GUI` → `C:\NVC_Network_NetHub` (`app_config.dist.json` 반영)
- `app_config.json` 자동 복원 — 파일 삭제 시 `app_config.dist.json`에서 자동 재생성
- `ap_inventory.json` 삭제 내성 — 파일 없을 때 빈 목록으로 안전 시작
- 설정 자동화 탭 기본값 변경 — AP 전체 조회, ELK Stuck Count, 동기화 후 집계, EFG SSH/API 기본 ON (`app_config.dist.json` 반영)
- 기본값 관리 원칙 확립 — `app_config.dist.json`이 단일 기본값 원본, 빌드 시 `app_config.json`으로 복사
- 시작 메시지 창 NAS 인증 로그 표시 — 로그인 단계 NAS SSH 인증 결과(성공/실패)를 저장해 시작 메시지 창에 출력
- AP Status 온라인/오프라인 카드 수정 — SSH 조회 완료 시 reachable 기준으로 카드 즉시 업데이트
- ELK Stuck Count 날짜 범위 0=전체 지원 — 0 입력 시 전체 로그 집계, 기본값 0으로 변경
- NAS ELK Sync 날짜 범위 단축 버튼 추가 — 1주/2주/3주/4주 버튼으로 From/To 자동 설정, 기본 1주
- `app_config.json` Git 제외 — 로컬 경로 포함 개인 설정 파일 `.gitignore` 등록

---

## v0.6  (2026-06-04)

### 인증 및 보안
- 7단계 로그인 인증 UI 개선 — 단계별 파랑/초록/빨강 색상 표시
- 보안 설정 서브탭 복원 — 팝업 없이 관리자 탭 인라인 표시
- TOTP 등록 완료 시 credentials.json 자동 추가
- 로그인 시 NAS SSH 중복 방지 — 4단계 인증 성공 시 시작 동기화 skip
- 개발자 모드 스플래시 화면 추가 — 인증 화면 없이 2초 스플래시 후 메인 진입
- 팀 공유 비밀번호 통합 관리 — NAS/EFG/AP/EFG API Key credentials.json 암호화 저장

### 동기화 / 메시지
- 시작 메시지 창 좌우 분할 — 왼쪽(AP 조회) / 오른쪽(NAS·EFG·ELK) 색상 구분
- 동기화 탭 왼쪽 비율 설정 범위 확대 (3~35%), 실시간 반영 수정
- NAS SSH 진단 도구 — 연결 현황 탭에 좌우 분할 레이아웃

### ELK Stuck
- ELK Stuck Record 테이블 행 색상 — 값 크기별 회색/파랑/노랑/빨강
- ELK Stuck 그래프 팝업 — 날짜별 AP 추이 라인 그래프, AP 선택/해제, Y축 범위 지정, 호버 툴팁

### 기타
- 시작 시 빈 화면 제거 — 초기화 완료 후 창 표시, 로딩 중 메시지
- EFG SSH / API 진단 결과 메시지 창 연동
- 상단 변경 이력 버튼 추가 — docs/revision.md 팝업
- 문서 팝업 크기 확대 (1100×820)
- backup.ps1 개선 — 버전 자동 감지, nvc_nethub + nvc_security 동시 백업, 기존 버전 자동 삭제
- build.ps1 개선 — 기존 버전 자동 삭제 후 재빌드
- CLAUDE.md `all` 명령 워크플로 추가 — mdreport → mdupdate → push → sync → build → baseline → backup

---

## v0.5  (2026-05-31)

### 인증
- GitHub PAT 기반 팀원 인증 재설계
- admin 포함 모든 사용자 동일 인증 흐름 적용
- 세션 전용 credentials — 종료 시 메모리 데이터 삭제

### 관리자
- 사용자 탭 → 관리자 탭으로 재편, 서브탭 구조 도입
- credentials manager — NAS/EFG/AP/EFG API Key 통합 암호화 관리
- 로그인 시 Keychain 비밀번호 자동 복원

### ELK
- ELK Stuck Record 워크플로 추가
- EFG relay 기반 SSH 오류 메시지 개선

---

## v0.4  (2026-05-31)

- 사용자 이메일 필드 추가 — 모델, 추가/편집 다이얼로그, 목록 테이블

---

## v0.3  (2026-05-31)

- 배포 안내 문서 추가 — 빌드/배포/계정 등록/PAT 발급 절차

---

## v0.2  (2026-05-29)

- 안정성 및 코드 품질 개선
- GUI Version 0.2

---

## v0.1  (2026-05-29)

- 초기 릴리즈 기준선
