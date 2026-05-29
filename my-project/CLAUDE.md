# Multi-Agent Code Review System

This project uses a multi-agent workflow to automatically review GitHub Pull Requests across three dimensions: security, performance, and style.

---

## Workflow Overview

When a PR is opened or updated, the orchestrator:
1. Fetches the unified diff via `gh pr diff`
2. Spins up three isolated git worktrees
3. Runs security, performance, and style agents in parallel — each writes results to `/tmp/review/`
4. Aggregates the three result files into `/tmp/review/final.md`
5. Posts inline comments to the PR from the final report

---

## Agent Definitions

Agent prompts live in `.claude/agents/`. Each file is loaded by the orchestrator and passed as the system prompt to its respective subagent.

| File | Responsibility |
|------|---------------|
| `agents/security-reviewer.md` | OWASP Top 10, secrets exposure, auth flaws, CVE patterns |
| `agents/performance-reviewer.md` | N+1 queries, O(n²) loops, memory leaks, blocking I/O |
| `agents/style-reviewer.md` | Naming conventions, cyclomatic complexity, dead code, missing docs |

---

## Running a Review

```bash
# Review the current open PR on this branch
gh pr view --json number -q .number | xargs -I{} claude "Review PR #{}"

# Review a specific PR number
claude "Review PR #42"
```

The orchestrator will fan out to all three agents automatically.

---

## Orchestrator Instructions

When asked to review a PR, follow this exact sequence:

### Step 1 — Fetch diff
```bash
gh pr diff <PR_NUMBER>
```
Parse the output into a list of changed files and their hunks.

### Step 2 — Spin up worktrees
Create one worktree per agent so they cannot interfere with each other:
```bash
git worktree add /tmp/wt-security   HEAD
git worktree add /tmp/wt-performance HEAD
git worktree add /tmp/wt-style       HEAD
```

### Step 3 — Run agents in parallel
Create the output directory, then invoke all three agents concurrently using `isolation: worktree`:
```bash
mkdir -p /tmp/review
```

- **SecurityAgent** — prompt from `agents/security-reviewer.md` → writes to `/tmp/review/security.md`
- **PerformanceAgent** — prompt from `agents/performance-reviewer.md` → writes to `/tmp/review/performance.md`
- **StyleAgent** — prompt from `agents/style-reviewer.md` → writes to `/tmp/review/style.md`

Each agent receives: the diff text, its assigned worktree path, and the list of changed files.  
Each agent **must** write its Markdown findings to the designated path before returning.

### Step 4 — Clean up worktrees
```bash
git worktree remove /tmp/wt-security    --force
git worktree remove /tmp/wt-performance --force
git worktree remove /tmp/wt-style       --force
```

### Step 5 — Aggregate findings into `/tmp/review/final.md`

Read the three agent result files, then apply the rules below to produce the final report.

#### Rule 1 — Deduplication
Group findings by `(file, line)`. When multiple agents flag the same location, keep only the entry with the highest severity. Severity order: `CRITICAL > HIGH > MEDIUM > LOW > INFO`.

#### Rule 2 — Priority Matrix
After deduplication, assign each finding a `difficulty` (수정 난이도) and an `impact` (영향도), then place it in the matrix:

| | 영향도: 높음 | 영향도: 중간 | 영향도: 낮음 |
|---|---|---|---|
| **난이도: 쉬움** | 🔴 즉시 수정 | 🟠 다음 스프린트 | 🟡 여유 시 처리 |
| **난이도: 중간** | 🟠 다음 스프린트 | 🟡 여유 시 처리 | 🟢 백로그 |
| **난이도: 어려움** | 🟡 여유 시 처리 | 🟢 백로그 | 🟢 백로그 |

**난이도 기준**
- `쉬움` — 한 줄 수정, 상수 추출, 이름 변경
- `중간` — 함수 분리, 쿼리 리팩토링, 캐시 레이어 추가
- `어려움` — 아키텍처 변경, 외부 의존성 교체, 대규모 리팩토링

**영향도 기준**
- `높음` — 보안 취약점, 서비스 장애 가능성, 데이터 손실 위험
- `중간` — 성능 저하, 유지보수 비용 증가
- `낮음` — 가독성·스타일 개선, 코드 품질 향상

#### Rule 3 — High Severity Summary
Extract all `CRITICAL` and `HIGH` findings into a separate **"즉시 조치 필요"** section at the top of `final.md`, formatted as:

```
## 🚨 즉시 조치 필요 (CRITICAL / HIGH)

| # | 심각도 | 파일 | 라인 | 제목 | 에이전트 |
|---|--------|------|------|------|---------|
| 1 | HIGH   | src/auth/login.ts | 42 | SQL Injection risk | security |
```

This section appears before all other findings and is the only content included in the GitHub PR summary comment.

### Step 6 — Post to GitHub
```bash
# Overall summary comment
gh pr comment <PR_NUMBER> --body "<markdown_summary>"

# Inline comment per finding
gh api POST /repos/{owner}/{repo}/pulls/<PR_NUMBER>/comments \
  --field path=<file> \
  --field line=<line> \
  --field side=RIGHT \
  --field body=<finding_body>
```

---

## Finding Schema

Each agent must return findings in this structure:

```json
{
  "findings": [
    {
      "file": "src/auth/login.ts",
      "line": 42,
      "severity": "HIGH",
      "category": "security",
      "title": "SQL Injection risk",
      "description": "User input is interpolated directly into the query string.",
      "suggestion": "Use parameterized queries or a query builder.",
      "difficulty": "쉬움",
      "impact": "높음"
    }
  ]
}
```

---

## Severity Guidelines

| Severity | Meaning | Action |
|----------|---------|--------|
| `CRITICAL` | Exploitable vulnerability or data loss risk | Block merge |
| `HIGH` | Significant bug or security flaw | Request changes |
| `MEDIUM` | Notable issue, should be fixed | Comment |
| `LOW` | Minor style or best-practice deviation | Suggestion |
| `INFO` | Informational note | Optional |

---

## Project Conventions

- Language: TypeScript (strict mode)
- Formatter: Prettier (`.prettierrc`)
- Linter: ESLint (`eslint.config.js`)
- Tests: Vitest — run with `npm test`
- Do not commit secrets, `.env` files, or large binaries
