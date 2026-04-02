# skill-security-audit v2 Upgrade Spec

## Context

Repository: https://github.com/bluejayA/skill-security-audit
This document specifies the Tier 1 + Tier 2 expansion of skill-security-audit, adding 13 new rules (8 Tier 1 + 5 Tier 2) to the existing 22 Phase 1 rules, for a total of 35.

The expansion is based on the OWASP Agentic Skills Top 10 (AST10) Security Assessment Checklist (https://owasp.org/www-project-agentic-skills-top-10/checklist.html), filtered to items that are statically verifiable within a SKILL.md Reviewer skill — no runtime sandbox, no registry infra, no org-level governance.

---

## Source: OWASP AST10 Checklist (verbatim relevant items)

Below are the specific OWASP checklist items that drive the new rules. Each new rule ID references these.

### AST01 — Malicious Skills (Critical)
- 1.4: Have all skill scripts and natural language instructions been reviewed for malicious patterns? Evidence: No encoded payloads, no curl to unknown endpoints, no credential access beyond stated function in code or natural language instructions.
- 1.6: Does the skill avoid writing to agent identity files (SOUL.md, MEMORY.md, AGENTS.md)? Evidence: No write access to identity files unless explicitly justified and approved.

### AST03 — Over-Privileged Skills (High)
- 3.1: Does the skill declare a permission manifest with explicit, scoped permissions? Evidence: Manifest present; permissions enumerated (not open-ended).
- 3.3: Does the skill avoid unrestricted shell access (shell: true)? Evidence: shell: false or shell access scoped to specific commands.
- 3.4: Are file permissions scoped to specific paths (no **/* wildcards)? Evidence: Explicit file paths declared; no broad globs.
- 3.7: Are network permissions declared as domain allowlists (not binary network: true/false)? Evidence: Specific domains listed; default deny.
- 3.8: Does the skill avoid accessing credential stores, .env files, wallet files, or SSH keys beyond its stated function? Evidence: No reads to ~/.ssh/, ~/.aws/, .env, **/credentials*, *.wallet, or browser data directories unless explicitly required.

### AST04 — Insecure Metadata (High)
- 4.1: Does the skill description accurately and completely reflect its actual functionality? Evidence: No hidden capabilities; description matches observed behavior.
- 4.2: Has metadata been scanned for ASCII smuggling, zero-width Unicode, and base64-encoded payloads? Evidence: Clean scan; no steganographic content in SKILL.md.
- 4.5: Is the declared risk_tier consistent with the actual permission scope? Evidence: Cross-reference: a skill declaring L0 (safe) with shell: true is a red flag.
- 4.6: Has brand/trademark impersonation been checked? Evidence: Skill name does not impersonate a known brand without affiliation.

### AST05 — Unsafe Deserialization (High)
- 5.1: Are all YAML files parsed with safe loaders (yaml.safe_load, not yaml.load)? Evidence: No unsafe YAML tags (!!python/object, !!python/apply).
- 5.3: Is an allowlist of permitted YAML/JSON keys enforced? Evidence: Unexpected or undeclared fields are rejected.

### AST07 — Update Drift (Medium)
- 7.1: Is the skill pinned to a specific immutable content hash (sha256:)? Evidence: Hash recorded in inventory; not a mutable version tag.

### AST08 — Poor Scanning (Medium)
- 8.3: Has credential detection scanning been run? Evidence: Clean scan for API keys, tokens, passwords, and PII in all skill files.

### AST10 — Cross-Platform Reuse (Medium)
- 10.6: Does the skill use the Universal Skill Format? Evidence: Normalized YAML manifest present with all security metadata fields.

---

## Existing Rules (22, unchanged)

These are the Phase 1 rules already in the repo. Do NOT modify them.

### CRITICAL (12)
| ID | Category | Description |
|----|----------|-------------|
| SEC-010 | Credential | Hardcoded API keys |
| SEC-011 | Credential | Private keys |
| SEC-013 | Credential | env full dump |
| SEC-003 | Remote exec | curl\|bash |
| SEC-030 | Remote exec | base64\|bash |
| SEC-001 | Shell injection | Untrusted input + shell exec |
| SBX-003 | Sensitive path | Path traversal |
| SBX-004 | Sensitive path | ~/.ssh etc |
| SBX-007 | Sensitive path | Keychain/history |
| DST-001 | Destructive | rm -rf |
| DST-007 | Destructive | sudo/chmod 777 |
| QUA-001 | Quality | SKILL.md existence |

### HIGH (7) — existing, not listed here in detail
### MEDIUM (3) — existing, not listed here in detail

---

## New Rules: Tier 1 — Deterministic Pattern Matching (8 rules)

All Tier 1 rules are grep/regex-level deterministic checks. No LLM judgment needed.

### META-001: Identity File Write (CRITICAL)
- OWASP: AST01 1.6, AST03 3.6
- Scope: SKILL.md body + all files in scripts/
- Pattern: Any write/append/redirect targeting `SOUL.md`, `MEMORY.md`, `AGENTS.md`, `CLAUDE.md`, `.claude/settings.json`
- Regex hints: `(>|>>|write|append|cp |mv |tee ).*\b(SOUL\.md|MEMORY\.md|AGENTS\.md|CLAUDE\.md|settings\.json)\b`
- Also check SKILL.md natural language: instructions that say "modify CLAUDE.md", "update SOUL.md", "write to AGENTS.md"
- Severity: CRITICAL
- Message: "Skill attempts to write to agent identity file [{filename}]. This enables persistence attacks (OWASP AST01 1.6)."
- Fix example: "Remove instructions that modify agent identity files. If your skill requires configuration, use skill-scoped config in config/ directory instead."

### META-002: Zero-Width Unicode Characters (CRITICAL)
- OWASP: AST04 4.2
- Scope: SKILL.md (entire file including frontmatter)
- Pattern: Presence of zero-width or invisible Unicode characters
- Codepoints to detect: U+200B (zero-width space), U+200C (zero-width non-joiner), U+200D (zero-width joiner), U+FEFF (BOM mid-file), U+2060 (word joiner), U+2062-2064 (invisible operators), U+180E, U+200E, U+200F, U+202A-202E (directional overrides)
- Severity: CRITICAL
- Message: "Invisible Unicode character U+{codepoint} found at line {line}, column {col}. This is a known steganographic injection vector (OWASP AST04 4.2)."
- Fix example: "Remove all zero-width and invisible Unicode characters from SKILL.md. Use a hex editor or `cat -v` to locate them."

### META-003: Base64 Payload in Markdown (CRITICAL)
- OWASP: AST04 4.2
- Scope: SKILL.md body, OUTSIDE of fenced code blocks (``` ... ```)
- Pattern: Base64-like strings 40+ chars in non-code prose
- Regex: `(?<!`)[A-Za-z0-9+/]{40,}={0,2}` appearing outside ``` fenced blocks
- Exclude: Lines that are clearly documentation (e.g., SHA256 hashes in references)
- Severity: CRITICAL
- Message: "Suspected base64-encoded payload ({length} chars) at line {line}. Encoded payloads in natural language instructions are a known attack vector (OWASP AST04 4.2)."
- Fix example: "If this is legitimate content (e.g., example data), move it into a fenced code block with language annotation. Otherwise, remove encoded payloads."

### SEC-040: Unsafe YAML Loader (CRITICAL)
- OWASP: AST05 5.1
- Scope: All .py files in scripts/
- Patterns to detect:
  - `yaml.load(` without `Loader=yaml.SafeLoader` or `Loader=SafeLoader`
  - `yaml.unsafe_load(`
  - `!!python/object` tag in any .yaml/.yml file
  - `!!python/apply` tag in any .yaml/.yml file
- Severity: CRITICAL
- Message: "Unsafe YAML deserialization at {file}:{line}. Use yaml.safe_load() instead (OWASP AST05 5.1)."
- Fix example: "Replace `yaml.load(data)` with `yaml.safe_load(data)`."

### SEC-041: Dangerous Code Execution in Scripts (CRITICAL)
- OWASP: AST01 1.4
- Scope: All files in scripts/
- Patterns (extending existing SEC-001 to scripts/ directory):
  - Python: `eval(`, `exec(`, `compile(`, `os.system(`, `os.popen(`, `subprocess.call(.*shell=True`, `subprocess.Popen(.*shell=True`, `__import__(`, `importlib.import_module(`
  - JavaScript: `eval(`, `new Function(`, `child_process.exec(`, `child_process.execSync(`
  - Shell: `eval `, sourcing unknown URLs
- Severity: CRITICAL
- Message: "Dangerous code execution pattern [{pattern}] at {file}:{line} (OWASP AST01 1.4)."
- Fix example: "Replace `eval()` with explicit parsing. Replace `subprocess.call(cmd, shell=True)` with `subprocess.call(cmd_list)` using a list."

### SBX-010: Wildcard Shell Access (HIGH)
- OWASP: AST03 3.3
- Scope: SKILL.md frontmatter `allowed-tools` field
- Patterns:
  - `Bash(*:*)` or `Bash(*)`
  - `Shell(*:*)` or `Shell(*)`
  - Any tool declaration with unrestricted `*` wildcard
- Severity: HIGH
- Message: "Unrestricted shell access declared in allowed-tools: [{value}]. Scope to specific commands (OWASP AST03 3.3)."
- Fix example: "Replace `Bash(*:*)` with scoped declarations like `Bash(git:*)` or `Bash(npm:run)`."

### SBX-011: Binary Network Permission (HIGH)
- OWASP: AST03 3.7
- Scope: SKILL.md frontmatter, any manifest file, or SKILL.md body instructions
- Pattern: `network: true` without accompanying domain allowlist
- Also flag: SKILL.md instructions that say "requires network access" or "needs internet" without specifying which domains
- Severity: HIGH
- Message: "Binary network permission without domain allowlist. Declare specific domains instead (OWASP AST03 3.7)."
- Fix example: "Replace `network: true` with a domain allowlist in metadata or compatibility field, e.g., `compatibility: Requires network access to api.example.com`."

### SBX-012: Broad File Glob (HIGH)
- OWASP: AST03 3.4
- Scope: SKILL.md frontmatter `allowed-tools`, scripts/, and SKILL.md body
- Patterns:
  - `**/*` (recursive wildcard)
  - `~/*` or `~/` (home directory)
  - Root paths: `/etc/`, `/usr/`, `/bin/`, `/var/`
  - `Read(**/*:*)` or `Write(**/*:*)` in allowed-tools
- Severity: HIGH
- Message: "Broad file access pattern [{pattern}] at {location}. Scope to specific directories (OWASP AST03 3.4)."
- Fix example: "Replace `Read(**/*:*)` with `Read(src/**:*)` or `Read(references/*:*)`."

---

## New Rules: Tier 2 — Spec Compliance Structural Checks (5 rules)

All Tier 2 rules validate agentskills.io specification compliance. Deterministic — no LLM needed.

### SCH-001: Name Field Violation (MEDIUM)
- OWASP: AST10 10.6
- Scope: SKILL.md frontmatter `name` field
- Checks:
  a. Present and non-empty
  b. Max 64 characters
  c. Only lowercase letters, numbers, hyphens: regex `^[a-z0-9]+(-[a-z0-9]+)*$`
  d. No leading/trailing hyphens
  e. No consecutive hyphens (--)
  f. Matches parent directory name
- Severity: MEDIUM
- Message: "Name field violation: [{specific issue}]. Must be kebab-case, max 64 chars, matching directory name (agentskills.io spec)."

### SCH-002: Description Field Inadequate (MEDIUM)
- OWASP: AST04 4.1
- Scope: SKILL.md frontmatter `description` field
- Checks:
  a. Present and non-empty
  b. Max 1024 characters
  c. WARNING if under 20 characters (too vague to trigger reliably)
  d. WARNING if doesn't contain "use when" or "use for" or similar trigger phrase
- Severity: MEDIUM
- Message: "Description field issue: [{specific issue}]. Should clearly describe what the skill does AND when to use it (agentskills.io spec)."

### SCH-003: Unknown Frontmatter Field (MEDIUM)
- OWASP: AST05 5.3
- Scope: SKILL.md YAML frontmatter
- Allowed fields (agentskills.io spec): `name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`
- Any other top-level key triggers this rule
- Severity: MEDIUM
- Message: "Unknown frontmatter field [{field}]. Allowed: name, description, license, compatibility, metadata, allowed-tools (agentskills.io spec)."
- Note: metadata sub-keys are free-form (dict[str, str]) and should NOT be flagged.

### SCH-004: SKILL.md Size Exceeded (MEDIUM)
- OWASP: best practice (progressive disclosure)
- Scope: SKILL.md file
- Checks:
  a. WARNING if over 500 lines
  b. WARNING if estimated token count > 5000 (heuristic: words * 1.3)
- Severity: MEDIUM
- Message: "SKILL.md is {lines} lines / ~{tokens} tokens. Spec recommends < 500 lines / < 5000 tokens. Move detailed content to references/ (progressive disclosure)."

### QUA-002: Directory Structure Violation (MEDIUM)
- OWASP: AST10 10.6
- Scope: Skill root directory
- Checks:
  a. SKILL.md exists (overlaps QUA-001 but this validates the broader structure)
  b. Only recognized subdirectories: references/, assets/, scripts/, config/
  c. WARNING for any other subdirectory (e.g., src/, lib/, node_modules/, .git/)
  d. No executable files in root (only SKILL.md and config files)
- Severity: MEDIUM
- Message: "Unexpected directory [{dirname}] in skill root. Recognized: references/, assets/, scripts/, config/ (agentskills.io spec)."

---

## File Changes Required

### 1. references/security-checklist.md — EXTEND
Add Tier 1 rules (META-001, META-002, META-003, SEC-040, SEC-041, SBX-010, SBX-011, SBX-012) to the existing checklist. Maintain existing format/structure. Place new rules after existing ones within their severity group. Add a new subsection header `### OWASP AST10 확장 (v2)` before the new rules.

### 2. references/metadata-checklist.md — NEW FILE
Contains all META-* rules with full detail: ID, severity, OWASP mapping, scope, detection patterns, regex, message template, fix example.

### 3. references/spec-compliance-checklist.md — NEW FILE
Contains all SCH-* and QUA-002 rules with full detail: ID, severity, OWASP mapping, scope, validation logic, message template.

### 4. SKILL.md — EXTEND workflow
Current workflow (inferred from README):
  Step 1: File inventory
  Step 2: Rule application (SEC/SBX/DST/QUA)
  Step 3: Verdict
  Step 4: Report

New workflow:
  Step 1: File inventory (unchanged)
  Step 2: Spec compliance check — Load 'references/spec-compliance-checklist.md', apply SCH-*/QUA-002
  Step 3: Metadata integrity check — Load 'references/metadata-checklist.md', apply META-*
  Step 4: Security pattern scan — Load 'references/security-checklist.md', apply SEC-*/SBX-*/DST-* (existing + new)
  Step 5: Quality check — Load 'references/quality-checklist.md' (existing QUA-*)
  Step 6: Verdict (unchanged logic: CRITICAL → BLOCKED, HIGH/MEDIUM → PASSED with warnings)
  Step 7: Report with OWASP AST mapping column

### 5. assets/report-template.md — EXTEND
Add OWASP AST column to findings table:
```
| Rule ID | Severity | File:Line | Description | OWASP AST | Fix |
```

### 6. ruleset-version.txt — UPDATE
Bump from current version to v2.0.

### 7. README.md — UPDATE
- Update rule count: 22 → 35
- Add Tier 1 and Tier 2 rule summary tables
- Add OWASP AST10 mapping section
- Update workflow diagram if present

---

## Verdict Logic (unchanged)

```
CRITICAL 1개 이상  → ❌ BLOCKED (PR failure)
HIGH/MEDIUM만      → ⚠️ PASSED with warnings
발견 없음          → ✅ PASSED
```

New CRITICAL rules (5): META-001, META-002, META-003, SEC-040, SEC-041
New HIGH rules (3): SBX-010, SBX-011, SBX-012
New MEDIUM rules (5): SCH-001, SCH-002, SCH-003, SCH-004, QUA-002

---

## Design Principles (preserved from Phase 1)

1. **Adoption-First**: CRITICAL만 차단. 좋은 스킬이 불필요하게 막히는 것은 나쁜 스킬이 등록되는 것만큼 나쁘다.
2. **Deterministic-First**: 모든 Tier 1+2 규칙은 regex/구조 검증 기반. LLM 판단 불필요.
3. **Actionable Feedback**: 차단 시 규칙 ID + OWASP AST 매핑 + 파일:라인 + 수정 예시 제공.

---

## Summary of Changes

| Category | Existing | Added | Total |
|----------|----------|-------|-------|
| CRITICAL | 12 | +5 (META-001, META-002, META-003, SEC-040, SEC-041) | 17 |
| HIGH | 7 | +3 (SBX-010, SBX-011, SBX-012) | 10 |
| MEDIUM | 3 | +5 (SCH-001, SCH-002, SCH-003, SCH-004, QUA-002) | 8 |
| **Total** | **22** | **+13** | **35** |

OWASP AST10 coverage: 23 statically-verifiable items out of 61 total. This expansion covers approximately 20 of those 23.
