# Spec Compliance Checklist

> agentskills.io 사양 준수 규칙. 스킬 메타데이터 형식, 프론트매터 필드 검증, 디렉토리 구조를 검사한다.
> 이 파일은 SKILL.md에서 참조하는 규칙 정의 문서이며, 여기에 포함된 패턴 예시는 **설명 텍스트**이다.

---

## SCH-001 — Name 필드 위반 (Name Field Violation)

- **심각도**: MEDIUM
- **OWASP**: AST10 10.6
- **범위**: SKILL.md frontmatter `name` 필드
- **검사 항목**:
  a. 존재하고 비어있지 않을 것
  b. 최대 64자
  c. 소문자, 숫자, 하이픈만 허용: `^[a-z0-9]+(-[a-z0-9]+)*$`
  d. 선행/후행 하이픈 금지
  e. 연속 하이픈(`--`) 금지
  f. 부모 디렉토리 이름과 일치할 것
- **메시지**: "Name field violation: [{specific issue}]. Must be kebab-case, max 64 chars, matching directory name (agentskills.io spec)."
- **수정 제안**: `name` 필드를 kebab-case(소문자+하이픈)로 수정하고, 스킬 디렉토리 이름과 일치시키세요. 예: `name: my-skill-name`

---

## SCH-002 — Description 필드 부적절 (Description Field Inadequate)

- **심각도**: MEDIUM
- **OWASP**: AST04 4.1
- **범위**: SKILL.md frontmatter `description` 필드
- **검사 항목**:
  a. 존재하고 비어있지 않을 것
  b. 최대 1024자
  c. WARNING: 20자 미만이면 트리거 신뢰성 부족
  d. WARNING: "use when", "use for", "trigger on" 등 트리거 구문이 없으면 CSO 매칭 어려움
- **메시지**: "Description field issue: [{specific issue}]. Should clearly describe what the skill does AND when to use it (agentskills.io spec)."
- **수정 제안**: description에 "Use when ..." 형태로 스킬의 기능과 트리거 조건을 명시하세요. 20자 이상, 1024자 이하로 작성하세요.

---

## SCH-003 — 알 수 없는 프론트매터 필드 (Unknown Frontmatter Field)

- **심각도**: MEDIUM
- **OWASP**: AST05 5.3
- **범위**: SKILL.md YAML frontmatter
- **허용 필드** (agentskills.io spec + skill-security-audit 확장):
  - `name`
  - `description`
  - `license`
  - `compatibility`
  - `metadata`
  - `allowed-tools`
  - `audit-ignore`
- **검사 방법**: 위 목록에 없는 최상위 키가 존재하면 플래그
- **참고**: `metadata` 하위 키는 자유 형식(dict[str, str])이므로 플래그하지 않는다. `audit-ignore`는 보안 감사 예외 선언용 필드이다.
- **허용 필드 기준 버전**: agentskills.io spec (2026-04 기준). 스펙 업데이트 시 이 목록도 갱신 필요.
- **메시지**: "Unknown frontmatter field [{field}]. Allowed: name, description, license, compatibility, metadata, allowed-tools, audit-ignore (agentskills.io spec)."
- **수정 제안**: 비표준 필드를 제거하거나 `metadata:` 하위로 이동하세요. 예: `metadata:\n  version: "1.0"\n  author: "name"`

---

## SCH-004 — SKILL.md 크기 초과 (SKILL.md Size Exceeded)

- **심각도**: MEDIUM
- **OWASP**: best practice (progressive disclosure)
- **범위**: SKILL.md 파일
- **검사 항목**:
  a. WARNING: 500줄 초과
  b. WARNING: 추정 토큰 수 5000 초과 (휴리스틱: 단어 수 × 1.3)
- **메시지**: "SKILL.md is {lines} lines / ~{tokens} tokens. Spec recommends < 500 lines / < 5000 tokens. Move detailed content to references/ (progressive disclosure)."
- **수정 제안**: 상세 내용을 `references/` 디렉토리로 분리하세요. SKILL.md는 개요와 네비게이션 역할만 담당해야 합니다.

---

## SCH-005 — 디렉토리 구조 위반 (Directory Structure Violation)

- **심각도**: MEDIUM
- **OWASP**: AST10 10.6
- **범위**: 스킬 루트 디렉토리
- **검사 항목**:
  a. `SKILL.md` 존재 (QUA-001과 중복되나, 여기서는 전체 구조 맥락에서 검증)
  b. 인정 하위 디렉토리만 허용: `references/`, `assets/`, `scripts/`, `config/`
  c. WARNING: 그 외 하위 디렉토리 존재 시 (예: `src/`, `lib/`, `node_modules/`, `.git/`)
  d. 루트에 실행 파일 금지 (SKILL.md와 설정 파일만 허용)
- **메시지**: "Unexpected directory [{dirname}] in skill root. Recognized: references/, assets/, scripts/, config/ (agentskills.io spec, SCH-005)."
- **수정 제안**: 비표준 디렉토리를 인정 디렉토리로 이동하세요. 예: `src/` → `scripts/`, `lib/` → `references/` 또는 `scripts/`. `node_modules/`는 `.gitignore`에 추가하세요.
