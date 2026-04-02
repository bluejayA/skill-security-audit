# Local Verification Guide

Run skill-security-audit locally to catch issues before submitting a PR.

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- Anthropic API key configured (`claude login` or `ANTHROPIC_API_KEY` env var)

## Install the Plugin

```bash
claude plugins install https://github.com/bluejayA/skill-security-audit.git
```

## Run an Audit

### Basic usage

```bash
# Audit a skill directory
claude "skill-security-audit 스킬로 ./skills/my-skill 을 검사해줘"

# Or in English
claude "Use skill-security-audit to audit ./skills/my-skill"
```

### With specific options

```bash
# Include PR submitter info
claude "skill-security-audit 스킬로 ./skills/my-skill 을 검사해줘. PR 제출자: @my-username"

# Request JSON output
claude "skill-security-audit 스킬로 ./skills/my-skill 을 검사해줘. JSON 결과도 함께 출력해줘."
```

### Non-interactive mode

```bash
claude --print \
  "skill-security-audit 스킬로 ./skills/my-skill 을 검사해줘. JSON 결과도 함께 출력해줘."
```

## Understanding Results

### Verdict

| Verdict | Meaning | Action |
|---------|---------|--------|
| **PASSED** | No findings | Safe to submit |
| **PASSED with warnings** | HIGH/MEDIUM findings | Review warnings, submit when ready |
| **BLOCKED** | 1+ CRITICAL findings | Must fix all CRITICAL issues before submitting |

### Report sections

1. **Target files** — List of files scanned
2. **Findings** — Grouped by severity (CRITICAL > HIGH > MEDIUM)
3. **Each finding includes**:
   - Rule ID (e.g., SEC-010)
   - File path and line number
   - Evidence snippet
   - OWASP AST reference
   - Fix suggestion

## Example: Fixing a BLOCKED Result

```
❌ BLOCKED — 2 CRITICAL findings

[SEC-010] Hardcoded API Key
  File: SKILL.md:15
  Evidence: sk-ant-api03-xxxxx
  Fix: Remove the key and use environment variables instead.

[DST-001] Recursive Delete
  File: SKILL.md:22
  Evidence: rm -rf $DIR
  Fix: Use targeted file removal instead of recursive delete.
```

**Steps:**
1. Remove the hardcoded key from line 15
2. Replace `rm -rf` with specific file deletion on line 22
3. Re-run the audit to verify

## Using `audit-ignore`

If a finding is a false positive, add an `audit-ignore` block to your SKILL.md frontmatter:

```yaml
---
name: my-skill
description: "..."
audit-ignore:
  - rule: SEC-010
    reason: "Example key in documentation, not a real credential"
    reviewer: "@marketplace-maintainer"
---
```

**Note:** The `reviewer` must be listed in the marketplace's `approved-reviewers.yml`. Otherwise the exception is ignored.

## CI Integration

For automated auditing on every PR, see the [CI Integration Guide](ci-integration-guide.md).

---

## 한국어 요약

PR 제출 전에 로컬에서 skill-security-audit을 실행하여 보안 문제를 사전에 잡을 수 있습니다.

```bash
# 플러그인 설치
claude plugins install https://github.com/bluejayA/skill-security-audit.git

# 감사 실행
claude "skill-security-audit 스킬로 ./skills/my-skill 을 검사해줘"
```

- **PASSED** → 제출 가능
- **PASSED with warnings** → 경고 확인 후 제출
- **BLOCKED** → CRITICAL 수정 필수
