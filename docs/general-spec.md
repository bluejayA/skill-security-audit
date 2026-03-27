# skill-security-audit — General Specification

> **skill-security-audit**은 Claude Code 마켓플레이스에 제출되는 서드파티 스킬을 등록 전에 자동으로 검사하는 게이트키퍼 스킬입니다. 이 문서는 Phase 1(내부 확산기)과 Phase 2(상용 배포기)를 통합한 전체 스펙입니다. 각 항목에 적용 단계가 명시되어 있습니다.

| 항목 | 내용 |
|------|------|
| **버전** | 2.0-general |
| **작성일** | 2026-03-27 |
| **상태** | Reference — Phase 1 + Phase 2 통합본 |
| **작성자** | jay |
| **참고** | Phase 1 운영 스펙은 skill-security-audit-spec-v1.5 참고 |

---

## 1. 개요

### 1.1 목적

제3자가 Claude Code 마켓플레이스에 제출한 스킬을 **보안, 안전성, 품질** 기준으로 자동 검사하는 독립 스킬. 마켓플레이스 게이트키퍼 역할로, 위험한 스킬이 등록되는 것을 사전에 차단한다.

### 1.2 적용 단계 정의

이 스펙의 모든 항목은 적용 단계가 명시되어 있다.

| 단계 | 대상 환경 | 게이트 철학 |
|------|-----------|-------------|
| **Phase 1** | 조직 내부 마켓플레이스, AI 확산기 | 즉각적 실질 피해를 유발하는 것만 차단. 좋은 스킬이 막히는 것을 피한다 |
| **Phase 2** | 외부 공개 마켓플레이스, 상용 환경 | 잠재적 위험까지 차단. 보안 완결성 우선 |

### 1.3 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **Deterministic-First** | 규칙이 판정한다. LLM은 Markdown 문맥 분류에만 사용한다 |
| **Actionable Feedback** | 차단 시 반드시 규칙 ID + 위치 + 수정 예시 제공 |
| **Local = CI** | 동일한 ruleset 버전과 판정 기준 적용. LLM 실행 엔진 차이로 세부 메시지는 달라질 수 있다 |
| **Fail-safe** | Slack 등 부가 기능 실패가 검사 결과에 영향 없음 |

---

## 2. 실행 경로

### 2.1 제출자 자체 실행 (Local)

제출자가 `claude` CLI로 직접 실행. CI와 동일한 ruleset 적용. 출력: Markdown 보고서.

- `SLACK_WEBHOOK_URL` 미설정 시: Slack 전송을 skip하고 보고서 하단에 `(Slack 미설정 — 로컬 실행)` 표기.
- `ruleset-version.txt` 기준으로 ruleset 버전 고정.

### 2.2 GitHub Action 자동 실행

```
PR 생성/업데이트
  → Action 트리거 (skills/** 변경 시)
  → Claude Code → skill-security-audit 로드 → 검사
  → PR comment 업데이트 (기존 봇 댓글 edit, 없으면 신규 생성)
  → Check Run 결과 설정
  → (비동기) Slack 알림
```

- Check Run: BLOCKED → `failure`, 그 외 → `success`
- Slack: `continue-on-error: true`

### 2.3 관리자 수동 실행

`workflow_dispatch` 지원. 특정 PR 번호 또는 스킬 디렉토리 경로 지정 가능.

---

## 3. 분석 방식

### 3.1 설계 원칙: Context-Aware Deterministic-First

> **LLM이 판정하지 않는다. LLM은 문맥을 분류하고, 규칙이 판정한다.**

스킬 파일에는 두 종류의 텍스트가 섞여 있다.

- **지시문 (directive)**: Claude에게 실제 실행을 지시하는 문장
- **설명 텍스트 (descriptive)**: 규칙 설명, 사용 예시, 주석

### 3.2 파일 유형별 분석 전략

