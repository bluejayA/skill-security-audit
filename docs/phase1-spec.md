# skill-security-audit — Specification

| 항목 | 내용 |
|------|------|
| **버전** | 1.5-draft |
| **작성일** | 2026-03-27 |
| **상태** | Draft — 4차 리뷰 반영 |
| **작성자** | jay |
| **변경 이력** | v1.4 → v1.5: SEC-001 CRITICAL 판정 조건 세분화, trusted/untrusted input 정의 추가 |

---

## 0. 이 문서의 전제

### Phase 1 — 조직 내 AI 확산 지원 (현재 문서)

이 스펙은 **조직 내 AI 도구 확산을 저해하지 않는 것을 제1 원칙**으로 한다.

규칙을 추가하는 기준은 단 하나다:

> **"이 규칙이 없으면 조직에 실질적 피해(데이터 유출, 시스템 손상, 자격증명 노출)가 발생하는가?"**
>
> YES → Phase 1에 포함. NO → Phase 2로 이관.

좋은 스킬이 불필요하게 차단되는 것은 나쁜 스킬이 등록되는 것만큼 나쁘다. Phase 1의 검사는 최소한의 안전망이며, 품질·운영·보안 고도화는 Phase 2에서 다룬다.

### Phase 2 — 상용 마켓플레이스 배포 (별도 문서)

조직 내 AI 활용이 충분히 확산된 이후, 외부 제출·상용 환경에 맞는 강화된 감사 스펙을 별도로 정의한다. Phase 2 항목은 이 문서 말미 **18절**에 명시한다.

---

## 1. 개요

### 1.1 목적

제3자가 Claude Code 마켓플레이스에 제출한 스킬을 **자격증명 보호, 시스템 안전, 최소 구조 품질** 기준으로 검사하는 독립 스킬.

Phase 1의 게이트키퍼 역할은 **"즉각적으로 실질 피해가 발생하는 것만 차단"** 이다. 위험 가능성이 있지만 맥락에 따라 정상인 것은 경고로 남기고 제출자가 판단하게 한다.

### 1.2 비전

- **Phase 1 (현재 scope)**: 필수 보안 규칙 + 패턴/문맥 하이브리드 검사 + GitHub Actions 자동화 + Slack 알림
- **Phase 2 (향후)**: 강화된 규칙셋, SARIF 출력, 동적 샌드박스, policy-as-data 엔진

### 1.3 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **Adoption-First** | 규칙은 AI 확산을 저해하지 않는 최소 범위로 제한 |
| **Deterministic-First** | 규칙이 판정한다. LLM은 Markdown 문맥 분류에만 사용한다 |
| **Actionable Feedback** | 차단 시 반드시 규칙 ID + 위치 + 수정 예시 제공 |
| **Local = CI** | 동일한 ruleset 버전과 판정 기준 적용. 단, LLM 실행 엔진 차이로 세부 메시지는 달라질 수 있다 |
| **Fail-safe** | Slack 등 부가 기능 실패가 검사 결과에 영향 없음 |

---

## 2. 실행 경로

### 2.1 제출자 자체 실행 (Local)

제출자가 `claude` CLI로 직접 실행. CI와 동일한 ruleset 적용. 출력: Markdown 보고서.

- `SLACK_WEBHOOK_URL` 미설정 시: Slack 전송을 skip하고 보고서 하단에 `(Slack 미설정 — 로컬 실행)` 표기. 검사 결과에는 영향 없음.
- 로컬과 CI 모두 `ruleset-version.txt` 기준으로 ruleset 버전을 고정.

### 2.2 GitHub Action 자동 실행

```
PR 생성/업데이트
  → Action 트리거 (skills/** 변경 시)
  → Claude Code → skill-security-audit 로드 → 검사
  → PR comment 업데이트 (기존 봇 댓글 edit, 없으면 신규 생성)
  → Check Run 결과 설정
  → (비동기) Slack 알림
```

