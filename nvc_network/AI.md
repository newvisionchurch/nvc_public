# NVC Network 요약

뉴비전 교회(NVC) 네트워크 운영을 위한 공개 AI 컨텍스트 저장소입니다.  
목적은 **AP Stuck / AP No-Service 문제를 로그 분석 + GUI 제어 + EFG 참조 문서**로 빠르게 해결하는 것입니다.

---

## 전체 구조

### 1) ELK Stack — NAS Docker
- NAS에서 Docker Compose로 운영
- UniFi AP / EFG syslog 수집, 분류, 저장
- 입력 포트: UDP 51415
- 출력:
  - Elasticsearch 인덱스
    - `unifi-network-v27-ap-*`
    - `unifi-network-v27-low-*`
    - `unifi-network-v27-noise-*`
  - JSONL 파일 저장
    - `logstash/outputs/ap/`
    - `logstash/outputs/low/`
    - `logstash/outputs/noise/`
- 역할:
  - AP 장애 원인 분석
  - system_issue / qos_error / rrm_scan_trigger / ap_stuck / ap_no_service 분류
  - RCA 타임라인 추적

### 2) GUI — Python + Tkinter
- Windows 로컬 관리 도구
- 실행 시 최대화
- 주요 기능:
  - VPN 연결 확인
  - 로그인 + bcrypt + TOTP 2FA
  - 역할 기반 권한 관리
  - NAS 로그 동기화
  - AP Stuck 집계 / 상세 분석
  - AP SSH 원격 조회
  - AP 다중 Reset
  - EFG 대시보드 조회
  - 로그 Export
  - Windows Credential Manager로 자격증명 보호

### 3) EFG
- 네트워크 코어 장비 참조 문서
- GUI 코드가 직접 참조하는 기준 문서
- 경로 변경 금지

---

## 핵심 목표

이 프로젝트의 목표는:

1. AP Stuck / AP No-Service의 반복 원인을 찾고
2. NAS ELK로 로그를 수집·분류·저장하고
3. GUI 하나로 문제 AP를 찾고
4. EFG / AP를 SSH로 원격 제어하는 것

즉, **Kibana나 수동 SSH 없이 GUI 중심으로 장애 대응**하는 것을 목표로 합니다.

---

## 프로젝트 폴더

- `elk/`  
  NAS Docker ELK Stack 및 Logstash 파이프라인

- `gui/`  
  Python/Tkinter 관리 도구

- `efg/`  
  EFG 장비 참조 문서

- `codex/`  
  Codex 시대 작업물 백업

---

## GUI 주요 파일

- `main.py`  
  Tkinter 시작점

- `modules/auth.py`  
  bcrypt 로그인 + TOTP + 개발자 모드

- `modules/vpn_check.py`  
  VPN 연결 확인

- `modules/nas_sync.py`  
  NAS JSONL → 로컬 캐시 동기화

- `modules/ap_count.py`  
  AP Stuck 이벤트 집계

- `modules/ap_detail.py`  
  문제 AP 상세 분석

- `modules/ap_remote.py`  
  AP SSH 스캔 / 진단 / 다중 Reset

- `modules/efg_remote.py`  
  EFG 시스템 대시보드 조회

- `modules/log_export.py`  
  조건별 로그 Export

- `modules/keychain.py`  
  Windows Credential Manager 래퍼

---

## GUI 탭 기능

- Log 동기화
- AP ELK Log
- AP Remote
- EFG Remote
- Log Export
- 설정
- 사용자 관리
- EFG 참조

---

## AP Remote 진단 항목

AP에 SSH로 들어가 `mca-dump`와 `logread`를 이용해 진단합니다.

주요 컬럼:
- ELK Stuck
- 로그오류
- 재시작2G/5G
- VAP지연2G/5G
- CU2G% / CU5G%

임계값 초과 시 `⚠` 표시.

---

## AP Reset 방식

AP에 직접 붙기 어려운 경우가 있어서 다음 경로를 사용합니다:

`GUI (paramiko) → EFG (OpenSSH) → sshpass → AP (Dropbear)`

즉, EFG를 relay로 사용해 AP를 안전하게 재부팅합니다.

---

## 로그/분석 철학

- AP Stuck은 증상이지 근본 원인이 아님
- 반드시 EFG / system_issue / QoS / RRM까지 같이 봐야 함
- 분석 순서:
  1. `system_issue`
  2. `rrm_scan_risk`
  3. `ap_stuck`
  4. `ap_no_service`

---

## ELK Stack 핵심

- Logstash가 syslog를 받아서 분류
- `ap / low / noise`로 라우팅
- RCA 중심 분류 규칙 사용
- small delta 방식으로만 파이프라인 수정
- 보호된 baseline 구간은 수정 금지

---

## 빠른 실행

```powershell
cd C:\Projects\nvc_network
pip install -r gui\requirements.txt
python gui\main.py
```

또는:

```powershell
.\gui\run-gui.ps1
```

---

## 주의사항

- 비밀번호, API 키, `.env` 파일은 커밋 금지
- `users.json`, `ssh_targets.json`은 git 제외
- `efg/EFG.md` 경로 변경 금지
- 원본 대용량 로그는 커밋하지 않음
- Logstash 핵심 baseline은 절대 수정 금지