| 파일 유형 | 분석 전략 | 이유 |
|-----------|-----------|------|
| `*.sh`, `*.py`, `*.js`, `*.ts` | 패턴 매칭으로 즉시 판정 | 구조화 코드 — 문맥 오탐 가능성 낮음 |
| `Makefile`, `package.json` scripts | 패턴 매칭으로 즉시 판정 | 명시적 실행 구문 |
| `*.md` (SKILL.md, references) | LLM 문맥 분류 후 규칙 적용 | 자연어 — directive/descriptive 구분 필요 |
| `*.json`, `*.yaml` (assets) | 패턴 매칭 (자격증명 중심) | 설정 파일 |

**Markdown 문맥 분류 기준:**

```
해당 구문이 Claude에게 실행을 지시하는가?
  YES (directive)   → 규칙 기본값으로 판정
  NO  (descriptive) → 탐지하지 않음
  애매함             → MEDIUM + "(context uncertain)" 표기
```

**최종 심각도 결정:**

```
구조화 코드에서 패턴 탐지     → 규칙 기본값으로 즉시 판정
Markdown에서 directive 확인   → 규칙 기본값으로 판정
Markdown에서 애매한 경우      → MEDIUM (규칙 기본값에 관계없이)
```

### 3.3 Phase 2 추가: 동적 샌드박스 분석

| 단계 | 방식 | 적용 |
|------|------|------|
| Phase 1 | 정적 분석 (패턴 + LLM 문맥 분류) | ✅ 현재 |
| Phase 2 | strace/sysdig 기반 dry-run 실행 모니터링 | 향후 |
| Phase 2 | 외부 URL 링크 대상 본문 동적 fetch 후 검사 | 향후 |

### 3.4 Self-Audit 테스트

`skill-security-audit` 자신의 `references/` 파일에는 `rm -rf`, `curl | bash` 등 규칙 설명 텍스트가 포함된다. CI에 Self-Audit 테스트를 포함하여 FALSE POSITIVE가 발생하지 않는지 검증한다.

- **통과 조건**: `skill-security-audit` 자신을 입력으로 넣었을 때 FALSE POSITIVE 0건
- 이 테스트가 실패하면 ruleset 배포를 차단한다

---

## 4. 심각도 레벨 및 판정 기준

### 4.1 심각도 정의

| 레벨 | 의미 |
|------|------|
| **CRITICAL** | 즉각적으로 실질 피해 발생 가능. 자격증명 노출·원격 코드 실행·시스템 파괴 |
| **HIGH** | 실질 피해 가능성이 높으나 맥락에 따라 정상일 수 있음 |
| **MEDIUM** | 위험 가능성 있음. 제출자 판단 필요 |
| **LOW** | 품질·가독성 관련 개선 제안 |

### 4.2 판정 로직

| Phase | CRITICAL | HIGH | MEDIUM | LOW |
|-------|----------|------|--------|-----|
| **Phase 1** | ❌ BLOCKED | ⚠️ WARNING (통과) | ⚠️ WARNING (통과) | ℹ️ INFO (통과) |
| **Phase 2** | ❌ BLOCKED | ❌ BLOCKED | ⚠️ WARNING (통과) | ℹ️ INFO (통과) |

```
[Phase 1]
CRITICAL 1개 이상              →  ❌ BLOCKED
HIGH 존재, CRITICAL 없음        →  ⚠️ PASSED with warnings
MEDIUM만 존재                  →  ⚠️ PASSED with warnings
LOW만 존재 또는 발견 없음        →  ✅ PASSED

[Phase 2]
CRITICAL 또는 HIGH 1개 이상    →  ❌ BLOCKED
MEDIUM만 존재                  →  ⚠️ PASSED with warnings
LOW만 존재 또는 발견 없음        →  ✅ PASSED
```

---

## 5. 검사 범위

### 5.1 검사 대상