- Check Run: CRITICAL 존재 → `failure`, 그 외 → `success`
- Slack: `continue-on-error: true` — 전송 실패가 검사 결과에 영향 없음

### 2.3 관리자 수동 실행

`workflow_dispatch` 지원. 특정 PR 번호 또는 스킬 디렉토리 경로 지정 가능.

---

## 3. 분석 방식

### 3.1 설계 원칙: Context-Aware Deterministic-First

> **LLM이 판정하지 않는다. LLM은 문맥을 분류하고, 규칙이 판정한다.**

스킬 파일에는 두 종류의 텍스트가 섞여 있다.

- **지시문 (directive)**: Claude에게 실제 실행을 지시하는 문장
- **설명 텍스트 (descriptive)**: 규칙 설명, 사용 예시, 주석

단순 패턴 매칭만 쓰면 `"rm -rf 사용을 금지한다"` 같은 설명 텍스트가 탐지된다. 이를 막기 위해 파일 종류에 따라 분석 전략을 다르게 적용한다.

### 3.2 파일 유형별 분석 전략

| 파일 유형 | 분석 전략 | 이유 |
|-----------|-----------|------|
| `*.sh`, `*.py`, `*.js`, `*.ts` | 패턴 매칭으로 즉시 판정 | 구조화 코드 — 문맥 오탐 가능성 낮음 |
| `Makefile`, `package.json` scripts | 패턴 매칭으로 즉시 판정 | 명시적 실행 구문 |
| `*.md` (SKILL.md, references) | LLM 문맥 분류 후 규칙 적용 | 자연어 — directive/descriptive 구분 필요 |
| `*.json`, `*.yaml` (assets) | 패턴 매칭 (자격증명 중심) | 설정 파일 |

**Markdown 문맥 분류 기준:**

```
해당 구문이 Claude에게 실행을 지시하는가?
  YES (directive)   → 규칙 기본값으로 판정
  NO  (descriptive) → 탐지하지 않음
  애매함             → MEDIUM + "(context uncertain)" 표기
```

**최종 심각도 결정:**

```
구조화 코드에서 패턴 탐지     → 규칙 기본값으로 즉시 판정
Markdown에서 directive 확인   → 규칙 기본값으로 판정
Markdown에서 애매한 경우      → MEDIUM (규칙 기본값에 관계없이)
```

> 고신뢰 패턴(하드코딩 자격증명, `curl|bash`)은 Markdown에서도 LLM 확인 없이 판정 가능.

### 3.3 Self-Audit 테스트

`skill-security-audit` 자신의 `references/` 파일에는 `rm -rf`, `curl | bash` 등 규칙 설명 텍스트가 포함된다. CI에 Self-Audit 테스트를 포함하여 FALSE POSITIVE가 발생하지 않는지 검증한다.

- **통과 조건**: `skill-security-audit` 자신을 입력으로 넣었을 때 FALSE POSITIVE 0건
- 이 테스트가 실패하면 ruleset 배포를 차단한다

---

## 4. 심각도 레벨 및 판정 기준

### 4.1 심각도 정의

| 레벨 | 의미 | 판정 |
|------|------|------|
| **CRITICAL** | 즉각적으로 실질 피해 발생 가능. 자격증명 노출·원격 코드 실행·시스템 파괴 | ❌ BLOCKED |
| **HIGH** | 실질 피해 가능성이 높으나 맥락에 따라 정상일 수 있음. 반드시 인지 필요 | ⚠️ WARNING (통과) |
| **MEDIUM** | 위험 가능성 있음. 제출자 판단 필요 | ⚠️ WARNING (통과) |
| **LOW** | 품질·가독성 관련 개선 제안 | ℹ️ INFO (통과) |

### 4.2 최종 판정 로직

```
CRITICAL 1개 이상              →  ❌ BLOCKED
HIGH 존재, CRITICAL 없음        →  ⚠️ PASSED with warnings
MEDIUM만 존재 (HIGH 없음)       →  ⚠️ PASSED with warnings
LOW만 존재 또는 발견 없음        →  ✅ PASSED
```

