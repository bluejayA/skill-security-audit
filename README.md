# skill-security-audit

제3자가 Claude Code 마켓플레이스에 제출한 스킬을 자격증명 보호, 시스템 안전, 최소 구조 품질 기준으로 검사하는 독립 스킬.

**Phase 1** — 조직 내부 AI 확산 환경 대상. "즉각적 실질 피해가 발생하는 것만 차단"하는 게이트키퍼.

## 핵심 원칙

- **Adoption-First**: CRITICAL만 차단. 좋은 스킬이 불필요하게 막히지 않는다
- **Deterministic-First**: LLM은 Markdown 문맥 분류에만 사용. 최종 판정은 규칙 기반
- **Actionable Feedback**: 차단 시 규칙 ID + 파일:라인 + 수정 예시 제공

## 사용법

### 로컬 실행 (제출자 사전 검사)

```bash
claude "제출 전에 스킬을 검사해줘: /path/to/my-skill"
```

CI와 동일한 ruleset, 동일한 판정 기준이 적용된다.

### GitHub Actions (자동)

PR에서 `skills/**` 경로가 변경되면 자동으로 트리거된다.

```yaml
# .github/workflows/skill-audit.yml
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

**필요한 Secrets:**
- `ANTHROPIC_API_KEY` — Claude API 키 (필수)
- `SLACK_WEBHOOK_URL` — Slack Incoming Webhook URL (선택)

## 판정 기준

```
CRITICAL 1개 이상  → ❌ BLOCKED (PR failure)
HIGH만 존재        → ⚠️ PASSED with warnings
MEDIUM만 존재      → ⚠️ PASSED with warnings
발견 없음          → ✅ PASSED
```

Phase 1에서는 **CRITICAL만 차단**한다. HIGH는 경고이지 차단 사유가 아니다.

## Phase 1 검사 규칙 (22개)

### CRITICAL (11개) — 차단

| ID | 검사 항목 |
|----|-----------|
| SEC-010 | 하드코딩된 API 키 (`ghp_`, `sk-`, `AKIA` 등) |
| SEC-011 | 프라이빗 키 (`BEGIN PRIVATE KEY`) |
| SEC-013 | `process.env` 전체 덤프/외부 전송 |
| SEC-003 | 파이프를 통한 원격 실행 (`curl \| bash`) |
| SEC-030 | Base64 디코딩 후 실행 |
| SEC-001 | 셸 인젝션 (untrusted input + 셸 실행 경로) |
| SBX-003 | 경로 탈출 (`../../etc/passwd`) |
| SBX-004 | 홈 디렉토리 민감 경로 참조 (`~/.ssh`, `~/.aws`) |
| SBX-007 | 키체인/히스토리 읽기 |
| DST-001 | 재귀적 삭제 (`rm -rf`) |
| DST-007 | 시스템 수준 권한 변경 (`chmod 777`, `sudo`) |
| QUA-001 | SKILL.md 파일 존재 |

### HIGH (6개) — 경고

| ID | 검사 항목 |
|----|-----------|
| SEC-001 | 셸 인젝션 (trusted input / 단순 변수) |
| SEC-002 | 동적 코드 실행 (`eval`, `exec`) |
| SEC-020H | 외부 HTTP + 민감 데이터 페이로드 |
| SEC-022 | 직접 네트워크 도구 (`ssh`, `nc`) |
| SBX-001 | 스킬 디렉토리 외부 파일 쓰기 |
| DST-003 | `git push --force` |
| QUA-002 | frontmatter 필수 필드 누락 |

### MEDIUM (5개) — 참고

SEC-012, SEC-020, DST-002, QUA-003~006, QUA-010, QUA-011

## 통과/실패 예시

### PASSED

```
skills/markdown-formatter/
├── SKILL.md   # frontmatter 완전, 300줄
└── references/rules.md

→ ✅ PASSED (0 findings)
```

### BLOCKED

```
skills/data-sync/
├── SKILL.md          # curl $URL | bash 포함
└── references/config.md   # sk-xxxxxxxx 하드코딩

→ ❌ BLOCKED
  SEC-003  CRITICAL  SKILL.md:45              curl pipe to bash
  SEC-010  CRITICAL  references/config.md:12  Hardcoded API key
```

### PASSED with warnings

```
skills/deploy-helper/
├── SKILL.md          # WebFetch + audit-ignore: DST-003 (승인됨)
└── scripts/deploy.sh # git push --force (audit-ignore로 예외)

→ ⚠️ PASSED with warnings
  DST-003   EXEMPTED  (reviewer: @admin, exp: 2026-09-30)
  SBX-001   HIGH      SKILL.md:18  Write outside skill directory
  SEC-020   MEDIUM    SKILL.md:30  External HTTP request
```

## 예외 처리 (audit-ignore)

SKILL.md frontmatter에 예외를 선언할 수 있다:

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

**필수 필드**: `rule`, `reason`, `reviewer`, `expires` — 하나라도 누락 시 예외 무효.

`reviewer`는 `config/approved-reviewers.yml` 목록과 대조하여 검증한다.

## 프로젝트 구조

```
skill-security-audit/
├── .claude-plugin/
│   └── plugin.json             # 플러그인 메타데이터
├── skills/
│   └── skill-security-audit/
│       ├── SKILL.md                    # 메인 스킬 (검사 워크플로우)
│       ├── ruleset-version.txt         # 룰셋 버전 고정
│       ├── references/
│       │   ├── security-checklist.md   # SEC-*, SBX-* 규칙
│       │   ├── destructive-ops-checklist.md  # DST-* 규칙
│       │   └── quality-checklist.md    # QUA-* 규칙
│       ├── assets/
│       │   ├── report-template.md      # Markdown 보고서 템플릿
│       │   └── slack-message-template.json  # Slack Block Kit 템플릿
│       └── config/
│           └── approved-reviewers.yml  # audit-ignore 승인자 목록
├── .github/workflows/
│   └── skill-audit.yml         # GitHub Actions 워크플로우
└── README.md
```

## 제출자 가이드

1. **로컬에서 먼저 검사**: `claude "제출 전에 스킬을 검사해줘: ./my-skill"` 실행
2. **CRITICAL 발견 시**: 보고서의 수정 제안을 따라 수정 후 재검사
3. **의도된 동작이 차단된 경우**: `audit-ignore`로 예외 선언 (관리자 승인 필요)
4. **오탐이라고 판단되면**: PR에 `audit-review-requested` 라벨을 추가하고 사유를 comment로 작성

## 라이선스

MIT
