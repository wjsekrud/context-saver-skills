# Context-Preserving Skills for Claude Code

Three Claude Code skills that keep your main agent's context window clean by delegating work to sub-agents and receiving only compressed results.

**The problem**: Long coding sessions fill the context window with file contents, debug traces, and intermediate work. Eventually the agent loses track of project-level context and starts making mistakes.

**The solution**: Delegate reading, analysis, and implementation to sub-agents. The main agent retains *what was done*, not *how it was done*.

```
Main agent context usage comparison:

Without skills:  Read 15 docs + 87 source files  →  ~15,000 lines in context
With skills:     Receive compressed summaries     →  ~500 lines in context
                                                     (~97% reduction)
```

## Skills

| Skill | Purpose | Trigger |
|-------|---------|---------|
| **explore-docs** | Summarize all project documentation without reading it | `/explore-docs` |
| **design-map** | Map design decisions to implementation status | `/design-map` |
| **delegate** | Offload any task and get a compressed result | `/delegate <task>` |

### /explore-docs

Spawns parallel agents to read all markdown files and produces a **hierarchical briefing** — a small index file (~50 lines, fixed size) plus per-area detail files loaded on demand.

```
> /explore-docs
> /explore-docs path/to/docs
```

Output:
```
.claude/docs-briefing/
├── index.md          ← Always loadable (~50 lines regardless of doc count)
└── area/
    ├── data-model.md ← Loaded on demand (~100-200 lines each)
    ├── editor-ux.md
    └── ...
```

The index stays small even for 100+ document projects. Area files are only loaded when needed.

### /design-map

Cross-references design documents against the codebase. Every design decision is classified as **Implemented**, **Not implemented**, or **Mismatch**. Produces a **hierarchical mapping** — the index contains only actionable items (Not implemented + Mismatch), while Implemented items live in area detail files. Supports incremental updates via `git diff`.

```
> /design-map              # Full initial generation
> /design-map update       # Incremental update after changes
```

Output:
```
.claude/design-map/
├── index.md          ← Actionable items only (~80 lines, zero Implemented items)
└── area/
    ├── data-model.md ← Full detail including Implemented (loaded on demand)
    ├── editor-ux.md
    └── ...
```

As items get implemented, they disappear from the index — it shrinks over time.

Prerequisite: Run `/explore-docs` first.

### /delegate

The general-purpose context saver. Send a one-line task description, get back a ~20-line summary of what was done, which files were touched, and what needs follow-up.

```
> /delegate Fix the authentication bug where users get 401 on valid tokens
> /delegate Add unit tests for the payment processing module
> /delegate Research how the caching layer works and summarize
```

Output: `.claude/delegate-results/{timestamp}-{task-slug}.md`

## Installation

Copy the `.claude/skills/` directory into your project:

```bash
# Clone this repo
git clone https://github.com/YOUR_USERNAME/context-saver-skills.git

# Copy skills to your project
cp -r context-saver-skills/.claude/skills/explore-docs your-project/.claude/skills/
cp -r context-saver-skills/.claude/skills/design-map your-project/.claude/skills/
cp -r context-saver-skills/.claude/skills/delegate your-project/.claude/skills/
```

Or copy all three at once:

```bash
cp -r context-saver-skills/.claude/skills/{explore-docs,design-map,delegate} your-project/.claude/skills/
```

No dependencies required. Skills are pure markdown — they work with any Claude Code project.

## Typical Workflow

```
# 1. Onboard to a new project
> /explore-docs

# 2. Understand what's implemented vs designed
> /design-map

# 3. Implement features without cluttering context
> /delegate Implement NarrativeBlock source/lock_state fields per S-D §2-1
> /delegate Add StateDimension enum and auto-tagging

# 4. Update the map after implementation
> /design-map update
```

## How It Works

All three skills follow the same architecture:

```
┌─────────────┐     task      ┌──────────────────┐    file     ┌──────────────────┐
│  Main Agent │ ──────────▶   │  Analysis Agents │ ─────────▶  │  Assembly Agent  │
│  (context   │               │  (parallel,      │  (temp      │  (reads parts,   │
│   clean)    │               │   read & work)   │   files)    │   writes final)  │
│             │  ◀──────────  │                  │             │                  │
│             │   header only │                  │             │                  │
└─────────────┘               └──────────────────┘             └──────────────────┘
```

1. **Main agent** orchestrates but never reads documents, code, or raw results
2. **Analysis agents** (parallel) do the actual reading and analysis, write results to temp files
3. **Assembly agent** compiles temp files into a single output file, then cleans up
4. Main agent reads only the output file's header (~10 lines) for reporting
5. Internal operations use **English** to save tokens; only the final user report is localized

## File Structure

```
your-project/
├── .claude/
│   ├── skills/
│   │   ├── explore-docs/
│   │   │   └── SKILL.md
│   │   ├── design-map/
│   │   │   └── SKILL.md
│   │   └── delegate/
│   │       └── SKILL.md
│   ├── docs-briefing/             ← generated by /explore-docs
│   │   ├── index.md              ←   Layer 1: quick ref (~50 lines)
│   │   └── area/                 ←   Layer 2: detail per area
│   │       ├── data-model.md
│   │       └── pipeline.md
│   ├── design-map/               ← generated by /design-map
│   │   ├── index.md              ←   Actionable items only
│   │   └── area/                 ←   Full detail per area
│   │       ├── data-model.md
│   │       └── pipeline.md
│   └── delegate-results/         ← generated by /delegate
│       ├── 20260304-1023-fix-auth-bug.md
│       └── 20260304-1045-add-unit-tests.md
```

## Configuration

No configuration needed. Each skill auto-detects:
- Document locations (Glob for `**/*.md`)
- Source code directories (standard project structures)
- Git state (for incremental updates)

## License

MIT
