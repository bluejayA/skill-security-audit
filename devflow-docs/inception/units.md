# Units

**Timestamp**: 2026-03-27T13:00:00+09:00
**Source**: requirements.md (FR 13개), phase1-spec.md 14절
**Total Units**: 3

## Implementation Order

```
Unit 1: Rule Definitions → Unit 2: Core Skill → Unit 3: CI Integration
```

---

### Unit 1: rule-definitions
**Responsibility**: Phase 1 검사 규칙 22개를 카테고리별 체크리스트 파일로 정의한다
**Dependencies**: none
**Interfaces**:
- `references/security-checklist.md` — SEC-*, SBX-* 규칙 (자격증명, RCE, 셸 인젝션, 민감 경로)
- `references/destructive-ops-checklist.md` — DST-* 규칙 (파괴적 동작)
- `references/quality-checklist.md` — QUA-* 규칙 (최소 품질)
- `ruleset-version.txt` — 룰셋 버전 고정
**Implementation order**: 1
**Traces**: FR-3 (규칙 적용), FR-13 (룰셋 버전)
**Notes**:
- 각 규칙에 ID, 패턴, 심각도, 파일 유형별 분석 전략, 수정 제안 포함
- Markdown 문맥 분류 기준 (directive/descriptive) 명시
- Self-Audit 시 오탐되지 않도록 설명 텍스트와 실행 지시 구분 가이드 포함

### Unit 2: core-skill
**Responsibility**: SKILL.md 메인 스킬 + 보고서/Slack 출력 템플릿을 작성한다
**Dependencies**: Unit 1 (references/ 참조)
**Interfaces**:
- `SKILL.md` — 메인 스킬 파일 (스캔 → 분석 → 판정 → 보고서 생성 워크플로우)
- `assets/report-template.md` — Markdown 보고서 템플릿
- `assets/slack-message-template.json` — Slack Block Kit 메시지 템플릿
**Implementation order**: 2
**Traces**: FR-1 (파일 스캔), FR-2 (분석 방식), FR-4 (판정 로직), FR-5 (audit-ignore), FR-6 (보고서), FR-7 (JSON), FR-9 (Slack), FR-10 (멀티 스킬)
**Notes**:
- SKILL.md 워크플로우: 대상 식별 → 파일 유형별 분석 → 규칙 적용 → 판정 → 보고서 생성
- audit-ignore 검증 로직 포함 (approved-reviewers.yml 참조)
- 보고서에 ruleset version + SHA 포함

### Unit 3: ci-integration
**Responsibility**: GitHub Actions 워크플로우, 승인자 설정, README를 작성한다
**Dependencies**: Unit 2 (SKILL.md가 동작해야 CI에서 실행 가능)
**Interfaces**:
- `.github/workflows/skill-audit.yml` — PR 트리거 + workflow_dispatch
- `config/approved-reviewers.yml` — audit-ignore 승인자 목록
- `README.md` — 사용법, 통과/실패 예시, 제출자 가이드
**Implementation order**: 3
**Traces**: FR-8 (Actions 통합), FR-9 (Slack 알림), FR-11 (Self-Audit), FR-12 (오탐 재검토)
**Notes**:
- PR comment 업데이트 전략 (<!-- skill-audit-report --> 식별)
- Check Run: CRITICAL → failure
- Slack: continue-on-error: true
- Self-Audit 테스트 step 포함
- README에 통과/실패 예시 3가지 (phase1-spec 16절)
