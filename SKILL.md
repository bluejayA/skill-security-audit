---
name: skill-security-audit
description: >
  Use when auditing a third-party Claude Code skill for security risks,
  destructive operations, and minimum quality standards.
  Trigger on "스킬 검사", "스킬 보안 검토", "skill audit", "security review",
  "스킬 등록 검증", "marketplace submission review", "제출 스킬 검사".
metadata:
  version: 1.5.0
  author: jay
  category: security
  phase: 1
  invoke_mode:
    - reviewer        # PR/CI 컨텍스트에서 게이트키퍼로 호출
    - user-invocable  # 제출자 로컬 사전 검사로 호출
  return_behavior: report
---

# Skill Security Audit

제3자가 Claude Code 마켓플레이스에 제출한 스킬을 **자격증명 보호, 시스템 안전, 최소 구조 품질** 기준으로 검사한다.

## 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **Adoption-First** | CRITICAL만 차단. 좋은 스킬이 불필요하게 막히지 않는다 |
| **Deterministic-First** | LLM은 Markdown 문맥 분류에만 사용. 최종 판정은 규칙 기반 |
| **Actionable Feedback** | 차단 시 반드시 규칙 ID + 파일:라인 + 수정 예시 제공 |

---

## Step 1: 대상 식별

검사 대상 스킬 디렉토리를 재귀적으로 스캔하여 파일 목록을 수집한다.

### 검사 대상 파일

| 대상 | 검사 내용 |
|------|-----------|
| `SKILL.md` | 구조·품질·보안 패턴 |
| `references/**`, `assets/**` | 보안 패턴 |
| `scripts/**`, `*.sh`, `*.py`, `*.js`, `*.ts` | 보안 + 파괴적 동작 |
| `package.json` (`scripts` 섹션) | 명령어 검사 |
| `Makefile` | 타겟 내 명령어 검사 |

### 제외 대상

- `node_modules/`, `.git/`, 바이너리 파일
- 저장소 내 파일만 검사한다. 외부 URL 링크 대상은 검사하지 않는다.

### 실행

1. Glob tool로 `**/*` 패턴으로 스킬 디렉토리 전체 파일 목록을 수집한다
2. 위 대상 파일 확장자와 경로에 해당하는 파일만 필터링한다
3. 대상 파일 목록을 기록한다 (보고서 Step 5에서 사용)

---

## Step 2: 파일 유형별 분석

각 대상 파일을 Read tool로 읽고, 파일 유형에 따라 다른 분석 전략을 적용한다.

### 분석 전략 테이블

| 파일 유형 | 전략 |
|-----------|------|
| `*.sh`, `*.py`, `*.js`, `*.ts`, `Makefile`, `package.json` scripts | **패턴 매칭으로 즉시 판정** |
| `*.md` (SKILL.md, references/, assets/) | **LLM 문맥 분류 후 판정** |
| `*.json`, `*.yaml` (assets/) | **패턴 매칭** (자격증명 중심) |

### Markdown 문맥 분류 기준

Markdown 파일에서 패턴이 탐지되면, 해당 구문이 어떤 목적인지 분류한다:

```
해당 구문이 Claude에게 실행을 지시하는가?
  YES (directive)   → 규칙 기본 심각도로 판정
  NO  (descriptive) → 탐지하지 않음 (finding에 포함하지 않음)
  애매함             → MEDIUM + "(context uncertain)" 표기
```

**고신뢰 패턴 예외**: 하드코딩된 API 키(`sk-`, `ghp_`, `AKIA` 등), 프라이빗 키(`BEGIN PRIVATE KEY`), `curl|bash` 패턴은 Markdown에서도 문맥 분류 없이 즉시 판정한다. 단, 아래 Self-Audit 보호 규칙이 우선 적용된다.

### Self-Audit 보호 규칙 (CRITICAL)

이 스킬 자신의 `references/` 파일에는 규칙 설명 목적으로 위험 패턴 예시가 포함된다.
**오탐을 방지하기 위해 다음을 반드시 적용한다** (오탐 발생 시 Troubleshooting 참조):

1. 각 references/ 체크리스트 파일 상단에 다음 문구가 있는지 확인한다:
   > "이 파일은 SKILL.md에서 참조하는 규칙 정의 문서이며, 여기에 포함된 패턴 예시는 **설명 텍스트**이다."
