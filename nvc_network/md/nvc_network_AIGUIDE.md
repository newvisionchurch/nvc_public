# NewVision Church Network - GitHub & AI 협업 완전 가이드
> San Jose, CA | UniFi 29 AP | 2개 건물 | ELK Logstash + Python GUI 네트워크 관리 시스템

---

## 1. GitHub 전체 구성

### 1-1. Repository 구조

```
GitHub Account: chrisjeong7-crypto
│
├── 📁 nvc_network (Private) ← 메인 Repo
│   ├── 실제 코드 (Python GUI, Logstash 등)
│   ├── ELK 설정 파일
│   ├── AP 인벤토리 (29대, 192.168.11.x)
│   └── MD 파일들
│       ├── README.md          ← 프로젝트 개요 및 현황
│       ├── CLAUDE.md          ← Claude Code 작업 지침
│       ├── AIGUIDE.md         ← AI 협업 가이드 (본 파일)
│       ├── elk/
│       │   ├── README.md      ← ELK V27 베이스라인, RCA 모델
│       │   ├── MEMO.md        ← NAS 운영 명령어
│       │   └── docs/          ← ELK 세부 문서
│       ├── gui/
│       │   ├── README.md      ← 기술 퀵스타트
│       │   ├── MANUAL.md      ← 사용자 매뉴얼
│       │   └── DESIGN.md      ← 설계 구상서 및 구현 이력
│       └── efg/
│           └── EFG.md         ← EFG 참조 (경로 변경 금지)
│
└── 📁 nvc_public (Public) ← AI 컨텍스트 공유용
    ├── chatgpt_master.md  ← AI 공유 컨텍스트 (항상 최신)
    └── logs/              ← 마스킹된 로그 (분석후 삭제)
        ├── log_part01.txt (임시)
        └── ...
```

### 1-2. Private vs Public 원칙

| 항목 | Private (nvc_network) | Public (nvc_public) |
|------|----------------------|---------------------|
| 실제 코드 | ✅ 보관 | ❌ 금지 |
| ELK/UniFi 설정 | ✅ 보관 | ❌ 금지 |
| 원본 로그 | ✅ 보관 | ❌ 금지 |
| IP/MAC 정보 | ✅ 보관 | ❌ 금지 |
| 요약 MD | ✅ 원본 | ✅ 요약본만 |
| 마스킹 로그 | - | ✅ 임시 (분석후 삭제) |
| chatgpt_master.md | - | ✅ 항상 유지 |

### 1-3. Raw URL 형식

```
# 일반 URL (GitHub 웹페이지)
https://github.com/AINVC/nvc_public/blob/main/chatgpt_master.md

# Raw URL (AI가 직접 읽는 텍스트)
https://raw.githubusercontent.com/AINVC/nvc_public/main/chatgpt_master.md

# 변환 규칙
github.com → raw.githubusercontent.com
/blob/ → 제거 (슬래시만 남김)
```

---

## 2. VS Code 작업 환경 구성

### 2-1. Multi-root Workspace 설정

```
VS Code
├── 📁 nvc_network/     → Private GitHub 연결
│   └── .git/
│
└── 📁 nvc_public/      → Public GitHub 연결
    └── .git/
```

### 2-2. Public Repo 초기 설정 (터미널)

```bash
# VS Code 터미널에서 실행
mkdir nvc_public
cd nvc_public
git init
git remote add origin https://github.com/AINVC/nvc_public.git
```

### 2-3. VS Code Multi-root 추가

```
File → Add Folder to Workspace → nvc_public 폴더 선택
→ 두 Repo 동시 관리 가능
→ 각각 독립적으로 Git 동작
```

---

## 3. MD 파일 동기화 — mdcopy.ps1

### 3-1. mdcopy — Google Drive 자동 동기화

`nvc_network`의 모든 MD 파일을 Google Drive로 복사하는 스크립트가 구현되어 있습니다.

```powershell
# 터미널 어디서나 실행 가능
mdcopy
```

