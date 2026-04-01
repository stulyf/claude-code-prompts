**Language**: **English** | [中文](README_CN.md)

# Coding Agent Prompt Patterns

> **Disclaimer**
>
> This repository is for technical research and learning purposes only. Do not use for commercial purposes.
> Prompt design patterns are inspired by [Anthropic](https://www.anthropic.com)'s Claude Code architecture.
> If there is any infringement, please contact for removal.

A collection of prompt engineering patterns for building AI coding agents, organized into **72 reference files** across 5 categories. These patterns are derived from studying how production-grade coding agents structure their prompts.

> Files are in TypeScript source form because most prompts are **dynamically assembled** (generated based on environment, tools, settings, etc.) and cannot be fully represented as plain text. Read the string templates to see the prompt content.

---

## Directory Structure

```
coding_agent-prompts/
├── system/          # Core system prompts (identity + behavior rules)
├── tools/           # Tool prompts (descriptions for 36 tools)
├── compact/         # Context compression / summarization prompts
├── agents/          # Sub-agent prompts (built-in agent definitions)
└── misc/            # Other auxiliary prompts (permissions, memory, etc.)
```

---

## system/ — Core System Prompts

Defines the agent's "personality" and behavior rules. These are concatenated and sent as the API's System Prompt.

| File | Purpose |
|------|---------|
| **prompts.ts** | **The most critical file.** Dynamically assembles the full system prompt, containing sections for:<br>- Identity intro<br>- System rules (tool permissions, system-reminder tags)<br>- Task guidelines (code style, no over-engineering, read before edit)<br>- Careful actions (distinguish safe vs dangerous operations)<br>- Tool usage rules (prefer dedicated tools over shell)<br>- Tone and style (concise, reference format)<br>- Output efficiency (get to the point)<br>- Environment info (working directory, git status, model name, knowledge cutoff)<br>- Autonomous mode instructions |
| **system.ts** | CLI identity prefix variants used for prompt cache key identification |
| **systemPrompt.ts** | The final system prompt assembler. Merges by priority: override → coordinator mode → agent definition → custom → default |
| **systemPromptSections.ts** | Section-based caching mechanism for system prompts with cache invalidation support |
| **outputStyles.ts** | Output style configuration. Users can customize the agent's response style |
| **context.ts** | User context injection. Collects project config, current date, git status, etc. |

---

## tools/ — Tool Prompts (36 tools)

Each tool's `prompt.ts` defines the `description` sent to the model. The model uses these descriptions to decide when and how to call each tool.

### Core Tools

| File | Tool Name | Purpose |
|------|-----------|---------|
| **BashTool_prompt.ts** | Bash | Most complex tool prompt (~370 lines). Includes: command execution rules, tool preferences, parallel/sequential multi-command rules, Git commit workflow, PR creation workflow, sandbox restrictions, background execution |
| **FileReadTool_prompt.ts** | Read | Read file contents, supports line ranges and image files |
| **FileEditTool_prompt.ts** | Edit | Exact string replacement editing. Includes: must read before edit, preserve indentation, uniqueness requirement |
| **FileWriteTool_prompt.ts** | Write | Create/overwrite files |
| **GlobTool_prompt.ts** | Glob | Search files by name pattern (replaces `find`) |
| **GrepTool_prompt.ts** | Grep | Search file contents by regex (replaces `grep`/`rg`) |

### Agent & Task Tools

| File | Tool Name | Purpose |
|------|-----------|---------|
| **AgentTool_prompt.ts** | Agent | ~288 lines. Launch sub-agents/forks. Includes: available agent types, fork vs sub-agent modes, when to use/not use, prompt writing guidance, detailed examples |
| **TodoWriteTool_prompt.ts** | TodoWrite | Create and manage task lists |
| **TaskCreateTool_prompt.ts** | TaskCreate | Create background tasks |
| **TaskGetTool_prompt.ts** | TaskGet | Get background task results |
| **TaskListTool_prompt.ts** | TaskList | List background tasks |
| **TaskStopTool_prompt.ts** | TaskStop | Stop background tasks |
| **TaskUpdateTool_prompt.ts** | TaskUpdate | Update background tasks |
| **SendMessageTool_prompt.ts** | SendMessage | Send messages to existing agents (resume conversations) |
| **TeamCreateTool_prompt.ts** | TeamCreate | Create teams |
| **TeamDeleteTool_prompt.ts** | TeamDelete | Delete teams |

### Mode Switching Tools

| File | Tool Name | Purpose |
|------|-----------|---------|
| **EnterPlanModeTool_prompt.ts** | EnterPlanMode | Enter Plan mode (read-only discussion) |
| **ExitPlanModeTool_prompt.ts** | ExitPlanMode | Exit Plan mode |
| **EnterWorktreeTool_prompt.ts** | EnterWorktree | Enter isolated Git Worktree environment |
| **ExitWorktreeTool_prompt.ts** | ExitWorktree | Exit Git Worktree |

### MCP Tools

| File | Tool Name | Purpose |
|------|-----------|---------|
| **MCPTool_prompt.ts** | MCP | Call MCP server tools |
| **ListMcpResourcesTool_prompt.ts** | ListMcpResources | List MCP resources |
| **ReadMcpResourceTool_prompt.ts** | ReadMcpResource | Read MCP resource content |

### Information Retrieval Tools

| File | Tool Name | Purpose |
|------|-----------|---------|
| **WebSearchTool_prompt.ts** | WebSearch | Web search (note: use current year) |
| **WebFetchTool_prompt.ts** | WebFetch | Fetch webpage content |
| **LSPTool_prompt.ts** | LSP | Language Server Protocol calls |

### Other Tools

| File | Tool Name | Purpose |
|------|-----------|---------|
| **AskUserQuestionTool_prompt.ts** | AskUserQuestion | Ask the user questions |
| **BriefTool_prompt.ts** | Brief | Brief mode (short notifications in autonomous mode) |
| **ConfigTool_prompt.ts** | Config | Manage configuration |
| **NotebookEditTool_prompt.ts** | NotebookEdit | Edit Jupyter Notebooks |
| **PowerShellTool_prompt.ts** | PowerShell | Windows PowerShell execution |
| **SkillTool_prompt.ts** | Skill | Invoke user-defined skills |
| **SleepTool_prompt.ts** | Sleep | Wait/delay |
| **ToolSearchTool_prompt.ts** | ToolSearch | Search available tools |
| **ScheduleCronTool_prompt.ts** | ScheduleCron | Scheduled tasks |
| **RemoteTriggerTool_prompt.ts** | RemoteTrigger | Remote triggers |

---

## compact/ — Context Compression Prompts

Prompt patterns for context management when conversation approaches token limits.

| File | Purpose |
|------|---------|
| **compact_prompt.ts** | Summary prompt template. Defines the prompt for conversation summarization, requiring a structured summary with 9 sections: user intent, technical concepts, files & code, errors & fixes, problem solving, user messages, pending tasks, current work, next steps. Uses `<analysis>` scratchpad + `<summary>` formal section pattern |
| **autoCompact.ts** | Auto-compact trigger logic. Contains threshold constants, trigger conditions, execution flow, and circuit breaker pattern |
| **compact.ts** | Core compaction logic. Summary generation → cache cleanup → key attachment restoration → post-compact message construction |
| **microCompact.ts** | Fine-grained tool result cleanup. Two modes:<br>- Time-based cleanup (clear old tool results after inactivity)<br>- Cache editing mode (delete via API without breaking prompt cache) |

---

## agents/ — Sub-Agent Prompts

Built-in agent definitions, each with independent system prompts and capability configurations.

| File | Agent Type | Purpose |
|------|-----------|---------|
| **generalPurposeAgent.ts** | generalPurpose | General-purpose agent for autonomous execution of complex multi-step tasks |
| **exploreAgent.ts** | explore | Code exploration agent. Fast file search, code search, codebase Q&A. Supports three depth levels |
| **planAgent.ts** | plan | Planning agent. Research and planning only, no code writing |
| **verificationAgent.ts** | verification | Adversarial verification agent. Independently verifies implementation correctness |
| **claudeCodeGuideAgent.ts** | guide | Usage guide agent |
| **statuslineSetup.ts** | — | Agent status bar configuration |
| **forkSubagent.ts** | fork | Fork subprocess. Inherits parent's full context |
| **coordinatorMode.ts** | coordinator | Coordinator mode for orchestrating multiple sub-agents |
| **generateAgent.ts** | — | AI-powered custom agent configuration generation |

---

## misc/ — Other Auxiliary Prompts

Prompt patterns for various special scenarios.

### Permissions & Security

| File | Purpose |
|------|---------|
| **yoloClassifier.ts** | Auto-mode permission classifier. Determines if tool calls are safe for auto-approval |
| **permissionExplainer.ts** | Shell command risk explainer. Explains command purposes and potential risks to users |
| **cyberRiskInstruction.ts** | Security risk instructions. Safety guidelines for code generation |
| **undercover.ts** | Mode for hiding internal information when working on open-source repos |

### Memory System

| File | Purpose |
|------|---------|
| **claudemd.ts** | Project configuration file scanning and loading logic. Supports multi-layer configuration |
| **memdir.ts** | Automatic memory directory. Loads memories and injects them into the system prompt |
| **findRelevantMemories.ts** | Memory retrieval selector. Selects task-relevant memories from the memory store |
| **extractMemories_prompts.ts** | Memory extraction prompts. Automatically extracts notable information from conversations |

### Command Prompts

| File | Purpose |
|------|---------|
| **review_command.ts** | Code review command with review-specific system prompts |
| **compact_command.ts** | Manual compaction command entry point |

### Auxiliary Features

| File | Purpose |
|------|---------|
| **sideQuestion.ts** | Lightweight agent for quick answers without full context |
| **sessionTitle.ts** | Session title auto-generation |
| **dateTimeParser.ts** | Natural language time parser |
| **agenticSessionSearch.ts** | Session history search prompts |
| **teammatePromptAddendum.ts** | Team mode role descriptions for multi-agent collaboration |
| **chromePrompt.ts** | Browser extension prompts |
| **buddy_prompt.ts** | Companion feature prompts |

---

## Prompt Assembly Flow

A typical coding agent assembles prompts in this structure:

```
System Prompt:
  ├─ [Static / cacheable] Identity + Rules + Tool guidance + Style
  ├─ ── dynamic boundary ──
  └─ [Dynamic / per-session] Environment info + Language + Memory + Extensions

Tools (each tool's description):
  ├─ Shell tool: command execution rules + Git workflow + Sandbox
  ├─ File tools: Read / Edit / Write
  ├─ Search tools: Glob / Grep
  ├─ Agent tool + sub-agent listing
  └─ ... 36 tools total

Messages:
  ├─ [meta user] Project config + date + git status
  ├─ [user] User messages
  ├─ [assistant] Agent responses
  └─ ... or post-compact summarized messages
```

---

## Inspired By

Prompt patterns in this repository are inspired by studying [Anthropic](https://www.anthropic.com)'s Claude Code architecture. This is a research reference, not an official release.

---

## Disclaimer

This repository is for technical research and learning purposes only. Do not use for commercial purposes. Prompt design patterns are inspired by [Anthropic](https://www.anthropic.com)'s product architecture. If there is any infringement, please contact for removal.