> Phase 1에서는 **CRITICAL만 차단**한다. HIGH는 "반드시 확인해야 할 경고"이지 차단 사유가 아니다.
> HIGH를 무시하고 계속 제출하는 것은 제출자 책임이며, Phase 2에서 HIGH 차단 여부를 재검토한다.

---

## 5. 검사 범위

### 5.1 검사 대상

| 대상 | 검사 내용 |
|------|-----------|
| `SKILL.md` | 구조·품질·보안 패턴 검사 |
| `references/**`, `assets/**` | 보안 패턴 검사 |
| `scripts/**`, `*.sh`, `*.py`, `*.js`, `*.ts` | 보안 + 파괴적 동작 검사 |
| `package.json` (`scripts` 섹션) | 명령어 검사 |
| `Makefile` | 타겟 내 명령어 검사 |

**범위 제한:**

- **저장소 내 파일만** 검사한다. 외부 URL이 포함돼 있어도 링크 대상 본문은 검사하지 않는다.
- `node_modules/`, `.git/`, 바이너리 파일은 제외한다.

---

## 6. Phase 1 검사 규칙

> **Phase 1 규칙 선정 기준**: "이 규칙 없이는 조직에 실질적 피해가 발생하는가?"
>
> CRITICAL: 즉각적 피해 → 차단. HIGH: 높은 위험 → 경고. 나머지는 MEDIUM 경고 또는 Phase 2 이관.

---

### 6.1 자격증명 노출 (Credential Exposure)

스킬 파일에 자격증명이 포함되면 저장소에 커밋되는 순간 노출된다.

| 규칙 ID | 검사 항목 | 심각도 |
|---------|-----------|--------|
| SEC-010 | 하드코딩된 API 키 패턴 (`ghp_`, `sk-`, `xoxb-`, `AKIA`) | 🔴 CRITICAL |
| SEC-011 | 프라이빗 키 패턴 (`BEGIN PRIVATE KEY`, `BEGIN RSA PRIVATE KEY`) | 🔴 CRITICAL |
| SEC-013 | `process.env` 전체 덤프 또는 외부 URL로 전송 | 🔴 CRITICAL |

> **SEC-010 비고**: JWT 헤더 prefix(`eyJ`)는 secret 자체가 아니며 오탐이 많아 제외한다. 완전한 JWT 토큰 탐지는 Phase 2에서 정교한 패턴으로 다룬다.

---

### 6.2 원격 코드 실행 (Remote Code Execution)

외부에서 코드를 가져와 실행하는 패턴. 내용을 알 수 없는 코드가 실행된다.

| 규칙 ID | 검사 항목 | 심각도 |
|---------|-----------|--------|
| SEC-003 | 파이프를 통한 원격 실행 (`curl \| bash`, `wget \| sh`) | 🔴 CRITICAL |
| SEC-030 | Base64 디코딩 후 실행 (`echo ... \| base64 -d \| bash`) | 🔴 CRITICAL |

---

### 6.3 셸 인젝션 (Shell Injection)

신뢰할 수 없는 입력(untrusted input)이 검증 없이 직접 셸 명령에 삽입되는 패턴.

#### Trusted / Untrusted Input 정의

SEC-001 판정의 핵심 기준이다. 제출자와 리뷰어 모두 이 정의를 기준으로 판단한다.

| 구분 | 예시 |
|------|------|
| **Untrusted input** (검증 필요) | CLI argument, prompt input, PR 내용, repository content, 파일 읽기 결과, 환경변수(`process.env`, `$ENV`), 외부 API 응답 |
| **Trusted input** (검증 불필요) | 코드 내부 상수(하드코딩된 값), allowlist로 제한된 값, 검증된 enum/whitelist 값 |