| 대상 | 검사 내용 |
|------|-----------|
| `SKILL.md` | 구조·품질·보안 패턴 검사 |
| `references/**`, `assets/**` | 보안 패턴 검사 |
| `scripts/**`, `*.sh`, `*.py`, `*.js`, `*.ts` | 보안 + 파괴적 동작 검사 |
| `package.json` (`scripts` 섹션) | 명령어 검사 |
| `Makefile` | 타겟 내 명령어 검사 |

**범위 제한 (Phase 1):**
- 저장소 내 파일만 검사. 외부 URL 링크 대상 본문은 검사하지 않음.

**범위 확장 (Phase 2):**
- 외부 URL 링크 대상 본문을 동적 fetch 후 검사 추가.

### 5.2 검사 제외

| 제외 대상 | 이유 |
|-----------|------|
| `node_modules/`, `.git/` | 생성·관리 파일 |
| 바이너리 파일 (이미지·폰트 등) | 텍스트 분석 불가 |

---

## 6. 검사 규칙 전체 목록

규칙표 범례:
- **P1**: Phase 1 적용 (현재)
- **P2**: Phase 2 적용 또는 강화
- **P1→P2**: Phase 1에서 경고, Phase 2에서 차단으로 상향

---

### 6.1 자격증명 노출 (Credential Exposure)

스킬 파일에 자격증명이 포함되면 저장소에 커밋되는 순간 노출된다.

| 규칙 ID | 검사 항목 | Phase 1 | Phase 2 |
|---------|-----------|---------|---------|
| SEC-010 | 하드코딩된 API 키 (`ghp_`, `sk-`, `xoxb-`, `AKIA`) | 🔴 CRITICAL | 🔴 CRITICAL |
| SEC-010B | JWT 완전한 토큰 패턴 (`eyJ`+`.`+`eyJ` 형태) | 미탐지 | 🔴 CRITICAL |
| SEC-011 | 프라이빗 키 (`BEGIN PRIVATE KEY`, `BEGIN RSA PRIVATE KEY`) | 🔴 CRITICAL | 🔴 CRITICAL |
| SEC-012 | `.env`, `credentials.json`, `token.json` 직접 참조 | 🟡 MEDIUM | 🟠 HIGH |
| SEC-013 | `process.env` 전체 덤프 또는 외부 전송 | 🔴 CRITICAL | 🔴 CRITICAL |

> **SEC-010 비고**: JWT 헤더 prefix(`eyJ`) 단독은 오탐이 많아 Phase 1에서 제외. Phase 2에서 완전한 JWT 토큰 패턴(SEC-010B)으로 탐지.

---

### 6.2 원격 코드 실행 (Remote Code Execution)

외부에서 코드를 가져와 실행하는 패턴.

| 규칙 ID | 검사 항목 | Phase 1 | Phase 2 |
|---------|-----------|---------|---------|
| SEC-003 | 파이프를 통한 원격 실행 (`curl \| bash`, `wget \| sh`) | 🔴 CRITICAL | 🔴 CRITICAL |
| SEC-030 | Base64 디코딩 후 실행 (`echo ... \| base64 -d \| bash`) | 🔴 CRITICAL | 🔴 CRITICAL |
| SEC-032 | `xxd`, `od` 등 바이너리 변환 후 실행 | 미탐지 | 🔴 CRITICAL |

---

### 6.3 셸 인젝션 (Shell Injection)

#### Trusted / Untrusted Input 정의

SEC-001 판정의 핵심 기준이다.

| 구분 | 예시 |
|------|------|
| **Untrusted input** (검증 필요) | CLI argument, prompt input, PR 내용, repository content, 파일 읽기 결과, 환경변수(`process.env`, `$ENV`), 외부 API 응답 |
| **Trusted input** (검증 불필요) | 코드 내부 상수(하드코딩된 값), allowlist로 제한된 값, 검증된 enum/whitelist 값 |

