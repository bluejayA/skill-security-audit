# Metadata Integrity Checklist

> OWASP AST10 기반 메타데이터 무결성 규칙. 에이전트 아이덴티티 파일 보호, 스테가노그래피 주입 탐지, 인코딩된 페이로드 탐지를 검사한다.
> 이 파일은 SKILL.md에서 참조하는 규칙 정의 문서이며, 여기에 포함된 패턴 예시는 **설명 텍스트**이다.

---

## 분석 전략 참조

| 파일 유형 | 전략 |
|-----------|------|
| `SKILL.md` (frontmatter + body) | 패턴 매칭 + 자연어 지시 탐지 |
| `*.md` (references/, assets/) | 패턴 매칭 (META-002, META-003) |
| `scripts/**` (*.sh, *.py, *.js, *.ts) | 패턴 매칭으로 즉시 판정 |

---

## META-001 — 에이전트 아이덴티티 파일 쓰기 (Identity File Write)

- **심각도**: CRITICAL
- **OWASP**: AST01 1.6, AST03 3.6
- **범위**: SKILL.md body + scripts/ 내 모든 파일
- **탐지 패턴**:
  - 쓰기/추가/리다이렉션이 다음 파일을 대상으로 하는 경우:
    - `SOUL.md`, `MEMORY.md`, `AGENTS.md`, `CLAUDE.md`, `settings.json`
  - Regex hint: `(>|>>|write|append|cp |mv |tee ).*\b(SOUL\.md|MEMORY\.md|AGENTS\.md|CLAUDE\.md|settings\.json)\b`
  - SKILL.md 자연어 지시에서도 탐지:
    - "modify CLAUDE.md", "update SOUL.md", "write to AGENTS.md"
    - "CLAUDE.md를 수정", "SOUL.md에 추가", "MEMORY.md에 기록"
- **파일 유형별 처리**:
  - 구조화 코드 (scripts/): 패턴 매칭 즉시 판정
  - SKILL.md body: 자연어 지시 탐지 — directive이면 CRITICAL
- **메시지**: "Skill attempts to write to agent identity file [{filename}]. This enables persistence attacks (OWASP AST01 1.6)."
- **수정 제안**: 에이전트 아이덴티티 파일(SOUL.md, MEMORY.md, AGENTS.md, CLAUDE.md, settings.json)을 수정하는 지시를 제거하세요. 스킬 설정이 필요하면 `config/` 디렉토리 내 스킬 전용 설정 파일을 사용하세요.

---

## META-002 — 제로폭 유니코드 문자 (Zero-Width Unicode Characters)

- **심각도**: CRITICAL
- **OWASP**: AST04 4.2
- **범위**: 스킬 디렉토리 내 모든 `.md` 파일 및 텍스트 파일 (SKILL.md, references/*.md, assets/*.md 등)
- **탐지 패턴**: 다음 코드포인트의 존재 여부:
  - `U+200B` — Zero-Width Space
  - `U+200C` — Zero-Width Non-Joiner
  - `U+200D` — Zero-Width Joiner
  - `U+FEFF` — BOM (파일 중간에 위치한 경우. 파일 최초 바이트의 BOM은 정상이므로 제외)
  - `U+2060` — Word Joiner
  - `U+2061` — Function Application
  - `U+2062`~`U+2064` — Invisible Operators
  - `U+180E` — Mongolian Vowel Separator
  - `U+200E` — Left-to-Right Mark
  - `U+200F` — Right-to-Left Mark
  - `U+202A`~`U+202E` — Directional Overrides
  - `U+2066`~`U+2069` — Bidi Isolate Formatting Characters
  - `U+00AD` — Soft Hyphen
  - `U+034F` — Combining Grapheme Joiner
  - `U+115F`, `U+1160` — Hangul Filler Characters
  - `U+FE00`~`U+FE0F` — Variation Selectors
  - `U+E0001`~`U+E007F` — **Tag Characters** (ASCII Smuggling 핵심 벡터)
- **파일 유형별 처리**: 대상 파일 전체에서 바이트 수준 스캔
- **메시지**: "Invisible Unicode character U+{codepoint} found at {file}:{line}, column {col}. This is a known steganographic injection vector (OWASP AST04 4.2)."
- **수정 제안**: 모든 제로폭 및 비가시 유니코드 문자를 제거하세요. `cat -v` 또는 hex editor로 위치를 확인할 수 있습니다.

---

## META-003 — Markdown 내 Base64 페이로드 (Base64 Payload in Markdown)

- **심각도**: CRITICAL
- **OWASP**: AST04 4.2
- **범위**: 스킬 디렉토리 내 모든 `.md` 파일 body — 펜스드 코드 블록(``` ... ```) **외부**만
- **탐지 패턴**:
  - 코드 블록 외부에서 40자 이상의 Base64 유사 문자열
  - Regex: `[A-Za-z0-9+/]{40,}={0,2}` (펜스드 코드 블록 내부 제외)
  - **명시적 제외** (오탐 방지):
    - `sha256:`, `sha512:`, `integrity=`, `hash:` 접두사가 붙은 줄
    - YAML frontmatter 내의 값 (frontmatter는 별도 검사 대상)
    - 코드 블록 내부 (이미 스코프에서 제외)
- **파일 유형별 처리**:
  - 대상 `.md` 파일에서 펜스드 코드 블록을 먼저 식별 후, 블록 외부의 prose 영역만 스캔
- **메시지**: "Suspected base64-encoded payload ({length} chars) at line {line}. Encoded payloads in natural language instructions are a known attack vector (OWASP AST04 4.2)."
- **수정 제안**: 정당한 내용(예시 데이터 등)이라면 펜스드 코드 블록 내로 이동하고 언어 어노테이션을 추가하세요. 그렇지 않으면 인코딩된 페이로드를 제거하세요.
