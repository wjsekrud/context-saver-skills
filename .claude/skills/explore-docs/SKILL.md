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

Explore a project's markdown documentation using parallel sub-agents. Produce a compressed briefing file that tells exactly what each document covers and when to read it — without the main agent ever opening the documents.

## Language Policy

All internal operations (agent prompts, briefing file contents) are conducted in English to minimize token consumption. Only the final summary message to the user is in their language.

## Step 1: Discover Documents

Use Glob to find all `.md` files in the target directory (recursively). The user may provide a directory path as an argument — if not, use the current working directory.

```
Glob pattern: **/*.md
```

After collecting the file list, count the documents.

**Exclude** common non-documentation files: `CHANGELOG.md`, `LICENSE.md`, `node_modules/**/*.md`, `.github/**/*.md` unless the user specifically asks to include them.

**DO NOT read any document contents.** Only look at file paths and names.

## Step 2: Plan Agent Dispatch

Divide documents into batches for parallel processing:

| Document count | Number of agents | Docs per agent |
|---|---|---|
| 1-4 | 2 | 1-2 |
| 5-16 | 3 | 2-6 |
| 17-30 | 4 | 5-8 |
| 31-48 | 5 | 7-10 |
| 49+ | 6 | ~8-10 |

Group related documents together when obvious from filenames (e.g., `api-auth.md` and `api-users.md`).

## Step 3: Spawn Analysis Agents

Launch all agents in a single message. Each agent must be:
- **Type**: `general-purpose` (NOT Explore — Explore agents cannot write files)

**Critical: Each agent must write its result to a temp file.** This prevents raw results from entering the main agent's context.

Output file paths: `.claude/docs-briefing-parts/batch-{N}.md`

Create the directory if it doesn't exist.

Agent prompt template:

```
You are a documentation analyst. Read each of the following markdown files thoroughly and produce a structured summary for each one.

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

IMPORTANT: Write your complete result to this file: {output_file_path}
Do not return the result as a message. Write it to the file.
```

## Step 4: Spawn Assembly Agent

After all analysis agents complete, spawn **one general-purpose agent** to assemble the final briefing file. The main agent does NOT read the part files.

Assembly agent prompt template:

```
You are a briefing assembler. Combine partial documentation analysis results into a single briefing file.

## Input Files
{list of .claude/docs-briefing-parts/batch-*.md paths}

## Assembly Instructions

1. Read all input files.

2. Produce a single `.claude/docs-briefing.md` with this structure:

# Documentation Briefing

**Project**: {directory name}
**Generated**: {date}
**Total documents**: {count}
**Explored by**: {N} agents

---

{All summaries, organized as a continuous list}

---

## Quick Reference

| Document | Read when... |
|---|---|
| {filename} | {short trigger condition} |
| ... | ... |

## Cross-References
{Documents that reference each other, shown as a list or graph}

## Document Hierarchy
{Layered organization if documents have natural groupings}

3. The Quick Reference table is critical — it provides at-a-glance lookup for finding the right document for any task.

4. Write the result to: .claude/docs-briefing.md

5. After writing, delete the .claude/docs-briefing-parts/ directory.
```

## Step 5: Verify and Report

After the assembly agent completes:

1. Verify `.claude/docs-briefing.md` exists (Glob check only).
2. Read only the **header section** (first ~10 lines) to extract document count and agent count.
3. Report to the user in their language: project name, document count, that the briefing is saved, and the file path.

**The main agent never reads the full briefing file or any part files.**

## Important Constraints

- **Never read document contents directly.** Only sub-agents read files. The main agent only sees paths, names, and the briefing header.
- **Never skip parallel dispatch.** Even for small sets, use at least 2 agents for consistent independent analysis.
- **Always save to file.** The briefing must be written to `.claude/docs-briefing.md` so it persists across sessions and can be used by other skills (e.g., design-map).
- **If the user provides a path argument**, use it as the root directory for the Glob search. Otherwise default to the current working directory.