| 규칙 ID | 검사 항목 | 심각도 |
|---------|-----------|--------|
| SEC-001 | Untrusted input이 셸 명령에 직접 삽입 | 🔴 CRITICAL / 🟠 HIGH |
| SEC-002 | `eval`, `exec`, `Function(string)` 등 문자열 기반 동적 코드 실행 | 🟠 HIGH |

**SEC-001 심각도 판정 조건:**

아래 조건 중 하나를 만족하면 **CRITICAL**:
- `sh -c`, `bash -c`, `subprocess(..., shell=True)` 등 셸 실행 경로에 untrusted input이 직접 삽입
- 입력 검증 / allowlist / escaping 없이 untrusted input으로 명령 문자열을 직접 구성
- untrusted input이 검증 없이 명령 실행에 연결 (파일 읽기 → 바로 실행 등)

위 조건 불충족 (단순 변수 할당, trusted input만 사용 등): **HIGH**

Markdown에서 directive/descriptive 판단이 애매한 경우: **MEDIUM**

> **SEC-002**: 합법적 사용도 많으므로 HIGH(경고). Markdown에서 애매한 경우 MEDIUM으로 처리.

---

### 6.4 민감 경로 접근 (Sensitive Path Access)

자격증명·개인 데이터가 저장된 경로에 대한 접근.

| 규칙 ID | 검사 항목 | 심각도 |
|---------|-----------|--------|
| SBX-003 | `..` 경로 탈출 (`../../etc/passwd`) | 🔴 CRITICAL |
| SBX-004 | 홈 디렉토리 민감 경로 참조 (`~/.ssh`, `~/.aws`, `~/.gnupg`, `~/.netrc`) | 🔴 CRITICAL |
| SBX-007 | 키체인·히스토리 읽기 (`~/.bash_history`, `security find-generic-password`) | 🔴 CRITICAL |

---

### 6.5 시스템 파괴적 동작 (Destructive Operations)

되돌리기 어렵거나 불가능한 파괴적 동작.

| 규칙 ID | 검사 항목 | 심각도 |
|---------|-----------|--------|
| DST-001 | 재귀적 삭제 (`rm -rf`, `rimraf`, `shutil.rmtree`) | 🔴 CRITICAL |
| DST-007 | 시스템 수준 권한·커널·소유권 변경 (`chmod 777`, `chown root`, `sysctl -w`) | 🔴 CRITICAL |
| DST-003 | `git push --force` (원격 히스토리 파괴) | 🟠 HIGH |

> `git push`(force 없음)는 Phase 1에서 탐지하지 않는다.

---

### 6.6 최소 구조 품질 (Minimum Quality)

스킬이 Claude Code에 의해 올바르게 로드·실행되기 위한 최소 조건.

| 규칙 ID | 검사 항목 | 심각도 |
|---------|-----------|--------|
| QUA-001 | `SKILL.md` 파일 존재 | 🔴 CRITICAL |
| QUA-002 | frontmatter 필수 필드 (`name`, `description`) 존재 | 🟠 HIGH |
| QUA-003 | `description`이 `"Use when"`으로 시작 | 🟡 MEDIUM |
| QUA-004 | `description` 길이 1024자 이하 | 🟡 MEDIUM |
| QUA-005 | `SKILL.md` 본문 500줄 이하 | 🟡 MEDIUM |
| QUA-006 | 참조 파일 깊이 1단계까지 (`SKILL.md → ref` OK, `ref → ref2` NG) | 🟡 MEDIUM |

> **QUA-001**: `SKILL.md`가 없으면 스킬 자체가 존재하지 않으므로 CRITICAL.
>
> **QUA-002**: frontmatter 누락은 보안 위험이 아닌 로딩 실패 문제이므로 HIGH(경고). 실제 구현에서는 스킬 로딩 실패 시 별도 에러 메시지로 표시한다.

---

### 6.7 경고 규칙 (HIGH/MEDIUM — 통과하지만 인지 필요)

