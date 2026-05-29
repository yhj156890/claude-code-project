# Multi-Agent Code Review Report

**분석 파일:** CLAUDE.md, .claude/settings.json, .claude/agents/*.md, .github/workflows/claude-review.yml  
**실행 에이전트:** security-reviewer (sonnet) · performance-reviewer (sonnet) · style-reviewer (haiku)  
**총 발견:** 35건 — 높음 4 · 중간 19 · 낮음 12

---

## 높음 (HIGH) — 4건

### [보안] CLAUDE.md:87 — PR 번호 미검증 shell 삽입
**에이전트:** security-reviewer · CWE-78  
`gh pr comment <PR_NUMBER>` 및 `gh api .../pulls/<PR_NUMBER>/comments` 에서 PR 번호를 외부 데이터(GitHub 이벤트 페이로드)로부터 검증 없이 shell에 삽입. 공격자가 PR 번호 필드를 제어할 경우 임의 shell 인수 주입 가능.  
**수정:** PR_NUMBER가 양의 정수인지 사전 검증 후 사용. `--` 구분자 및 `--number` 플래그 활용.

---

### [보안] CLAUDE.md:86 — gh api --field 값 미인용으로 명령어 인젝션
**에이전트:** security-reviewer · CWE-78  
`--field body=<finding_body>`, `--field path=<file>` 값에 쌍따옴표 및 이스케이프 없음. finding body나 파일 경로에 backtick, `$(...)`등 메타문자가 포함되면 shell 탈출 가능.  
**수정:** 모든 `--field` 값을 쌍따옴표로 감싸거나 `--field body=@tmpfile` 방식으로 파일 경유 전달.

---

### [성능] CLAUDE.md:87 — 인라인 코멘트 N번 순차 API 호출
**에이전트:** performance-reviewer  
finding 수(N)만큼 GitHub API에 개별 POST를 순차 전송. N개 HTTP 왕복이 직렬로 발생하며 rate limit 위험.  
**수정:** GitHub Pull Request Review API(`POST .../reviews`)의 `comments` 배열을 사용해 단일 요청으로 일괄 전송.

---

### [성능] CLAUDE.md:49 — diff 전체 무제한 로드
**에이전트:** performance-reviewer  
`gh pr diff <PR_NUMBER>` 에 파일 필터나 라인 수 제한 없음. 대형 PR(lock 파일, 자동 생성 파일 포함)의 경우 전체 diff가 오케스트레이터 컨텍스트로 로드되어 토큰 비용 급증.  
**수정:** `--name-only`로 범위 파악 후 생성 파일 제외. 에이전트별로 관련 파일 subset만 전달.

---

## 중간 (MEDIUM) — 19건

### [보안] CLAUDE.md:32 — xargs로 미검증 외부 데이터 claude CLI 전달
CWE-78. `gh pr view ... | xargs -I{} claude "Review PR #{}"` 패턴에서 API 응답값이 검증 없이 xargs에 주입됨.  
**수정:** PR 번호를 변수로 캡처 후 정수 검증(`[[ $PR =~ ^[0-9]+$ ]]`)하고 직접 실행.

### [보안] CLAUDE.md:33 — 오케스트레이터 문서에 PR_NUMBER 검증 단계 없음
CWE-20. Shell 명령어 사용 전 입력값 검증 지침이 문서에 없어 구현자가 이를 생략할 위험.  
**수정:** 오케스트레이터 시퀀스에 "PR 번호가 양의 정수인지 검증" 단계를 명시적으로 추가.

### [보안] .claude/agents/security-reviewer.md:4 — Bash 툴 불필요하게 부여
CWE-284. security-reviewer에만 `Bash` 툴이 부여되어 있어, diff 콘텐츠를 통한 프롬프트 인젝션 시 shell 실행 능력 노출. performance-reviewer, style-reviewer는 Bash 없이 구성됨.  
**수정:** Bash 툴 제거. 패턴 스캔은 Grep으로 대체.

### [보안] .github/workflows/claude-review.yml — 워크플로우 파일 비어 있음(보안 통제 없음)
CWE-284. 파일이 0바이트로 트리거 조건, 권한 범위, 시크릿 관리 없음.  
**수정:** `permissions: pull-requests: write, contents: read` 최소 권한 적용. `pull_request` 트리거 사용(pull_request_target 사용 금지). 모든 액션 버전을 full commit SHA로 고정.

### [성능] CLAUDE.md:77 — worktree 정리 순차 실행으로 취합 지연
세 개의 `git worktree remove`가 직렬 실행되어 aggregation 전 불필요한 I/O 대기 발생.  
**수정:** 백그라운드 병렬 실행 후 `wait`: `git worktree remove ... & ... & wait`

### [성능] CLAUDE.md:55 — worktree 생성 병렬 실행 명시 없음
Step 2에서 3개 worktree 생성이 순차 실행으로 문서화됨. 대형 레포에서 각 add가 수 초 소요.  
**수정:** 병렬 실행을 명시적으로 문서화.

### [성능] CLAUDE.md:77 — 동일 커밋 재실행 시 캐싱 전략 없음
매 실행마다 diff 재취득 및 worktree 재생성. `(PR_NUMBER, HEAD_SHA)` 키로 단기 TTL 캐시 적용 필요.

### [성능] .claude/agents/security-reviewer.md — 에이전트별 파일 전체 독립 읽기
security/performance/style 세 에이전트가 동일 파일을 각각 독립적으로 전체 읽기. 파일 I/O 3배 낭비.  
**수정:** 오케스트레이터가 파일을 1회 읽어 각 에이전트에 전달.

### [성능] .claude/agents/security-reviewer.md — Grep 4회 독립 순회
SQL, secrets, XSS, 명령어 인젝션 패턴을 각각 별도 Grep으로 실행. 단일 OR 패턴으로 통합 가능.

### [성능] .claude/agents/performance-reviewer.md — Grep 5회 독립 순회
5개 성능 패턴을 별도 Grep으로 순차 실행. 단일 패스로 통합 후 카테고리별 분류 권장.

### [성능] .github/workflows/claude-review.yml — CI 레벨 병렬성·캐싱 미정의
워크플로우 파일 비어 있어 job matrix, actions/cache, 병렬 job 없음. 매 실행마다 의존성 재설치.

### [스타일] .claude/agents/*.md:3 — frontmatter description 한국어/영어 혼용
세 에이전트 파일 모두 frontmatter `description`은 한국어, body는 영어로 언어 불일치.  
**수정:** 전체를 영어로 통일하거나 프로젝트 정책을 수립해 일관 적용.

### [스타일] .claude/agents/style-reviewer.md:3 — model 이름 포맷 불일치
`claude-haiku-4-5-20251001`(날짜 접미사 형식) vs `claude-sonnet-4-6`(간략 형식) 혼용.  
**수정:** 세 파일 모두 동일한 버전 표기 규칙 적용.

### [스타일] .claude/agents/*.md — 심각도 한국어 레이블 매직 스트링 중복
`[높음]`, `[중간]`, `[낮음]`이 인라인 코멘트 예시와 매핑 테이블에 각 파일마다 중복 정의.  
**수정:** CLAUDE.md에 레이블 정의 한 곳으로 통합.

### [스타일] CLAUDE.md:87 — `{owner}`, `{repo}` 플레이스홀더 미정의
GitHub API 호출에 `{owner}`, `{repo}`가 쓰이나 정의/설명 없음.  
**수정:** `${{ github.repository_owner }}` 등 GitHub Actions 표현식으로 교체하거나 주석 추가.

### [스타일] .github/workflows/claude-review.yml — 내용 없는 파일 존재
0바이트 파일이 존재하는 것이 빈 파일이 없는 것보다 혼란스러움.  
**수정:** 최소한의 워크플로우 구현 또는 파일 제거 후 CLAUDE.md 업데이트.

### [스타일] .claude/agents/performance-reviewer.md:108 — Grep 줄마다 "in changed files" 반복
Search Strategy의 모든 패턴 줄에 `in changed files`가 반복되며 섹션 제목과 중복.  
**수정:** 블록 상단에 한 번만 명시.

### [스타일] .claude/agents/security-reviewer.md:93 — 동일 반복 문제
성능 에이전트와 동일하게 "in changed files" 반복.

### [스타일] CLAUDE.md:63 — worktree 경로 `/tmp/wt-*` 하드코딩 중복
Step 2와 Step 4에 동일 경로가 반복 정의됨. 변경 시 두 곳 모두 수정 필요.  
**수정:** 섹션 상단에 `WORKTREE_BASE=/tmp` 변수 정의 후 파생.

---

## 낮음 (LOW) — 12건

| # | 에이전트 | 파일 | 제목 |
|---|---------|------|------|
| 1 | 보안 | security-reviewer.md:130 | finding body를 GitHub 코멘트에 삽입 전 HTML/Markdown 이스케이프 미언급 (CWE-79) |
| 2 | 보안 | security-reviewer.md:160 | 인라인 코멘트 파일 경로/제목 삽입 시 sanitization 미언급 (CWE-116) |
| 3 | 보안 | CLAUDE.md:57 | `/tmp/wt-*` worktree 경로 사전 존재 여부 미확인 — 공유 CI에서 심볼릭 링크 공격 가능 (CWE-61) |
| 4 | 성능 | style-reviewer.md:5 | Haiku 모델 사용에도 파일 전체 읽기 수행 — diff hunk ±30줄로 제한 권장 |
| 5 | 성능 | style-reviewer.md:73 | 함수 길이 체크를 수동 라인 카운트로 지시 — Grep 패턴으로 대체 가능 |
| 6 | 스타일 | security-reviewer.md:14 | `## Input` 섹션이 세 파일에 동일 반복 (DRY 위반) |
| 7 | 스타일 | security-reviewer.md:24 | `## Your Task` 보일러플레이트 세 파일에 거의 동일 반복 |
| 8 | 스타일 | security-reviewer.md:128 | 출력 형식 지시문 세 파일에 동일 반복 |
| 9 | 스타일 | performance-reviewer.md:108 | Search Strategy Grep 줄마다 "in changed files" 반복 |
| 10 | 스타일 | security-reviewer.md:93 | 동일 반복 문제 |
| 11 | 스타일 | CLAUDE.md:63 | `/tmp/wt-*` 경로 Step 2·4에 하드코딩 반복 |
| 12 | 스타일 | style-reviewer.md:184 | Style Pattern Reference 표에서 산문과 코드 스니펫 형식 혼용 |

---

## 수정 우선순위 요약

| 우선순위 | 항목 | 이유 |
|---------|------|------|
| **즉시** | PR_NUMBER shell 삽입 검증 추가 | 실제 배포 시 명령어 인젝션 직접 위험 |
| **즉시** | `claude-review.yml` 최소 구현 | 보안 통제 없이 비어 있는 상태 |
| **즉시** | security-reviewer에서 Bash 툴 제거 | 불필요한 공격 표면 |
| **단기** | GitHub Review API 일괄 코멘트 전환 | N번 API 호출 → 1번으로 성능 개선 |
| **단기** | worktree 생성/정리 병렬화 문서화 | CI 소요 시간 단축 |
| **단기** | 세 에이전트 공통 섹션 CLAUDE.md로 통합 | DRY 위반 해소 |
| **장기** | diff 크기 제한 및 캐싱 전략 수립 | 대형 PR 처리 비용 절감 |
