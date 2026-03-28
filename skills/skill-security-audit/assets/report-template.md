# Skill Security Audit Report

**Skill**: {{skill_name}}
**Submitted by**: {{submitted_by}}
**Audit date**: {{audit_date}}
**Ruleset version**: v{{ruleset_version}} (sha: {{ruleset_sha}})
**Phase**: 1
**Verdict**: {{verdict_emoji}} {{verdict}}

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | {{critical_count}} |
| HIGH     | {{high_count}} |
| MEDIUM   | {{medium_count}} |
| EXEMPTED | {{exempted_count}} |

## Findings

{{#if has_critical}}
### 🔴 CRITICAL — 차단 사유

{{#each critical_findings}}
#### [{{id}}] {{title}}
- **File**: {{file}}:{{line}}
- **Evidence**: `{{evidence}}`
- **Fix**: {{fix}}

{{/each}}
{{/if}}

{{#if has_high}}
### 🟠 HIGH — 반드시 확인 (차단하지 않음)

{{#each high_findings}}
#### [{{id}}] {{title}}
- **File**: {{file}}:{{line}}
- **Note**: {{fix}}

{{/each}}
{{/if}}

{{#if has_medium}}
### 🟡 MEDIUM — 참고 사항

{{#each medium_findings}}
#### [{{id}}] {{title}}
- **File**: {{file}}:{{line}}
- **Note**: {{fix}}

{{/each}}
{{/if}}

{{#if has_exempted}}
### ⬜ EXEMPTED — 예외 처리됨

{{#each exempted_findings}}
#### [{{id}}] {{title}}
- **Status**: EXEMPTED (reviewer: {{reviewer}}, expires: {{expires}})
- **Reason**: {{reason}}

{{/each}}
{{/if}}

---

*Ruleset v{{ruleset_version}} · Phase 1 · {{slack_status}}*
