**Language**: [English](README.md) | **中文**

# Coding Agent 提示词模式参考

> **声明**
>
> 本仓库仅供技术研究与学习使用，请勿用于商业用途。
> 提示词设计模式借鉴了 [Anthropic](https://www.anthropic.com) 的 Claude Code 产品架构。
> 如有侵权，请联系删除。

一组用于构建 AI 编码代理的提示词工程模式参考，共 **72 个参考文件**，按用途分为 5 个目录。这些模式来源于对生产级编码代理提示词结构的研究分析。

> 文件以 TypeScript 源码形式保留，因为大量提示词是**动态拼接**的（根据环境、工具、配置等条件生成），无法完全以纯文本呈现。直接阅读源码中的字符串模板即可看到提示词内容。

---

## 目录结构

```
coding_agent-prompts/
├── system/          # 系统主提示词（核心身份 + 行为规则）
├── tools/           # 工具提示词（36个工具的使用说明）
├── compact/         # 上下文压缩/摘要提示词
├── agents/          # 子代理提示词（内置 Agent 定义）
└── misc/            # 其他辅助提示词（权限、记忆、特殊场景等）
```

---

## system/ — 系统主提示词

定义编码代理的"人格"和行为规则。这些内容拼接后作为 API 的 System Prompt 发送。

| 文件 | 作用 |
|------|------|
| **prompts.ts** | **最核心的文件**。动态组装完整的系统提示词，包含以下段落：<br>- 身份介绍<br>- 系统规则（工具权限、标签说明）<br>- 做任务的规则（代码风格、不要过度设计、先读再改等）<br>- 谨慎行动（区分安全操作和危险操作）<br>- 使用工具的规则（优先用专用工具而非 Shell）<br>- 语气和风格（简洁、引用格式）<br>- 输出效率（直入正题）<br>- 环境信息（工作目录、git 状态、模型名、知识截止日期）<br>- 自主模式指令 |
| **system.ts** | CLI 身份前缀变体，用于 prompt cache 缓存键识别 |
| **systemPrompt.ts** | 系统提示词的最终组装器。按优先级合并：override → 协调器模式 → Agent 定义 → 自定义 → 默认 |
| **systemPromptSections.ts** | 系统提示词的分段缓存机制，支持缓存和失效 |
| **outputStyles.ts** | 输出风格配置。用户可自定义代理的回复风格 |
| **context.ts** | 用户上下文注入。收集项目配置、当前日期、git 状态等 |

---

## tools/ — 工具提示词（36个）

每个工具的 `prompt.ts` 定义了发送给模型的 `description`（使用说明）。模型根据这些说明决定何时、如何调用工具。

### 核心工具

| 文件 | 工具名 | 作用 |
|------|--------|------|
| **BashTool_prompt.ts** | Bash | 最复杂的工具提示词（约 370 行）。包含：命令执行规则、工具偏好、多命令并行/串行规则、Git commit 流程、PR 创建流程、沙箱限制、后台运行 |
| **FileReadTool_prompt.ts** | Read | 读取文件内容，支持行号范围、图片文件 |
| **FileEditTool_prompt.ts** | Edit | 精确字符串替换编辑。包含：必须先读再编辑、保留缩进、唯一性要求 |
| **FileWriteTool_prompt.ts** | Write | 创建/覆盖文件 |
| **GlobTool_prompt.ts** | Glob | 按文件名模式搜索（替代 find 命令） |
| **GrepTool_prompt.ts** | Grep | 按正则搜索文件内容（替代 grep/rg 命令） |

### 代理与任务工具

| 文件 | 工具名 | 作用 |
|------|--------|------|
| **AgentTool_prompt.ts** | Agent | 约 288 行。启动子代理/Fork。包含：agent 类型列表、Fork vs 子代理模式、使用指导、详细示例 |
| **TodoWriteTool_prompt.ts** | TodoWrite | 创建和管理任务清单 |
| **TaskCreateTool_prompt.ts** | TaskCreate | 创建后台任务 |
| **TaskGetTool_prompt.ts** | TaskGet | 获取后台任务结果 |
| **TaskListTool_prompt.ts** | TaskList | 列出后台任务 |
| **TaskStopTool_prompt.ts** | TaskStop | 停止后台任务 |
| **TaskUpdateTool_prompt.ts** | TaskUpdate | 更新后台任务 |
| **SendMessageTool_prompt.ts** | SendMessage | 向已有 Agent 发送消息 |
| **TeamCreateTool_prompt.ts** | TeamCreate | 创建团队 |
| **TeamDeleteTool_prompt.ts** | TeamDelete | 删除团队 |

### 模式切换工具

| 文件 | 工具名 | 作用 |
|------|--------|------|
| **EnterPlanModeTool_prompt.ts** | EnterPlanMode | 进入 Plan 模式（只读讨论） |
| **ExitPlanModeTool_prompt.ts** | ExitPlanMode | 退出 Plan 模式 |
| **EnterWorktreeTool_prompt.ts** | EnterWorktree | 进入 Git Worktree 隔离环境 |
| **ExitWorktreeTool_prompt.ts** | ExitWorktree | 退出 Git Worktree |

### MCP 相关工具

| 文件 | 工具名 | 作用 |
|------|--------|------|
| **MCPTool_prompt.ts** | MCP | 调用 MCP 服务器工具 |
| **ListMcpResourcesTool_prompt.ts** | ListMcpResources | 列出 MCP 资源 |
| **ReadMcpResourceTool_prompt.ts** | ReadMcpResource | 读取 MCP 资源内容 |

### 信息获取工具

| 文件 | 工具名 | 作用 |
|------|--------|------|
| **WebSearchTool_prompt.ts** | WebSearch | 网络搜索 |
| **WebFetchTool_prompt.ts** | WebFetch | 抓取网页内容 |
| **LSPTool_prompt.ts** | LSP | 语言服务器协议调用 |

### 其他工具

| 文件 | 工具名 | 作用 |
|------|--------|------|
| **AskUserQuestionTool_prompt.ts** | AskUserQuestion | 向用户提问 |
| **BriefTool_prompt.ts** | Brief | 简报模式（自主模式下的简短通知） |
| **ConfigTool_prompt.ts** | Config | 管理配置 |
| **NotebookEditTool_prompt.ts** | NotebookEdit | 编辑 Jupyter Notebook |
| **PowerShellTool_prompt.ts** | PowerShell | Windows PowerShell 执行 |
| **SkillTool_prompt.ts** | Skill | 调用用户定义的技能 |
| **SleepTool_prompt.ts** | Sleep | 等待/延迟 |
| **ToolSearchTool_prompt.ts** | ToolSearch | 搜索可用工具 |
| **ScheduleCronTool_prompt.ts** | ScheduleCron | 定时任务 |
| **RemoteTriggerTool_prompt.ts** | RemoteTrigger | 远程触发 |

---

## compact/ — 上下文压缩提示词

当对话上下文接近 token 上限时，自动触发的压缩/摘要机制的提示词模式。

| 文件 | 作用 |
|------|------|
| **compact_prompt.ts** | 摘要提示词模板。定义了对话总结的 prompt，要求生成包含 9 个章节的结构化摘要。使用 `<analysis>` 草稿区 + `<summary>` 正式区的模式 |
| **autoCompact.ts** | 自动压缩的触发判断逻辑。包含阈值常量、触发条件、执行流程、熔断器模式 |
| **compact.ts** | 压缩的核心执行逻辑。摘要生成 → 缓存清理 → 关键附件恢复 → 压缩后消息构建 |
| **microCompact.ts** | 细粒度工具结果清理。两种模式：<br>- 基于时间的清理（长时间未操作后清空旧工具结果）<br>- 缓存编辑模式（不破坏 prompt cache） |

---

## agents/ — 子代理提示词

内置的各种 Agent 定义，每个有独立的系统提示词和能力配置。

| 文件 | Agent 类型 | 作用 |
|------|-----------|------|
| **generalPurposeAgent.ts** | generalPurpose | 通用代理，用于复杂多步任务的自主执行 |
| **exploreAgent.ts** | explore | 代码探索代理。快速搜索文件、搜索代码、回答代码库问题 |
| **planAgent.ts** | plan | 规划代理。只做研究和规划，不写代码 |
| **verificationAgent.ts** | verification | 对抗性验证代理。独立验证实现是否正确 |
| **claudeCodeGuideAgent.ts** | guide | 使用指南代理 |
| **statuslineSetup.ts** | — | Agent 状态栏配置 |
| **forkSubagent.ts** | fork | Fork 子进程，继承父进程完整上下文 |
| **coordinatorMode.ts** | coordinator | 协调器模式，用于编排多个子代理协同工作 |
| **generateAgent.ts** | — | 用 AI 生成自定义 Agent 配置 |

---

## misc/ — 其他辅助提示词

各种特殊场景下的提示词模式。

### 权限与安全

| 文件 | 作用 |
|------|------|
| **yoloClassifier.ts** | 自动模式权限分类器。判断工具调用是否安全可自动批准 |
| **permissionExplainer.ts** | Shell 命令风险解释器。向用户解释命令的用途和潜在风险 |
| **cyberRiskInstruction.ts** | 安全风险指令。代码生成的安全准则 |
| **undercover.ts** | 在开源仓库工作时隐藏内部信息的模式 |

### 记忆系统

| 文件 | 作用 |
|------|------|
| **claudemd.ts** | 项目配置文件的扫描与加载逻辑，支持多层配置 |
| **memdir.ts** | 自动记忆目录。加载记忆并注入到系统提示词 |
| **findRelevantMemories.ts** | 记忆检索选择器。从记忆库中选取与当前任务相关的记忆 |
| **extractMemories_prompts.ts** | 记忆提取提示词。从对话中自动提取值得记住的信息 |

### 命令提示词

| 文件 | 作用 |
|------|------|
| **review_command.ts** | 代码审查命令，包含审查专用的系统提示词 |
| **compact_command.ts** | 手动压缩命令入口 |

### 辅助功能

| 文件 | 作用 |
|------|------|
| **sideQuestion.ts** | 轻量代理，用于快速回答不需要完整上下文的简单问题 |
| **sessionTitle.ts** | 会话标题自动生成 |
| **dateTimeParser.ts** | 自然语言时间解析 |
| **agenticSessionSearch.ts** | 会话历史搜索提示词 |
| **teammatePromptAddendum.ts** | 团队模式追加提示，多 Agent 协作时的角色说明 |
| **chromePrompt.ts** | 浏览器扩展提示词 |
| **buddy_prompt.ts** | 伴侣功能提示词 |

---

## 提示词组装流程

一个典型编码代理的提示词组装结构：

```
System Prompt:
  ├─ [静态/可缓存] 身份 + 规则 + 工具指导 + 风格
  ├─ ── 动态边界 ──
  └─ [动态/每会话不同] 环境信息 + 语言 + 记忆 + 扩展指令

Tools (每个工具的 description):
  ├─ Shell 工具: 命令执行规则 + Git 流程 + 沙箱
  ├─ 文件工具: Read / Edit / Write
  ├─ 搜索工具: Glob / Grep
  ├─ Agent 工具 + 子代理列表
  └─ ... 共 36 个工具

Messages:
  ├─ [meta user] 项目配置 + 日期 + git 状态
  ├─ [user] 用户消息
  ├─ [assistant] 代理回复
  └─ ... 或经过压缩后的摘要消息
```

---

## 借鉴来源

本仓库的提示词模式借鉴了 [Anthropic](https://www.anthropic.com) 的 Claude Code 产品架构设计，仅作为研究参考，非官方发布。

---

## 声明

本仓库仅供技术研究与学习使用，请勿用于商业用途。提示词设计模式借鉴了 [Anthropic](https://www.anthropic.com) 的产品架构。如有侵权，请联系删除。
