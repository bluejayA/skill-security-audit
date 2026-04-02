# Test Expected Results

각 픽스처에 대한 기대 감사 결과.

## 1. clean-skill → PASSED

기대 findings: 없음 (또는 MEDIUM만)
- 가능한 MEDIUM: QUA-003 (description이 "Use when"으로 시작하므로 통과할 수 있음)

## 2. critical-violations → BLOCKED

| 기대 규칙 | 심각도 | 증거 |
|-----------|--------|------|
| SEC-010 | CRITICAL | `sk-test1234567890abcdefghijklmnop` |
| SEC-003 | CRITICAL | `curl ... \| bash` |
| DST-001 | CRITICAL | `rm -rf /tmp/deploy-*` |
| META-001 | CRITICAL | "CLAUDE.md를 수정" |

## 3. meta-unicode-attack → BLOCKED

| 기대 규칙 | 심각도 | 증거 |
|-----------|--------|------|
| META-002 | CRITICAL | U+200B at "파일을​" 뒤 |

## 4. base64-payload → BLOCKED

| 기대 규칙 | 심각도 | 증거 |
|-----------|--------|------|
| META-003 | CRITICAL | 80자 Base64 문자열 (코드 블록 외부) |

## 5. bad-frontmatter → PASSED with warnings

| 기대 규칙 | 심각도 | 증거 |
|-----------|--------|------|
| SCH-001 | MEDIUM | name `Bad-Frontmatter_Skill` — 대문자, 밑줄 포함 |
| SCH-002 | MEDIUM | description "short" — 20자 미만 |
| SCH-003 | MEDIUM | `custom_field`, `another_bad_field` — 비표준 필드 |
| SCH-005 | MEDIUM | `src/` — 비표준 디렉토리 |

## 6. wildcard-tools → PASSED with warnings (또는 BLOCKED)

| 기대 규칙 | 심각도 | 증거 |
|-----------|--------|------|
| SBX-010 | HIGH | `Bash(*:*)` |
| SBX-012 | HIGH | `Read(**/*:*)`, `Write(**/*:*)`, `~/*`, `/etc/` |
| SBX-011 | HIGH | "requires network access" 도메인 미지정 |

## 7. self-audit-copycat → BLOCKED

면책 문구를 복사했지만 Self-Audit 보호가 적용되지 않아야 함:

| 기대 규칙 | 심각도 | 증거 |
|-----------|--------|------|
| SBX-004 | CRITICAL | `~/.ssh/id_rsa` |
| SEC-003 | CRITICAL | `curl ... -d "$(cat ...)"` + 파이프 아님이지만 SEC-020H/SBX-004 |
| META-001 | CRITICAL | "MEMORY.md에 ... 추가한다" |

핵심 검증: references/patterns.md에 면책 문구가 있어도, 이 스킬은 `skill-security-audit` 자신이 아니므로 descriptive 처리하지 않는다.
