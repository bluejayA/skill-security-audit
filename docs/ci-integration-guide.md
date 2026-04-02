# CI Integration Guide

Add automated skill security auditing to your GitHub repository using GitHub Actions.

## Two Integration Modes

| Mode | When to use | Trigger |
|------|-------------|---------|
| **Direct** | Skills are submitted directly to your repo's `skills/` directory | `pull_request` on `skills/**` |
| **Remote** | Plugins are registered via URL in a `marketplace.json` file | `pull_request` on `marketplace.json` |

You can use one or both depending on your setup.

---

## Option A: Direct Skill Auditing

For repos where skill files are submitted directly (e.g., a marketplace repo with `skills/` directory).

### 1. Add skill-security-audit as a submodule

```bash
cd your-repo
git submodule add https://github.com/bluejayA/skill-security-audit.git _audit-tool
git commit -m "chore: add skill-security-audit submodule"
```

### 2. Create the workflow

Create `.github/workflows/skill-audit-direct.yml`:

```yaml
name: "Skill Audit: Direct Submission"

on:
  pull_request:
    types: [opened, synchronize, reopened]
    paths:
      - 'skills/**'

concurrency:
  group: skill-audit-direct-${{ github.event.pull_request.number || github.run_id }}
  cancel-in-progress: true

jobs:
  scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    outputs:
      verdict: ${{ steps.audit.outputs.verdict }}
      skill_count: ${{ steps.detect.outputs.count }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Checkout audit tool from base branch
        env:
          BASE_SHA: ${{ github.event.pull_request.base.sha }}
        run: |
          git checkout "$BASE_SHA" -- _audit-tool .gitmodules
          git submodule update --init _audit-tool

      - name: Detect changed skills
        id: detect
        env:
          BASE_SHA: ${{ github.event.pull_request.base.sha }}
          HEAD_SHA: ${{ github.sha }}
        run: |
          CHANGED_SKILLS=$(git diff --name-only "$BASE_SHA" "$HEAD_SHA" \
            | grep '^skills/' \
            | grep -v '^skills/\.gitkeep' \
            | cut -d'/' -f1-2 \
            | sort -u \
            | tr '\n' ' ')
          COUNT=$(echo "$CHANGED_SKILLS" | xargs -n1 2>/dev/null | wc -l | tr -d ' ')
          echo "skills=$CHANGED_SKILLS" >> "$GITHUB_OUTPUT"
          echo "count=$COUNT" >> "$GITHUB_OUTPUT"

      - name: Install Claude CLI
        if: steps.detect.outputs.count != '0'
        run: npm install -g @anthropic-ai/claude-code@latest

      - name: Run audit
        if: steps.detect.outputs.count != '0'
        id: audit
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          SKILL_LIST: ${{ steps.detect.outputs.skills }}
          PR_ACTOR: ${{ github.actor }}
        run: |
          OVERALL_VERDICT="PASSED"
          REPORT=""
          AUDIT_PATH="_audit-tool/skills/skill-security-audit"

          for SKILL_DIR in $SKILL_LIST; do
            RESULT=$(claude --print \
              "${AUDIT_PATH} 스킬을 사용하여 ${SKILL_DIR} 스킬을 검사해줘. PR 제출자: ${PR_ACTOR}. JSON 결과도 함께 출력해줘." \
              2>&1)
            CLAUDE_EXIT=$?

            if [ $CLAUDE_EXIT -ne 0 ] || [ -z "$RESULT" ]; then
              OVERALL_VERDICT="BLOCKED"
              continue
            fi

            VERDICT_JSON=$(echo "$RESULT" | grep -oP '"verdict"\s*:\s*"\K[^"]+' | tail -1 || true)
            if [ "$VERDICT_JSON" = "BLOCKED" ]; then
              OVERALL_VERDICT="BLOCKED"
            elif [ "$VERDICT_JSON" = "PASSED_WITH_WARNINGS" ] && [ "$OVERALL_VERDICT" != "BLOCKED" ]; then
              OVERALL_VERDICT="PASSED_WITH_WARNINGS"
            fi

            REPORT="${REPORT}$(printf '\n%s\n' "$RESULT")"
          done

          echo "$REPORT" > /tmp/audit-report.md
          echo "verdict=$OVERALL_VERDICT" >> "$GITHUB_OUTPUT"

      - name: Upload report
        if: steps.detect.outputs.count != '0'
        uses: actions/upload-artifact@v4
        with:
          name: audit-report
          path: /tmp/audit-report.md
          retention-days: 30

  report:
    needs: scan
    if: needs.scan.outputs.skill_count != '0'
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      checks: write
    steps:
      - name: Download report
        uses: actions/download-artifact@v4
        with:
          name: audit-report
          path: /tmp

      - name: Update PR comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('/tmp/audit-report.md', 'utf8');
            const verdict = '${{ needs.scan.outputs.verdict }}';
            const marker = '<!-- skill-audit -->';
            const icon = verdict === 'BLOCKED' ? '❌' : verdict === 'PASSED_WITH_WARNINGS' ? '⚠️' : '✅';
            const body = `${marker}\n## ${icon} Skill Audit Results\n\n**Verdict: ${verdict}**\n\n${report}`;
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner, repo: context.repo.repo,
              issue_number: context.issue.number,
            });
            const existing = comments.find(c => c.body.includes(marker));
            const params = { owner: context.repo.owner, repo: context.repo.repo, body };
            if (existing) {
              await github.rest.issues.updateComment({ ...params, comment_id: existing.id });
            } else {
              await github.rest.issues.createComment({ ...params, issue_number: context.issue.number });
            }

      - name: Set check status
        uses: actions/github-script@v7
        with:
          script: |
            const verdict = '${{ needs.scan.outputs.verdict }}';
            await github.rest.checks.create({
              owner: context.repo.owner, repo: context.repo.repo,
              name: 'Skill Security Audit',
              head_sha: '${{ github.event.pull_request.head.sha }}',
              status: 'completed',
              conclusion: verdict === 'BLOCKED' ? 'failure' : 'success',
              output: {
                title: verdict === 'BLOCKED' ? '❌ BLOCKED' : verdict === 'PASSED_WITH_WARNINGS' ? '⚠️ PASSED with warnings' : '✅ PASSED',
                summary: `Verdict: ${verdict}`,
              },
            });
