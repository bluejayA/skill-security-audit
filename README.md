# skill-security-audit

Claude Code 마켓플레이스에 제출된 서드파티 스킬을 **보안, 안전성, 품질** 기준으로 자동 검사하는 게이트키퍼 스킬.

> **Phase 2** — OWASP Agentic Skills Top 10 (AST10) 기반 35개 규칙. "즉각적 실질 피해 차단 + 메타데이터 무결성 + 사양 준수"

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

## 규칙 (35개)

### CRITICAL (17개) — 1개라도 발견 시 차단

| 카테고리 | 규칙 |
|----------|------|
| 자격증명 | SEC-010 하드코딩 API 키, SEC-011 프라이빗 키, SEC-013 env 전체 덤프 |
| 원격 실행 | SEC-003 curl\|bash, SEC-030 base64\|bash |
| 셸 인젝션 | SEC-001 untrusted input + 셸 실행 |
| 민감 경로 | SBX-003 경로 탈출, SBX-004 ~/.ssh 등, SBX-007 키체인/히스토리 |
| 파괴적 동작 | DST-001 rm -rf, DST-007 sudo/chmod 777 |
| 품질 | QUA-001 SKILL.md 존재 |
| 메타데이터 | META-001 아이덴티티 파일 쓰기, META-002 제로폭 유니코드, META-003 Base64 페이로드 |
| YAML 안전성 | SEC-040 안전하지 않은 YAML 로더 |
| 코드 실행 | SEC-041 스크립트 내 위험한 코드 실행 |

### HIGH (10개) — 경고

| 카테고리 | 규칙 |
|----------|------|
| 기존 | SEC-001(trusted), SEC-002 eval, SEC-020H HTTP+민감, SEC-022 네트워크 도구, SBX-001 외부 쓰기, DST-003 force push, QUA-002 필수 필드 |
| v2 추가 | SBX-010 무제한 셸 접근, SBX-011 바이너리 네트워크 권한, SBX-012 광범위 파일 글로브 |

### MEDIUM (8개) — 참고

| 카테고리 | 규칙 |
|----------|------|
| 기존 | SEC-012 민감 설정 참조, SEC-020 HTTP 요청, DST-002 단일 삭제, QUA-003~006 품질, QUA-010~011 모호 표현 |
| v2 추가 | SCH-001 name 필드 위반, SCH-002 description 부적절, SCH-003 비표준 필드, SCH-004 크기 초과, SCH-005 디렉토리 구조 |

상세는 `references/` 체크리스트 참조.

### OWASP AST10 매핑

| OWASP | 항목 | 대응 규칙 |
|-------|------|-----------|
| AST01 1.4 | 악성 패턴 검사 | SEC-041 |
| AST01 1.6 | 아이덴티티 파일 보호 | META-001 |
| AST03 3.3 | 셸 접근 제한 | SBX-010 |
| AST03 3.4 | 파일 경로 스코핑 | SBX-012 |
| AST03 3.7 | 네트워크 도메인 허용목록 | SBX-011 |
| AST04 4.1 | 설명 정확성 | SCH-002 |
| AST04 4.2 | 스테가노그래피/인코딩 탐지 | META-002, META-003 |
| AST05 5.1 | 안전한 YAML 로더 | SEC-040 |
| AST05 5.3 | 허용 필드 목록 | SCH-003 |
| AST10 10.6 | 유니버설 스킬 포맷 | SCH-001, SCH-005 |

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
├── .claude-plugin/
│   └── plugin.json             # 플러그인 메타데이터
├── skills/
│   └── skill-security-audit/
│       ├── SKILL.md                    # 메인 스킬 (검사 워크플로우)
│       ├── ruleset-version.txt         # 룰셋 버전 고정
│       ├── references/
│       │   ├── security-checklist.md   # SEC-*, SBX-* 규칙 (+ v2 OWASP 확장)
│       │   ├── destructive-ops-checklist.md  # DST-* 규칙
│       │   ├── quality-checklist.md    # QUA-* 규칙
│       │   ├── metadata-checklist.md   # META-* 규칙 (v2)
│       │   └── spec-compliance-checklist.md  # SCH-* 규칙 (v2)
│       ├── assets/
│       │   ├── report-template.md      # Markdown 보고서 템플릿
│       │   └── slack-message-template.json  # Slack Block Kit 템플릿
│       └── config/
│           └── approved-reviewers.yml  # audit-ignore 승인자 목록
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
