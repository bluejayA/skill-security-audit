# Requirements Analysis

**Depth**: Standard
**Timestamp**: 2026-03-27T12:00:00+09:00
**Source**: docs/phase1-spec.md (v1.5), docs/general-spec.md (v2.0)

## User Intent

제3자가 Claude Code 마켓플레이스에 제출하는 스킬을 자동으로 검사하는 독립 스킬을 구축한다. Phase 1은 조직 내부 AI 확산 환경을 대상으로 하며, "즉각적 실질 피해가 발생하는 것만 차단"하는 게이트키퍼 역할을 한다. 좋은 스킬이 불필요하게 막히지 않는 것(Adoption-First)을 제1 원칙으로 한다.

## Functional Requirements

### FR-1: 파일 스캔 및 대상 식별
- 스킬 디렉토리를 재귀적으로 스캔하여 검사 대상 파일을 식별한다
- 대상: SKILL.md, references/**, assets/**, scripts/**, *.sh, *.py, *.js, *.ts, package.json(scripts), Makefile
- 제외: node_modules/, .git/, 바이너리 파일

### FR-2: Context-Aware Deterministic-First 분석
- 구조화 코드(sh, py, js, ts, Makefile, package.json): 패턴 매칭으로 즉시 판정
- Markdown 파일: LLM 문맥 분류(directive vs descriptive) 후 규칙 적용
- 문맥 애매한 경우: MEDIUM + "(context uncertain)" 표기

### FR-3: Phase 1 규칙 적용 (22개)
- CRITICAL 11개: SEC-010, SEC-011, SEC-013, SEC-003, SEC-030, SEC-001(조건부), SBX-003, SBX-004, SBX-007, DST-001, DST-007, QUA-001
- HIGH 6개: SEC-001(조건부), SEC-002, SEC-020H, SEC-022, SBX-001, DST-003, QUA-002
- MEDIUM 5개: SEC-012, SEC-020, DST-002, QUA-003~006, QUA-010, QUA-011
- SEC-001은 trusted/untrusted input 판정 기준에 따라 CRITICAL/HIGH 분기

### FR-4: 판정 로직
- CRITICAL 1개 이상 → ❌ BLOCKED
- HIGH만 존재 → ⚠️ PASSED with warnings
- MEDIUM만 존재 → ⚠️ PASSED with warnings
- LOW만 또는 발견 없음 → ✅ PASSED

### FR-5: audit-ignore 예외 처리
- SKILL.md frontmatter에 audit-ignore 선언 지원
- 필수 필드: rule, reason, reviewer, expires
- reviewer는 config/approved-reviewers.yml과 대조하여 검증
- 만료된 예외 → 원래 심각도로 보고
- 유효한 예외 → EXEMPTED로 표시 (숨기지 않음)

### FR-6: Markdown 보고서 출력
- 스킬명, 제출자, 감사 일시, 룰셋 버전, Phase, 판정 결과 포함
- 심각도별 그룹핑 (CRITICAL → HIGH → MEDIUM)
- 각 finding: 규칙 ID + 파일:라인 + 증거 + 수정 제안
- EXEMPTED 항목 별도 섹션

### FR-7: Machine-readable JSON 출력
- audit-result.json으로 Actions 아티팩트 저장
- version, phase, ruleset_sha, skill, verdict, counts, findings 필드

### FR-8: GitHub Actions 통합
- PR 생성/업데이트 시 자동 트리거 (skills/** 변경)
- workflow_dispatch 지원 (수동 실행)
- 변경된 스킬 디렉토리 단위 증분 스캔
- PR comment 업데이트 (<!-- skill-audit-report --> 식별, 기존 댓글 edit)
- 스킬별 Check Run (CRITICAL 시 failure)

### FR-9: Slack 알림
- BLOCKED만 즉시 전송 (PASSED/WARNING은 미전송)
- Incoming Webhook 방식 (SLACK_WEBHOOK_URL secret)
- continue-on-error: true (전송 실패 → 검사 결과 미영향)
- D-30 만료 경고 (audit-ignore 만료 30일 전)

### FR-10: 멀티 스킬 PR 처리
- PR 내 여러 스킬 변경 시 스킬별 독립 검사
- 단일 PR comment에 스킬별 결과 통합 표시
- 하나라도 BLOCKED → PR failure

### FR-11: Self-Audit 테스트
- skill-security-audit 자신을 입력으로 넣었을 때 FALSE POSITIVE 0건 통과
- references/에 포함된 규칙 설명 텍스트가 오탐되지 않는지 검증

### FR-12: 오탐 재검토 요청
- audit-review-requested 라벨 또는 PR comment 형식으로 요청
- 관리자 수동 검토 후 false-positive-confirmed 라벨 → Check Run override

### FR-13: 룰셋 버전 관리
- ruleset-version.txt로 버전 고정
- 보고서에 버전 + SHA 기록
- 신규 CRITICAL 규칙 추가 시 30일 내 기존 스킬 재검사

## Non-Functional Requirements

### NFR-1: Adoption-First
- Phase 1에서 CRITICAL만 차단. HIGH는 경고만
- 오탐(false positive)을 최소화하여 불필요한 차단 방지

### NFR-2: Local = CI 일관성
- 동일한 ruleset 버전과 판정 기준 적용
- LLM 엔진 차이로 세부 메시지는 달라질 수 있음 (허용)

### NFR-3: Fail-safe
- Slack 등 부가 기능 실패가 검사 결과에 영향 없음

### NFR-4: 보고서 가독성
- Actionable feedback: 규칙 ID + 파일:라인 + 수정 예시 필수

## Technology Stack

| 계층 | 선택 | 소스 | 비고 |
|------|------|------|------|
| 실행 환경 | Claude Code | 프로젝트 특성 | SKILL.md 기반 스킬 |
| CI | GitHub Actions | spec 정의 | claude CLI 실행 |
| 알림 | Slack Incoming Webhook | spec 정의 | continue-on-error |
| 설정 | YAML (approved-reviewers.yml) | spec 정의 | 승인자 목록 |

## Constraints

1. **Phase 1 scope만 구현** — Phase 2 항목(동적 샌드박스, SARIF, policy-as-data 등)은 제외
2. **스킬 패키지와 마켓플레이스 인프라 분리** — SKILL.md/references/assets는 스킬 패키지, .github/config는 마켓플레이스 저장소
3. **LLM은 판정하지 않음** — Markdown 문맥 분류에만 사용, 최종 판정은 규칙 기반

## Assumptions

1. 마켓플레이스 저장소가 별도로 존재하며, 스킬은 skills/ 하위에 디렉토리로 제출된다
2. GitHub Actions에서 claude CLI 실행이 가능하다 (Anthropic API 키 설정)
3. Slack Incoming Webhook이 이미 설정되어 있거나, 미설정 시 graceful skip
4. 제출자는 로컬에서 claude CLI를 통해 동일한 검사를 실행할 수 있다

## Open Questions

없음. spec 문서(v1.5)에서 Phase 1 scope, 규칙, 판정 기준, 출력 형식, CI 통합이 모두 구체적으로 정의되어 있다.
