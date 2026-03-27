# User Stories

**Timestamp**: 2026-03-27T12:15:00+09:00
**Source**: devflow-docs/inception/requirements.md

## Actors

- **스킬 제출자**: 마켓플레이스에 스킬을 제출하는 제3자 개발자. 로컬 사전 검사 및 PR 제출
- **마켓플레이스 관리자**: 스킬 등록 승인/거부, audit-ignore 승인, 오탐 재검토 처리
- **CI 시스템**: GitHub Actions를 통해 자동으로 검사를 트리거하고 결과를 게시하는 시스템

---

## Stories

### US-001: 로컬 사전 검사 실행
**Actor**: 스킬 제출자
**Story**: As a 스킬 제출자, I want to run the security audit locally before submitting a PR so that I can fix issues early and avoid CI rejection.
**Acceptance Criteria**:
- Given 스킬 디렉토리가 존재할 때, When claude CLI로 skill-security-audit를 실행하면, Then 해당 디렉토리의 모든 대상 파일이 스캔된다
- Given 검사가 완료되었을 때, When 결과를 확인하면, Then Markdown 보고서에 심각도별 findings가 표시된다
- Given SLACK_WEBHOOK_URL이 미설정일 때, When 검사를 실행하면, Then Slack 전송을 skip하고 보고서 하단에 "(Slack 미설정 — 로컬 실행)" 표기한다
**Priority**: Must
**Traces**: FR-1, FR-2, FR-6

### US-002: 보안 규칙 자동 판정
**Actor**: CI 시스템
**Story**: As a CI 시스템, I want to apply deterministic security rules to skill files so that dangerous patterns are detected without human review.
**Acceptance Criteria**:
- Given 구조화 코드 파일(sh, py, js, ts)에서 패턴이 탐지되면, When 규칙을 적용하면, Then 규칙 기본값으로 즉시 판정한다
- Given Markdown 파일에서 패턴이 탐지되면, When LLM 문맥 분류 후 directive로 확인되면, Then 규칙 기본값으로 판정한다
- Given Markdown에서 directive/descriptive 판단이 애매하면, When 판정하면, Then MEDIUM + "(context uncertain)"으로 표기한다
- Given 자격증명 패턴(ghp_, sk-, AKIA 등)이 탐지되면, When 어떤 파일이든, Then CRITICAL로 즉시 판정한다
**Priority**: Must
**Traces**: FR-2, FR-3

### US-003: CRITICAL 차단 판정
**Actor**: CI 시스템
**Story**: As a CI 시스템, I want to block skills with CRITICAL findings so that immediate security threats are prevented from entering the marketplace.
**Acceptance Criteria**:
- Given CRITICAL finding이 1개 이상 존재할 때, When 최종 판정하면, Then ❌ BLOCKED를 반환한다
- Given HIGH만 존재하고 CRITICAL이 없을 때, When 최종 판정하면, Then ⚠️ PASSED with warnings를 반환한다
- Given finding이 없거나 LOW만 있을 때, When 최종 판정하면, Then ✅ PASSED를 반환한다
**Priority**: Must
**Traces**: FR-4

### US-004: Actionable 보고서 확인
**Actor**: 스킬 제출자
**Story**: As a 스킬 제출자, I want each finding to include a rule ID, file location, and fix suggestion so that I can quickly resolve issues without guessing.
**Acceptance Criteria**:
- Given finding이 보고될 때, When 보고서를 읽으면, Then 각 항목에 규칙 ID, 파일:라인, 증거(evidence), 수정 제안(fix)이 포함되어 있다
- Given CRITICAL finding이 있을 때, When 보고서를 읽으면, Then "차단 사유" 섹션에 별도로 그룹핑된다
- Given EXEMPTED 항목이 있을 때, When 보고서를 읽으면, Then 숨기지 않고 EXEMPTED 섹션에 표시된다
**Priority**: Must
**Traces**: FR-6, NFR-4

### US-005: audit-ignore 예외 선언
**Actor**: 스킬 제출자
**Story**: As a 스킬 제출자, I want to declare audit-ignore exceptions for intentional rule violations so that legitimate use cases are not blocked.
**Acceptance Criteria**:
- Given SKILL.md frontmatter에 audit-ignore를 선언할 때, When rule, reason, reviewer, expires가 모두 있으면, Then 해당 규칙은 EXEMPTED로 처리된다
- Given reviewer가 approved-reviewers.yml에 없을 때, When 예외를 검증하면, Then 예외가 무효 처리되고 원래 심각도로 보고된다
- Given expires가 지난 예외일 때, When 검사하면, Then 만료된 예외로 원래 심각도가 적용된다
**Priority**: Must
**Traces**: FR-5