2. 이 문구가 있는 파일 내의 패턴 예시는 **모두 descriptive**로 분류한다 — 고신뢰 패턴(API 키, curl|bash 등)도 예외 없이 descriptive 처리
3. 탐지 패턴이 `- **탐지 패턴**:` 또는 `- **검사 방법**:` 하위 목록에 포함된 경우 → descriptive
4. 탐지 패턴이 `**수정 제안**:` 내에 포함된 경우 → descriptive
5. SKILL.md 본문의 `curl` 명령(Step 6 Slack 전송)은 스킬 자체의 동작 지시이므로 SEC-020 대상에서 제외한다

---

## Step 3: 규칙 적용

Step 2에서 분석한 각 파일에 Phase 1 규칙을 적용한다. 규칙 상세는 references/ 파일을 참조한다.

### 규칙 참조 파일

| 파일 | 내용 |
|------|------|
| `references/security-checklist.md` | SEC-*, SBX-* 규칙 (자격증명, RCE, 셸 인젝션, 민감 경로, 네트워크) |
| `references/destructive-ops-checklist.md` | DST-* 규칙 (파괴적 동작) |
| `references/quality-checklist.md` | QUA-* 규칙 (최소 품질) |

**각 체크리스트 파일을 Read tool로 로드**하여 규칙 ID, 탐지 패턴, 심각도, 파일 유형별 처리, 수정 제안을 확인한 후 적용한다.

### Phase 1 규칙 요약 (22개)

#### CRITICAL (11개) — 1개라도 발견 시 BLOCKED

| ID | 검사 항목 |
|----|-----------|
| SEC-010 | 하드코딩된 API 키 |
| SEC-011 | 프라이빗 키 |
| SEC-013 | process.env 전체 덤프/외부 전송 |
| SEC-003 | 파이프를 통한 원격 실행 |
| SEC-030 | Base64 디코딩 후 실행 |
| SEC-001 | 셸 인젝션 (untrusted input + 셸 실행 경로) |
| SBX-003 | 경로 탈출 |
| SBX-004 | 홈 디렉토리 민감 경로 참조 |
| SBX-007 | 키체인·히스토리 읽기 |
| DST-001 | 재귀적 삭제 |
| DST-007 | 시스템 수준 권한·커널 변경 |

#### HIGH (6개) — 경고 (차단하지 않음)

| ID | 검사 항목 |
|----|-----------|
| SEC-001 | 셸 인젝션 (trusted input / 단순 변수) |
| SEC-002 | 동적 코드 실행 (eval, exec, Function) |
| SEC-020H | 외부 HTTP + 민감 데이터 페이로드 |
| SEC-022 | 직접 네트워크 도구 (ssh, nc, netcat) |
| SBX-001 | 스킬 디렉토리 외부 파일 쓰기 |
| DST-003 | git push --force |

#### MEDIUM (5개) — 참고 사항

| ID | 검사 항목 |
|----|-----------|
| SEC-012 | 민감 설정 파일 직접 참조 |
| SEC-020 | 외부 HTTP 요청 (기본) |
| DST-002 | 단일 파일 삭제 |
| QUA-003~006 | description 형식, 길이, 본문 줄 수, 참조 깊이 |
| QUA-010, QUA-011 | 모호 표현, description 내부 구현 노출 |

#### CRITICAL (품질)

| ID | 검사 항목 |
|----|-----------|
| QUA-001 | SKILL.md 파일 존재 |

#### HIGH (품질)

| ID | 검사 항목 |
|----|-----------|
| QUA-002 | frontmatter 필수 필드 (name, description) |

### SEC-001 심각도 분기

SEC-001은 trusted/untrusted input 판정에 따라 CRITICAL 또는 HIGH로 분기한다:

- **CRITICAL**: `sh -c`, `bash -c`, `subprocess(shell=True)` 등 셸 실행 경로에 untrusted input 직접 삽입
- **HIGH**: 단순 변수 할당, trusted input만 사용
- **MEDIUM**: Markdown에서 directive/descriptive 판단이 애매한 경우

### 최종 심각도 결정

```
구조화 코드에서 패턴 탐지     → 규칙 기본 심각도로 즉시 판정
Markdown에서 directive 확인   → 규칙 기본 심각도로 판정
Markdown에서 애매한 경우      → MEDIUM (규칙 기본값에 관계없이) + "(context uncertain)"
```

