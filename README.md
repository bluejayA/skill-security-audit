# skill-security-audit

Automated security gatekeeper for Claude Code skills — scans third-party skills for credential leaks, destructive operations, metadata manipulation, and quality issues before marketplace registration.

> **v2.0** — 35 rules based on OWASP Agentic Skills Top 10 (AST10). Blocks only immediate, real-world threats while keeping the barrier low for good skills.

## Principles

| Principle | Description |
|-----------|-------------|
| **Adoption-First** | Only CRITICAL findings block. HIGH/MEDIUM are warnings |
| **Deterministic-First** | LLM only classifies Markdown context. All verdicts are rule-based |
| **Actionable Feedback** | Every blocking finding includes rule ID, file:line, evidence, and fix suggestion |

## Verdict Logic

```
1+ CRITICAL finding  → ❌ BLOCKED    (PR check failure)
HIGH/MEDIUM only     → ⚠️ PASSED with warnings
No findings          → ✅ PASSED
```

## Quick Start

### Local Verification

Audit your skill before submitting to the marketplace:

```bash
# Install the plugin
claude plugins install https://github.com/bluejayA/skill-security-audit.git

# Run audit on your skill directory
claude "skill-security-audit 스킬로 /path/to/my-skill 을 검사해줘"
```

### CI Integration

Add automated auditing to your plugin repository or marketplace. See the guides below for detailed setup.

## Guides

| Guide | Description |
|-------|-------------|
| [Local Verification Guide](docs/local-verification-guide.md) | Run audits locally with Claude CLI before PR submission |
| [CI Integration Guide](docs/ci-integration-guide.md) | Add GitHub Actions workflows for automated PR auditing |
| [User Guide](docs/user-guide.md) | Installation, audit-ignore syntax, troubleshooting |
| [Integration Guide](docs/integration-guide.md) | Integrate into existing marketplace repos (submodule setup) |

## Rules (35 total)

### CRITICAL (17) — Blocks submission

| Category | Rules |
|----------|-------|
| Credentials | SEC-010 hardcoded API keys, SEC-011 private keys, SEC-013 env dump + exfil |
| Remote exec | SEC-003 curl\|bash, SEC-030 base64\|bash |
| Shell injection | SEC-001 untrusted input + shell exec |
| Sensitive paths | SBX-003 path traversal, SBX-004 ~/.ssh etc, SBX-007 keychain/history |
| Destructive | DST-001 rm -rf, DST-007 sudo/chmod 777 |
| Quality | QUA-001 SKILL.md existence |
| Metadata | META-001 identity file writes, META-002 zero-width unicode, META-003 base64 payloads |
| Code safety | SEC-040 unsafe YAML loaders, SEC-041 dangerous code execution |

### HIGH (10) — Warnings only

SEC-001(trusted), SEC-002 eval/exec, SEC-020H HTTP+sensitive, SEC-022 network tools, SBX-001 external writes, SBX-010 unrestricted shell, SBX-011 binary network, SBX-012 wildcard globs, DST-003 force push, QUA-002 required fields

### MEDIUM (8) — Informational

SEC-012 sensitive config refs, SEC-020 HTTP requests, DST-002 single delete, QUA-003~006 quality, QUA-010~011 ambiguous expressions, SCH-001~005 spec compliance

Full rule definitions in `skills/skill-security-audit/references/`.

### OWASP AST10 Mapping

| OWASP | Item | Rules |
|-------|------|-------|
| AST01 1.4 | Malicious patterns | SEC-041 |
| AST01 1.6 | Identity file protection | META-001 |
| AST03 3.3 | Shell access restriction | SBX-010 |
| AST03 3.4 | File path scoping | SBX-012 |
| AST03 3.7 | Network domain allowlist | SBX-011 |
| AST04 4.1 | Description accuracy | SCH-002 |
| AST04 4.2 | Steganography/encoding detection | META-002, META-003 |
| AST05 5.1 | Safe YAML loaders | SEC-040 |
| AST05 5.3 | Allowed field list | SCH-003 |
| AST10 10.6 | Universal skill format | SCH-001, SCH-005 |

## Project Structure

```
skill-security-audit/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── skill-security-audit/
│       ├── SKILL.md                     # Main audit skill (8-step pipeline)
│       ├── ruleset-version.txt          # Ruleset version lock
│       ├── references/                  # Rule checklists
│       │   ├── security-checklist.md    # SEC-*, SBX-* (17 rules)
│       │   ├── destructive-ops-checklist.md  # DST-* (4 rules)
│       │   ├── quality-checklist.md     # QUA-* (8 rules)
│       │   ├── metadata-checklist.md    # META-* (3 rules)
│       │   └── spec-compliance-checklist.md  # SCH-* (5 rules)
│       ├── assets/
│       │   ├── report-template.md       # Markdown report template
│       │   └── slack-message-template.json
│       └── config/
│           └── approved-reviewers.yml   # audit-ignore reviewer list
├── .github/workflows/
│   └── skill-audit.yml                 # GitHub Actions workflow
├── docs/                               # Documentation
└── tests/fixtures/                     # Test skills (7 fixtures)
```

## Test Results

Full end-to-end CI verification performed on 2026-04-02:

| Test | Scenario | Expected | Result | Duration |
|------|----------|----------|--------|----------|
| Gate Only | PR with no skill/marketplace changes | Gate passes, Direct/Remote skip | **PASS** | 12s |
| Direct Clean | Safe skill submitted to `skills/` | PASSED verdict, PR comment posted | **PASS** | 1m 52s |
| Direct Dangerous | Malicious skill (SEC-010, SEC-003, DST-001, SBX-004) | BLOCKED verdict, check failure | **PASS** | 2m 6s |
| Remote Plugin | marketplace.json revision change | External repo cloned, audited | **PASS** | 13m 48s |
| URL Allowlist | `file:///etc/passwd` in marketplace.json | Blocked immediately | **PASS** | 5s |
| Fail-Closed | Missing ANTHROPIC_API_KEY | BLOCKED (not PASSED) | **PASS** | 13s |

**6/6 tests passed.**

---

## 한국어 요약

Claude Code 마켓플레이스에 제출되는 서드파티 스킬을 **보안, 안전성, 품질** 기준으로 자동 검사하는 게이트키퍼 스킬입니다.

- **35개 규칙** (OWASP AST10 기반) — CRITICAL 17개, HIGH 10개, MEDIUM 8개
- **Adoption-First** — CRITICAL만 차단, 나머지는 경고
- **로컬 검증** — PR 제출 전 CLI로 직접 감사 가능 ([가이드](docs/local-verification-guide.md))
- **CI 자동화** — GitHub Actions로 PR 감사 자동화 ([가이드](docs/ci-integration-guide.md))
- **Fail-Closed** — 감사 실패 시 차단 (PASSED가 아님)

## License

MIT