**복사 위치:** `C:\Users\admin\Google Drive\IT\NVC_NETWORK\`

**명명 규칙:**
| 원본 경로 | 복사 파일명 |
|---|---|
| `README.md` (루트) | `nvc_network_README.md` |
| `CLAUDE.md` (루트) | `nvc_network_CLAUDE.md` |
| `elk/README.md` | `elk_README.md` |
| `gui/MANUAL.md` | `gui_MANUAL.md` |
| `gui/DESIGN.md` | `gui_DESIGN.md` |
| `elk/docs/index.md` | `elk_docs/index.md` |

**스크립트 위치:** `C:\Projects\nvc_network\mdcopy.ps1`  
PowerShell 프로필에 `mdcopy` 함수 등록되어 있어 언제든 바로 실행 가능.

### 3-2. chatgpt_master.md 업데이트 방법

```
1. mdcopy 실행 → Google Drive MD 최신화
2. chatgpt_master.md 내용 수동 업데이트
3. nvc_public Repo에 push
```

### 3-3. log_prepare — 로그 분석용 준비

```
로그 분석이 필요할 때 Claude Code에게 요청:
"로그 파일을 보안 마스킹 + 5MB 분할해서 Public Repo에 올려줘"

처리:
1. IP/MAC/도메인/비밀번호 자동 마스킹
2. 5MB 단위 분할 → log_masked_part01.txt, part02.txt ...
3. Public Repo logs/ 폴더에 업로드
4. Raw URL 목록 출력
5. 분석 완료 후 삭제 명령어 출력
```

---

## 4. chatgpt_master.md 구조 및 템플릿

```markdown
# NewVision Network - AI Context
# Updated: [TIMESTAMP] | Version: [자동증가]

---

## 🔴 현재 이슈 (즉시 확인)
- AP Stuck 지속 모니터링 중 — GUI로 주기적 확인 운영
- [추가 이슈]

## ✅ 최근 완료
- GUI V1 완성 (2026-05)
  - bcrypt + TOTP 2FA 로그인
  - AP 29대 Remote SSH 스캔 (mca-dump + logread)
  - 다중 AP Reset (빨강 표시 → 일괄 실행)
  - EFG 시스템 대시보드
  - NAS 로그 동기화
  - 기본 정렬 설정, 개발자 모드, 최대화 시작
- AP 29대 전체 192.168.11.x 통합 완료
- ELK V27 안정 운영중

---

## 📋 파트별 현황 요약

### 1. ELK Logstash Pipeline
상태: ✅ V27 운영중
- UDP 51415 수신, 인덱스 unifi-network-v27-{ap,low,noise}-*
- JSONL 출력: NAS logstash/outputs/{ap,low,noise}/YYYY/M/D/
- RCA 모델: system_issue → qos_error → ap_stuck / ap_no_service
상세보기: [elk/README.md — Private]

### 2. Python GUI 관리 도구
상태: ✅ V1 완성
- 탭: Log 동기화 / AP ELK Log / AP Remote / EFG Remote / Log Export / 설정 / 사용자 관리
- AP Remote: 29대 동시 SSH 스캔, mca-dump 진단 컬럼, 다중 Reset
- 인증: bcrypt + TOTP 2FA, 역할 기반 권한 (admin/operator/viewer)
- 개발자 모드: Windows MachineGuid 기반 자동 로그인
상세보기: [gui/README.md, gui/MANUAL.md — Private]

### 3. UniFi AP
상태: ✅ 정상 운영
- AP 29대 / 2개 건물 (Main + Education)
- 전체 192.168.11.x 통합 완료
- EFG relay → sshpass → AP(Dropbear) SSH 구조
상세보기: [gui/config/ap_inventory.json — Private]

### 4. EFG
상태: ✅ 운영중
- GUI EFG Remote 탭에서 시스템/네트워크/트래픽/ARP 실시간 조회
상세보기: [efg/EFG.md — Private]

---

## ⏭️ 다음 단계 (V2 후보)
1. EFG system_issue 자동 감지 및 알림
2. AP 상태 실시간 모니터링
3. AP 리셋 후 복구 여부 자동 확인
4. Google OAuth 로그인 (Workspace 이메일 기반)
5. Daily Report 자동 생성

