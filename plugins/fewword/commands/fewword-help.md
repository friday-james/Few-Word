---
description: "Explain FewWord plugin, how it works, and available commands"
---

# FewWord Plugin Help

Please explain the following to the user:

## What is FewWord?

FewWord is a context engineering plugin for Claude Code that automatically offloads large command outputs to the filesystem, keeping your conversation context clean and efficient.

**The Problem:** When you run commands that produce large outputs (test suites, find, logs), those outputs sit in your context forever, eating tokens and eventually getting lost when context summarizes.

**The Solution:** FewWord intercepts large outputs (>8KB), writes them to disk, and returns a smart pointer with preview + retrieval commands. You get 82% token savings on message context.

## Does it work automatically?

**Yes!** After installation, FewWord works automatically with zero configuration:

1. Install: `claude plugin install fewword@sheeki03-Few-Word`
2. Start a new session
3. Run commands as normal - large outputs are automatically offloaded

**No configuration needed.** The plugin hooks into Claude Code's PreToolUse event and intercepts Bash commands automatically.

## How It Works

```
You run a command → Output >8KB? → Yes → Write to .fewword/scratch/ → Return pointer + preview
                                → No  → Show output normally
```

**What you see for large outputs:**
```
=== [FewWord: Output offloaded] ===
File: .fewword/scratch/tool_outputs/pytest_143022.txt
Size: 45678 bytes, 882 lines
Exit: 0

=== First 10 lines ===
[preview of start]

=== Last 10 lines ===
[preview of end]

=== Retrieval commands ===
  Full: cat .fewword/scratch/tool_outputs/pytest_143022.txt
  Grep: grep 'pattern' .fewword/scratch/tool_outputs/pytest_143022.txt
```

## Available Commands

### /context-init

Set up FewWord directory structure manually (usually not needed - SessionStart does this automatically).

### /context-cleanup

View storage statistics and clean up old scratch files.

**Shows:**
- Total scratch storage size
- Number of files by age
- Option to clean files older than threshold

### /context-search <term>

Search through all offloaded context files.

**Usage:**
```
/context-search "error"
/context-search "FAILED"
/context-search "function_name"
```

Searches all files in `.fewword/scratch/` for the given term.

### /fewword-help

Show this help information (what you're reading now).

---

## What Gets Offloaded

| Offloaded | NOT Offloaded |
|-----------|---------------|
| Outputs >8KB | Small outputs (<8KB) |
| find, grep results | Interactive commands (vim, ssh) |
| Test suite output | Commands with existing redirects |
| Build logs | Pipelines (v1 limitation) |

## Auto-Cleanup

FewWord automatically cleans up old files:
- **Tool outputs**: Deleted after 60 minutes
- **Subagent workspaces**: Deleted after 120 minutes
- Cleanup runs on every SessionStart

## Escape Hatch

If you need to disable FewWord temporarily:

```bash
# Option 1: Create disable file
touch .fewword/DISABLE_OFFLOAD

# Option 2: Environment variable
export FEWWORD_DISABLE=1
```

## Privacy

FewWord logs only metadata (tool names, timestamps), never your actual data:
- Command arguments: NOT logged (may contain secrets)
- Output content: Written to local disk only, never transmitted
- MCP tools: Only parameter keys logged, not values

## Directory Structure

```
.fewword/
├── scratch/           # Ephemeral (auto-cleaned)
│   ├── tool_outputs/  # Command outputs
│   └── subagents/     # Agent workspaces
├── memory/            # Persistent
│   └── plans/         # Archived plans
└── index/             # Metadata
```

## Configuration Defaults

| Setting | Value |
|---------|-------|
| Size threshold | 8KB (~2000 tokens) |
| Preview lines | 10 (first + last) |
| Output retention | 60 minutes |
| Scratch size warning | 100MB |

## Learn More

- GitHub: https://github.com/sheeki03/Few-Word
- Inspired by: [Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus), [LangChain](https://blog.langchain.com/how-agents-can-use-filesystems-for-context-engineering/)