아래 항목은 차단하지 않는다. 위험 수준에 따라 HIGH(강한 경고) 또는 MEDIUM(권고)으로 분류한다.

#### HIGH 경고 (위험 가능성 높음 — 반드시 확인)

| 규칙 ID | 검사 항목 | 심각도 |
|---------|-----------|--------|
| SEC-020H | 외부 HTTP 요청 + 자격증명·환경변수·PII를 페이로드에 포함 | 🟠 HIGH |
| SEC-022 | `ssh`, `nc`, `netcat` 등 직접 네트워크 도구 사용 | 🟠 HIGH |
| SBX-001 | 스킬 디렉토리 외부에 파일 쓰기·편집 지시 | 🟠 HIGH |

#### MEDIUM 경고 (위험 가능성 있음 — 제출자 판단)

| 규칙 ID | 검사 항목 | 심각도 |
|---------|-----------|--------|
| SEC-012 | `.env`, `credentials.json`, `token.json` 파일 직접 참조 | 🟡 MEDIUM |
| SEC-020 | 외부 HTTP 요청 (`WebFetch`, `curl`, `wget`, `http.get`) — 기본 | 🟡 MEDIUM |
| DST-002 | 단일 파일 삭제 (`rm`, `unlink`) — 재귀 아닌 경우 | 🟡 MEDIUM |
| QUA-010 | 모호 표현 사용 (`적절히`, `잘`, `필요 시`, `상황에 따라`) | 🟡 MEDIUM |
| QUA-011 | `description`에 내부 구현 상세 노출 (파일 경로, 위임 체인) | 🟡 MEDIUM |

> **Phase 1 미탐지 항목**: `git clone/fetch`, 패키지 설치(`npm install`, `pip install`), 프로세스 종료(`kill`), 절대경로 사용, 인라인 실행(`python -c`), 패키지 제거. 내부 개발 스킬에서 정상적으로 필요한 경우가 많다. → 18절 Phase 2 참고.

---

## 7. 예외 처리 (audit-ignore)

### 7.1 선언 방식

```yaml
---
name: my-deployment-skill
description: "Use when deploying to production..."
audit-ignore:
  - rule: DST-003
    reason: "배포 파이프라인 스킬 — git push --force가 핵심 기능"
    reviewer: "@admin"
    expires: "2026-09-30"
---
```

### 7.2 필수 필드

| 필드 | 필수 | 설명 |
|------|:----:|------|
| `rule` | ✅ | 예외 대상 규칙 ID |
| `reason` | ✅ | 구체적 사유 |
| `reviewer` | ✅ | 마켓플레이스 관리자 `@handle` |
| `expires` | ✅ | 만료일 — 최대 6개월 |

### 7.3 승인 검증

`reviewer` 값은 마켓플레이스 저장소의 `config/approved-reviewers.yml` 목록과 대조한다. 목록에 없으면 예외 무효.

```yaml
# config/approved-reviewers.yml
approved_reviewers:
  - "@admin"
  - "@marketplace-maintainer"
```

### 7.4 검사 시 처리

- 유효한 예외: 보고서에 **EXEMPTED**로 표시 (숨기지 않음)
- 만료된 예외: 원래 심각도로 보고
- 미승인 reviewer: 예외 무효, 원래 심각도로 보고

### 7.5 만료 사전 경고 (D-30)

만료 30일 전 Slack 경고 발송 (Step 6, `continue-on-error: true`).

```
⏰ audit-ignore 만료 예정 — "my-deployment-skill"

규칙: DST-003  |  만료: 2026-09-30 (D-28)  |  승인자: @admin
갱신하지 않으면 해당 규칙이 재활성화됩니다.
→ https://github.com/org/marketplace/pull/42
```

---

## 8. 오탐 재검토 요청

Phase 1에서는 경량 절차를 사용한다.

### 8.1 요청 방법

