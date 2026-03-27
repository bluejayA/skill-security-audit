# skill-security-audit

Claude Code 마켓플레이스에 제출된 서드파티 스킬을 **보안, 안전성, 품질** 기준으로 자동 검사하는 게이트키퍼 스킬.

> **Phase 1** — 조직 내부 AI 확산 환경 대상. "즉각적 실질 피해가 발생하는 것만 차단"

## Quick Start

```bash
# 로컬에서 스킬 검사
claude "제출 전에 스킬을 검사해줘: /path/to/my-skill"
```

GitHub Actions에서는 `skills/**` 경로 변경 시 자동 트리거됩니다.

## 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **Adoption-First** | CRITICAL만 차단. 좋은 스킬이 불필요하게 막히지 않는다 |
| **Deterministic-First** | LLM은 Markdown 문맥 분류에만 사용. 최종 판정은 규칙 기반 |
| **Actionable Feedback** | 차단 시 규칙 ID + 파일:라인 + 수정 예시 제공 |

## 판정 기준

```
CRITICAL 1개 이상  → ❌ BLOCKED (PR failure)
HIGH/MEDIUM만      → ⚠️ PASSED with warnings
발견 없음          → ✅ PASSED
```

## Phase 1 규칙 (22개)

**CRITICAL (12개)** — 1개라도 발견 시 차단

| 카테고리 | 규칙 |
|----------|------|
| 자격증명 | SEC-010 하드코딩 API 키, SEC-011 프라이빗 키, SEC-013 env 전체 덤프 |
| 원격 실행 | SEC-003 curl\|bash, SEC-030 base64\|bash |
| 셸 인젝션 | SEC-001 untrusted input + 셸 실행 |
| 민감 경로 | SBX-003 경로 탈출, SBX-004 ~/.ssh 등, SBX-007 키체인/히스토리 |
| 파괴적 동작 | DST-001 rm -rf, DST-007 sudo/chmod 777 |
| 품질 | QUA-001 SKILL.md 존재 |

**HIGH (7개)** — 경고, **MEDIUM (5개)** — 참고. 상세는 `references/` 체크리스트 참조.

## 문서

| 문서 | 내용 |
|------|------|
| [Introduction](docs/introduction.md) | 프로젝트 배경, 설계 철학, 아키텍처 |
| [User Guide](docs/user-guide.md) | 설치, 로컬 실행, CI 설정, audit-ignore, 트러블슈팅 |
| [Integration Guide](docs/integration-guide.md) | 기존 마켓플레이스 저장소에 통합하는 방법 |
| [Phase 1 Spec](docs/phase1-spec.md) | Phase 1 상세 스펙 (규칙 정의, 판정 로직) |
| [General Spec](docs/general-spec.md) | Phase 1+2 통합 스펙 (참조용) |

## 프로젝트 구조

```
skill-security-audit/
├── SKILL.md                          # 메인 스킬 (검사 워크플로우)
├── ruleset-version.txt               # 룰셋 버전 고정 (1.0.0)
├── references/
│   ├── security-checklist.md         # SEC-*, SBX-* 규칙
│   ├── destructive-ops-checklist.md  # DST-* 규칙
│   └── quality-checklist.md          # QUA-* 규칙
├── assets/
│   ├── report-template.md            # Markdown 보고서 템플릿
│   └── slack-message-template.json   # Slack Block Kit 템플릿
├── config/
│   └── approved-reviewers.yml        # audit-ignore 승인자 목록
├── .github/workflows/
│   └── skill-audit.yml               # GitHub Actions 워크플로우
└── docs/
    ├── introduction.md               # 소개 문서
    ├── user-guide.md                 # 사용자 가이드
    ├── phase1-spec.md                # Phase 1 스펙
    └── general-spec.md               # 통합 스펙
```

## 라이선스

MIT
