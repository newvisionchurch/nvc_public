# ELK — Claude Code 작업 지침

이 파일은 `elk/` 디렉토리에서 작업할 때 Claude Code가 따르는 규칙이다.
root의 `CLAUDE.md`와 함께 읽힌다.

---

## 작업 전 필수 확인

1. `elk/README.md` — 현재 V27 베이스라인, 파이프라인 구조 맵, RCA 모델, 검증 체크리스트
2. `elk/docs/logstash-rules.md` — 필드 전체 목록, 감지 키워드, 이벤트 매핑 상세
3. `elk/logstash/pipeline/logstash.conf` — 활성 파이프라인 코드
4. 문제 클래스 추가 시: `elk/docs/root_cause_kr.md` — 실제 RCA 증거 및 가설

---

## 절대 금지 (보호된 baseline)

다음 구간은 어떤 이유로도 수정 금지. 변경 요청이 오면 대신 이유를 설명하고 대안 제시.

```
input { udp { port => 51415 ... } }          ← 포트 변경 금지
섹션 2: Base syslog grok (8개 패턴)          ← 파서 기반 — 수정 시 전체 파괴
섹션 4: observer/dataset base 로직           ← AP vs EFG 분리 기반
output { elasticsearch { ... } }             ← ES 인덱스 출력 구조
output { file { ... } }                      ← JSONL 파일 출력
output { stdout { ... } }                    ← AP-only 디버그
```

---

## 변경 원칙

1. **Small delta only** — 기존 블록 수정이 아니라 새 `if` 블록 추가
2. **한 번에 1개 기능** — 복수 문제 클래스를 한 번에 추가하지 않는다
3. **unclassified guard 유지** — 새 rule은 반드시 `[event][action] == "unclassified"` 조건 포함
4. **기존 event.action 수정 금지** — 새 action을 추가할 수는 있으나 기존 것 변경 금지
5. **정상 lifecycle drop 금지** — 저우선순위 이벤트도 `low_priority`로 유지, 삭제하지 않는다

---

## 새 문제 클래스 추가 패턴

```logstash
# 섹션 번호. 설명 (버전 표시)
if [event][action] == "unclassified" and (
  [message] =~ /패턴1/ or
  [message] =~ /패턴2/
) {
  mutate {
    replace => {
      "[event][action]"        => "action_name"
      "[event][category]"      => "category"
      "[event][problem_class]" => "problem_class"
      "[event][problem_name]"  => "Problem Name"
      "[event][tier]"          => "high"
      "[event][outcome]"       => "failure"
    }
    add_field => {
      "[event][reason]" => "reason_string"
    }
  }
}
```

correlation helper가 필요한 경우 V25.2 블록 패턴 참조 (`correlation_stage`, `correlation_domain`, `blast_scope`, `correlation_target`).

---

## 변경 후 필수 작업

1. `docker compose exec logstash logstash --config.test_and_exit` — config test
2. `docker compose restart logstash` — 적용
3. Kibana에서 새 event.action / problem_class 이벤트 확인
4. `elk/docs/logstash-rules.md` 업데이트 — 새 패턴/필드/tier 추가
5. `elk/README.md`의 버전 이력 행 추가

---

## V27 현재 상태 요약

- storage_class: `ap` (RCA focus), `low` (컨텍스트), `noise` (known-noise)
- rca_step 1~7: `rrm_scan_trigger` → `retry_high` → `tx_overflow` → `radio_reset` → `vap_timeout` → `channel_invalid` → `qos_error/ap_no_service`
- 확인된 취약 모델: `UAP-AC-Pro-Gen2`, fw `6.8.2.15592` — `iwpriv wifi1ap3 acsrrm` 반복이 VAP stuck 유발

## V28 다음 구현 후보

우선순위 순:
1. `auth_issue` — EAPOL timeout, auth fail, RADIUS reject (오탐 방지 최우선)
2. `roaming_issue` — FT decrypt fail, roaming failure
3. DNS/DHCP 세분화

---

## 금지된 분석 오류

- RF 문제(`rf_issue`)와 서비스 실패(`ap_no_service`)를 같은 problem_class로 처리
- `system_issue`를 부차적 이벤트로 취급
- AP 로그만 보고 EFG/gateway 이벤트 확인 생략
- 단일 로그 패턴 하나로 root cause 결론
