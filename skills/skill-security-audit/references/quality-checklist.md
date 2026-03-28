# Quality Checklist

> Phase 1 최소 품질 규칙. 스킬이 Claude Code에 의해 올바르게 로드·실행되기 위한 최소 구조 조건과 기본 품질을 검사한다.
> 이 파일은 SKILL.md에서 참조하는 규칙 정의 문서이며, 여기에 포함된 패턴 예시는 **설명 텍스트**이다.

---

## 구조 검사 (Structure)

### QUA-001 — SKILL.md 파일 존재
- **심각도**: CRITICAL
- **검사 방법**: 스킬 디렉토리 루트에 `SKILL.md` 파일이 존재하는지 확인
- **수정 제안**: 스킬 디렉토리에 SKILL.md 파일이 없습니다. Claude Code 스킬은 반드시 SKILL.md를 포함해야 합니다.

### QUA-002 — frontmatter 필수 필드
- **심각도**: HIGH
- **검사 방법**: SKILL.md 상단의 YAML frontmatter에서 `name`과 `description` 필드 존재 여부 확인
- **필수 필드**:
  - `name`: 스킬 식별자
  - `description`: 스킬 설명 (CSO 트리거용)
- **수정 제안**: SKILL.md의 frontmatter에 `name`과 `description` 필드를 추가하세요.
  ```yaml
  ---
  name: my-skill-name
  description: "Use when ..."
  ---
  ```

### QUA-003 — description 시작 형식
- **심각도**: MEDIUM
- **검사 방법**: `description` 값이 `"Use when"`으로 시작하는지 확인 (대소문자 무시)
- **수정 제안**: description은 "Use when"으로 시작해야 Claude Code가 적절한 시점에 스킬을 트리거할 수 있습니다.

### QUA-004 — description 길이
- **심각도**: MEDIUM
- **검사 방법**: `description` 값의 문자 수가 1024자 이하인지 확인
- **수정 제안**: description이 1024자를 초과합니다. 핵심 트리거 조건만 포함하도록 줄여주세요.

### QUA-005 — SKILL.md 본문 줄 수
- **심각도**: MEDIUM
- **검사 방법**: SKILL.md의 총 줄 수가 500줄 이하인지 확인 (frontmatter 포함)
- **수정 제안**: SKILL.md가 500줄을 초과합니다. 상세 내용은 references/ 디렉토리로 분리하세요.

### QUA-006 — 참조 파일 깊이
- **심각도**: MEDIUM
- **검사 방법**: SKILL.md에서 참조하는 파일이 또 다른 파일을 참조하는지 확인 (2단계 이상 참조 탐지)
- **탐지 패턴**: references/ 파일 내에서 다른 references/ 파일을 Read하도록 지시하는 구문
- **수정 제안**: 참조 파일이 또 다른 파일을 참조합니다. SKILL.md → 참조 파일(1단계)까지만 허용합니다. 중첩 참조를 SKILL.md에서 직접 참조하도록 변경하세요.

---

## 내용 검사 (Content)

### QUA-010 — 모호 표현 사용
- **심각도**: MEDIUM
- **탐지 패턴** (SKILL.md 핵심 지시 섹션에서만):
  - `적절히`, `적절하게`, `적당히`
  - `잘`, `잘 처리`
  - `필요 시`, `필요에 따라`
  - `상황에 따라`, `경우에 따라`
  - `appropriately`, `as needed`, `if necessary`
  - `properly`, `correctly` (단독 사용 — 구체적 기준 없이)
- **검사 범위**: SKILL.md 본문의 지시 섹션 (## 이하). frontmatter, 주석, Examples 섹션은 제외.
- **수정 제안**: 모호한 표현 대신 구체적인 기준을 명시하세요. 예: "적절히 처리한다" → "500ms 이내에 응답한다" 또는 "에러 시 재시도 3회 후 실패로 처리한다"

### QUA-011 — description에 내부 구현 상세 노출
- **심각도**: MEDIUM
- **탐지 패턴** (description 필드에서만):
  - 파일 경로 (`references/`, `_shared/`, `assets/`)
  - 위임 체인 (`dispatch`, `delegate`, `호출`)
  - 실행 순서 (`Step 1`, `먼저 ... 그 다음`)
  - 내부 구현 키워드 (`subprocess`, `parse`, `regex`)
- **검사 범위**: SKILL.md frontmatter의 `description` 필드만
- **수정 제안**: description에는 "무엇을 하는가" + "언제 사용하는가"만 기술하세요. 내부 구현 상세(파일 경로, 실행 순서, 위임 체인)는 제거하세요.
