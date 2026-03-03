# Skills User Guide

## Overview

Three skills that work together to manage large projects with minimal context consumption.

```
/explore-docs  →  문서 이해 (브리핑 생성)
/design-map    →  설계 vs 코드 비교 (매핑 생성)
/delegate      →  범용 작업 위임 (컨텍스트 보존)
```

Typical workflow: explore-docs → design-map → delegate for implementation tasks.

---

## 1. /explore-docs

**What it does**: Reads all markdown files in a project using parallel agents and produces a compressed briefing. The main agent never opens any document.

**When to use**:
- New project onboarding — "이 프로젝트 문서들이 뭐가 있어?"
- Before diving into code — "어떤 문서를 먼저 읽어야 해?"
- After documentation changes — "문서 현황을 다시 정리해줘"

**Commands**:
```
/explore-docs              # Explore docs in current directory
/explore-docs path/to/docs # Explore a specific directory

"Documents에 프로젝트를 설명하는 문서들이 들어 있으니 /explore-docs 이 프로젝트의 개요를 파악하고, 그 다음
  claude.md를 작성하자. 참고로 이 문서를 작성할 때 코드베이스에서 얻을 수 있는 정보를 포함해서는 안 되고, teamcreate를
  언제 할 지, claude in chrome을 언제 사용할 지, 문서를 언제 열어볼 지, 스킬을 언제 사용할 지 등 필수적인 정보만
  압축해서 간단하게 만들어야 해."
```

**Example Prompt**:

```
"@Documents에 프로젝트를 설명하는 문서들이 들어 있으니 이 프로젝트의 개요를 파악하고, 그 다음
  claude.md를 작성하자. 이 문서를 작성할 때 코드베이스에서 얻을 수 있는 정보를 포함해서는 안 되고, teamcreate를
  언제 할 지, 문서를 언제 열어볼 지, 스킬을 언제 사용할 지 등 필수 정보와 도구 사용 지침만 압축해서 간단하게 만들어야 해."
```

**Output**: `docs-briefing.md` — a structured briefing with:
- Per-document summary (scope, key topics, "read this when" conditions)
- Quick reference table
- Cross-references between documents
- Document hierarchy

**Token savings**: ~97% vs reading all documents directly.

---

## 2. /design-map

**What it does**: Cross-references design documents against the codebase to determine what's implemented, what's missing, and what doesn't match.

**When to use**:
- Starting implementation — "뭐가 구현되고 뭐가 안 됐어?"
- After code changes — "구현 현황을 갱신해줘"
- Before planning work — "미구현 항목이 뭐야?"

**Commands**:
```
/design-map          # Full initial generation
/design-map init     # Same as above
/design-map update   # Incremental update (uses git diff)
```

**Prerequisite**: Run `/explore-docs` first. design-map uses the briefing to reduce exploration costs.

**Output**: `.claude/design-map.md` — a mapping table with:
- Every design decision mapped to implementation status
- Three statuses: `Implemented` / `Not implemented` / `Mismatch`
- Dependency graph (what blocks what)
- Prioritized implementation suggestions

**Incremental updates**: After code or doc changes, `/design-map update` detects what changed via `git diff` and only re-verifies affected items.

---

## 3. /delegate

**What it does**: Offloads any task to a sub-agent. The main agent sends a one-line task description, the sub-agent does all the work (reading, editing, debugging), and returns a compressed summary (~20 lines). The main agent's context stays clean.

**When to use**:
- Any code change that touches 3+ files
- Debugging (unknown cause, needs exploration)
- Self-contained features, fixes, or refactors
- Research tasks — "이 모듈이 어떻게 작동하는지 알아봐"

**Commands**:
```
/delegate Fix the auth bug where users get 401 on valid tokens
/delegate Add unit tests for payment processing
/delegate Refactor database connection pool to use async
/delegate Research how the caching layer works
```

**Output**: `.claude/delegate-results/{timestamp}-{task-slug}.md` — a fixed 5-section summary:
- **Result**: 1-2 sentence summary
- **Files**: list of files read/modified
- **Decisions**: choices made when task was ambiguous
- **Follow-up**: issues found or next steps needed
- **Verification**: what was checked for correctness

**Key property**: The sub-agent handles ambiguity autonomously — it makes reasonable choices and documents them rather than blocking.

**Chaining**: Break large tasks into multiple delegates. Each one returns only a compressed summary, so the main agent accumulates awareness without accumulating file contents.

---

## Typical Session

```
# 1. Understand the documentation landscape
> /explore-docs

# 2. Map design decisions to code
> /design-map

# 3. Implement features without filling context
> /delegate Implement the NarrativeBlock source/lock_state fields per S-D §2-1
> /delegate Add StateDimension enum and auto-tagging in effect_applicator

# 4. After implementation, update the map
> /design-map update

# 5. After documentation changes, refresh the briefing
> /explore-docs
```

---

## How It Works (Under the Hood)

All three skills follow the same pattern:

1. **Main agent** orchestrates but never reads documents or code
2. **Analysis agents** (parallel) do the actual reading and analysis
3. **Assembly agent** compiles results into a single output file
4. Results are passed between agents via **temp files**, not through the main agent's context
5. Internal operations are conducted in **English** to save tokens
6. Only the **final user report** is in Korean

This means your context window stays clean regardless of project size.

---

## Tips

- **Large projects (20+ docs)**: explore-docs will spawn more agents (up to 5). This is slower but still faster than reading everything yourself.
- **design-map granularity**: Each mapping item is one independent decision. You can reference specific items by their Source code (e.g., "S-D §2") when asking follow-up questions.
- **Mismatch ≠ bug**: A mismatch means code differs from design. It might be intentional (noted in the mapping) or something that needs fixing.
- **Git integration**: design-map update works best when you commit regularly. It uses `git diff HEAD~1` to detect changes.