| 규칙 ID | 검사 항목 | Phase 1 | Phase 2 |
|---------|-----------|---------|---------|
| SEC-001 | Untrusted input이 셸 명령에 직접 삽입 | 🔴 CRITICAL / 🟠 HIGH | 🔴 CRITICAL / 🟠 HIGH |
| SEC-002 | `eval`, `exec`, `Function(string)` 문자열 기반 동적 실행 | 🟠 HIGH | 🟠 HIGH |
| SEC-031 | `python -c`, `node -e`, `ruby -e` 인라인 실행 | 미탐지 | 🟠 HIGH* |
| SEC-033 | 문자열 결합으로 명령어 구성 (`'r'+'m'+' '+'-rf'`) | 미탐지 | 🟠 HIGH |
| SEC-034 | 16진수/유니코드 이스케이프를 통한 명령 은닉 | 미탐지 | 🟠 HIGH |

**SEC-001 심각도 판정 조건:**

아래 조건 중 하나를 만족하면 **CRITICAL**:
- `sh -c`, `bash -c`, `subprocess(..., shell=True)` 등 셸 실행 경로에 untrusted input 직접 삽입
- 입력 검증 / allowlist / escaping 없이 untrusted input으로 명령 문자열 직접 구성
- untrusted input이 검증 없이 명령 실행에 연결

위 조건 불충족 (단순 변수 할당, trusted input만 사용): **HIGH**

Markdown에서 애매한 경우: **MEDIUM**

`*` SEC-031은 Phase 2에서 다른 난독화 패턴과 결합된 경우에만 HIGH. 단독 발견 시 MEDIUM.

---

### 6.4 민감 경로 접근 (Sensitive Path Access)

| 규칙 ID | 검사 항목 | Phase 1 | Phase 2 |
|---------|-----------|---------|---------|
| SBX-001 | 스킬 디렉토리 외부에 파일 쓰기·편집 지시 | 🟠 HIGH | 🔴 CRITICAL |
| SBX-002 | 허용 경로 외 절대경로 사용 (`/etc/`, `/usr/` 등) | 미탐지 | 🟠 HIGH |
| SBX-003 | `..` 경로 탈출 (`../../etc/passwd`) | 🔴 CRITICAL | 🔴 CRITICAL |
| SBX-004 | 홈 디렉토리 민감 경로 참조 (`~/.ssh`, `~/.aws`, `~/.gnupg`) | 🔴 CRITICAL | 🔴 CRITICAL |
| SBX-005 | 심볼릭 링크를 통한 경로 탈출 (`ln -s`) | 미탐지 | 🔴 CRITICAL |
| SBX-006 | 경로 정규화 우회 (`../`, URL 인코딩) | 미탐지 | 🟠 HIGH |
| SBX-007 | 키체인·히스토리 읽기 (`~/.bash_history`, `security find-generic-password`) | 🔴 CRITICAL | 🔴 CRITICAL |

**SBX-002 허용 절대경로 Allowlist (Phase 2 기준):**

| 허용 경로 | 설명 |
|-----------|------|
| `/tmp`, `/private/tmp`, `/var/tmp` | 임시 파일 (Linux/macOS) |
| `/workspace`, `/home/runner/work/**` | CI 런타임 작업 공간 |
| `$TMPDIR` 확장 경로 | 런타임 임시 디렉토리 |

---

### 6.5 네트워크 접근 (Network Access)

> **마켓플레이스 기본 정책**: 스킬은 저장소 내 자료만 사용하는 것을 원칙으로 한다. 원격 코드/콘텐츠 가져오기는 제한하며, 필요 시 audit-ignore 승인으로만 허용한다.