### US-006: PR 자동 검사 트리거
**Actor**: CI 시스템
**Story**: As a CI 시스템, I want to automatically trigger audits when skills/** files change in a PR so that every submission is checked without manual intervention.
**Acceptance Criteria**:
- Given skills/** 경로에 파일 변경이 있는 PR이 생성/업데이트될 때, When Action이 트리거되면, Then 변경된 스킬 디렉토리만 증분 스캔한다
- Given 검사가 완료되면, When PR comment를 게시할 때, Then <!-- skill-audit-report --> 식별자로 기존 댓글을 찾아 edit한다 (없으면 신규 생성)
- Given CRITICAL이 발견되면, When Check Run을 설정할 때, Then failure로 표시한다
**Priority**: Must
**Traces**: FR-8

### US-007: Slack BLOCKED 알림 수신
**Actor**: 마켓플레이스 관리자
**Story**: As a 마켓플레이스 관리자, I want to receive Slack notifications when a skill is BLOCKED so that I can quickly review and respond to submissions.
**Acceptance Criteria**:
- Given 스킬이 BLOCKED 판정을 받으면, When Slack 알림을 전송하면, Then CRITICAL/HIGH findings 요약 + 제출자 + PR 링크가 포함된다
- Given PASSED/WARNING 판정이면, When 검사가 완료되면, Then Slack 알림을 전송하지 않는다
- Given Slack webhook이 실패하면, When 검사 결과를 확인하면, Then 검사 결과에는 영향이 없다
**Priority**: Should
**Traces**: FR-9, NFR-3

### US-008: 멀티 스킬 PR 통합 리포팅
**Actor**: 스킬 제출자
**Story**: As a 스킬 제출자, I want to see all skill audit results in a single PR comment so that I can review multiple skills at once.
**Acceptance Criteria**:
- Given PR에 3개 스킬이 변경되었을 때, When 검사가 완료되면, Then 단일 PR comment에 스킬별 결과가 통합 표시된다
- Given 3개 중 1개가 BLOCKED일 때, When Check Run을 설정할 때, Then PR 전체가 failure로 표시된다
**Priority**: Should
**Traces**: FR-10

### US-009: Self-Audit FALSE POSITIVE 검증
**Actor**: CI 시스템
**Story**: As a CI 시스템, I want to run a self-audit test on skill-security-audit itself so that rule description text in references/ does not trigger false positives.
**Acceptance Criteria**:
- Given skill-security-audit 자신을 입력으로 넣었을 때, When 검사를 실행하면, Then FALSE POSITIVE가 0건이다
- Given self-audit 테스트가 실패하면, When ruleset 배포를 시도하면, Then 배포가 차단된다
**Priority**: Should
**Traces**: FR-11

### US-010: audit-result.json 아티팩트 저장
**Actor**: CI 시스템
**Story**: As a CI 시스템, I want to save audit-result.json as an Actions artifact so that downstream tools can consume machine-readable results.
**Acceptance Criteria**:
- Given 검사가 완료되면, When 아티팩트를 저장할 때, Then version, phase, ruleset_sha, skill, verdict, counts, findings 필드가 포함된다
**Priority**: Should
**Traces**: FR-7

### US-011: 수동 검사 실행
**Actor**: 마켓플레이스 관리자
**Story**: As a 마켓플레이스 관리자, I want to manually trigger an audit via workflow_dispatch so that I can re-check existing skills or test specific directories.
**Acceptance Criteria**:
- Given workflow_dispatch에 skill_path를 입력할 때, When Action을 실행하면, Then 해당 경로의 스킬만 검사한다
**Priority**: Could
**Traces**: FR-8

### US-012: 오탐 재검토 요청
**Actor**: 스킬 제출자
**Story**: As a 스킬 제출자, I want to request a false-positive review so that incorrectly flagged findings can be overridden.
**Acceptance Criteria**:
- Given audit-review-requested 라벨을 추가하거나 PR comment에 재검토 요청을 작성할 때, When 관리자가 검토 후 승인하면, Then false-positive-confirmed 라벨이 부여되고 Check Run이 override된다
**Priority**: Could
**Traces**: FR-12

### US-013: D-30 만료 사전 경고
**Actor**: 마켓플레이스 관리자
**Story**: As a 마켓플레이스 관리자, I want to receive Slack warnings 30 days before audit-ignore exemptions expire so that I can prompt skill owners to renew.
**Acceptance Criteria**:
- Given audit-ignore의 expires가 30일 이내일 때, When 검사가 실행되면, Then Slack에 만료 예정 경고가 전송된다
**Priority**: Could
**Traces**: FR-9

### US-014: 룰셋 버전 추적
**Actor**: 마켓플레이스 관리자
**Story**: As a 마켓플레이스 관리자, I want each audit report to include the ruleset version and SHA so that I can trace which rules were applied.
**Acceptance Criteria**:
- Given 검사가 완료되면, When 보고서를 확인할 때, Then ruleset-version.txt의 버전과 SHA가 포함되어 있다
**Priority**: Could
**Traces**: FR-13