PR에 `audit-review-requested` 라벨을 추가하거나, PR comment에 아래 형식으로 작성:

```
재검토 요청: SEC-001
사유: 해당 구문은 사용 예시 설명이며 실행 지시가 아닙니다.
```

### 8.2 처리 방식

관리자가 수동으로 검토 후 finding을 인정하거나 유지한다.
인정 시: `false-positive-confirmed` 라벨 부여 → Check Run 수동 override.

> 봇 기반 상태 머신(DISPUTED/DISMISSED 자동화)은 Phase 2에서 구현한다.

---

## 9. 멀티 스킬 PR 처리

### 9.1 스캔 단위

- 변경된 파일이 속한 스킬 디렉토리 단위로 독립 검사
- 스킬별로 별도 Check Run 생성

### 9.2 PR comment 전략

기존 봇 댓글을 찾아 업데이트한다 (스팸 방지).
봇 댓글 식별: `<!-- skill-audit-report -->` HTML 주석.

```markdown
<!-- skill-audit-report -->
## Skill Audit Results

### ❌ auto-deploy-helper — BLOCKED
...

### ⚠️ api-connector — PASSED with warnings
...

### ✅ markdown-formatter — PASSED
...
```

### 9.3 최종 판정

| 조건 | PR 결과 |
|------|---------|
| 하나라도 BLOCKED (CRITICAL 존재) | ❌ failure |
| BLOCKED 없음, WARNING 존재 | ✅ success (warnings 표시) |
| 전체 PASSED | ✅ success |

---

## 10. 출력 형식

### 10.1 Markdown 보고서

```markdown
# Skill Security Audit Report

**Skill**: my-skill-name
**Submitted by**: @username
**Audit date**: 2026-03-27
**Ruleset version**: v1.5.0 (sha: a1b2c3d)
**Phase**: 1
**Verdict**: ❌ BLOCKED

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | 1     |
| HIGH     | 1     |
| MEDIUM   | 2     |

## Findings

### 🔴 CRITICAL — 차단 사유

#### [SEC-010] Hardcoded API key
- **File**: references/config.md:12
- **Evidence**: `sk-...`
- **Fix**: 자격증명을 파일에 직접 포함하지 마세요.
  환경변수 또는 시크릿 매니저를 사용하세요.

### 🟠 HIGH — 반드시 확인 (차단하지 않음)

#### [DST-003] git push --force
- **File**: scripts/deploy.sh:8
- **Note**: 원격 히스토리를 파괴할 수 있습니다.
  의도된 동작이면 audit-ignore로 예외 선언하세요.

### 🟡 MEDIUM — 참고 사항

#### [SEC-020] External HTTP request
- **File**: SKILL.md:30
- **Note**: 외부 HTTP 요청이 감지되었습니다. 의도된 동작이면 무시해도 됩니다.
```

### 10.2 Machine-readable JSON

보고서와 별도로 `audit-result.json`을 Actions 아티팩트로 저장.

```json
{
  "version": "1.5.0",
  "phase": 1,
  "ruleset_sha": "a1b2c3d",
  "skill": "my-skill-name",
  "verdict": "BLOCKED",
  "counts": { "critical": 1, "high": 1, "medium": 2, "low": 0, "exempted": 0 },
  "findings": [
    {
      "id": "SEC-010",
      "severity": "CRITICAL",
      "file": "references/config.md",
      "line": 12,
      "message": "Hardcoded API key",
      "fix": "환경변수 또는 시크릿 매니저를 사용하세요."
    },
    {
      "id": "DST-003",
      "severity": "HIGH",
      "file": "scripts/deploy.sh",
      "line": 8,
      "message": "git push --force",
      "fix": "의도된 동작이면 audit-ignore로 예외 선언하세요."
    }
  ]
}
```

---

## 11. Slack 알림

| 항목 | 값 |
|------|-----|
| **방식** | Incoming Webhook |
| **secret** | `SLACK_WEBHOOK_URL` |
| **실패 시** | `continue-on-error: true` |
| **v1 전송 정책** | BLOCKED만 즉시 전송. PASSED/WARNING은 전송하지 않음 |