| 규칙 ID | 검사 항목 | Phase 1 | Phase 2 |
|---------|-----------|---------|---------|
| SEC-020 | 외부 HTTP 요청 (`WebFetch`, `curl`, `wget`) — 기본 | 🟡 MEDIUM | 🟡 MEDIUM |
| SEC-020H | 외부 HTTP 요청 + 자격증명·환경변수·PII 페이로드 포함 | 🟠 HIGH | 🟠 HIGH |
| SEC-021 | `git clone`, `git fetch` 원격 저장소 접근 | 미탐지 | 🟠 HIGH |
| SEC-022 | `ssh`, `nc`, `netcat` 직접 네트워크 도구 | 🟠 HIGH | 🔴 CRITICAL |
| SEC-023 | DNS 기반 데이터 유출 (`nslookup`, `dig`) | 미탐지 | 🟠 HIGH |
| SEC-024 | 패키지 매니저 외부 설치 (`npm install`, `pip install`) | 미탐지 | 🟠 HIGH |

---

### 6.6 파괴적 동작 (Destructive Operations)

| 규칙 ID | 검사 항목 | Phase 1 | Phase 2 |
|---------|-----------|---------|---------|
| DST-001 | 재귀적 삭제 (`rm -rf`, `rimraf`, `shutil.rmtree`) | 🔴 CRITICAL | 🔴 CRITICAL |
| DST-002 | 단일 파일 삭제 (`rm`, `unlink`) — 재귀 아닌 경우 | 🟡 MEDIUM | 🟡 MEDIUM |
| DST-003 | `git push --force` (원격 히스토리 파괴) | 🟠 HIGH | 🟠 HIGH |
| DST-004 | `git reset --hard`, `git clean -f` | 미탐지 | 🟠 HIGH |
| DST-005 | 프로세스 종료 (`kill`, `pkill`, `killall`) | 미탐지 | 🟡 MEDIUM |
| DST-006 | 패키지 제거 (`npm uninstall`, `pip uninstall`) | 미탐지 | 🟡 MEDIUM |
| DST-007 | 시스템 수준 권한·커널·소유권 변경 (`chmod 777`, `chown root`, `sysctl -w`) | 🔴 CRITICAL | 🔴 CRITICAL |
| DST-008 | 민감 파일 overwrite (`> .env`, `> *.pem`, `> *.key`) | 미탐지 | 🟠 HIGH |

---

### 6.7 최소 품질 기준 (Quality)

| 규칙 ID | 검사 항목 | Phase 1 | Phase 2 |
|---------|-----------|---------|---------|
| QUA-001 | `SKILL.md` 파일 존재 | 🔴 CRITICAL | 🔴 CRITICAL |
| QUA-002 | frontmatter 필수 필드 (`name`, `description`) 존재 | 🟠 HIGH | 🟠 HIGH |
| QUA-003 | `description`이 `"Use when"`으로 시작 | 🟡 MEDIUM | 🟡 MEDIUM |
| QUA-004 | `description` 길이 1024자 이하 | 🟡 MEDIUM | 🟡 MEDIUM |
| QUA-005 | `SKILL.md` 본문 500줄 이하 | 🟡 MEDIUM | 🟡 MEDIUM |
| QUA-006 | 참조 파일 깊이 1단계까지 | 🟡 MEDIUM | 🟡 MEDIUM |
| QUA-010 | 모호 표현 사용 (`적절히`, `잘`, `필요 시`) | 🟡 MEDIUM | 🟡 MEDIUM |
| QUA-011 | `description`에 내부 구현 상세 노출 | 🟡 MEDIUM | 🟡 MEDIUM |
| QUA-012 | 한국어/영어 키워드 병기 (CSO 트리거 품질) | 미탐지 | 🔵 LOW |
| QUA-013 | Examples 섹션 존재 (최소 2개) | 미탐지 | 🔵 LOW |
| QUA-014 | Troubleshooting/Error Handling 섹션 존재 | 미탐지 | 🔵 LOW |
| QUA-015 | 토큰 수 4,000 이하 (컨텍스트 로드 비용) | 미탐지 | 🟡 MEDIUM |

---

## 7. 예외 처리 (audit-ignore)

### 7.1 선언 방식

```yaml
---
name: my-deployment-skill
description: "Use when deploying to production..."
audit-ignore:
  - rule: DST-003
    reason: "배포 파이프라인 스킬 — git push --force가 핵심 기능"
    reviewer: "@admin"
    expires: "2026-09-30"
---
```

