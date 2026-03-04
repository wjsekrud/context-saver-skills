---
name: explore-docs
description: >
  Parallel document exploration skill that reads and summarizes all markdown documentation
  in a project without the main agent ever opening the files. Use this skill whenever
  the user wants to understand a large set of documentation, get a project overview,
  or needs to know which documents to consult for a specific task. Trigger on
  '/explore-docs' or when the user asks things like "summarize the docs", "explore
  the documentation", "give me an overview of the project docs", "what docs do we have",
  "brief me on the project", or "I need to understand this project's documentation".
  Also trigger when the user has a large project with many .md files and wants to
  navigate them efficiently without reading everything.
---

# Explore Docs

Explore a project's markdown documentation using parallel sub-agents. Produce a **hierarchical briefing** — a small index file that's always in context, plus per-area detail files loaded only on demand.

## Language Policy

All internal operations (agent prompts, briefing file contents) are conducted in English to minimize token consumption. Only the final summary message to the user is in their language.

## Briefing Architecture

The briefing is split into three layers to prevent linear growth:

```
Layer 1: index.md          (~50 lines, fixed size, always loadable)
Layer 2: area/*.md          (~100-200 lines each, loaded on demand)
Layer 3: original documents  (never read by main agent)
```

```
50 docs with flat briefing:   ~2500 lines always in context
50 docs with layered briefing: ~50 lines + ~200 on demand = ~250 lines
```

Output structure:

```
.claude/docs-briefing/
├── index.md              ← Layer 1: quick reference + cross-refs
├── area/
│   ├── data-model.md     ← Layer 2: detailed briefing per group
│   ├── editor-ux.md
│   ├── pipeline.md
│   └── ...
```

## Step 1: Discover Documents

Use Glob to find all `.md` files in the target directory (recursively). The user may provide a directory path as an argument — if not, use the current working directory.

```
Glob pattern: **/*.md
```

After collecting the file list, count the documents.

**Exclude** common non-documentation files: `CHANGELOG.md`, `LICENSE.md`, `node_modules/**/*.md`, `.github/**/*.md` unless the user specifically asks to include them.

**DO NOT read any document contents.** Only look at file paths and names.

## Step 2: Plan Agent Dispatch

Divide documents into batches for parallel processing. **Each batch becomes one Layer 2 area file**, so group by functional area:

| Document count | Number of agents | Docs per agent |
|---|---|---|
| 1-4 | 2 | 1-2 |
| 5-16 | 3 | 2-6 |
| 17-30 | 4 | 5-8 |
| 31-48 | 5 | 7-10 |
| 49+ | 6 | ~8-10 |

**Assign a descriptive area name to each batch** based on the documents it contains (e.g., `data-model`, `editor-ux`, `pipeline`, `api-reference`). This name becomes the Layer 2 filename.

Group related documents together when obvious from filenames (e.g., `api-auth.md` and `api-users.md`). The grouping quality directly affects how useful the Layer 2 files are.

## Step 3: Spawn Analysis Agents

Launch all agents in a single message. Each agent must be:
- **Type**: `general-purpose` (NOT Explore — Explore agents cannot write files)

**Critical: Each agent writes its result to a temp file.** This prevents raw results from entering the main agent's context.

Output file paths: `.claude/docs-briefing-parts/{area-name}.md`

Create the directory if it doesn't exist.

Agent prompt template:

```
You are a documentation analyst. Read each of the following markdown files thoroughly and produce a structured summary for each one.

This batch covers the "{area_name}" area.

Files to analyze:
{list of file paths}

For EACH file, produce this exact structure:

### {filename}
**Path**: {full path}
**Scope**: 1-2 sentence description of what this document covers.
**Key topics**: Comma-separated list of main topics, concepts, APIs, or entities defined.
**Read this when**: Specific conditions or tasks that require this document. Be precise — not "when you need info about X" but "when implementing X, when debugging Y, when configuring Z".
**References**: Other documents this one links to or depends on (file paths if found, or "None").
**Critical details**: Crucial constraints, warnings, version requirements, or non-obvious information (1-2 sentences, or "None").

Be thorough when reading but concise when summarizing. The goal is to give someone who has never seen these files enough information to know exactly when they need to read each one. Do not include raw content — only your structured analysis.

At the END of your output, add a one-line summary for each document in this format (this will be used for the index):

## Quick Reference Lines
| {filename} | {one-line "read when" trigger} |

IMPORTANT: Write your complete result to this file: {output_file_path}
Do not return the result as a message. Write it to the file.
```

## Step 4: Spawn Assembly Agent

After all analysis agents complete, spawn **one general-purpose agent** to produce the hierarchical briefing. The main agent does NOT read the part files.

Assembly agent prompt template:

```
You are a briefing assembler. Transform partial analysis results into a hierarchical briefing structure.

## Input Files
{list of .claude/docs-briefing-parts/*.md paths}

## Assembly Instructions

### 1. Create Layer 2 area files

For each input file, create a corresponding area file at:
  .claude/docs-briefing/area/{area-name}.md

Each area file should contain:

# {Area Name} — Documentation Briefing

**Documents in this area**: {count}

---

{All document summaries from this batch, as-is from the input}

---

## Area Cross-References
{References between documents within this area}

### 2. Create Layer 1 index file

Create `.claude/docs-briefing/index.md` with this structure:

# Documentation Briefing — Index

**Project**: {directory name}
**Generated**: {date}
**Total documents**: {count}
**Areas**: {count}

---

## Quick Reference

| Document | Area | Read when... |
|---|---|---|
| {filename} | {area-name} | {one-line trigger from Quick Reference Lines} |
| ... | ... | ... |

---

## Areas

| Area | Documents | Detail file |
|---|---|---|
| {area-name} | {doc count} | `.claude/docs-briefing/area/{area-name}.md` |
| ... | ... | ... |

---

## Cross-References
{Documents that reference each other across areas}

## Document Hierarchy
{Layered organization if documents have natural groupings}

### 3. Key rules

- The index file MUST stay under 80 lines regardless of document count. One row per document in the Quick Reference table, nothing more.
- Each area file should be self-contained — readable without the index.
- Extract the "Quick Reference Lines" from each input file for the index table.

### 4. Write all files, then delete .claude/docs-briefing-parts/
```

## Step 5: Verify and Report

After the assembly agent completes:

1. Verify `.claude/docs-briefing/index.md` exists (Glob check only).
2. Read only the **index file header** (first ~10 lines) to extract document count and area count.
3. Report to the user in their language: project name, document count, area count, and note that they can load area files on demand.

**The main agent never reads the full area files or any part files.**

## How Other Skills Use the Briefing

**design-map**: Reads only `index.md` to identify which area files are relevant for each analysis agent group, then passes the corresponding area file path to each agent. This way each design-map agent gets only the briefing for its functional area, not the entire briefing.

**Main agent on-demand**: When the main agent needs to understand a specific area (e.g., before delegating a task about the data model), it reads only `.claude/docs-briefing/area/data-model.md` — not the entire briefing.

## Important Constraints

- **Never read document contents directly.** Only sub-agents read files. The main agent only sees paths, names, and the index header.
- **Never skip parallel dispatch.** Even for small sets, use at least 2 agents for consistent independent analysis.
- **Always produce hierarchical output.** Even for small projects, maintain the index + area file structure for consistency.
- **Index must stay small.** The index file must not exceed ~80 lines. If it grows beyond this, the grouping is too granular — merge areas.
- **If the user provides a path argument**, use it as the root directory for the Glob search. Otherwise default to the current working directory.