**BLOCKED 알림:**

```
❌ Skill Audit BLOCKED — "data-sync"
제출자: @username | PR: #42

CRITICAL (1):
• [SEC-010] references/config.md:12 — Hardcoded API key

HIGH (1) — 참고:
• [DST-003] scripts/deploy.sh:8 — git push --force

→ https://github.com/org/marketplace/pull/42
```

---

## 12. GitHub Actions 통합

### 12.1 트리거

```yaml
on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - 'skills/**'
  workflow_dispatch:
    inputs:
      skill_path:
        description: '검사할 스킬 디렉토리 경로'
        required: true
```

### 12.2 워크플로우 구조

```
Job: skill-audit
├── Step 1: Checkout
├── Step 2: Detect changed skill directories
├── Step 3: Run skill-security-audit — 스킬별 병렬 실행
├── Step 4: Update PR comment (기존 봇 댓글 edit, 없으면 신규)
├── Step 5: Set Check Run per skill (CRITICAL 존재 시 failure)
└── Step 6: Slack notification + D-30 만료 경고 (continue-on-error: true)
```

---

## 13. 룰셋 버전 정책

- `ruleset-version.txt`로 버전 고정. 보고서에 버전 + SHA 기록.
- 로컬 실행과 CI 모두 동일 버전 사용.

| 변경 유형 | 재검사 여부 | 처리 |
|-----------|:-----------:|------|
| 신규 CRITICAL 규칙 추가 | ✅ 필요 | 배포 후 30일 내 기존 스킬 재검사. 실패 시 `REVIEW_REQUIRED` 표시 |
| HIGH/MEDIUM/LOW 추가·변경 | 권장 | 다음 PR 제출 시 자동 적용 |
| 심각도 하향 | ❌ 불필요 | — |
| 버그 픽스·문구 변경 | ❌ 불필요 | — |

---

## 14. 프로젝트 구조

### 14.1 스킬 패키지

```
skills/skill-security-audit/
├── SKILL.md
├── ruleset-version.txt
├── references/
│   ├── security-checklist.md
│   ├── destructive-ops-checklist.md
│   └── quality-checklist.md
├── assets/
│   ├── report-template.md
│   └── slack-message-template.json
└── README.md
```

### 14.2 마켓플레이스 운영 인프라 (별도 관리)

```
marketplace-repo/
├── .github/workflows/skill-audit.yml   # GitHub Action
└── config/approved-reviewers.yml       # audit-ignore 승인자 목록
```

> 스킬 제출자는 `.github/`, `config/` 디렉토리에 접근할 수 없다.

---

## 15. 스킬 메타데이터

```yaml
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
    - reviewer        # 주 호출 경로: PR/CI 컨텍스트 게이트키퍼
    - user-invocable  # 보조 호출 경로: 제출자 로컬 사전 검사
  return_behavior: report
---
```

---

## 16. 통과/실패 예시

### 16.1 PASSED

```
skills/markdown-formatter/
├── SKILL.md   # frontmatter 완전, 300줄
└── references/rules.md

→ ✅ PASSED (0 findings)
```

### 16.2 BLOCKED

```
skills/data-sync/
├── SKILL.md          # curl $URL | bash 포함
└── references/config.md   # sk-xxxxxxxx 하드코딩

→ ❌ BLOCKED
  SEC-003  CRITICAL  SKILL.md:45              curl pipe to bash
  SEC-010  CRITICAL  references/config.md:12  Hardcoded API key
```

### 16.3 PASSED with warnings

```
skills/deploy-helper/
├── SKILL.md          # WebFetch + audit-ignore: DST-003 (승인됨)
└── scripts/deploy.sh # git push --force (audit-ignore로 예외)

→ ⚠️ PASSED with warnings
  DST-003   EXEMPTED  (reviewer: @admin, exp: 2026-09-30)
  SBX-001   HIGH      SKILL.md:18  Write outside skill directory
  SEC-020   MEDIUM    SKILL.md:30  External HTTP request
```

