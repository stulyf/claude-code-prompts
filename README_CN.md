**Language**: [English](README.md) | **中文**

# Claude Code 提示词提取（v2.1.88）

> **声明**
>
> 本仓库仅供技术研究与学习使用，请勿用于商业用途。
> 源码版权归 [Anthropic](https://www.anthropic.com) 所有。
> 如有侵权，请联系删除。

从 Claude Code 反编译源码中原封不动提取的所有提示词相关文件，共 **72 个文件**，按用途分为 5 个目录。

> 提示词以 TypeScript 源码形式保留，因为大量提示词是**动态拼接**的（根据环境、工具、配置等条件生成），无法完全以纯文本呈现。直接阅读源码中的字符串模板即可看到完整的提示词内容。

---

## 目录结构

```
claude-code_prompt/
├── system/          # 系统主提示词（核心身份 + 行为规则）
├── tools/           # 工具提示词（36个工具的使用说明）
├── compact/         # 上下文压缩/摘要提示词
├── agents/          # 子代理提示词（内置 Agent 定义）
└── misc/            # 其他辅助提示词（权限、记忆、特殊场景等）
```

---

## system/ — 系统主提示词

Claude Code 的"人格"和行为规则定义。这些内容拼接后作为 API 的 System Prompt 发送。


| 文件                          | 作用                                                                                                                                                                                                                                                                                                                                                                               |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **prompts.ts**              | **最核心的文件**。`getSystemPrompt()` 函数动态组装完整的系统提示词，包含以下段落： - 身份介绍（"You are an interactive agent..."） - 系统规则（工具权限、system-reminder 标签说明） - 做任务的规则（代码风格、不要过度设计、先读再改等） - 谨慎行动（区分安全操作和危险操作） - 使用工具的规则（优先用专用工具而非 Bash） - 语气和风格（不用 emoji、简洁、引用格式） - 输出效率（直入正题、不废话） - 环境信息（工作目录、git 状态、模型名、知识截止日期） - 自主模式（Proactive 模式下的特殊指令） 还包含 `DEFAULT_AGENT_PROMPT`（默认子代理人格）和 `computeEnvInfo()`（环境信息注入） |
| **system.ts**               | CLI 身份前缀，包含三种变体： - `"You are Claude Code, Anthropic's official CLI for Claude."` - `"...running within the Claude Agent SDK."` - `"You are a Claude agent, built on Anthropic's Claude Agent SDK."` 用于 prompt cache 的缓存键识别                                                                                                                                                       |
| **systemPrompt.ts**         | `buildEffectiveSystemPrompt()` — 系统提示词的最终组装器。按优先级合并：override → 协调器模式 → Agent 定义 → 自定义 → 默认                                                                                                                                                                                                                                                                                       |
| **systemPromptSections.ts** | 系统提示词的分段缓存机制。通过 `systemPromptSection()` 包装动态段落，支持缓存和失效（`/clear`、`/compact` 时清除）                                                                                                                                                                                                                                                                                                  |
| **outputStyles.ts**         | 输出风格配置。用户可自定义 Claude 的回复风格，会注入到系统提示词中                                                                                                                                                                                                                                                                                                                                            |
| **context.ts**              | 用户上下文注入。`getUserContext()` 收集 CLAUDE.md 内容、当前日期、git 状态等，作为对话的第一条 meta user 消息发送（不是 System Prompt 的一部分）                                                                                                                                                                                                                                                                           |


---

## tools/ — 工具提示词（36个）

每个工具的 `prompt.ts` 定义了该工具发送给模型的 `description`（使用说明）。模型根据这些说明决定何时、如何调用工具。

### 核心工具


| 文件                          | 工具名   | 作用                                                                                                     |
| --------------------------- | ----- | ------------------------------------------------------------------------------------------------------ |
| **BashTool_prompt.ts**      | Bash  | **最复杂的工具提示词（约 370 行）**。包含：命令执行规则、工具偏好（不要用 cat/grep 等）、多命令并行/串行规则、Git commit 完整流程、PR 创建流程、沙箱限制说明、后台运行说明 |
| **FileReadTool_prompt.ts**  | Read  | 读取文件内容，支持行号范围、图片文件                                                                                     |
| **FileEditTool_prompt.ts**  | Edit  | 精确字符串替换编辑文件。包含：必须先读再编辑、保留缩进、old_string 唯一性要求                                                           |
| **FileWriteTool_prompt.ts** | Write | 创建/覆盖文件                                                                                                |
| **GlobTool_prompt.ts**      | Glob  | 按文件名模式搜索（替代 find 命令）                                                                                   |
| **GrepTool_prompt.ts**      | Grep  | 按正则搜索文件内容（替代 grep/rg 命令）                                                                               |


### 代理与任务工具


| 文件                            | 工具名         | 作用                                                                              |
| ----------------------------- | ----------- | ------------------------------------------------------------------------------- |
| **AgentTool_prompt.ts**       | Agent       | **约 288 行**。启动子代理/Fork。包含：可用 agent 类型列表、Fork vs 子代理模式、何时用/何时不用、prompt 写法指导、详细示例 |
| **TodoWriteTool_prompt.ts**   | TodoWrite   | 创建和管理任务清单                                                                       |
| **TaskCreateTool_prompt.ts**  | TaskCreate  | 创建后台任务                                                                          |
| **TaskGetTool_prompt.ts**     | TaskGet     | 获取后台任务结果                                                                        |
| **TaskListTool_prompt.ts**    | TaskList    | 列出后台任务                                                                          |
| **TaskStopTool_prompt.ts**    | TaskStop    | 停止后台任务                                                                          |
| **TaskUpdateTool_prompt.ts**  | TaskUpdate  | 更新后台任务                                                                          |
| **SendMessageTool_prompt.ts** | SendMessage | 向已有 Agent 发送消息（恢复对话）                                                            |
| **TeamCreateTool_prompt.ts**  | TeamCreate  | 创建团队                                                                            |
| **TeamDeleteTool_prompt.ts**  | TeamDelete  | 删除团队                                                                            |


### 模式切换工具


| 文件                              | 工具名           | 作用                   |
| ------------------------------- | ------------- | -------------------- |
| **EnterPlanModeTool_prompt.ts** | EnterPlanMode | 进入 Plan 模式（只读讨论）     |
| **ExitPlanModeTool_prompt.ts**  | ExitPlanMode  | 退出 Plan 模式           |
| **EnterWorktreeTool_prompt.ts** | EnterWorktree | 进入 Git Worktree 隔离环境 |
| **ExitWorktreeTool_prompt.ts**  | ExitWorktree  | 退出 Git Worktree      |


### MCP 相关工具


| 文件                                 | 工具名              | 作用            |
| ---------------------------------- | ---------------- | ------------- |
| **MCPTool_prompt.ts**              | MCP              | 调用 MCP 服务器的工具 |
| **ListMcpResourcesTool_prompt.ts** | ListMcpResources | 列出 MCP 资源     |
| **ReadMcpResourceTool_prompt.ts**  | ReadMcpResource  | 读取 MCP 资源内容   |


### 信息获取工具


| 文件                          | 工具名       | 作用             |
| --------------------------- | --------- | -------------- |
| **WebSearchTool_prompt.ts** | WebSearch | 网络搜索（注意使用当前年份） |
| **WebFetchTool_prompt.ts**  | WebFetch  | 抓取网页内容         |
| **LSPTool_prompt.ts**       | LSP       | 调用语言服务器协议      |


### 其他工具


| 文件                                | 工具名             | 作用                       |
| --------------------------------- | --------------- | ------------------------ |
| **AskUserQuestionTool_prompt.ts** | AskUserQuestion | 向用户提问                    |
| **BriefTool_prompt.ts**           | Brief           | 简报模式（Proactive 模式下的简短通知） |
| **ConfigTool_prompt.ts**          | Config          | 管理配置                     |
| **NotebookEditTool_prompt.ts**    | NotebookEdit    | 编辑 Jupyter Notebook      |
| **PowerShellTool_prompt.ts**      | PowerShell      | Windows PowerShell 执行    |
| **SkillTool_prompt.ts**           | Skill           | 调用用户定义的技能                |
| **SleepTool_prompt.ts**           | Sleep           | 等待/延迟                    |
| **ToolSearchTool_prompt.ts**      | ToolSearch      | 搜索可用工具                   |
| **ScheduleCronTool_prompt.ts**    | ScheduleCron    | 定时任务                     |
| **RemoteTriggerTool_prompt.ts**   | RemoteTrigger   | 远程触发                     |


---

## compact/ — 上下文压缩提示词

当对话上下文接近 200K token 上限时，自动触发的压缩/摘要机制。


| 文件                    | 作用                                                                                                                                                                                                         |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **compact_prompt.ts** | **摘要提示词模板**。定义了让 Claude 总结对话的完整 prompt，要求生成包含 9 个章节的结构化摘要：用户意图、技术概念、文件代码、错误修复、问题解决、用户消息、待完成任务、当前工作、下一步。包含 `<analysis>` 草稿区（提高总结质量后被丢弃）和 `<summary>` 正式区。还有部分压缩（partial compact）和 up_to 方向的变体               |
| **autoCompact.ts**    | 自动压缩的触发判断逻辑。包含阈值常量（AUTOCOMPACT_BUFFER_TOKENS=13000）、`shouldAutoCompact()` 判断函数、`autoCompactIfNeeded()` 执行流程、熔断器（连续失败 3 次停止重试）                                                                              |
| **compact.ts**        | 压缩的核心执行逻辑。`compactConversation()` 函数：调用 Claude API 生成摘要 → 清理文件缓存 → 恢复关键附件（最近的文件/Plan/Skills）→ 构建压缩后消息。还包含 `stripImagesFromMessages()`、`truncateHeadForPTLRetry()`（prompt-too-long 紧急截断）、post-compact 文件恢复等 |
| **microCompact.ts**   | 细粒度工具结果清理。两种模式： - 基于时间的清理（长时间未操作后清空旧工具结果） - 缓存编辑模式（通过 API cache_edits 删除，不破坏 prompt cache）                                                                                                                 |


---

## agents/ — 子代理提示词

Claude Code 内置的各种 Agent 定义，每个有独立的系统提示词和能力配置。


| 文件                          | Agent 类型       | 作用                                                                                |
| --------------------------- | -------------- | --------------------------------------------------------------------------------- |
| **generalPurposeAgent.ts**  | generalPurpose | 通用代理。`"You are an agent for Claude Code..."` 用于复杂多步任务的自主执行                        |
| **exploreAgent.ts**         | explore        | 代码探索代理。快速搜索文件、搜索代码、回答代码库问题。支持三种深度：quick/medium/very thorough                      |
| **planAgent.ts**            | plan           | 规划代理。只做研究和规划，不写代码                                                                 |
| **verificationAgent.ts**    | verification   | 对抗性验证代理。独立验证实现是否正确，给出 PASS/FAIL/PARTIAL 判定                                        |
| **claudeCodeGuideAgent.ts** | guide          | Claude Code 使用指南代理                                                                |
| **statuslineSetup.ts**      | —              | Agent 状态栏配置                                                                       |
| **forkSubagent.ts**         | fork           | Fork 子进程。继承父进程完整上下文，`"You are a forked worker process..."`                        |
| **coordinatorMode.ts**      | coordinator    | 协调器模式。`"You are Claude Code, an AI assistant that orchestrates..."` 用于编排多个子代理协同工作 |
| **generateAgent.ts**        | —              | 用 AI 生成自定义 Agent 配置。`"You are an elite AI agent architect..."`                    |


---

## misc/ — 其他辅助提示词

各种特殊场景下使用的提示词和辅助逻辑。

### 权限与安全


| 文件                          | 作用                                                                                                                        |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **yoloClassifier.ts**       | 自动模式（YOLO）权限分类器。判断工具调用是否安全可自动批准，包含 `buildYoloSystemPrompt()` 和 CLAUDE.md 注入逻辑。引用外部 `auto_mode_system_prompt.txt`（未包含在源码中） |
| **permissionExplainer.ts**  | Shell 命令风险解释器。`"You are an expert..."` 向用户解释 bash 命令的用途和潜在风险                                                              |
| **cyberRiskInstruction.ts** | 网络安全风险指令。禁止生成恶意代码、漏洞利用等内容                                                                                                 |
| **undercover.ts**           | 卧底模式。在开源仓库工作时隐藏 Anthropic 内部信息（模型名、内部代号等）                                                                                 |


### 记忆系统


| 文件                             | 作用                                                           |
| ------------------------------ | ------------------------------------------------------------ |
| **claudemd.ts**                | CLAUDE.md 文件扫描与加载逻辑。支持三层：Managed（组织级）、User（用户级）、Project（项目级） |
| **memdir.ts**                  | 自动记忆目录。`loadMemoryPrompt()` 加载记忆并注入到系统提示词的 `memory` 段落       |
| **findRelevantMemories.ts**    | 记忆检索选择器。`"You are selecting memories..."` 从记忆库中选取与当前任务相关的记忆  |
| **extractMemories_prompts.ts** | 记忆提取提示词。从对话中自动提取值得记住的信息                                      |


### 命令提示词


| 文件                     | 作用                                   |
| ---------------------- | ------------------------------------ |
| **review_command.ts**  | `/review` 代码审查命令。包含代码审查专用的系统提示词      |
| **compact_command.ts** | `/compact` 手动压缩命令。调用 compact 流程的命令入口 |


### 辅助功能


| 文件                            | 作用                                                                                  |
| ----------------------------- | ----------------------------------------------------------------------------------- |
| **sideQuestion.ts**           | Side Question 轻量代理。`"You are a separate, lightweight agent..."` 用于快速回答不需要完整上下文的简单问题 |
| **sessionTitle.ts**           | 会话标题生成。`"You are coming up with a succinct title..."` 自动为对话生成简短标题                   |
| **dateTimeParser.ts**         | 自然语言时间解析。`"You are a date/time parser..."` 将 "下周一" 等文本转为标准时间                        |
| **agenticSessionSearch.ts**   | 会话搜索。搜索历史对话记录的系统提示词                                                                 |
| **teammatePromptAddendum.ts** | 团队模式追加提示。`"You are running as an agent in a team..."` 多 Agent 协作时的角色说明              |
| **chromePrompt.ts**           | Chrome 扩展提示词。在浏览器中使用 Claude Code 时的专用系统提示词                                          |
| **buddy_prompt.ts**           | Companion 气泡提示。桌面端伴侣功能的文案（非模型 API 使用）                                               |


---

## 提示词组装流程

所有提示词最终按以下结构发送给 Claude API：

```
System Prompt (system/ 目录的内容拼接):
  ├─ [静态/可全局缓存] 身份 + 规则 + 工具指导 + 风格
  ├─ ── __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__ ──
  └─ [动态/每会话不同] 环境信息 + 语言 + 记忆 + MCP 指令

Tools (tools/ 目录，每个工具的 description):
  ├─ BashTool: "执行 bash 命令..." + Git 规则 + 沙箱
  ├─ FileReadTool / FileEditTool / FileWriteTool
  ├─ GlobTool / GrepTool
  ├─ AgentTool + 子代理列表
  └─ ... 共 36 个工具

Messages:
  ├─ [meta user] CLAUDE.md + 日期 + git 状态 (context.ts)
  ├─ [user] 用户消息
  ├─ [assistant] Claude 回复
  └─ ... 或经过 compact/ 压缩后的摘要
```

---

## 来源

提取自 `@anthropic-ai/claude-code-source` v2.1.88 反编译源码。

---

## 声明

本仓库仅供技术研究与学习使用，请勿用于商业用途。源码版权归 [Anthropic](https://www.anthropic.com) 所有。如有侵权，请联系删除。