## 🖥️ 환경 정보
- UniFi AP: 29대 / 2개 건물 / 전체 192.168.11.x
- ELK: 7.17.10, NAS Docker Compose, Logstash V27
- GUI: Python 3.13 + Tkinter, Windows 11
- VPN: 교회 내부망 (192.168.11.1:22 체크)
```

---

## 5. AI별 역할 및 사용법

### 5-1. Claude Code (유료 $20/월) - 코딩/구현 전담

**역할:**
```
✅ Python GUI 코딩 및 구현 (V1 완성, V2 확장)
✅ Logstash 설정 작성/수정
✅ mdcopy.ps1 — MD 파일 Google Drive 동기화
✅ AP 인벤토리 관리 (ap_inventory.json)
✅ MD 문서 업데이트 (README, MANUAL, DESIGN)
✅ GitHub push 자동화
✅ 모든 코드 설계 및 디버깅
```

**사용 방법:**
```
1. VS Code에서 Claude Code 실행
2. CLAUDE.md 컨텍스트 자동 참조
3. 작업 완료시 mdcopy 실행
   → 모든 MD 파일 → Google Drive 동기화
4. 필요 시 chatgpt_master.md 업데이트 후 nvc_public push
```

**Claude Code 대화 시작 템플릿:**
```
CLAUDE.md 읽고 현재 상태 파악 후
아래 작업 진행해줘:
[작업 내용]
```

---

### 5-2. ChatGPT Plus (유료 $20/월) - 분석/추론/음성 전담

**역할:**
```
✅ 음성 대화로 문제 상황 설명 및 토론
✅ Root Cause 추론 및 방향 설정
✅ Gemini/Perplexity 분석 결과 종합
✅ 해결 방향 결정 및 Claude에 전달
✅ 프로젝트 전략 수립
```

**Project Instructions 고정 설정:**
```
매 대화 시작시 반드시 아래 URL을 먼저 읽고
현재 프로젝트 상태를 완전히 파악한 후 답변할 것:
https://raw.githubusercontent.com/chrisjeong7-crypto/nvc_public/main/chatgpt_master.md
```

**대화 시작 템플릿:**
```
[위 URL] 읽고 현재 상태 파악 후 답변해줘.

오늘 상황: [문제 설명]
Gemini 분석 결과: [붙여넣기]
Perplexity 검색 결과: [붙여넣기]

Root Cause 추론해줘.
```

**음성 대화 활용:**
```
ChatGPT 앱 → 음성 모드
"오늘 AP 3번이 이런 증상인데 원인이 뭘까?"
→ 자연스러운 한국어 음성 토론
→ 결론 나오면 Claude Code에 전달
```

---

### 5-3. Gemini (무료) - 대용량 로그 분석 전담

**역할:**
```
✅ 대용량 로그 파일 전체 분석 (1M 토큰)
✅ AP 동작 패턴 파악
✅ 시간대별 장애 상관관계 분석
✅ 간헐적 단절 규칙성 발견
✅ 여러 AP 동시 비교 분석
```

**로그 분석 워크플로우:**
```
1. Claude Code에게 로그 준비 요청
   → 보안 마스킹 + 5MB 분할
   → Public Repo 자동 업로드
   → Raw URL 목록 출력

2. Gemini에 순차 전달
   Part01 → "이 로그 분석해줘"
   Part02 → "이어서 분석해줘"
   ...
   마지막 → "전체 종합 결론내줘"

3. 분석 완료 후
   → Public Repo logs/ 폴더 삭제
   → 결과만 ChatGPT에 전달
```

**Gemini 분석 요청 템플릿:**
```
아래 URL의 로그를 분석해줘:
[log_part01 Raw URL]

분석 요청:
1. UniFi AP Stuck / 간헐적 단절 패턴
2. 에러 발생 시간대
3. 특정 AP 집중 발생 여부
4. 비정상 패턴 요약
5. 의심되는 원인
```

**파일 크기 기준:**
```
로그 크기    → 분할 개수
10MB 이하   → 2개 (5MB × 2)
50MB        → 10개 (5MB × 10)
100MB       → 20개 (5MB × 20)
무제한 확장 가능 ✅
```

---

### 5-4. Perplexity (무료) - 최신 정보 검색 전담

**역할:**
```
✅ UniFi 최신 버그/Known Issue 검색
✅ ELK Logstash 최신 문제 검색
✅ 펌웨어 업데이트 정보
✅ 유사 사례 및 해결 방법 검색
✅ 기술 문서 최신 동향
```

**검색 요청 템플릿:**
```
아래 URL을 읽고 이 환경 기반으로 검색해줘:
https://raw.githubusercontent.com/chrisjeong7-crypto/nvc_public/main/chatgpt_master.md