```

### 3. Add GitHub Secrets

Go to **Settings → Secrets and variables → Actions → Repository secrets**:

| Secret | Required | Purpose |
|--------|----------|---------|
| `ANTHROPIC_API_KEY` | **Yes** | Claude API calls |
| `SLACK_WEBHOOK_URL` | No | Slack notifications on BLOCKED |

### 4. (Optional) Branch Protection

Add `Skill Security Audit` as a required status check in branch protection rules.

> **Note:** If using `paths:` filter, use a gate workflow that always runs, and set that as required. See the devflow-marketplace repo for a reference implementation.

---

## Option B: Remote Plugin Auditing

For marketplace repos where plugins are registered via URL in `marketplace.json`.

### marketplace.json schema

Each plugin entry must include a `revision` field (full commit SHA):

```json
{
  "name": "my-plugin",
  "source": {
    "source": "url",
    "url": "https://github.com/owner/repo.git"
  },
  "revision": "abc123def456...",
  "description": "...",
  "version": "1.0.0"
}
```

### Workflow overview

The Remote workflow:
1. Parses `marketplace.json` diff (base vs head, name-keyed map comparison)
2. Validates URLs against allowlist (`https://github.com/` only)
3. Validates plugin names (alphanumeric, hyphens, underscores only)
4. Clones each changed plugin repo (`--depth 1`, no submodules, timeout 120s)
5. Checks out the specified `revision` SHA
6. Finds `skills/*/SKILL.md` in the cloned repo
7. Runs audit on each skill
8. Posts results to PR

For the full workflow file, see [`devflow-marketplace/.github/workflows/skill-audit-remote.yml`](https://github.com/bluejayA/devflow-marketplace/blob/main/.github/workflows/skill-audit-remote.yml).

### Security measures

| Measure | Purpose |
|---------|---------|
| **2-job split** (scan → report) | Isolate untrusted code execution from PR write permissions |
| **Base branch audit tool** | Prevent submodule pointer tampering in PRs |
| **URL allowlist** | Block `file://`, `ssh://`, localhost, IP-based URLs |
| **Revision pinning** | Ensure reproducible audits (checkout fails → BLOCKED) |
| **Fail-Closed** | Claude CLI failure → BLOCKED (never silent PASSED) |
| **Input validation** | Regex validation on plugin names and skill paths |
| **Concurrency** | `cancel-in-progress` prevents duplicate runs per PR |

---

## Cost Estimate

| Component | Tokens |
|-----------|--------|
| Audit tool (skill + references) | ~17,000 input |
| Target skill (recommended max) | ~15,000 input |
| System prompt + instructions | ~500 input |
| Output (report + JSON) | ~3,000 output |

**Per audit (Sonnet):** ~$0.14 | **Monthly (20 PRs, 2 skills each):** ~$5.60

---

## Reference Implementation

The [devflow-marketplace](https://github.com/bluejayA/devflow-marketplace) repo demonstrates both Direct and Remote workflows with full security hardening:

- [`skill-audit-direct.yml`](https://github.com/bluejayA/devflow-marketplace/blob/main/.github/workflows/skill-audit-direct.yml)
- [`skill-audit-remote.yml`](https://github.com/bluejayA/devflow-marketplace/blob/main/.github/workflows/skill-audit-remote.yml)
- [`skill-audit-gate.yml`](https://github.com/bluejayA/devflow-marketplace/blob/main/.github/workflows/skill-audit-gate.yml)

---

## 한국어 요약

GitHub Actions로 PR 제출 시 자동 보안 감사를 설정하는 가이드입니다.

- **Direct 모드**: `skills/` 디렉토리에 직접 스킬을 올리는 경우
- **Remote 모드**: `marketplace.json`에 외부 플러그인 URL을 등록하는 경우
- **필수 Secret**: `ANTHROPIC_API_KEY` (Repository secret으로 등록)
- **보안 설계**: 2-job 분리, base branch audit tool, URL allowlist, revision 고정, Fail-Closed
- **비용**: Sonnet 기준 스킬 1개 감사에 ~$0.14
