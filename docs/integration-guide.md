# Integration Guide

기존 마켓플레이스 저장소에 skill-security-audit을 통합하는 방법을 설명한다.

---

## 목차

1. [사전 요구사항](#1-사전-요구사항)
2. [스킬 추가](#2-스킬-추가)
3. [워크플로우 설치](#3-워크플로우-설치)
4. [GitHub Secrets 설정](#4-github-secrets-설정)
5. [승인자 설정](#5-승인자-설정)
6. [경로 커스터마이징](#6-경로-커스터마이징)
7. [동작 확인](#7-동작-확인)
8. [Submodule 업데이트](#8-submodule-업데이트)
9. [고급 설정](#9-고급-설정)

---

## 1. 사전 요구사항

| 항목 | 요구사항 |
|------|----------|
| GitHub 저장소 | 마켓플레이스 저장소에 admin 또는 write 권한 |
| Anthropic API Key | Claude API 호출용 키 |
| Claude Code CLI | `npm install -g @anthropic-ai/claude-code` (CI에서 자동 설치됨) |
| 스킬 디렉토리 구조 | 제출된 스킬이 단일 루트 디렉토리 하위에 위치 (예: `skills/`) |

---

## 2. 스킬 추가

마켓플레이스 저장소에 감사 스킬을 추가한다. 두 가지 방법 중 선택:

### 방법 A: Git Submodule (권장)

버전 추적이 가능하고 업데이트가 용이하다.

```bash
cd your-marketplace-repo
git submodule add https://github.com/bluejayA/skill-security-audit.git skills/skill-security-audit
git commit -m "chore: add skill-security-audit as submodule"
```

### 방법 B: 직접 복사

submodule을 사용하지 않는 조직에 적합하다.

```bash
cd your-marketplace-repo
git clone https://github.com/bluejayA/skill-security-audit.git /tmp/skill-security-audit
cp -r /tmp/skill-security-audit skills/skill-security-audit

# 불필요한 파일 제거
rm -rf skills/skill-security-audit/.git
rm -rf skills/skill-security-audit/.github
rm -rf skills/skill-security-audit/devflow-docs

git add skills/skill-security-audit
git commit -m "chore: add skill-security-audit"
```

### 최종 디렉토리 구조

```
your-marketplace-repo/
├── skills/
│   ├── skill-security-audit/     ← 감사 스킬
│   │   ├── SKILL.md
│   │   ├── references/
│   │   │   ├── security-checklist.md
│   │   │   ├── destructive-ops-checklist.md
│   │   │   └── quality-checklist.md
│   │   ├── assets/
│   │   │   ├── report-template.md
│   │   │   └── slack-message-template.json
│   │   ├── config/
│   │   │   └── approved-reviewers.yml
│   │   └── ruleset-version.txt
│   ├── submitted-skill-a/        ← 제출된 스킬들
│   ├── submitted-skill-b/
│   └── ...
└── .github/
    └── workflows/
        └── skill-audit.yml      ← 워크플로우
```

---

## 3. 워크플로우 설치

감사 스킬에 포함된 워크플로우를 마켓플레이스 저장소의 `.github/workflows/`에 복사한다.

```bash
mkdir -p .github/workflows
cp skills/skill-security-audit/.github/workflows/skill-audit.yml .github/workflows/
```

> **주의**: 방법 B(직접 복사)로 설치한 경우 감사 스킬 디렉토리의 `.github/`는 이미 제거했으므로, 원본 저장소에서 직접 가져온다:
>
> ```bash
> curl -o .github/workflows/skill-audit.yml \
>   https://raw.githubusercontent.com/bluejayA/skill-security-audit/main/.github/workflows/skill-audit.yml
> ```

워크플로우가 요구하는 권한:

```yaml
permissions:
  contents: read        # 코드 읽기
  pull-requests: write  # PR comment 업데이트
  checks: write         # Check Run 결과 설정
```

---

## 4. GitHub Secrets 설정

저장소 **Settings → Secrets and variables → Actions**에서 설정한다.

| Secret | 필수 | 용도 |
|--------|------|------|
| `ANTHROPIC_API_KEY` | **필수** | Claude API 호출 |
| `SLACK_WEBHOOK_URL` | 선택 | BLOCKED 시 Slack 알림 전송 |

### API Key 발급

1. [Anthropic Console](https://console.anthropic.com/)에서 API Key를 생성한다
2. GitHub Secrets에 `ANTHROPIC_API_KEY`로 등록한다

### Slack Webhook 설정 (선택)

1. Slack 워크스페이스에서 **Incoming Webhook** 앱을 추가한다
2. 알림을 받을 채널을 선택하고 Webhook URL을 생성한다
3. GitHub Secrets에 `SLACK_WEBHOOK_URL`로 등록한다

---

## 5. 승인자 설정

`skills/skill-security-audit/config/approved-reviewers.yml`을 마켓플레이스 관리자에 맞게 수정한다.

```yaml
# config/approved-reviewers.yml
approved_reviewers:
  - "@your-admin"
  - "@your-security-lead"
  - "@your-marketplace-maintainer"
```

이 목록에 있는 사람만 `audit-ignore` 예외를 승인할 수 있다. 제출자가 frontmatter에 `reviewer: "@someone"`을 선언해도, 이 목록에 없으면 예외가 무효 처리된다.

---

## 6. 경로 커스터마이징

기본 설정은 스킬이 `skills/` 디렉토리에 위치한다고 가정한다. 다른 경로를 사용하는 경우 워크플로우를 수정해야 한다.

### 스킬 디렉토리가 `skills/`가 아닌 경우

예: `marketplace/plugins/` 하위에 스킬이 위치하는 경우

**skill-audit.yml에서 수정할 곳 3곳:**

```yaml
# 1. 트리거 경로
on:
  pull_request:
    paths:
      - 'marketplace/plugins/**'    # ← 변경

# 2. 변경된 스킬 디렉토리 감지
- name: Detect changed skills
  run: |
    CHANGED_SKILLS=$(git diff --name-only ... \
      | grep '^marketplace/plugins/' \     # ← 변경
      | cut -d'/' -f1-3 \                  # ← depth 조정 (3단계)
      | sort -u \
      | tr '\n' ' ')

# 3. ruleset 버전 참조 경로
RULESET_VERSION=$(cat marketplace/plugins/skill-security-audit/ruleset-version.txt)  # ← 변경
```

### 감사 스킬 자체의 위치가 다른 경우

감사 스킬이 `skills/skill-security-audit/`가 아닌 다른 경로에 있다면, 워크플로우의 Claude 프롬프트에서 경로를 수정한다:

```yaml
# audit 실행 프롬프트
RESULT=$(claude --print \
  "your/custom/path/skill-security-audit 스킬을 사용하여 ...")  # ← 경로 변경
```

---

## 7. 동작 확인

### 테스트 PR로 검증

```bash
# 테스트 브랜치 생성
git checkout -b test/audit-integration

# 테스트용 더미 스킬 추가
mkdir -p skills/test-dummy
cat > skills/test-dummy/SKILL.md << 'EOF'
---
name: test-dummy
description: "Use when testing the audit integration"
---

# Test Dummy

테스트용 더미 스킬이다. 아무 동작도 하지 않는다.
EOF

# 커밋 & 푸시
git add skills/test-dummy
git commit -m "test: add dummy skill for audit integration test"
git push -u origin test/audit-integration

# PR 생성
gh pr create --title "test: audit integration" --body "감사 스킬 통합 테스트"
```

### 확인 사항

1. **Actions 탭**: "Skill Security Audit" 워크플로우가 트리거되었는지 확인
2. **PR Comment**: 감사 보고서가 댓글로 추가되었는지 확인
3. **Check Run**: PR의 Checks 탭에 "Skill Security Audit" 결과가 표시되는지 확인
4. **판정 결과**: 더미 스킬이 `✅ PASSED`인지 확인

### 테스트 후 정리

```bash
# 테스트 PR 닫기 & 브랜치 삭제
gh pr close test/audit-integration --delete-branch
```

---

## 8. Submodule 업데이트

방법 A(submodule)로 설치한 경우, 감사 스킬을 최신 버전으로 업데이트하는 방법:

```bash
cd your-marketplace-repo

# 최신 버전으로 업데이트
git submodule update --remote skills/skill-security-audit

# 변경 확인
cd skills/skill-security-audit
git log --oneline -3
cd ../..

# 커밋
git add skills/skill-security-audit
git commit -m "chore: update skill-security-audit to latest"
```

### 특정 버전으로 고정

```bash
cd skills/skill-security-audit
git checkout v1.0.0    # 특정 태그
cd ../..
git add skills/skill-security-audit
git commit -m "chore: pin skill-security-audit to v1.0.0"
```

---

## 9. 고급 설정

### 감사 스킬 변경과 일반 스킬 변경 분리

감사 스킬 자체가 변경될 때는 Self-Audit만 실행하고, 일반 스킬 변경 시에만 감사를 실행하도록 분리할 수 있다:

```yaml
# .github/workflows/skill-audit.yml
- name: Detect changed skills
  run: |
    # skill-security-audit 자체 변경은 별도 처리
    CHANGED_SKILLS=$(git diff --name-only ... \
      | grep '^skills/' \
      | grep -v '^skills/skill-security-audit/' \  # ← 감사 스킬 제외
      | cut -d'/' -f1-2 \
      | sort -u \
      | tr '\n' ' ')
```

### 특정 스킬만 검사 (수동 실행)

`workflow_dispatch`로 특정 스킬을 지정하여 수동 검사할 수 있다:

1. GitHub Actions 탭 → "Skill Security Audit" 워크플로우 선택
2. "Run workflow" 클릭
3. `skill_path`에 검사할 경로 입력 (예: `skills/my-new-skill`)

### BLOCKED 시 PR 머지 차단

GitHub Branch Protection을 설정하여 BLOCKED 판정 시 머지를 물리적으로 차단할 수 있다:

1. 저장소 **Settings → Branches → Branch protection rules**
2. `main` 브랜치에 규칙 추가
3. **Require status checks to pass before merging** 활성화
4. **Status checks**: "Skill Security Audit" 추가

이렇게 설정하면 CRITICAL finding이 있는 PR은 머지 버튼이 비활성화된다.

### CI 비용 최적화

Claude API 호출 비용을 줄이려면:

- `paths` 필터를 정확히 설정하여 불필요한 트리거를 방지한다
- `workflow_dispatch`를 활용하여 필요할 때만 수동 실행한다
- 대규모 PR은 스킬 단위로 나누어 제출한다