검색 요청:
1. UniFi [현재버전] 최신 Known Issue / 버그
2. ELK Logstash AP 로그 관련 최근 문제
3. UniFi AP Stuck / 간헐적 단절 최근 사례 및 해결법
4. 관련 펌웨어 업데이트 권장사항
5. 유사 환경 문제 해결 사례
```

---

## 6. 전체 문제 해결 워크플로우

### 일반적인 문제 발생시

```
문제 발생
    ↓
① GUI AP Remote 탭
   전체 조회 실행 → 문제 AP 식별
   (ELK Stuck, 로그오류, 재시작 횟수 확인)
    ↓
② Claude Code
   mdcopy 실행 → MD 파일 최신화
   chatgpt_master.md 업데이트 → nvc_public push
    ↓
③ Perplexity (무료)
   chatgpt_master.md Raw URL 제공
   → 관련 Known Issue 검색
    ↓
④ 로그 준비 (필요시)
   Claude Code에게 요청 → 마스킹 + 분할 + Public 업로드
    ↓
⑤ Gemini (무료)
   로그 Raw URL 순차 제공
   → 대용량 로그 전체 분석
    ↓
⑥ ChatGPT Plus
   Perplexity 결과 + Gemini 분석 결과 제공
   → Root Cause 추론
   → 해결 방향 결정
    ↓
⑦ Claude Code
   ChatGPT 결론 전달
   → 코드/설정 수정 구현
   → AP Reset 필요시 GUI에서 직접 실행
    ↓
⑧ Public Repo logs/ 삭제
   → 보안 유지
```

---

## 7. GitHub 용량 관리

```
nvc_public Repo 용량 원칙:

항상 유지 (소용량):
└── chatgpt_master.md  ← 200줄 이하

임시 사용 (분석후 즉시 삭제):
└── logs/
    ├── log_part01.txt  → Gemini 분석후 삭제
    ├── log_part02.txt  → Gemini 분석후 삭제
    └── ...

GitHub 무료 제한:
→ Repo 전체 1GB 권장
→ 파일 하나 100MB 제한
→ 임시 삭제 운영으로 용량 걱정 없음 ✅
```

---

## 8. 보안 체크리스트

```
Public Repo 업로드 전 반드시 확인:

□ IP 주소 마스킹 (192.168.x.x)
□ MAC 주소 마스킹 (xx:xx:xx:xx:xx:xx)
□ 비밀번호/API 키 제거 ([REDACTED])
□ 내부 도메인명 제거 ([DOMAIN])
□ 디바이스명 마스킹 ([DEVICE])
□ SSID 정보 확인
□ 교회 내부 정보 미포함 확인

→ Claude Code 보안 스캔 스크립트로 자동화
```

nvc_network Private Repo 보안:
```
git 제외 파일 (.gitignore):
□ gui/config/users.json    ← bcrypt 해시, TOTP secret
□ gui/config/ssh_targets.json  ← NAS/EFG/AP 접속 정보
□ runtime/               ← 로컬 캐시, 작업 파일
```

---

## 9. 비용 및 도구 요약

| AI/도구 | 비용 | 주요 역할 |
|---------|------|----------|
| Claude Code | $20/월 | 코딩, 구현, 자동화, MD 관리 |
| ChatGPT Plus | $20/월 | 분석, 음성, Root Cause 추론 |
| Gemini | 무료 | 대용량 로그 분석 |
| Perplexity | 무료 | 최신 정보 검색 |
| GitHub | 무료 | 코드/컨텍스트 공유 |
| Google Drive | 무료 | MD 파일 백업 (mdcopy.ps1) |
| **합계** | **$40/월** | **완전한 네트워크 관리** |

---

## 10. 즉시 시작 순서

```
신규 설치 시:

Step 1: GitHub에서 nvc_public (Public) Repo 생성

Step 2: VS Code에 nvc_public 폴더 추가
        (Multi-root Workspace)

Step 3: chatgpt_master.md 초기 생성
        (위 섹션 4 템플릿 참고)

Step 4: ChatGPT Project Instructions에
        chatgpt_master.md Raw URL 고정 설정

Step 5: mdcopy 실행 → Google Drive MD 동기화 확인

Step 6: GUI 실행 → VPN 연결 → 로그인 확인
```

```
일상 운영 시:

① GUI 실행 → AP Remote 탭 → 전체 조회
② 문제 AP 확인 → 필요시 AP Reset 실행
③ 중요 변경 후 → mdcopy 실행 + git push
```
