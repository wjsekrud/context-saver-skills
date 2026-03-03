---
name: delegate
description: >
  Context-preserving delegation skill. Offloads any task to a sub-agent so the main agent
  retains only a compressed summary of what was done, not the full details.
  Use '/delegate' followed by a task description, or trigger when you judge that a task
  would consume significant context (reading many files, debugging, implementing a feature,
  refactoring, etc.) but the main agent only needs to know the outcome.
  Also trigger when the user says "delegate this", "offload this task", "do this in background",
  "handle this without cluttering context", or when the main agent decides autonomously
  that a task should be delegated to preserve context window for project-level awareness.
  The main agent SHOULD proactively use this skill whenever it estimates a task will require
  reading 3+ files or making changes across multiple locations.
---

# Delegate

Offload a task to a sub-agent and receive only a compressed result summary. The main agent preserves its context window for project-level awareness instead of filling it with file contents, debug traces, and intermediate attempts.

## When to Delegate

The main agent should delegate when ANY of these apply:
- Task requires reading 3+ files
- Task involves debugging (unknown cause, needs exploration)
- Task is a self-contained code change (feature, fix, refactor)
- Task is research/exploration that produces a conclusion
- Main agent already knows WHAT needs to be done but doesn't need to see HOW

Do NOT delegate when:
- Task requires back-and-forth with the user (clarifications, preferences)
- Task is trivial (single-line fix in a known location)
- Task outcome affects the next immediate decision the main agent must make AND the main agent cannot proceed without seeing the details

## How to Use

The main agent passes a one-line task description. That's it. No need to specify files, approaches, or detailed instructions — the sub-agent figures it out.

```
/delegate Fix the authentication bug where users get 401 on valid tokens
/delegate Add unit tests for the payment processing module
/delegate Refactor the database connection pool to use async
/delegate Research how the caching layer works and summarize the architecture
```

## Execution

### Step 1: Dispatch

Spawn a **general-purpose** agent with the universal prompt below. Pass the task description as `{TASK}`.

**Critical**: Set `run_in_background: false` — the main agent waits for the result but does NOT read any files or do any work while waiting.

Universal prompt:

```
You are an autonomous task executor. Complete the task below independently.

## Task
{TASK}

## Working Rules

1. **Explore first**: Use Glob, Grep, and Read to understand the relevant code before making changes. Never edit code you haven't read.

2. **Make changes confidently**: When the task requires code changes, implement them fully. Do not leave TODOs or partial implementations.

3. **Ambiguity protocol**: If the task is ambiguous, choose the most reasonable interpretation. Document your choice in the result. Do NOT stop or ask for clarification — you cannot communicate with the user.

4. **Verify your work**: After making changes, verify correctness:
   - For code changes: check that modified files are syntactically valid, imports are correct, and the change is consistent with surrounding code patterns
   - For test additions: run the tests if a test runner is available
   - For research: cross-reference findings across multiple files

5. **Scope discipline**: Do exactly what was asked. Do not refactor surrounding code, add documentation, or make "improvements" beyond the task scope.

## Output Format

When done, write your result to the file: {RESULT_FILE}

The file MUST follow this exact format:

```
## Result
{1-2 sentence summary of what was done and why}

## Files
{list of files read or modified, with (read) or (modified) tag}
- path/to/file.py (modified)
- path/to/other.py (read)

## Decisions
{only if ambiguity was encountered — what you chose and why, one line each}

## Follow-up
{only if something needs attention — known limitations, related issues found, suggested next steps}
{write "None" if everything is clean}

## Verification
{what you checked to confirm correctness}
```

Keep the entire result under 20 lines. Be concise. The person reading this only needs to know WHAT changed, not HOW you arrived at the solution.
```

Result file path: `.claude/delegate-results/{timestamp}-{sanitized-task-slug}.md`

Create the directory if it doesn't exist. Generate `{timestamp}` as YYYYMMDD-HHMM and `{sanitized-task-slug}` as the first 4-5 words of the task, lowercased, joined by hyphens.

### Step 2: Receive Result

When the agent completes:

1. **Read the result file** (it's small — under 20 lines by design).
2. **Report to the user** in their language: summarize what was done, list modified files, and flag any follow-up items.
3. The result file stays in `.claude/delegate-results/` as a session log. It can be cleaned up later.

### Step 3: Context Preserved

The main agent now knows:
- What was done (1-2 sentences)
- Which files were touched (list)
- Whether anything needs follow-up

It does NOT know (and doesn't need to):
- The contents of any file that was read
- The debugging process
- Intermediate attempts or failures
- The full diff of changes

## Chaining Delegates

For large tasks, the main agent can break them into multiple delegate calls:

```
/delegate Implement the data model changes for user preferences
/delegate Add API endpoints for user preferences (data model already updated)
/delegate Add frontend components for user preferences settings page
```

Each delegation is independent. The main agent carries forward only the compressed results between them, maintaining project-level awareness without accumulating file-level details.

## Result File Cleanup

Result files accumulate in `.claude/delegate-results/`. They serve as a session log of what was delegated. The main agent may clean them up at session end or leave them for future reference.

## Key Principles

- **One-line task input**: The main agent should NOT spend context crafting detailed prompts. A single sentence describing the desired outcome is sufficient.
- **Fixed output format**: The result format is always the same 5-section structure. No surprises, no parsing needed.
- **File-based handoff**: Results go through temp files, not through the main agent's conversation context.
- **Autonomy over clarification**: Sub-agents make reasonable choices and document them, rather than blocking on ambiguity.
- **Scope discipline**: Sub-agents do exactly what's asked, nothing more. Expanding scope wastes the user's context when they read the result.
