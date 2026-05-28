# AINVC — Public AI Context Hub

AI가 프로젝트 컨텍스트를 직접 읽을 수 있도록 공개된 문서 저장소입니다.

---

## 📁 nvc_network — 뉴비전 교회 네트워크 관리 프로젝트

> UniFi 29 AP / 2개 건물 / ELK Logstash V27 + Python GUI  
> San Jose, CA

### 프로젝트 개요

| 문서 | 설명 | Raw URL |
|------|------|---------|
| **README** | 프로젝트 전체 개요, 구성, 빠른 시작 | [보기](https://raw.githubusercontent.com/AINVC/nvc_public/main/nvc_network/md/nvc_network_README.md) |
| **AIGUIDE** | AI 협업 가이드 — AI별 역할, 워크플로우, chatgpt_master.md 템플릿 | [보기](https://raw.githubusercontent.com/AINVC/nvc_public/main/nvc_network/md/nvc_network_AIGUIDE.md) |
| **CLAUDE** | Claude Code 작업 지침 — 파일 구조, 핵심 제약사항 | [보기](https://raw.githubusercontent.com/AINVC/nvc_public/main/nvc_network/md/nvc_network_CLAUDE.md) |

### GUI — Python/Tkinter 관리 도구 (V1 완성)

| 문서 | 설명 | Raw URL |
|------|------|---------|
| **GUI README** | 기술 퀵스타트 — 설치, 실행, 모듈 구조, SSH 설정 | [보기](https://raw.githubusercontent.com/AINVC/nvc_public/main/nvc_network/md/gui_README.md) |
| **GUI MANUAL** | 사용자 매뉴얼 — 로그인, 탭별 기능, AP Reset, 권한 구조 | [보기](https://raw.githubusercontent.com/AINVC/nvc_public/main/nvc_network/md/gui_MANUAL.md) |
| **GUI DESIGN** | 설계 구상서 — 초기 설계부터 V1 구현 이력, 기술 결정 | [보기](https://raw.githubusercontent.com/AINVC/nvc_public/main/nvc_network/md/gui_DESIGN.md) |

### ELK Stack — NAS Docker (V27)

| 문서 | 설명 | Raw URL |
|------|------|---------|
| **ELK README** | V27 베이스라인, RCA 모델, Kibana 분석 가이드 | [보기](https://raw.githubusercontent.com/AINVC/nvc_public/main/nvc_network/md/elk_README.md) |
| **ELK RUNBOOK** | NAS 운영 명령어 모음 (SSH, Docker, Logstash) | [보기](https://raw.githubusercontent.com/AINVC/nvc_public/main/nvc_network/md/elk_RUNBOOK.md) |
| **ELK CLAUDE** | ELK 작업용 Claude Code 지침 | [보기](https://raw.githubusercontent.com/AINVC/nvc_public/main/nvc_network/md/elk_CLAUDE.md) |

### EFG

| 문서 | 설명 | Raw URL |
|------|------|---------|
| **EFG Reference** | GUI에서 참조하는 EFG/UniFi 기준 정보 | [보기](https://raw.githubusercontent.com/AINVC/nvc_public/main/nvc_network/md/efg_EFG.md) |

---

## AI 사용 가이드

**프로젝트 전체 파악이 필요할 때:**
```
README Raw URL 먼저 읽고 → 필요한 세부 문서 Raw URL로 접근
```

**Claude Code와 작업 시작 템플릿:**
```
CLAUDE.md 읽고 현재 상태 파악 후 아래 작업 진행해줘:
[작업 내용]
```

**ChatGPT / Gemini에 컨텍스트 전달 시:**
```
아래 URL을 읽고 현재 프로젝트 상태를 파악한 후 답변해줘:
https://raw.githubusercontent.com/AINVC/nvc_public/main/nvc_network/md/nvc_network_README.md
```
