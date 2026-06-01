# AI 협업 구조 가이드

뉴비전 교회 NVC Network 프로젝트의 GitHub 저장소 구조와 AI 협업 방식.

---

## GitHub 계정 / 저장소 구조

| 저장소 | 접근 | 용도 |
|--------|------|------|
| `newvisionchurch/nvc_nethub` | Private | 메인 코드 (SW, ELK, 문서) |
| `newvisionchurch/nvc-auth` | Private | 인증 데이터 (users.json, ap_reset_schedule.json, feedback.json) |
| `newvisionchurch-it/nvc_release` | Private | 배포 패키지 (exe/source zip) |
| `newvisionchurch/nvc_public` | Public | MD 문서 공개 사본 (nvcpush로 동기화) |
| `newvisionchurch/nvc_security` | Private | 보안 감사 데이터 |

커밋 작성자: Chris Jeong `<chris.jeong7@gmail.com>`

---

## Claude Code 역할

Claude Code는 이 프로젝트의 **코딩/구현/문서 전담** AI이다.

**작업 범위:**
- `source/` Python/Tkinter GUI 코드 작성 및 수정
- `elk/logstash/pipeline/logstash.conf` Logstash 파이프라인 수정 (small delta)
- `docs/` AI context 문서 업데이트
- `source/config/ap_inventory.json` AP 인벤토리 관리
- `scripts/` PowerShell 스크립트 작성
- git commit / push (4개 repo)

**Claude Code가 작업할 때 자동 로드되는 컨텍스트:**
1. `CLAUDE.md` — 항상 자동 로드 (프로젝트 지침)
2. `docs/*.md` — settings.json `importedFiles`로 자동 로드

---

## 4개 저장소 역할 분리

### nvc_nethub (Private) — 메인 개발 저장소

```
source/       ← Python GUI 소스
elk/          ← ELK 설정 (compose.yaml, logstash.conf)
docs/         ← AI context 문서
scripts/      ← PowerShell 스크립트
CLAUDE.md     ← Claude Code 지침
README.md     ← GitHub 소개
```

git 제외 파일:
- `source/config/users.json` (bcrypt 해시, TOTP secret)
- `source/config/ssh_targets.json` (SSH 접속 정보)
- `runtime/` (로컬 캐시, 작업 파일)

### nvc-auth (Private) — 팀 공유 인증/데이터 저장소

GUI가 GitHub API (HTTPS + PAT)로 직접 읽고 쓰는 파일:

| 파일 | 내용 | 접근 방식 |
|------|------|---------|
| `users.json` | 사용자 계정 (bcrypt 해시, TOTP secret) | GUI 인증 원본 |
| `ap_reset_schedule.json` | AP Reset 예약 및 이력 | GUI github_schedule.py |
| `feedback.json` | 팀원 게시판 | GUI github_feedback.py |

PAT 권한: `repo` (nvc-auth read/write). Windows Keychain 보관.

### nvc_release (Private) — 배포 저장소

```
releases/
  v0.3/
    win/    nvc_nethub_GUI_v0.3.zip      (PyInstaller exe)
    source/ nvc_nethub_GUI_v0.3_source.zip (소스 패키지)
```

팀원은 이 repo에서 zip을 다운로드하여 설치한다.  
접근: `newvisionchurch-it` 조직 멤버만.

### nvc_public (Public) — 공개 문서 저장소

`scripts/sync.ps1` 실행 시 `nvc_nethub`의 모든 `.md` 파일 → `nvc_public/nethub/md/`로 복사.  
GUI 타이틀 바 [GUI 매뉴얼] / [GUI 개요] / [프로젝트 개요] 버튼이 이 경로의 파일을 직접 읽는다.

```
app_config.json → "md_base_path": "C:\\Projects\\nvc_public\\nvc_network\\md"
```

---

## AI context 문서 자동 로드 구조

```
CLAUDE.md (항상 자동 로드)
  ├── docs/elk.md     (ELK 작업 시 참조)
  ├── docs/nethub.md  (SW 작업 시 참조)
  ├── docs/efg.md     (EFG 장비 참조)
  ├── docs/manual.md  (사용자/배포 매뉴얼)
  └── docs/aiguide.md (이 파일 — AI 협업 구조)
```

자동 로드: `.claude/settings.json` → `importedFiles` 설정.

---

## 보안 원칙

| 항목 | Private (nvc_nethub) | Public (nvc_public) |
|------|----------------------|---------------------|
| 실제 코드 | ✅ | ❌ |
| IP/MAC 정보 | ✅ | ❌ |
| 원본 로그 | ❌ (runtime/ git 제외) | ❌ |
| bcrypt 해시 | ❌ (nvc-auth로 이동) | ❌ |
| MD 문서 | ✅ 원본 | ✅ 사본 |
| 비밀번호/API 키 | 절대 커밋 금지 | 절대 금지 |

git 제외 파일 체크리스트:
- `source/config/users.json` — bcrypt 해시, TOTP secret
- `source/config/ssh_targets.json` — NAS/EFG/AP SSH 접속 정보
- `runtime/` — 로컬 캐시, 작업 파일

---

## Push 워크플로우

```
Claude Code 작업 완료
  ↓
git commit (nvc_nethub)
  ↓
"push" 요청
  ↓ Claude Code가 4개 repo 순서대로 처리
  nvc_nethub  → origin/main
  nvc_public  → origin/main (변경 있을 때)
  nvc_security → origin/main (변경 있을 때)
  nvc_release → origin/main (변경 있을 때)

문서 변경 후 nvc_public 동기화가 필요하면:
  .\scripts\sync.ps1
```
