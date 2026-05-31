# 뉴비전 교회 GitHub Public

> NewVision Church — San Jose, CA  
> AI 협업 및 프로젝트 문서 공개 저장소

---

## 📁 nethub

**뉴비전 교회 네트워크 관리 시스템**

UniFi AP 29대 / 2개 건물(Main + Education) 환경에서 AP Stuck 문제를 분석하고  
원격으로 제어하기 위한 통합 관리 프로젝트입니다.

### 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| **ELK Stack** | NAS Docker 기반 Logstash V27 — AP/EFG syslog 수집·분류·저장 |
| **Python GUI** | Tkinter 로컬 관리 도구 V1 — AP Remote 스캔, 다중 Reset, EFG 대시보드 |
| **EFG** | 네트워크 코어 장비 참조 문서 |

### 폴더 구조

```
nethub/
├── md/          ← 프로젝트 전체 문서 (mdupdate로 자동 동기화)
├── log/         ← 보안 마스킹 로그 — Gemini 대용량 분석용 (임시, 분석 후 삭제)
├── code/        ← 코드 스니펫 — 디버깅·리뷰 공유용
├── issues/      ← 현재 조사 중인 이슈 컨텍스트 — ChatGPT·Claude 분석용
└── analysis/    ← AI 분석 결과 저장 — 세션 간 공유·재활용
```

### AI별 활용 방법

| AI | 주로 사용하는 폴더 | 역할 |
|----|-------------------|------|
| **Claude Code** | `md/`, `code/` | 코드 구현, 문서 작성, 직접 수정 |
| **ChatGPT Plus** | `md/`, `issues/`, `analysis/` | Root Cause 추론, 전략 수립, 음성 토론 |
| **Gemini** | `log/` | 대용량 로그 전체 분석 (1M 토큰) |
| **Perplexity** | `md/` | 최신 Known Issue 검색, 펌웨어 정보 |

### 문서 목록 및 Raw URL

👉 [nethub/README.md](nethub/README.md) — 전체 문서 목록 및 Raw URL

### 워크플로우

```
nvc_nethub (Private)
    │
    ├─ mdupdate          → md/ 자동 동기화
    ├─ 수동 copy         → log/, code/, issues/ 필요시 업로드
    └─ AI 분석 결과      → analysis/ 저장
```

---

*문의: ai@newvisionchurch.org*
