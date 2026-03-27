# Introduction

## 왜 스킬 보안 감사가 필요한가

Claude Code의 스킬 생태계가 성장하면서 마켓플레이스에 제출되는 서드파티 스킬이 늘어나고 있다. 스킬은 Claude Code 안에서 실행되므로, 악의적이거나 부주의한 스킬은 다음과 같은 실질적 피해를 유발할 수 있다:

- **자격증명 노출** — API 키, SSH 키, AWS 자격증명이 스킬 파일에 하드코딩되거나 외부로 전송됨
- **원격 코드 실행** — `curl | bash` 같은 패턴으로 검증 없이 외부 코드를 실행
- **시스템 손상** — `rm -rf /`, `chmod 777`, `sudo` 같은 파괴적 동작으로 호스트 시스템 훼손
- **데이터 유출** — 셸 히스토리, 키체인, 민감 설정 파일을 읽어 외부로 전송

**skill-security-audit**은 이런 위험을 스킬이 등록되기 전에 자동으로 탐지하는 게이트키퍼다.

---

## 설계 철학

### Adoption-First: 확산을 막지 않는 최소 안전망

Phase 1의 제1원칙은 **"좋은 스킬이 불필요하게 막히는 것은, 나쁜 스킬이 등록되는 것만큼 나쁘다"** 이다.

규칙을 추가하는 기준은 단 하나:

> *"이 규칙이 없으면 조직에 실질적 피해(데이터 유출, 시스템 손상, 자격증명 노출)가 발생하는가?"*
>
> YES → Phase 1에 포함. NO → Phase 2로 이관.

그래서 Phase 1은 CRITICAL만 차단한다. HIGH는 "반드시 확인해야 할 경고"이지 차단 사유가 아니다. 제출자가 경고를 인지한 상태에서 의도된 동작이라면 통과시킨다.

### Deterministic-First: LLM이 판정하지 않는다

```
구조화 코드 (.sh, .py, .js, .ts)  →  패턴 매칭으로 즉시 판정
Markdown (.md)                     →  LLM이 문맥 분류 (directive vs descriptive)
                                      → 규칙이 판정
```

LLM은 **문맥 분류**에만 사용한다. "이 구문이 Claude에게 실행을 지시하는가(directive), 아니면 설명 텍스트인가(descriptive)?"를 판단하는 역할이다. 최종 판정은 항상 규칙 기반이므로, 같은 입력에 대해 일관된 결과를 기대할 수 있다.

이 설계가 필요한 이유: 스킬 파일(SKILL.md, references/)에는 `rm -rf 사용을 금지한다` 같은 **설명 텍스트**와 `rm -rf ./cache` 같은 **실행 지시**가 섞여 있다. 단순 패턴 매칭만 쓰면 설명 텍스트가 오탐된다.

### Actionable Feedback: 차단하되 해결책을 함께 제공

BLOCKED 판정을 받았을 때 "뭘 고쳐야 하는지 모르겠다"면 게이트키퍼가 아니라 장벽이 된다. 모든 finding에는 다음이 포함된다:

- **규칙 ID** — 어떤 규칙에 걸렸는지 (예: SEC-010)
- **파일:라인** — 정확한 위치
- **증거** — 탐지된 패턴 또는 구문 발췌
- **수정 제안** — 구체적인 수정 방법

---

## 아키텍처

### 검사 파이프라인

```
Step 1: 대상 식별
  Glob으로 스킬 디렉토리 스캔 → 대상 파일 필터링
      ↓
Step 2: 파일 유형별 분석
  구조화 코드 → 패턴 매칭
  Markdown    → LLM 문맥 분류 (directive / descriptive / 애매)
  JSON/YAML   → 자격증명 패턴 매칭
      ↓
Step 3: 규칙 적용
  references/ 체크리스트 로드 → 각 파일에 22개 규칙 적용
      ↓
Step 4: 판정
  audit-ignore 예외 처리 → CRITICAL 존재 여부로 최종 판정
      ↓
Step 5: 보고서 생성
  Markdown 보고서 + JSON 출력
      ↓
Step 6: Slack 알림 (선택)
  BLOCKED일 때만 전송. 실패해도 검사 결과에 영향 없음
```

### 규칙 체계

22개 Phase 1 규칙은 3개 카테고리로 분류된다:

| 카테고리 | 파일 | 규칙 수 | 내용 |
|----------|------|---------|------|
| **Security** | `references/security-checklist.md` | 14개 | 자격증명, RCE, 셸 인젝션, 민감 경로, 네트워크 |
| **Destructive** | `references/destructive-ops-checklist.md` | 4개 | 재귀 삭제, force push, 시스템 권한 변경 |
| **Quality** | `references/quality-checklist.md` | 8개 | SKILL.md 존재, frontmatter, 구조 품질 |

각 규칙에는 ID, 탐지 패턴, 심각도, 파일 유형별 처리 전략, 수정 제안이 포함되어 있다.

### 심각도 레벨

| 레벨 | 의미 | 판정 영향 |
|------|------|-----------|
| **CRITICAL** | 즉각적 실질 피해 발생 가능 | ❌ BLOCKED — PR 차단 |
| **HIGH** | 높은 위험이나 맥락에 따라 정상일 수 있음 | ⚠️ 경고만 (통과) |
| **MEDIUM** | 위험 가능성 있음, 제출자 판단 필요 | ⚠️ 참고 (통과) |

### Self-Audit 보호

이 스킬 자신의 `references/` 파일에는 규칙 설명 목적으로 `rm -rf`, `curl | bash`, `sk-xxx` 같은 위험 패턴 예시가 포함된다. 자기 자신을 검사할 때 이들이 오탐되지 않도록 **Self-Audit 보호 규칙**이 적용된다:

1. references/ 파일 상단의 면책 문구 확인 → 해당 파일 내 모든 패턴을 descriptive로 분류
2. `- **탐지 패턴**:` 또는 `- **검사 방법**:` 하위 목록은 무조건 descriptive
3. `**수정 제안**:` 내 패턴도 descriptive
4. SKILL.md 본문의 curl 명령(Slack 전송)은 스킬 자체 동작이므로 제외

CI의 Self-Audit 테스트에서 FALSE POSITIVE 0건이 검증되어야 룰셋 배포가 허용된다.

---

## Phase 로드맵

| Phase | 대상 환경 | 게이트 철학 | 상태 |
|-------|-----------|-------------|------|
| **Phase 1** | 조직 내부 마켓플레이스 | 실질 피해만 차단 (Adoption-First) | **현재** |
| **Phase 2** | 외부 공개 마켓플레이스 | 잠재적 위험까지 차단 (Security-First) | 계획 |

Phase 2에서 추가될 예정인 항목:

- 강화된 규칙셋 (JWT 토큰, 동적 import, 난독화 탐지 등)
- SARIF 출력 형식 지원
- 동적 샌드박스 실행 검증
- Policy-as-data 엔진
- HIGH 차단 여부 재검토
