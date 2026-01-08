---
name: fewword
description: Use this skill when tool outputs exceed 2000 tokens, tasks span multiple conversation turns, sub-agents need to share state, context window is bloating, plans need to persist across summarization, terminal/log output needs selective querying, or when user mentions "offload context", "dynamic context discovery", "filesystem memory", "scratch pad", "reduce context bloat", or "just-in-time context loading".
---

# FewWord - Filesystem-Based Context Engineering

The filesystem provides a single interface for storing, retrieving, and updating effectively unlimited context. This addresses the fundamental constraint that context windows are limited while tasks often require more information.

**Core insight**: Files enable dynamic context discovery—pull relevant context on demand rather than carrying everything in the context window.

**v1 Feature**: Bash commands are automatically intercepted and their output is offloaded to files when large. You will see a pointer and preview instead of full output.

## Directory Structure

```
project/
└── .fewword/                          # All plugin data in one namespace
    ├── scratch/                     # Ephemeral (auto-cleaned hourly)
    │   ├── tool_outputs/            # Offloaded command outputs
    │   └── subagents/               # Agent workspace files
    ├── memory/                      # Persistent (survives cleanup)
    │   ├── plans/                   # Archived completed plans
    │   ├── history/                 # Archived sessions
    │   ├── patterns/                # Discovered patterns
    │   └── preferences.yaml         # User preferences
    ├── index/                       # Metadata tracking
    │   ├── current_plan.yaml        # THE canonical active plan
    │   ├── tool_log.jsonl           # Tool execution log
    │   └── mcp_metadata.jsonl       # MCP tool metadata
    └── DISABLE_OFFLOAD              # Create to disable auto-offloading
```

## Escape Hatch

If automatic offloading causes issues:
- Create file: `touch .fewword/DISABLE_OFFLOAD`
- Or set env: `export FEWWORD_DISABLE=1`

## Automatic Behaviors (v1)

### Bash Output Offloading with AI Summary

When you run a Bash command, the plugin automatically:
1. Captures stdout+stderr to a file
2. After completion, measures output size
3. If small (<8KB): shows full output normally, deletes temp file
4. If large: saves to file and outputs `[FEWWORD_SUMMARIZE]` marker
5. Preserves the original exit code

**What you see for large output:**
```
[FEWWORD_SUMMARIZE]
file: .fewword/scratch/tool_outputs/pytest_20250107_143022_a1b2c3d4.txt
command: pytest tests/ -v
bytes: 45678
lines: 1234
exit_code: 0

=== First 10 lines ===
...preview...

... (use Task tool with subagent to read and summarize full output) ...

=== Last 10 lines ===
...preview...
[/FEWWORD_SUMMARIZE]
```

**IMPORTANT - When you see `[FEWWORD_SUMMARIZE]`:**

You MUST immediately spawn a Task subagent to read and summarize the full output:

```
Task tool:
  subagent_type: "general-purpose"
  prompt: "Read the file at {file_path} and provide a comprehensive summary.
           The file contains output from: {command}
           Focus on: errors, warnings, key results, and actionable items.
           Return a structured summary that captures all important information."
```

This ensures you don't miss any important details from the full output while keeping context lean.

**Skipped commands** (v1 conservatively skips):
- Interactive: ssh, vim, less, top, watch, python, node, psql, etc.
- Already redirecting: commands with `>`, `2>`, `| tee`, `| less`
- Heredocs: commands containing `<<`
- Pipelines: commands containing `|`
- Trivial: very short commands

### MCP Tool Handling

- All MCP tool calls are logged to `.fewword/index/mcp_metadata.jsonl`
- Write-like operations (create, update, delete, commit, push) are gated
- Pagination parameters are automatically clamped to prevent excessive results

## Manual Patterns

### Pattern 1: Plan Persistence

For long-horizon tasks, use the canonical active plan:

```yaml
# .fewword/index/current_plan.yaml
objective: "Refactor authentication module"
status: in_progress
steps:
  - id: 1
    description: "Audit current auth endpoints"
    status: completed
  - id: 2
    description: "Design new token validation"
    status: in_progress
  - id: 3
    description: "Implement and test"
    status: pending
```

- Plan survives context summarization
- Re-read at turn start or when losing track
- When completed, automatically archived to `memory/plans/`

### Pattern 2: Sub-Agent File Workspaces

Sub-agents write findings directly to filesystem instead of message passing:

```
.fewword/scratch/subagents/
├── research_agent/
│   ├── findings.md
│   └── sources.jsonl
├── code_agent/
│   ├── changes.md
│   └── test_results.txt
└── synthesis.md
```

### Pattern 3: Chat History as File Reference

When context window fills:
1. Write full history to `.fewword/memory/history/session_{id}.txt`
2. Generate summary for new context window
3. Include reference: "Full history in .fewword/memory/history/session_{id}.txt"
4. Use grep to recover details lost in summarization

## Search Techniques

| Tool | Use Case | Example |
|------|----------|---------|
| `ls` / `find` | Discover structure | `find .fewword -name "*.txt" -mmin -30` |
| `grep` | Content search | `grep -rn "error" .fewword/scratch/` |
| `head`/`tail` | Boundary reads | `tail -100 .fewword/scratch/tool_outputs/log.txt` |
| `sed -n` | Line ranges | `sed -n '50,100p' file.txt` |

## References

For detailed implementation patterns, see:
- `references/implementation-patterns.md` - Code examples for each pattern
- `references/cleanup-strategies.md` - Scratch file lifecycle management