### 7.2 필수 필드

| 필드 | 필수 | 설명 |
|------|:----:|------|
| `rule` | ✅ | 예외 대상 규칙 ID |
| `reason` | ✅ | 구체적 사유 |
| `reviewer` | ✅ | 마켓플레이스 관리자 `@handle` |
| `expires` | ✅ | 만료일 — 최대 6개월 |

### 7.3 승인 검증

| 단계 | 검증 방식 |
|------|-----------|
| **Phase 1** | `config/approved-reviewers.yml` 목록 대조 |
| **Phase 2** | 서명된 승인 레코드 + Policy-as-data (Rego) |

**Phase 1 검증 흐름:**

```
audit-ignore 선언 발견
  → reviewer가 approved-reviewers.yml에 있는가?  NO → 예외 무효
  → 만료일이 유효한가?                           NO → 예외 무효
  → 모두 통과 → EXEMPTED로 보고
```

### 7.4 검사 시 처리

- 유효한 예외: 보고서에 **EXEMPTED**로 표시 (숨기지 않음)
- 만료된 예외: 원래 심각도로 보고
- 미승인 reviewer: 예외 무효

### 7.5 만료 사전 경고 (D-30)

만료 30일 전 Slack 경고 발송.

```
⏰ audit-ignore 만료 예정 — "my-deployment-skill"

규칙: DST-003  |  만료: 2026-09-30 (D-28)  |  승인자: @admin
갱신하지 않으면 해당 규칙이 재활성화됩니다.
→ https://github.com/org/marketplace/pull/42
```

---

## 8. 오탐 이의제기

| 항목 | Phase 1 | Phase 2 |
|------|---------|---------|
| **요청 방법** | `audit-review-requested` 라벨 또는 PR comment 템플릿 | 봇 명령 (`@marketplace-bot false-positive SEC-001`) |
| **처리 방식** | 관리자 수동 검토 후 승인/거부 | DISPUTED/DISMISSED 상태 머신 자동화 |
| **Check Run** | 관리자 승인 후 수동 override | 봇이 자동 override |
| **횟수 제한** | 없음 | PR당 규칙 ID별 3회 |

**Phase 1 요청 형식:**

```
재검토 요청: SEC-001
사유: 해당 구문은 사용 예시 설명이며 실행 지시가 아닙니다.
```

**Phase 2 요청 형식:**

```
@marketplace-bot false-positive SEC-001
사유: 해당 구문은 사용 예시 설명이며 실행 지시가 아닙니다.
```

---

## 9. 멀티 스킬 PR 처리

### 9.1 스캔 단위

- 변경된 파일이 속한 스킬 디렉토리 단위로 독립 검사
- 스킬별로 별도 Check Run 생성

### 9.2 PR comment 전략

기존 봇 댓글을 찾아 업데이트 (스팸 방지). 식별자: `<!-- skill-audit-report -->`.

```markdown
<!-- skill-audit-report -->
## Skill Audit Results

### ❌ auto-deploy-helper — BLOCKED
...

### ⚠️ api-connector — PASSED with warnings
...

### ✅ markdown-formatter — PASSED
...
```

### 9.3 최종 판정

| 조건 | PR 결과 |
|------|---------|
| 하나라도 BLOCKED | ❌ failure |
| BLOCKED 없음, WARNING 존재 | ✅ success (warnings 표시) |
| 전체 PASSED | ✅ success |

---

## 10. 출력 형식

### 10.1 Markdown 보고서

