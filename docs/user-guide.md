# User Guide

이 문서는 skill-security-audit의 설치부터 사용, CI 설정, 예외 처리, 트러블슈팅까지를 다룬다.

---

## 목차

1. [사전 요구사항](#1-사전-요구사항)
2. [설치](#2-설치)
3. [로컬 실행 (제출자 사전 검사)](#3-로컬-실행)
4. [보고서 읽는 법](#4-보고서-읽는-법)
5. [GitHub Actions CI 설정](#5-github-actions-ci-설정)
6. [예외 처리 (audit-ignore)](#6-예외-처리-audit-ignore)
7. [멀티 스킬 PR](#7-멀티-스킬-pr)
8. [Slack 알림 설정](#8-slack-알림-설정)
9. [트러블슈팅](#9-트러블슈팅)
10. [FAQ](#10-faq)

---

## 1. 사전 요구사항

- **Claude Code CLI** — `npm install -g @anthropic-ai/claude-code`
- **Anthropic API Key** — `ANTHROPIC_API_KEY` 환경변수 설정
- 검사 대상 스킬 디렉토리 (최소한 `SKILL.md` 파일 포함)

---

## 2. 설치

### 로컬 설치

skill-security-audit을 로컬에 클론한다:

```bash
git clone https://github.com/bluejayA/skill-security-audit.git
```

이 스킬은 독립 실행형이다. 별도 의존성 설치가 필요 없으며, Claude Code가 SKILL.md를 직접 읽어 실행한다.

### 조직 저장소에 통합

마켓플레이스 저장소의 `skills/` 디렉토리에 포함시켜 CI에서 자동 실행되도록 설정한다:

```
your-marketplace-repo/
├── skills/
│   ├── skill-security-audit/    ← 이 스킬
│   ├── submitted-skill-a/
│   └── submitted-skill-b/
└── .github/workflows/
    └── skill-audit.yml          ← CI 워크플로우
```

---

## 3. 로컬 실행

### 기본 사용법

```bash
claude "제출 전에 스킬을 검사해줘: /path/to/my-skill"
```

또는 영어로:

```bash
claude "Run a security audit on this skill: /path/to/my-skill"
```

### 실행 흐름

1. Claude Code가 skill-security-audit의 SKILL.md를 로드한다
2. 대상 스킬 디렉토리를 스캔하여 검사 대상 파일을 수집한다
3. 파일 유형별로 분석 전략을 적용한다
4. 22개 Phase 1 규칙을 적용하여 findings를 수집한다
5. 최종 판정 (PASSED / PASSED with warnings / BLOCKED)을 내린다
6. Markdown 보고서 + JSON 결과를 출력한다

### 검사 대상 파일

| 대상 | 검사 내용 |
|------|-----------|
| `SKILL.md` | 구조, 품질, 보안 패턴 |
| `references/**`, `assets/**` | 보안 패턴 |
| `scripts/**`, `*.sh`, `*.py`, `*.js`, `*.ts` | 보안 + 파괴적 동작 |
| `package.json` (scripts 섹션) | 명령어 검사 |
| `Makefile` | 타겟 내 명령어 검사 |

**제외 대상**: `node_modules/`, `.git/`, 바이너리 파일, 외부 URL 링크 대상

### 로컬 실행 시 제한사항

- `SLACK_WEBHOOK_URL` 미설정 → Slack 알림 skip (보고서에 표기됨)
- Ruleset SHA → `(local)` 표시 (git 저장소 밖에서 실행 시)
- `config/approved-reviewers.yml` 미발견 → 모든 audit-ignore 예외 무효

---

## 4. 보고서 읽는 법

### 보고서 구조

```markdown
# Skill Security Audit Report

**Skill**: my-skill
**Verdict**: ❌ BLOCKED

## Summary
| Severity | Count |
|----------|-------|
| CRITICAL | 2     |
| HIGH     | 1     |
| MEDIUM   | 0     |

## Findings

### 🔴 CRITICAL — 차단 사유
#### [SEC-010] 하드코딩된 API 키
- **File**: SKILL.md:15
- **Evidence**: `sk-proj-abc123...`
- **Fix**: 자격증명을 파일에 직접 포함하지 마세요. 환경변수를 사용하세요.

### 🟠 HIGH — 반드시 확인
#### [SEC-002] 동적 코드 실행
- **File**: SKILL.md:42
- **Note**: eval() 사용이 감지되었습니다. 정적 함수 호출로 대체하세요.
```

### 판정 의미

| 판정 | 의미 | 해야 할 일 |
|------|------|-----------|
| ✅ **PASSED** | 문제 없음 | 그대로 제출 |
| ⚠️ **PASSED with warnings** | 경고 있지만 통과 | 경고 내용 확인 후 판단. 의도된 동작이면 무시 가능 |
| ❌ **BLOCKED** | CRITICAL 발견 | 수정 제안을 따라 수정 후 재검사 |

### JSON 출력

보고서와 함께 machine-readable JSON도 출력된다:

```json
{
  "version": "1.0.0",
  "phase": 1,
  "skill": "my-skill",
  "verdict": "BLOCKED",
  "counts": { "critical": 2, "high": 1, "medium": 0, "exempted": 0 },
  "findings": [...]
}
```

---

## 5. GitHub Actions CI 설정

### 워크플로우 설치

`.github/workflows/skill-audit.yml`을 마켓플레이스 저장소에 복사한다.

### 필요한 Secrets

| Secret | 필수 | 용도 |
|--------|------|------|
| `ANTHROPIC_API_KEY` | **필수** | Claude API 호출 |
| `SLACK_WEBHOOK_URL` | 선택 | BLOCKED 시 Slack 알림 |

GitHub 저장소 Settings → Secrets and variables → Actions에서 설정한다.

### 필요한 Permissions

워크플로우에 다음 권한이 필요하다:

```yaml
permissions:
  contents: read
  pull-requests: write   # PR comment 업데이트
  checks: write          # Check Run 결과 설정
```

### 트리거 조건

```yaml
on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - 'skills/**'           # skills/ 하위 변경 시 자동 트리거
  workflow_dispatch:
    inputs:
      skill_path:
        description: '검사할 스킬 디렉토리 경로'
        required: true         # 수동 실행 지원
```

### CI 동작 흐름

```
1. PR에서 skills/** 변경 감지
2. 변경된 스킬 디렉토리 식별
3. 스킬별 독립 검사 실행
4. PR comment에 보고서 업데이트 (<!-- skill-audit-report --> 마커)
5. Check Run 결과 설정 (BLOCKED → failure, 그 외 → success)
6. BLOCKED 시 Slack 알림 전송 (선택, continue-on-error)
7. Self-Audit 테스트 실행 (skill-security-audit 자기 검사)
```

### PR Comment 업데이트 전략

- 기존 봇 댓글이 있으면 (`<!-- skill-audit-report -->` 마커로 식별) 업데이트
- 없으면 새 댓글 생성
- PR이 업데이트될 때마다 최신 결과로 갱신

---

## 6. 예외 처리 (audit-ignore)

특정 규칙이 의도된 동작을 차단하는 경우, SKILL.md frontmatter에 예외를 선언할 수 있다.

### 선언 방법

```yaml
---
name: my-deployment-skill
description: "Use when deploying to production..."
audit-ignore:
  - rule: DST-003
    reason: "배포 파이프라인 스킬 — git push --force가 핵심 기능"
    reviewer: "@admin"
    expires: "2026-09-30"
  - rule: SEC-020
    reason: "외부 API 호출이 스킬의 핵심 기능"
    reviewer: "@marketplace-maintainer"
    expires: "2026-12-31"
---
```

### 필수 필드

| 필드 | 설명 | 누락 시 |
|------|------|---------|
| `rule` | 예외 대상 규칙 ID (예: DST-003) | 예외 무효 |
| `reason` | 예외가 필요한 이유 | 예외 무효 |
| `reviewer` | 예외를 승인한 관리자 (예: @admin) | 예외 무효 |
| `expires` | 만료일 (YYYY-MM-DD) | 예외 무효 |

**4개 필드 모두 필수**. 하나라도 누락되면 예외가 무효 처리되고 원래 심각도로 보고된다.

### 승인자 검증

`reviewer` 값은 `config/approved-reviewers.yml` 목록과 대조하여 검증한다:

```yaml
# config/approved-reviewers.yml
approved_reviewers:
  - "@admin"
  - "@marketplace-maintainer"
```

- reviewer가 목록에 없으면 → 해당 예외만 무효
- `approved-reviewers.yml` 파일 자체가 없으면 → 모든 예외 무효

### 만료 정책

- 만료일이 지난 예외 → 무효, 원래 심각도로 보고
- 만료 30일 이내 → Slack 만료 경고 전송 대상

### 보고서에서의 표시

유효한 예외는 숨기지 않고 **EXEMPTED**로 표시된다:

```
### ⬜ EXEMPTED — 예외 처리됨
#### [DST-003] git push --force
- Status: EXEMPTED (reviewer: @admin, expires: 2026-09-30)
- Reason: 배포 파이프라인 스킬 — git push --force가 핵심 기능
```

---

## 7. 멀티 스킬 PR

하나의 PR에서 여러 스킬이 변경된 경우:

1. 변경된 파일이 속한 스킬 디렉토리를 자동 식별한다
2. 각 스킬을 **독립적으로** 검사한다 (규칙 적용, 판정 모두 개별)
3. 결과를 하나의 보고서에 스킬별 섹션으로 통합한다
4. **하나라도 BLOCKED → 전체 PR failure**

```
PR #42: skills/data-sync/ + skills/api-connector/ 변경

→ data-sync:     ❌ BLOCKED (SEC-010)
→ api-connector:  ✅ PASSED

→ 전체 판정: ❌ BLOCKED
```

---

## 8. Slack 알림 설정

### 설정 방법

GitHub Secrets에 `SLACK_WEBHOOK_URL`을 추가한다. Slack 워크스페이스에서 Incoming Webhook을 생성하여 URL을 얻는다.

### 전송 정책

| 상황 | 전송 여부 |
|------|-----------|
| ❌ BLOCKED | **즉시 전송** |
| ⚠️ PASSED with warnings | 전송하지 않음 |
| ✅ PASSED | 전송하지 않음 |
| audit-ignore 만료 D-30 | **경고 전송** |

### 실패 처리

Slack 전송은 `continue-on-error: true`로 설정되어 있다. 전송에 실패해도 검사 결과에 영향을 주지 않는다 (Fail-safe 원칙).

---

## 9. 트러블슈팅

### BLOCKED인데 오탐 같은 경우

1. 보고서의 규칙 ID와 증거를 확인한다
2. 해당 구문이 실제로 실행 지시인지, 설명 텍스트인지 판단한다
3. **설명 텍스트가 오탐된 경우**: PR에 `audit-review-requested` 라벨을 추가하고 사유를 comment로 작성한다
4. **의도된 동작인 경우**: `audit-ignore`로 예외를 선언한다 (관리자 승인 필요)

### approved-reviewers.yml 파일이 없는 경우

- 로컬 실행 시 이 파일이 없으면 모든 audit-ignore 예외가 무효로 처리된다
- 보고서에 `(approved-reviewers.yml 미발견 — 모든 예외 무효)` 표기
- CI 환경에서는 마켓플레이스 저장소에 이 파일이 포함되어 있어야 한다

### references/ 체크리스트 파일이 없는 경우

- 해당 카테고리 규칙 검사를 skip하고 보고서에 표기
- 다른 카테고리는 정상 실행. 전체 검사를 실패시키지 않는다

### Self-Audit에서 오탐이 발생하는 경우

- references/ 파일 상단에 면책 문구가 있는지 확인한다:
  > "이 파일은 SKILL.md에서 참조하는 규칙 정의 문서이며, 여기에 포함된 패턴 예시는 **설명 텍스트**이다."
- 문구가 누락되었으면 추가 후 재검사한다

### 대규모 스킬 (파일 50개 이상)

토큰 한계에 근접하는 경우:
- SKILL.md와 구조화 코드(`*.sh`, `*.py`, `*.js`, `*.ts`)를 우선 검사
- references/ 파일은 보안 패턴 중심으로 스캔
- 검사 범위가 제한되면 보고서에 표기

---

## 10. FAQ

### Q: HIGH 경고를 받았는데 무시해도 되나요?

Phase 1에서 HIGH는 차단 사유가 아니다. 의도된 동작이라면 무시해도 된다. 다만 보고서에는 계속 표시되므로, 반복적으로 같은 경고를 받는다면 `audit-ignore`로 예외 선언하는 것을 권장한다.

### Q: 로컬과 CI에서 결과가 다를 수 있나요?

동일한 ruleset 버전을 사용하므로 규칙 적용은 동일하다. 다만 LLM이 Markdown 문맥 분류에 관여하므로, 애매한 구문에 대한 판단이 약간 다를 수 있다. 애매한 경우에는 항상 MEDIUM으로 처리되므로 판정 결과(BLOCKED 여부)에는 영향이 없다.

### Q: 새 규칙을 추가하고 싶으면 어떻게 하나요?

1. `references/` 디렉토리의 해당 체크리스트 파일에 규칙을 추가한다
2. SKILL.md의 규칙 요약 테이블을 업데이트한다
3. `ruleset-version.txt`의 버전을 올린다
4. Self-Audit 테스트를 실행하여 오탐이 없는지 확인한다

### Q: Phase 2는 언제 나오나요?

Phase 1이 조직 내부에서 충분히 안정화된 후 Phase 2를 별도 문서로 정의할 예정이다. Phase 2에서는 강화된 규칙셋, SARIF 출력, 동적 샌드박스 등이 추가된다. 상세는 `docs/general-spec.md`의 Phase 2 항목을 참조.

### Q: 이 스킬 자체의 보안은 어떻게 보장되나요?

- CI에 Self-Audit 테스트가 포함되어 있어, 자기 자신에 대한 오탐 0건이 검증된다
- 스킬 개발 시 5개 대상(실제 스킬 2개 + 합성 테스트 3개)에 대한 Evaluation을 통과했다
- 규칙 변경 시마다 Self-Audit + Evaluation 재실행을 권장한다