---

## 17. v1 구현 로드맵

| 항목 | 설명 | 우선순위 |
|------|------|---------|
| Self-Audit CI 테스트 | ruleset 배포 전 FALSE POSITIVE 자동 검증 | P1 |
| PR comment 봇 (업데이트 방식) | 댓글 스팸 방지, `<!-- skill-audit-report -->` 식별 | P1 |
| D-30 만료 경고 | Slack 자동 경고 | P2 |
| `audit-result.json` 아티팩트 | Actions artifact 저장 | P2 |

---

## 18. Phase 2 이관 항목

> Phase 1에서 의도적으로 제외한 항목들이다. 조직 내 AI 활용이 충분히 확산된 후, 상용 마켓플레이스 환경에 맞춰 별도 스펙으로 구현한다.

### 18.1 강화될 검사 규칙

| 항목 | Phase 1 | Phase 2 방향 |
|------|---------|-------------|
| `git clone`, `git fetch` | 미탐지 | MEDIUM 또는 HIGH |
| `npm install`, `pip install` 외부 패키지 설치 | 미탐지 | MEDIUM |
| `python -c`, `node -e` 인라인 실행 | 미탐지 | HIGH (다른 난독화 패턴 결합 시) |
| `kill`, `pkill` 프로세스 종료 | 미탐지 | MEDIUM |
| 절대경로 사용 (`/etc/`, `/usr/`) | 미탐지 | HIGH (allowlist 기반) |
| 경로 정규화 우회 (`../`, URL 인코딩) | 미탐지 | HIGH |
| 심볼릭 링크 탈출 (`ln -s`) | 미탐지 | CRITICAL |
| `ssh`, `nc` 직접 네트워크 도구 | HIGH (경고) | CRITICAL (차단) |
| `SBX-001` 스킬 외부 Write | HIGH (경고) | CRITICAL (차단) |
| 민감 파일 overwrite (`> .env`, `> *.pem`) | 미탐지 | HIGH |
| DNS 기반 데이터 유출 | 미탐지 | HIGH |
| 문자열 결합·이스케이프 명령 은닉 | 미탐지 | HIGH |
| JWT 토큰 완전한 패턴 탐지 | 미탐지 | CRITICAL (SEC-010 강화) |
| 외부 URL 링크 대상 본문 검사 | 미탐지 | 동적 fetch 후 검사 |
| HIGH → 차단 여부 재검토 | 경고만 | 상용 환경에서 BLOCKED 전환 검토 |

### 18.2 고도화될 운영 기능

| 항목 | Phase 1 | Phase 2 방향 |
|------|---------|-------------|
| 오탐 이의제기 | 수동 라벨 + 관리자 검토 | 봇 명령, DISPUTED/DISMISSED 상태 머신 자동화 |
| audit-ignore 승인 | `approved-reviewers.yml` 대조 | 서명된 승인 레코드, Policy-as-data (Rego) |
| 보고서 형식 | Markdown + JSON | SARIF, GitHub Code Scanning UI 연동 |
| 분석 엔진 | 패턴 + LLM 문맥 분류 | strace/sysdig 기반 동적 샌드박스 |
| Slack 알림 | BLOCKED만 즉시 전송 | PASSED 일일 요약 집계 |
| QUA 품질 기준 | 최소 구조 6개 규칙 | 토큰 기반 지표, Examples/Troubleshooting 필수화 |
| Finding 그룹핑 | 개별 나열 | 공격 체인 단위 그룹 표시 |
| 보안 점수/배지 | 없음 | 100점 기반 점수 + README 배지 SVG |
| allowlist 외부화 | 코드 내 하드코딩 | `allowlist.yml` 환경별 오버라이드 |
