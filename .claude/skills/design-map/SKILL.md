---
name: design-map
description: >
  설계 문서의 확정된 결정사항과 코드베이스의 구현 상태를 매핑하는 스킬.
  설계는 되었으나 코드에 없는 것, 코드에 있지만 설계와 불일치하는 것, 완전히 일치하는 것을
  즉시 구분할 수 있는 매핑 테이블을 생성하고 유지한다.
  '/design-map' 또는 '/design-map update'로 트리거한다.
  사용자가 "설계와 코드 상태를 비교해줘", "뭐가 구현되고 뭐가 안 됐어",
  "설계 매핑을 갱신해줘", "구현 현황을 알려줘", "미구현 항목이 뭐야" 같은 요청을 할 때도
  이 스킬을 사용한다. 설계 문서와 코드베이스가 같은 레포에 있는 프로젝트에서 특히 유용하다.
---

# Design-Implementation Map

설계 문서의 결정사항들이 코드베이스에 어떤 상태로 존재하는지를 매핑한다. 에이전트가 미구현을 버그로 오인하거나, 의도적 구조를 리팩토링 대상으로 판단하거나, 폐기된 설계를 기준으로 코드를 작성하는 실수를 방지한다.

## 두 가지 모드

인자가 없거나 `init`이면 초기 생성, `update`이면 증분 업데이트를 실행한다.

---

## Language Policy

**Internal operations (agent prompts, mapping tables, file contents) are conducted entirely in English** to minimize token consumption. Korean characters cost ~2-3x more tokens than English.

- **English**: Agent prompt templates, mapping table columns (Source, Decision, Status, Notes), dependency graphs, priority suggestions, all temp/output files
- **Korean**: Only the final summary message printed to the user after completion

The "Decision" column may reference Korean terms from design documents (e.g., entity names, field names) — quote them as-is but keep the surrounding description in English.

---

## Mode 1: Initial Generation (`/design-map` or `/design-map init`)

### Step 1: Precondition Check

1. **docs-briefing.md**: Check if an explore-docs briefing file exists (typically `.claude/docs-briefing.md` or near project root). If found, use it as input. If not, instruct the user to run `/explore-docs` first.

2. **Project structure**: Use Glob to identify design document directories and source code directories. Check file paths and directory names only — do NOT read file contents.

### Step 2: Analysis Agent Dispatch

Group design documents by functional area and assign a general-purpose agent to each group. Agents need to read **both design docs and codebase**, so use general-purpose type (not Explore).

Grouping examples (adjust per project):
- Core data model docs + corresponding models/schemas code
- UI/UX design docs + corresponding frontend code
- Pipeline/service docs + corresponding services/routers code

Scale agent count to document/code volume: 2–5 agents.

**Critical: Each agent must write its result to a temp file**, not return it as a message. This prevents raw results from entering the main agent's context.

Agent prompt template:

```
You are a design-implementation mapping analyst. Read the design documents below, find corresponding implementations in the codebase, and produce a mapping table.

## Design Documents (read carefully)
{list of design doc paths}

## Codebase Search Scope
{source code directory paths}

## Briefing Reference (if available)
{explore-docs briefing excerpt for these documents}

## Instructions

1. Extract **implementable decisions** from each design document.
   - Data models/schemas, API endpoints, business rules, UI components, algorithms, etc.
   - Exclude philosophical principles or background explanations — only concrete, code-translatable decisions
   - One decision = one independent implementation unit

2. For each decision, search the codebase to determine implementation status.
   - Use Glob and Grep to find relevant files, Read to verify contents
   - Match by class names, function names, table names, field names, etc.

3. Format each item as a markdown table row:

| Source | Decision | Status | Notes |
|--------|----------|--------|-------|
| S-D §1 | Entity schema: stable_id, display_name, entity_type fields | Implemented | models/entity.py |
| S-D §2 | Attribute temporal_category field | Not implemented | |
| S-E §3 | lock_state transition: proposed→draft→locked | Mismatch | Only proposed→locked exists. Draft stage omitted (intentional simplification) |

Status values: Implemented / Not implemented / Mismatch

4. Note dependencies in the Notes column:
   - "Blocked by: {other decision}" format
   - Example: "Blocked by: time model finalization"

5. Add a one-line rationale ONLY for items with high deviation risk:
   - Example: "TEXT storage reason: LLM readability priority (S-B principle)"

Keep decisions to the minimum context needed for an agent to judge implementation status. For full design details, point to the original document name and section number.

## Output

Write your complete result to this file: {output_file_path}

The file must contain ONLY the markdown tables organized by document, with document headers (e.g., "## S-C: ..."). Do not include any preamble or summary — just the tables.
```