---

## Step 4: 판정

모든 파일에 대한 규칙 적용이 끝나면, findings 목록을 집계하여 최종 판정을 내린다.

### audit-ignore 예외 처리

판정 전에 SKILL.md frontmatter의 `audit-ignore` 선언을 확인한다.

1. `audit-ignore` 배열의 각 항목에서 `rule`, `reason`, `reviewer`, `expires` 필드를 확인한다
2. 4개 필드 중 하나라도 누락이면 → 예외 무효, 원래 심각도로 보고
3. `reviewer` 값을 `config/approved-reviewers.yml`과 대조한다
   - **파일이 없는 경우** (로컬 실행 등): 모든 audit-ignore 예외를 무효로 처리하고, 보고서에 `(approved-reviewers.yml 미발견 — 모든 예외 무효)` 표기
   - 파일은 있지만 reviewer가 목록에 없으면 → 해당 예외만 무효, 원래 심각도로 보고
4. `expires` 날짜가 오늘 이전이면 → 만료, 원래 심각도로 보고
5. 모든 조건 충족 → 해당 규칙의 finding을 **EXEMPTED**로 표시 (숨기지 않음)

### 만료 사전 경고 (D-30)

`expires` 날짜가 오늘부터 30일 이내이면 Slack 만료 경고 대상으로 표시한다.

### 최종 판정 로직

```
CRITICAL 1개 이상 (EXEMPTED 제외)  → ❌ BLOCKED
HIGH만 존재 (CRITICAL 없음)        → ⚠️ PASSED with warnings
MEDIUM만 존재 (HIGH 없음)          → ⚠️ PASSED with warnings
LOW만 또는 발견 없음                → ✅ PASSED
```

---

## Step 5: 보고서 생성

`assets/report-template.md`를 Read tool로 로드하여 템플릿 구조를 확인한 후, 다음 값을 채워 보고서를 출력한다.

### 보고서 필수 항목

| 항목 | 값 |
|------|-----|
| Skill | 스킬 디렉토리명 또는 SKILL.md의 `name` |
| Submitted by | PR 작성자 또는 `(local)` |
| Audit date | 검사 실행 일시 |
| Ruleset version | `ruleset-version.txt` 내용 (예: `1.0.0`) |
| Ruleset SHA | 스킬 디렉토리의 git SHA (가능한 경우) 또는 `(local)` |
| Phase | `1` |
| Verdict | 판정 결과 |

### findings 출력 순서

1. **CRITICAL** — 차단 사유 (🔴)
2. **HIGH** — 반드시 확인 (🟠)
3. **MEDIUM** — 참고 사항 (🟡)
4. **EXEMPTED** — 예외 처리됨 (별도 섹션)

각 finding에는 다음을 포함한다:
- 규칙 ID + 제목
- 파일:라인
- 증거 (탐지된 패턴 또는 구문 발췌)
- 수정 제안 (references/ 체크리스트의 수정 제안 참조)

### Machine-readable JSON 출력

보고서와 별도로 `audit-result.json` 형식의 JSON도 출력한다:

```json
{
  "version": "{{ruleset_version}}",
  "phase": 1,
  "ruleset_sha": "{{sha}}",
  "skill": "{{skill_name}}",
  "verdict": "BLOCKED | PASSED | PASSED_WITH_WARNINGS",
  "counts": { "critical": 0, "high": 0, "medium": 0, "low": 0, "exempted": 0 },
  "findings": []
}
```

---

## Step 6: Slack 알림 (선택)

`SLACK_WEBHOOK_URL` 환경변수가 설정된 경우에만 실행한다. 미설정 시 skip하고 보고서 하단에 `(Slack 미설정 — 로컬 실행)` 표기.

### 전송 정책

- **BLOCKED만 즉시 전송**. PASSED/WARNING은 전송하지 않는다.
- **D-30 만료 경고**: audit-ignore 만료 30일 이내 항목이 있으면 경고 전송

### 전송 방법

1. `assets/slack-message-template.json`을 Read tool로 로드한다
2. 템플릿의 플레이스홀더를 실제 값으로 치환한다
3. `curl`로 Slack Incoming Webhook에 POST한다

```bash
curl -s -X POST "$SLACK_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{{치환된 JSON}}'
```

### 실패 처리

Slack 전송 실패 시 검사 결과에 영향을 주지 않는다. 보고서에 `(Slack 전송 실패)` 표기만 추가한다.

