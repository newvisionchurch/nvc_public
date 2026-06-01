# 뉴비전 교회 GitHub Public

> NewVision Church — San Jose, CA  
> AI 협업 및 프로젝트 문서 공개 저장소

---

## 📁 nethub

**뉴비전 교회 네트워크 관리 시스템 (NVC Network Hub)**

UniFi AP 29대 / 2개 건물(Main + Education) 환경에서 AP Stuck 문제를 분석하고  
원격으로 제어하기 위한 통합 관리 프로젝트입니다.

### 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| **nethub SW** | Python/Tkinter 로컬 관리 GUI v0.3 — AP 스캔·Reset, EFG 대시보드, Log Export |
| **ELK Stack** | NAS Docker 기반 Logstash V27 — AP/EFG syslog 수집·분류·저장·RCA 분석 |
| **EFG** | UniFi Dream Machine Enterprise — 네트워크 코어 게이트웨이 |

### 폴더 구조

```
nethub/
└── md/          ← 프로젝트 전체 문서 (scripts/sync.ps1로 자동 동기화)
    ├── CLAUDE.md    ← AI 작업 지침
    ├── README.md    ← 프로젝트 소개
    ├── nethub.md    ← nethub SW 설계 및 모듈 구조
    ├── elk.md       ← ELK V27 베이스라인, RCA 모델, 운영 가이드
    ├── efg.md       ← EFG 장비 기준 정보
    ├── manual.md    ← 사용자 매뉴얼 및 배포 안내
    └── aiguide.md   ← AI 협업 구조 안내
```

### AI별 활용 방법

| AI | 주로 사용하는 파일 | 역할 |
|----|-------------------|------|
| **Claude Code** | `md/` 전체 | 코드 구현, 문서 작성, 직접 수정 |
| **ChatGPT Plus** | `md/elk.md`, `md/nethub.md` | Root Cause 추론, 전략 수립, 음성 토론 |
| **Gemini** | `md/elk.md` | 대용량 로그 전체 분석 (1M 토큰) |
| **Perplexity** | `md/` | 최신 Known Issue 검색, 펌웨어 정보 |

### 문서 Raw URL

👉 [nethub/md/](nethub/md/) — 전체 문서 목록

### 동기화 워크플로우

```
nvc_nethub (Private)
    │
    └─ scripts/sync.ps1 실행 → nethub/md/ 자동 동기화
```

---

*문의: ai@newvisionchurch.org*
