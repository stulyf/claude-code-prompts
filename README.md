**Language**: **English** | [中文](README_CN.md)

# Claude Code Prompt Extraction (v2.1.88)

> **Disclaimer**
>
> This repository is for technical research and learning purposes only. Do not use for commercial purposes.
> All source code copyright belongs to [Anthropic](https://www.anthropic.com).
> If there is any infringement, please contact for removal.

All prompt-related files extracted verbatim from the decompiled Claude Code source code — **72 files** in total, organized into 5 directories.

> Prompts are kept in TypeScript source form because most are **dynamically assembled** (generated based on environment, tools, settings, etc.) and cannot be fully represented as plain text. Read the string templates in the source to see the complete prompt content.

---

## Directory Structure

```
claude-code_prompt/
├── system/          # Core system prompts (identity + behavior rules)
├── tools/           # Tool prompts (descriptions for 36 tools)
├── compact/         # Context compression / summarization prompts
├── agents/          # Sub-agent prompts (built-in agent definitions)
└── misc/            # Other auxiliary prompts (permissions, memory, etc.)
```

---

## system/ — Core System Prompts

Defines Claude Code's "personality" and behavior rules. These are concatenated and sent as the API's System Prompt.

| File | Purpose |
|------|---------|
| **prompts.ts** | **The most critical file.** `getSystemPrompt()` dynamically assembles the full system prompt, containing sections for:<br>- Identity intro ("You are an interactive agent...")<br>- System rules (tool permissions, system-reminder tags)<br>- Task guidelines (code style, no over-engineering, read before edit)<br>- Careful actions (distinguish safe vs dangerous operations)<br>- Tool usage rules (prefer dedicated tools over Bash)<br>- Tone and style (no emoji, concise, reference format)<br>- Output efficiency (get to the point)<br>- Environment info (working directory, git status, model name, knowledge cutoff)<br>- Autonomous mode (Proactive mode instructions)<br>Also includes `DEFAULT_AGENT_PROMPT` and `computeEnvInfo()` |
| **system.ts** | CLI identity prefix with three variants:<br>- `"You are Claude Code, Anthropic's official CLI for Claude."`<br>- `"...running within the Claude Agent SDK."`<br>- `"You are a Claude agent, built on Anthropic's Claude Agent SDK."`<br>Used for prompt cache key identification |
| **systemPrompt.ts** | `buildEffectiveSystemPrompt()` — the final assembler. Merges by priority: override → coordinator mode → agent definition → custom → default |
| **systemPromptSections.ts** | Section-based caching mechanism for system prompts. Wraps dynamic sections via `systemPromptSection()`, with cache invalidation on `/clear` and `/compact` |
| **outputStyles.ts** | Output style configuration. Users can customize Claude's response style, injected into the system prompt |
| **context.ts** | User context injection. `getUserContext()` collects CLAUDE.md content, current date, git status, etc., sent as the first meta user message (not part of System Prompt) |

---

## tools/ — Tool Prompts (36 tools)

Each tool's `prompt.ts` defines the `description` sent to the model. The model uses these descriptions to decide when and how to call each tool.

### Core Tools

| File | Tool Name | Purpose |
|------|-----------|---------|
| **BashTool_prompt.ts** | Bash | **Most complex tool prompt (~370 lines).** Includes: command execution rules, tool preferences (don't use cat/grep), parallel/sequential multi-command rules, full Git commit workflow, PR creation workflow, sandbox restrictions, background execution |
| **FileReadTool_prompt.ts** | Read | Read file contents, supports line ranges and image files |
| **FileEditTool_prompt.ts** | Edit | Exact string replacement editing. Includes: must read before edit, preserve indentation, old_string uniqueness requirement |
| **FileWriteTool_prompt.ts** | Write | Create/overwrite files |
| **GlobTool_prompt.ts** | Glob | Search files by name pattern (replaces `find`) |
| **GrepTool_prompt.ts** | Grep | Search file contents by regex (replaces `grep`/`rg`) |

### Agent & Task Tools

| File | Tool Name | Purpose |
|------|-----------|---------|
| **AgentTool_prompt.ts** | Agent | **~288 lines.** Launch sub-agents/forks. Includes: available agent types, fork vs sub-agent modes, when to use/not use, prompt writing guidance, detailed examples |
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
| **BriefTool_prompt.ts** | Brief | Brief mode (short notifications in Proactive mode) |
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

Automatically triggered compression/summarization when conversation context approaches the 200K token limit.

| File | Purpose |
|------|---------|
| **compact_prompt.ts** | **Summary prompt template.** Defines the full prompt for Claude to summarize conversations, requiring a structured summary with 9 sections: user intent, technical concepts, files & code, errors & fixes, problem solving, user messages, pending tasks, current work, next steps. Includes `<analysis>` scratchpad (improves quality, then discarded) and `<summary>` formal section. Also has partial compact and up_to direction variants |
| **autoCompact.ts** | Auto-compact trigger logic. Contains threshold constants (AUTOCOMPACT_BUFFER_TOKENS=13000), `shouldAutoCompact()`, `autoCompactIfNeeded()` execution flow, circuit breaker (stops retrying after 3 consecutive failures) |
| **compact.ts** | Core compaction logic. `compactConversation()`: calls Claude API to generate summary → clears file cache → restores key attachments (recent files/Plan/Skills) → builds post-compact messages. Also includes `stripImagesFromMessages()`, `truncateHeadForPTLRetry()` (prompt-too-long emergency truncation), post-compact file restoration |
| **microCompact.ts** | Fine-grained tool result cleanup. Two modes:<br>- Time-based cleanup (clear old tool results after long inactivity)<br>- Cache editing mode (delete via API cache_edits without breaking prompt cache) |

---

## agents/ — Sub-Agent Prompts

Built-in agent definitions for Claude Code, each with independent system prompts and capability configurations.

| File | Agent Type | Purpose |
|------|-----------|---------|
| **generalPurposeAgent.ts** | generalPurpose | General-purpose agent. `"You are an agent for Claude Code..."` for autonomous execution of complex multi-step tasks |
| **exploreAgent.ts** | explore | Code exploration agent. Fast file search, code search, codebase Q&A. Supports three depth levels: quick/medium/very thorough |
| **planAgent.ts** | plan | Planning agent. Research and planning only, no code writing |
| **verificationAgent.ts** | verification | Adversarial verification agent. Independently verifies implementation correctness, gives PASS/FAIL/PARTIAL verdicts |
| **claudeCodeGuideAgent.ts** | guide | Claude Code usage guide agent |
| **statuslineSetup.ts** | — | Agent status bar configuration |
| **forkSubagent.ts** | fork | Fork subprocess. Inherits parent's full context, `"You are a forked worker process..."` |
| **coordinatorMode.ts** | coordinator | Coordinator mode. `"You are Claude Code, an AI assistant that orchestrates..."` for orchestrating multiple sub-agents |
| **generateAgent.ts** | — | AI-powered custom agent generation. `"You are an elite AI agent architect..."` |

---

## misc/ — Other Auxiliary Prompts

Prompts and auxiliary logic for various special scenarios.

### Permissions & Security

| File | Purpose |
|------|---------|
| **yoloClassifier.ts** | Auto-mode (YOLO) permission classifier. Determines if tool calls are safe for auto-approval. Contains `buildYoloSystemPrompt()` and CLAUDE.md injection logic. References external `auto_mode_system_prompt.txt` (not included in source) |
| **permissionExplainer.ts** | Shell command risk explainer. `"You are an expert..."` explains bash command purposes and potential risks to users |
| **cyberRiskInstruction.ts** | Cyber security risk instructions. Prohibits generating malicious code, exploits, etc. |
| **undercover.ts** | Undercover mode. Hides Anthropic internal information (model names, internal codenames, etc.) when working on open-source repos |

### Memory System

| File | Purpose |
|------|---------|
| **claudemd.ts** | CLAUDE.md file scanning and loading logic. Supports three layers: Managed (org-level), User (user-level), Project (project-level) |
| **memdir.ts** | Automatic memory directory. `loadMemoryPrompt()` loads memories and injects them into the system prompt's `memory` section |
| **findRelevantMemories.ts** | Memory retrieval selector. `"You are selecting memories..."` selects task-relevant memories from the memory store |
| **extractMemories_prompts.ts** | Memory extraction prompts. Automatically extracts memorable information from conversations |

### Command Prompts

| File | Purpose |
|------|---------|
| **review_command.ts** | `/review` code review command. Contains code review-specific system prompts |
| **compact_command.ts** | `/compact` manual compaction command. Command entry point for the compact workflow |

### Auxiliary Features

| File | Purpose |
|------|---------|
| **sideQuestion.ts** | Side Question lightweight agent. `"You are a separate, lightweight agent..."` for quick answers without full context |
| **sessionTitle.ts** | Session title generation. `"You are coming up with a succinct title..."` auto-generates short conversation titles |
| **dateTimeParser.ts** | Natural language time parser. `"You are a date/time parser..."` converts text like "next Monday" to standard timestamps |
| **agenticSessionSearch.ts** | Session search. System prompts for searching conversation history |
| **teammatePromptAddendum.ts** | Team mode addendum. `"You are running as an agent in a team..."` role descriptions for multi-agent collaboration |
| **chromePrompt.ts** | Chrome extension prompts. Dedicated system prompts for using Claude Code in the browser |
| **buddy_prompt.ts** | Companion bubble prompts. Desktop companion feature copy (not used for model API) |

---

## Prompt Assembly Flow

All prompts are ultimately sent to the Claude API in this structure:

```
System Prompt (concatenated from system/ files):
  ├─ [Static / globally cacheable] Identity + Rules + Tool guidance + Style
  ├─ ── __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__ ──
  └─ [Dynamic / per-session] Environment info + Language + Memory + MCP instructions

Tools (from tools/ files, each tool's description):
  ├─ BashTool: "Execute bash commands..." + Git rules + Sandbox
  ├─ FileReadTool / FileEditTool / FileWriteTool
  ├─ GlobTool / GrepTool
  ├─ AgentTool + sub-agent listing
  └─ ... 36 tools total

Messages:
  ├─ [meta user] CLAUDE.md + date + git status (context.ts)
  ├─ [user] User messages
  ├─ [assistant] Claude responses
  └─ ... or post-compact/ summarized messages
```

---

## Source

Extracted from `@anthropic-ai/claude-code-source` v2.1.88 decompiled source code. All files copied verbatim without modification.

---

## Disclaimer

This repository is for technical research and learning purposes only. Do not use for commercial purposes. All source code copyright belongs to [Anthropic](https://www.anthropic.com). If there is any infringement, please contact for removal.