```markdown
# Skill Security Audit Report

**Skill**: my-skill-name
**Submitted by**: @username
**Audit date**: 2026-03-27
**Ruleset version**: v2.0.0 (sha: a1b2c3d)
**Phase**: 1
**Verdict**: ❌ BLOCKED

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | 1     |
| HIGH     | 1     |
| MEDIUM   | 2     |

## Findings

### 🔴 CRITICAL — 차단 사유

#### [SEC-010] Hardcoded API key
- **File**: references/config.md:12
- **Evidence**: `sk-...`
- **Fix**: 자격증명을 파일에 직접 포함하지 마세요. 환경변수 또는 시크릿 매니저를 사용하세요.

### 🟠 HIGH — 반드시 확인 (차단하지 않음)

#### [DST-003] git push --force
- **File**: scripts/deploy.sh:8
- **Note**: 원격 히스토리를 파괴할 수 있습니다. 의도된 동작이면 audit-ignore로 예외 선언하세요.

### 🟡 MEDIUM — 참고 사항

#### [SEC-020] External HTTP request
- **File**: SKILL.md:30
- **Note**: 외부 HTTP 요청이 감지되었습니다. 의도된 동작이면 무시해도 됩니다.
```

### 10.2 Machine-readable JSON

보고서와 별도로 `audit-result.json`을 Actions 아티팩트로 저장.

```json
{
  "version": "2.0.0",
  "phase": 1,
  "ruleset_sha": "a1b2c3d",
  "skill": "my-skill-name",
  "verdict": "BLOCKED",
  "counts": { "critical": 1, "high": 1, "medium": 2, "low": 0, "exempted": 0 },
  "findings": [
    {
      "id": "SEC-010",
      "severity": "CRITICAL",
      "file": "references/config.md",
      "line": 12,
      "message": "Hardcoded API key",
      "fix": "환경변수 또는 시크릿 매니저를 사용하세요."
    }
  ]
}
```

### 10.3 Phase 2 추가: SARIF 출력

GitHub Code Scanning UI 연동을 위한 SARIF 형식 출력. Phase 2에서 구현.

---

## 11. Slack 알림

| 항목 | Phase 1 | Phase 2 |
|------|---------|---------|
| **전송 정책** | BLOCKED만 즉시 전송 | BLOCKED 즉시 + PASSED 일일 요약 집계 |
| **실패 처리** | `continue-on-error: true` | `continue-on-error: true` |

**BLOCKED 알림:**

```
❌ Skill Audit BLOCKED — "data-sync"
제출자: @username | PR: #42

CRITICAL (1): [SEC-010] references/config.md:12 — Hardcoded API key
HIGH (1):     [DST-003] scripts/deploy.sh:8 — git push --force

→ https://github.com/org/marketplace/pull/42
```

---

## 12. GitHub Actions 통합

### 12.1 트리거

```yaml
on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - 'skills/**'
  workflow_dispatch:
    inputs:
      skill_path:
        description: '검사할 스킬 디렉토리 경로'
        required: true
```

### 12.2 워크플로우 구조

```
Job: skill-audit
├── Step 1: Checkout
├── Step 2: Detect changed skill directories
├── Step 3: Run skill-security-audit — 스킬별 병렬 실행
├── Step 4: Update PR comment (기존 봇 댓글 edit, 없으면 신규)
├── Step 5: Set Check Run per skill
└── Step 6: Slack notification + D-30 만료 경고 (continue-on-error: true)
```

---

## 13. 룰셋 버전 정책

- `ruleset-version.txt`로 버전 고정. 보고서에 버전 + SHA 기록.

| 변경 유형 | 재검사 여부 | 처리 |
|-----------|:-----------:|------|
| 신규 CRITICAL 규칙 추가 | ✅ 필요 | 배포 후 30일 내 기존 스킬 재검사. 실패 시 `REVIEW_REQUIRED` 표시 |
| 신규 HIGH 규칙 추가 (Phase 2) | ✅ 필요 | 동일 |
| MEDIUM/LOW 추가·변경 | 권장 | 다음 PR 제출 시 자동 적용 |
| 심각도 하향 | ❌ 불필요 | — |
| 버그 픽스·문구 변경 | ❌ 불필요 | — |

---

