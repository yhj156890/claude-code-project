# Multi-Agent Code Review System — Communication Flow

```mermaid
sequenceDiagram
    autonumber

    participant GH as GitHub PR
    participant ORC as Orchestrator<br/>(Claude Sonnet)
    participant WT1 as Worktree · security
    participant WT2 as Worktree · performance
    participant WT3 as Worktree · style
    participant SA as SecurityAgent<br/>(Claude Sonnet)
    participant PA as PerformanceAgent<br/>(Claude Sonnet)
    participant STA as StyleAgent<br/>(Claude Sonnet)

    %% ── Phase 1: Diff Ingestion ──────────────────────────────
    rect rgb(230, 244, 255)
        Note over GH,ORC: Phase 1 — Diff Ingestion
        GH->>ORC: PR webhook (PR number, repo, sha)
        ORC->>GH: gh pr diff <PR#>
        GH-->>ORC: unified diff (patch text)
        ORC->>ORC: parse diff → file list, hunks
    end

    %% ── Phase 2: Parallel Worktree Isolation ─────────────────
    rect rgb(255, 248, 230)
        Note over ORC,WT3: Phase 2 — Spin up isolated worktrees (isolation: worktree)
        par create worktrees in parallel
            ORC->>WT1: git worktree add /tmp/wt-security <sha>
        and
            ORC->>WT2: git worktree add /tmp/wt-performance <sha>
        and
            ORC->>WT3: git worktree add /tmp/wt-style <sha>
        end
        Note over WT1,WT3: Each worktree is an independent checkout<br/>— agents cannot interfere with each other
    end

    %% ── Phase 3: Parallel Agent Execution ───────────────────
    rect rgb(230, 255, 236)
        Note over SA,STA: Phase 3 — Parallel review (agent() calls fan out concurrently)
        par agents run concurrently
            ORC->>SA: agent(securityPrompt, {isolation:"worktree", workdir:WT1})
            SA->>WT1: read changed files
            WT1-->>SA: file contents + diff context
            SA->>SA: analyze: SQLi, XSS, secrets,<br/>auth bypass, CVE patterns
            SA-->>ORC: SecurityFindings[]<br/>{file, line, severity, description, suggestion}
        and
            ORC->>PA: agent(perfPrompt, {isolation:"worktree", workdir:WT2})
            PA->>WT2: read changed files
            WT2-->>PA: file contents + diff context
            PA->>PA: analyze: N+1 queries, O(n²) loops,<br/>memory leaks, blocking I/O
            PA-->>ORC: PerfFindings[]<br/>{file, line, impact, description, suggestion}
        and
            ORC->>STA: agent(stylePrompt, {isolation:"worktree", workdir:WT3})
            STA->>WT3: read changed files
            WT3-->>SA: file contents + diff context
            STA->>STA: analyze: naming conventions,<br/>complexity, dead code, docs
            STA-->>ORC: StyleFindings[]<br/>{file, line, rule, description, suggestion}
        end
    end

    %% ── Phase 4: Cleanup worktrees ───────────────────────────
    rect rgb(255, 230, 230)
        Note over ORC,WT3: Phase 4 — Worktree cleanup
        par remove worktrees
            ORC->>WT1: git worktree remove /tmp/wt-security --force
        and
            ORC->>WT2: git worktree remove /tmp/wt-performance --force
        and
            ORC->>WT3: git worktree remove /tmp/wt-style --force
        end
    end

    %% ── Phase 5: Synthesis ───────────────────────────────────
    rect rgb(245, 230, 255)
        Note over ORC: Phase 5 — Aggregate & synthesize
        ORC->>ORC: merge findings → deduplicate by (file, line)
        ORC->>ORC: rank by severity (CRITICAL > HIGH > MEDIUM > LOW)
        ORC->>ORC: render Markdown summary<br/>— overall score, tables per category
    end

    %% ── Phase 6: Post to GitHub ──────────────────────────────
    rect rgb(230, 255, 255)
        Note over ORC,GH: Phase 6 — Post review back to GitHub
        ORC->>GH: gh pr comment <PR#> --body "<summary>"
        loop for each inline finding
            ORC->>GH: gh api POST /repos/{owner}/{repo}/pulls/{PR#}/comments<br/>body: {path, line, side:"RIGHT", body: finding}
        end
        GH-->>ORC: 201 Created (comment IDs)
        ORC-->>GH: ✅ Review complete — <N> findings posted
    end
```

## Flow summary

| Phase | Actor | Action |
|-------|-------|--------|
| 1 | Orchestrator | Receives PR webhook, fetches unified diff via `gh pr diff` |
| 2 | Orchestrator | Creates 3 independent git worktrees (`isolation: worktree`) |
| 3 | Security / Perf / Style agents | Run **in parallel**, each in its own worktree — zero cross-contamination |
| 4 | Orchestrator | Removes worktrees after agents complete |
| 5 | Orchestrator | Merges & deduplicates findings, ranks by severity |
| 6 | Orchestrator | Posts an overall summary comment + per-line inline comments to the PR |