---

## 멀티 스킬 PR 처리

PR에 여러 스킬이 변경된 경우:

1. 변경된 파일이 속한 스킬 디렉토리를 식별한다
2. 스킬별로 **독립적으로** Step 1~6을 실행한다
3. 결과를 하나의 보고서에 스킬별 섹션으로 통합한다
4. 하나라도 BLOCKED → 전체 PR failure

---

## Trigger

이 스킬은 다음 상황에서 트리거된다:

- 사용자가 "스킬 검사", "스킬 보안 검토", "skill audit", "security review" 등의 키워드를 사용할 때
- 스킬 디렉토리 경로를 지정하며 검사를 요청할 때
- PR 리뷰 과정에서 제출된 스킬의 보안 검증이 필요할 때
- GitHub Actions에서 `skills/**` 경로 변경으로 자동 트리거될 때

**트리거하지 않는 경우:**
- 일반 코드 리뷰 (→ `code-review` 스킬 사용)
- 스킬 작성/설계 (→ `skill-development` 또는 `writing-skills` 사용)
- 보안 취약점 일반 분석 (이 스킬은 Claude Code 스킬 전용)

---

## Examples

### 예시 1: 로컬 단일 스킬 검사

```
사용자: 제출 전에 이 스킬을 검사해줘: ~/projects/my-skill
```

→ Step 1~5 실행. SKILL.md + references/ + scripts/ 스캔.
→ Markdown 보고서 출력. Slack 미전송 (SLACK_WEBHOOK_URL 미설정).
→ Submitted by: `(local)`, Ruleset SHA: `(local)`

### 예시 2: PR 멀티 스킬 검사

```
사용자: PR #42에서 변경된 스킬들을 검사해줘
```

→ 변경된 스킬 디렉토리 식별 (예: `skills/data-sync/`, `skills/api-connector/`)
→ 스킬별 독립 검사, 단일 보고서에 통합 출력
→ 하나라도 BLOCKED → 전체 PR failure

### 예시 3: audit-ignore가 포함된 스킬

```
사용자: skills/deploy-helper 스킬을 검사해줘
```

→ SKILL.md frontmatter에서 audit-ignore 선언 확인
→ reviewer를 `config/approved-reviewers.yml`과 대조
→ 유효한 예외: EXEMPTED로 표시 (숨기지 않음)
→ 만료 30일 이내: Slack 경고 대상 표시

---

## Troubleshooting

### `config/approved-reviewers.yml` 파일이 없는 경우

audit-ignore에 `reviewer` 필드가 있지만 승인자 목록 파일이 없으면:
- 해당 예외를 **무효**로 처리하고 원래 심각도로 보고한다
- 보고서에 `(approved-reviewers.yml 미발견 — 예외 무효)` 표기

### references/ 체크리스트 파일 누락

`references/security-checklist.md` 등 체크리스트 파일을 Read할 수 없는 경우:
- 해당 카테고리 규칙 검사를 **skip**하고 보고서에 `(체크리스트 미발견 — 해당 카테고리 검사 생략)` 표기
- 다른 카테고리 규칙은 정상 적용. 검사 전체를 실패시키지 않는다

### Self-Audit 시 오탐 발생

이 스킬 자신의 `references/` 파일을 검사할 때 패턴 예시가 탐지되면:
- Step 2의 Self-Audit 보호 규칙이 정상 작동하는지 확인한다
- references/ 파일 상단에 설명 텍스트 면책 문구가 있는지 확인한다
- 문구가 누락되었으면 추가 후 재검사한다

### Slack 전송 실패

`SLACK_WEBHOOK_URL`이 설정되었지만 전송에 실패한 경우:
- 검사 결과에 영향 없음 (Fail-safe 원칙)
- 보고서 하단에 `(Slack 전송 실패)` 표기
- curl 종료 코드와 HTTP 응답을 확인하여 원인 파악

### 대규모 스킬 (파일 수 50개 이상)

파일 수가 많아 토큰 한계에 근접하는 경우:
- SKILL.md와 구조화 코드(*.sh, *.py, *.js, *.ts)를 우선 검사한다
- references/ 파일은 보안 패턴 중심으로 스캔한다
- 검사 범위가 제한되면 보고서에 `(일부 파일 미검사 — 파일 목록 참조)` 표기