## 14. 프로젝트 구조

### 14.1 스킬 패키지

```
skills/skill-security-audit/
├── SKILL.md
├── ruleset-version.txt
├── references/
│   ├── security-checklist.md
│   ├── destructive-ops-checklist.md
│   └── quality-checklist.md
├── assets/
│   ├── report-template.md
│   └── slack-message-template.json
└── README.md
```

### 14.2 마켓플레이스 운영 인프라 (별도 관리)

```
marketplace-repo/
├── .github/workflows/skill-audit.yml
└── config/approved-reviewers.yml
```

---

## 15. 스킬 메타데이터

```yaml
---
name: skill-security-audit
description: >
  Use when auditing a third-party Claude Code skill for security risks,
  destructive operations, and minimum quality standards.
  Trigger on "스킬 검사", "스킬 보안 검토", "skill audit", "security review",
  "스킬 등록 검증", "marketplace submission review", "제출 스킬 검사".
metadata:
  version: 2.0.0
  author: jay
  category: security
  invoke_mode:
    - reviewer        # 주 호출 경로: PR/CI 컨텍스트 게이트키퍼
    - user-invocable  # 보조 호출 경로: 제출자 로컬 사전 검사
  return_behavior: report
---
```

---

## 16. 통과/실패 예시

### 16.1 PASSED

```
skills/markdown-formatter/
├── SKILL.md
└── references/rules.md

→ ✅ PASSED (0 findings)
```

### 16.2 BLOCKED

```
skills/data-sync/
├── SKILL.md          # curl $URL | bash 포함
└── references/config.md   # sk-xxxxxxxx 하드코딩

→ ❌ BLOCKED
  SEC-003  CRITICAL  SKILL.md:45              curl pipe to bash
  SEC-010  CRITICAL  references/config.md:12  Hardcoded API key
```

### 16.3 PASSED with warnings

```
skills/deploy-helper/
├── SKILL.md
└── scripts/deploy.sh   # git push --force (audit-ignore 선언)

→ ⚠️ PASSED with warnings
  DST-003  EXEMPTED  (reviewer: @admin, exp: 2026-09-30)
  SBX-001  HIGH      SKILL.md:18  Write outside skill directory
  SEC-020  MEDIUM    SKILL.md:30  External HTTP request
```

---

## 17. 구현 로드맵

### Phase 1 우선순위

| 항목 | 설명 | 우선순위 |
|------|------|---------|
| Self-Audit CI 테스트 | ruleset 배포 전 FALSE POSITIVE 자동 검증 | P1 |
| PR comment 봇 (업데이트 방식) | `<!-- skill-audit-report -->` 식별, 댓글 스팸 방지 | P1 |
| D-30 만료 경고 | Slack 자동 경고 | P2 |
| `audit-result.json` 아티팩트 | Actions artifact 저장 | P2 |

### Phase 2 우선순위

| 항목 | 설명 | 우선순위 |
|------|------|---------|
| SARIF 출력 | GitHub Code Scanning UI 연동 | P1 |
| 룰셋 해시 체인 강화 | 감사 대응용 전체 룰셋 해시 기록 | P1 |
| BLOCKED 수정 체크리스트 | PR comment에 재제출 가이드 자동 생성 | P2 |
| 동적 샌드박스 | strace/sysdig 기반 dry-run 실행 모니터링 | P2 |
| Policy-as-data | YAML/Rego 기반 룰 엔진 전환 | P2 |
| 오탐 이의제기 자동화 | 봇 명령, DISPUTED/DISMISSED 상태 머신 | P2 |
| Finding 그룹핑 | 공격 체인 단위 그룹 표시 | P2 |
| 보안 점수/배지 | 100점 기반 점수 + README 배지 SVG | P3 |
| PASSED 일일 Slack 집계 | PASSED 건 하루 1회 요약 전송 | P3 |
| allowlist 외부화 | `allowlist.yml` 환경별 오버라이드 | P3 |