**Output file paths**: `.claude/design-map-parts/group-{N}.md` (e.g., `group-1.md`, `group-2.md`, `group-3.md`). Create the directory if needed.

### Step 3: Assembly Agent Dispatch

After all analysis agents complete, spawn **one more general-purpose agent** to assemble the final mapping file. The main agent does NOT read the part files.

Assembly agent prompt template:

```
You are a mapping assembler. Combine the partial mapping results into a single design-map.md file.

## Input Files
{list of .claude/design-map-parts/group-*.md paths}

## Assembly Instructions

1. Read all input files.

2. Produce a single `.claude/design-map.md` with this structure:

# Design-Implementation Map

**Project**: {project name}
**Generated**: {date}
**Base commit**: {commit hash}
**Design docs**: {count} | **Mapped items**: {count}
**Status summary**: Implemented {N} | Not implemented {N} | Mismatch {N}

---

## {Functional Area 1} ({document codes})

| Source | Decision | Status | Notes |
|--------|----------|--------|-------|
| ... | ... | ... | ... |

## {Functional Area 2} ({document codes})
...

---

## Dependency Graph

{Blocked items with their blocker relationships, in text or ascii format}

## Implementation Priority Suggestions

### Priority 1: Immediate (no blockers, data integrity)
{table of items}

### Priority 2: Independent feature blocks (no blockers, feature extension)
{table of items}

### Priority 3: Dependency chains (must follow order)
{numbered chains}

### Deferred (by design intent or out of scope)
{table of items with reasons}

3. Count statuses accurately from the actual table rows.

4. Organize sections by functional area, merging related documents.

5. Write the result to: .claude/design-map.md

6. After writing, delete the .claude/design-map-parts/ directory.
```

### Step 4: Verify and Report

After the assembly agent completes:

1. Verify `.claude/design-map.md` exists (Glob check only — do NOT read the full file).
2. Read only the **header section** (first ~10 lines) to extract the status summary counts.
3. Report to the user **in Korean**: project name, item counts, status breakdown, and note any high-risk mismatches if the assembly agent flagged them.

**The main agent never reads the full mapping file or any part files.**

---

## Mode 2: Incremental Update (`/design-map update`)

### Step 1: Change Detection

```bash
git diff HEAD~1 --name-only
```

Classify changed files:
- **Design doc changes**: .md files in the design docs directory
- **Code changes**: source code files (.py, .ts, .tsx, etc.)

If no changes detected, report "Mapping is up to date" and exit.

### Step 2: Scope Identification

Read only the **header and section headers** of the existing `design-map.md` (not the full content) to understand the current structure. Then:

- **Design doc changes**: Identify which Source codes (e.g., S-D) are affected → those sections need re-verification.
- **Code changes**: Use Grep on design-map.md for changed filenames in the Notes column → those items need re-verification.

### Step 3: Selective Re-verification

Spawn agents only for affected items. If few (~10 or less), use a single agent. If many, split into 2–3 agents.

Each agent receives:
- The specific section(s) to re-verify (extracted from design-map.md)
- The changed files to examine
- Instructions to output updated rows in the same table format
- Output file path: `.claude/design-map-parts/update-{N}.md`

### Step 4: Merge Update

Spawn an assembly agent to:
- Read the existing `.claude/design-map.md`
- Read the update part files
- Replace/insert/remove affected rows
- Update the header (commit hash, date, status counts)
- Write the updated file back to `.claude/design-map.md`
- Delete the parts directory

### Step 5: Verify and Report

Same as Mode 1 Step 4 — read only the header, report summary in Korean.

---

## Key Principles

- **Granularity**: "Minimum context needed for an agent to judge implementation status." One item = 1–2 lines.
- **Independence**: Each item must be independently updatable. Changing one item should never require modifying another item's text.
- **Source reference**: For detailed design, point to the original document name and section — don't copy design content into the mapping table.
- **Minimal rationale**: Only for high-deviation-risk items. Copying all "why" explanations bloats the table.
- **Mismatch justification**: Every Mismatch item MUST include a one-line reason (intentional simplification, technical constraint, not yet applied, etc.).
- **Main agent reads NO documents**: All design doc reading, code exploration, and result assembly is performed by sub-agents.
- **Main agent reads NO raw results**: Analysis results are passed between agents via temp files. The main agent only reads the final file's header for reporting.
- **English internals**: All agent prompts, mapping tables, temp files, and analysis are in English. Only the user-facing summary is in Korean.
