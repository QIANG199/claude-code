# System Prompt 构建详解

> system prompt 是 agent 的"人格说明书"——它告诉 Claude "你是谁、你能做什么、你该怎么做"。

---

## 一、整体流程概览

```mermaid
flowchart TB
    Entry["getSystemPrompt(tools, model)\n入口函数"] --> Check{"CLAUDE_CODE_SIMPLE\n模式?"}
    Check -- "是" --> Simple["返回最简 prompt:\nYou are Claude Code...\nCWD + Date"]
    Check -- "否" --> Build

    subgraph build["构建完整 prompt"]
        direction TB
        Static["静态部分 (可缓存)"]
        Boundary["__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__\n缓存分界线"]
        Dynamic["动态部分 (每轮可能变化)"]
    end

    Build --> Static --> Boundary --> Dynamic --> Final["string[]\nprompt 数组"]

    style Entry fill:#4A90D9,color:#fff
    style Boundary fill:#E74C3C,color:#fff
    style Final fill:#27AE60,color:#fff
```

**关键设计**：prompt 被分成**静态部分**和**动态部分**，中间用一个分界线隔开。静态部分可以被 Anthropic API 缓存（省 token 费用），动态部分每轮可能变化。

---

## 二、getSystemPrompt() 函数详解

入口在 `src/constants/prompts.ts:444`：

```mermaid
flowchart TB
    Start["getSystemPrompt(tools, model)"] --> Parallel["并行获取三个配置"]

    subgraph parallel["三个并行任务"]
        direction LR
        P1["getSkillToolCommands()\n获取可用的 Skill 命令"]
        P2["getOutputStyleConfig()\n获取输出风格配置"]
        P3["computeSimpleEnvInfo()\n获取环境信息"]
    end

    Parallel --> Settings["获取初始设置"]
    Settings --> EnabledTools["enabledTools = Set(工具名列表)"]

    EnabledTools --> Sections["构建 dynamicSections 数组"]
    Sections --> Resolve["resolveSystemPromptSections()\n解析所有 section（带缓存）"]
    Resolve --> Assemble["组装最终 prompt 数组"]
    Assemble --> Filter[".filter(s => s !== null)\n过滤掉空值"]

    style Start fill:#4A90D9,color:#fff
    style Assemble fill:#27AE60,color:#fff
```

---

## 三、静态部分（可缓存，不变）

这些内容在同一个会话内**不会变化**，所以 Anthropic API 可以缓存它们：

```mermaid
flowchart TB
    subgraph static["静态部分 — 每次启动时计算一次"]
        direction TB
        S1["getSimpleIntroSection()\n身份定义 + 安全指令"]
        S2["getSimpleSystemSection()\n系统行为规则"]
        S3["getSimpleDoingTasksSection()\n任务执行指南"]
        S4["getActionsSection()\n危险操作警告"]
        S5["getUsingYourToolsSection()\n工具使用指南"]
        S6["getSimpleToneAndStyleSection()\n语气和风格"]
        S7["getOutputEfficiencySection()\n输出效率要求"]
    end

    S1 --> S2 --> S3 --> S4 --> S5 --> S6 --> S7

    style static fill:#2C3E50,color:#fff
```

### 3.1 身份定义（getSimpleIntroSection）

```text
You are an interactive agent that helps users with software engineering tasks.
Use the instructions below and the tools available to you to assist the user.

IMPORTANT: Assist with authorized security testing...
IMPORTANT: You must NEVER generate or guess URLs...
```

**作用**：告诉 Claude "你是谁"——一个帮助用户做软件工程任务的 agent。

### 3.2 系统行为规则（getSimpleSystemSection）

包含 6 条核心规则：

| 规则 | 说明 |
|------|------|
| 输出方式 | 用 GitHub-flavored markdown，渲染为等宽字体 |
| 权限模式 | 工具执行受权限控制，用户可批准/拒绝 |
| system-reminder | 消息中的 `<system-reminder>` 标签来自系统 |
| 外部数据警告 | 工具结果可能包含外部数据，注意 prompt injection |
| Hooks | 用户可配置 hooks，在工具调用时执行 shell 命令 |
| 自动压缩 | 对话会自动压缩，不受上下文窗口限制 |

### 3.3 任务执行指南（getSimpleDoingTasksSection）

这是**最长的一段**，包含大量行为约束：

```mermaid
flowchart TB
    D1["主要任务：软件工程\n修bug、加功能、重构、解释代码"]
    D3["用户交互规范"]
    D4["安全要求"]

    subgraph code_rules["代码风格约束 (8条)"]
        direction LR
        CS1["不加多余功能"] --- CS2["不加多余错误处理"] --- CS3["不做过度抽象"]
        CS4["默认不写注释"] --- CS5["不解释代码做了什么"] --- CS6["不删已有注释"]
        CS7["完成后要验证"] --- CS8["如实报告结果"]
    end

    D1 --> code_rules
    D1 --> D3
    D1 --> D4
```

### 3.4 工具使用指南（getUsingYourToolsSection）

告诉 Claude **优先用专用工具，不要用 Bash**：

```text
Do NOT use the Bash to run commands when a relevant dedicated tool is provided.

- To read files use Read instead of cat, head, tail, or sed
- To edit files use Edit instead of sed or awk
- To create files use Write instead of cat with heredoc
- To search for files use Glob instead of find or ls
- To search content use Grep instead of grep or rg
- Reserve Bash exclusively for system commands...
```

还包含并行执行指南：
```text
You can call multiple tools in a single response.
If tools are independent, make all independent tool calls in parallel.
```

### 3.5 危险操作警告（getActionsSection）

列出需要用户确认的操作类型：

| 类型 | 示例 |
|------|------|
| 破坏性操作 | 删除文件/分支、drop table、rm -rf |
| 不可逆操作 | force-push、git reset --hard、修改 CI/CD |
| 影响他人 | push 代码、创建 PR、发消息 |
| 上传敏感内容 | 上传到第三方工具 |

---

## 四、缓存分界线

```text
__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__
```

这是一个特殊的标记字符串。在它**之前**的内容使用 `scope: 'global'` 缓存（跨组织可共享），在它**之后**的内容包含用户/会话特定信息，不缓存。

```mermaid
flowchart LR
    subgraph before["分界线之前 — 可全局缓存"]
        direction TB
        B1["身份定义"]
        B2["系统规则"]
        B3["任务指南"]
        B4["工具指南"]
        B5["语气风格"]
    end

    Boundary["__DYNAMIC_BOUNDARY__"]

    subgraph after["分界线之后 — 会话级"]
        direction TB
        A1["session_guidance"]
        A2["memory"]
        A3["env_info"]
        A4["language"]
        A5["mcp_instructions"]
    end

    before --> Boundary --> after

    style Boundary fill:#E74C3C,color:#fff
    style before fill:#27AE60,color:#fff
    style after fill:#F39C12,color:#fff
```

---

## 五、动态部分（每轮可能变化）

这些 section 通过 `systemPromptSection()` 或 `DANGEROUS_uncachedSystemPromptSection()` 创建：

```mermaid
flowchart TB
    subgraph dynamic["动态 sections"]
        direction TB
        D1["session_guidance\n会话特定指南（Skill、Agent 使用说明）"]
        D2["memory\n记忆系统提示（persistent memory 路径等）"]
        D3["ant_model_override\n内部模型覆盖（ant-only）"]
        D4["env_info_simple\n环境信息（CWD、平台、模型名、知识截止日期）"]
        D5["language\n语言偏好（如: Always respond in 简体中文）"]
        D6["output_style\n输出风格配置"]
        D7["mcp_instructions\nMCP 服务器使用说明"]
        D8["scratchpad\n临时文件目录说明"]
        D9["frc\nFunction Result Clearing 配置"]
        D10["token_budget\nToken 预算说明（feature-gated）"]
    end

    style dynamic fill:#34495E,color:#fff
```

### 5.1 缓存策略

有两种 section 类型：

```mermaid
flowchart LR
    subgraph cached["systemPromptSection()"]
        direction TB
        CC["计算一次，缓存结果"]
        CC2["/clear 或 /compact 时清除"]
        CC3["例子: language, memory"]
    end

    subgraph uncached["DANGEROUS_uncachedSystemPromptSection()"]
        direction TB
        UC["每轮重新计算"]
        UC2["会破坏 prompt cache"]
        UC3["需要说明理由"]
        UC4["例子: mcp_instructions\n(MCP 服务器可能中途连接/断开)"]
    end

    style cached fill:#27AE60,color:#fff
    style uncached fill:#E74C3C,color:#fff
```

### 5.2 resolveSystemPromptSections() 解析过程

```mermaid
flowchart TB
    Input["sections 数组"] --> Loop["遍历每个 section"]
    Loop --> Check{"是 uncached\n且有缓存?"}
    Check -- "有缓存" --> Cache["返回缓存值"]
    Check -- "无缓存或 uncached" --> Compute["await section.compute()"]
    Compute --> Save["保存到缓存"]
    Save --> Return["返回值"]
    Cache --> Return

    style Check fill:#E67E22,color:#fff
    style Compute fill:#4A90D9,color:#fff
```

---

## 六、优先级链（buildEffectiveSystemPrompt）

在 `getSystemPrompt()` 生成默认 prompt 之后，还有一个**优先级覆盖**机制：

```mermaid
flowchart TB
    Default["getSystemPrompt() 生成的默认 prompt"] --> BES["buildEffectiveSystemPrompt()"]

    BES --> Check1{"overrideSystemPrompt\n存在?"}
    Check1 -- "是" --> Override["[overrideSystemPrompt]\n替换一切"]

    Check1 -- "否" --> Check2{"Coordinator\n模式?"}
    Check2 -- "是" --> Coord["[coordinatorPrompt + append]"]

    Check2 -- "否" --> Check3{"agent prompt\n存在?"}
    Check3 -- "是" --> Check3b{"Proactive\n模式?"}
    Check3b -- "是" --> AgentProactive["[default + agent + append]\n追加模式"]
    Check3b -- "否" --> AgentReplace["[agent + append]\n替换模式"]

    Check3 -- "否" --> Check4{"custom prompt\n存在?"}
    Check4 -- "是" --> Custom["[custom + append]"]
    Check4 -- "否" --> DefaultResult["[default + append]"]

    Override --> Result["最终 SystemPrompt"]
    Coord --> Result
    AgentProactive --> Result
    AgentReplace --> Result
    Custom --> Result
    DefaultResult --> Result

    style Override fill:#E74C3C,color:#fff
    style Result fill:#27AE60,color:#fff
```

**appendSystemPrompt** 总是附加在最后（除非 override 模式）。

---

## 七、上下文注入（context.ts）

除了 system prompt 本身，还有两个上下文会被注入到 API 请求中：

```mermaid
flowchart TB
    subgraph system["getSystemContext() — 缓存"]
        direction TB
        SC["gitStatus:\nCurrent branch: main\nMain branch: main\nStatus: (clean)\nRecent commits:\nabc123 fix bug\ndef456 add feature"]
        SC2["cacheBreaker:\n[调试用，ant-only]"]
    end

    subgraph user["getUserContext() — 缓存"]
        direction TB
        UC["claudeMd:\nContents of CLAUDE.md\n(项目指令文件内容)"]
        UC2["currentDate:\nToday's date is 2026/05/02."]
    end

    subgraph inject["注入位置"]
        direction TB
        I1["systemContext → appendSystemContext()\n附加到 system prompt 末尾"]
        I2["userContext → prependUserContext()\n注入到用户消息前面"]
    end

    system --> I1
    user --> I2

    style system fill:#3498DB,color:#fff
    style user fill:#2ECC71,color:#fff
```

### 7.1 gitStatus 内容示例

```text
This is the git status at the start of the conversation.
Note that this status is a snapshot in time, and will not update
during the conversation.

Current branch: main
Main branch (you will usually use this for PRs): main
Git user: xinyi
Status:
(clean)

Recent commits:
3b0a5e4 docs: 更新说明文档
c57e6ee docs: 文档优化完成
```

### 7.2 claudeMd 内容

就是 CLAUDE.md 文件的内容。Claude Code 会从项目目录层级中自动发现并加载所有 CLAUDE.md 文件。

---

## 八、环境信息（computeSimpleEnvInfo）

```text
Here is useful information about the environment you are running in:
<env>
Primary working directory: /path/to/project
Is a git repository: true
Platform: darwin
Shell: /bin/zsh
OS Version: macOS 14.0
</env>
You are powered by the model named Claude Opus 4.6.
The exact model ID is claude-opus-4-6.
Assistant knowledge cutoff is early 2025.
```

---

## 九、MCP 服务器指令

当有 MCP 服务器连接时，会注入它们的使用说明：

```text
# MCP Server Instructions

The following MCP servers have provided instructions for how to use
their tools and resources:

## my-server
Use the search tool to query the database.
The query tool accepts SQL syntax...
```

---

## 十、完整 prompt 结构总览

```mermaid
flowchart TB
    subgraph full["最终发给 Claude API 的 system prompt"]
        direction TB

        subgraph static["静态部分 (可全局缓存)"]
            direction TB
            ST1["身份定义: You are an interactive agent..."]
            ST2["系统规则: 输出方式、权限、hooks..."]
            ST3["任务指南: 代码风格、安全要求..."]
            ST4["危险操作警告: 破坏性操作需确认..."]
            ST5["工具使用指南: 优先用专用工具..."]
            ST6["语气风格: 不用emoji、简洁..."]
            ST7["输出效率: 直奔主题..."]
        end

        Boundary["__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__"]

        subgraph dynamic["动态部分 (会话级缓存)"]
            direction TB
            DY1["session_guidance: Skill/Agent 使用说明"]
            DY2["memory: 记忆系统路径"]
            DY3["env_info: CWD、平台、模型名"]
            DY4["language: Always respond in 简体中文"]
            DY5["output_style: 输出风格配置"]
            DY6["mcp_instructions: MCP 服务器说明"]
        end

        ST1 --> ST2 --> ST3 --> ST4 --> ST5 --> ST6 --> ST7
        ST7 --> Boundary
        Boundary --> DY1 --> DY2 --> DY3 --> DY4 --> DY5 --> DY6
    end

    subgraph external["外部注入"]
        direction LR
        EX1["systemContext\n(gitStatus) → 附加到末尾"]
        EX2["userContext\n(claudeMd + date) → 注入到用户消息前"]
    end

    full --> API["发给 Claude API"]
    external --> API

    style Boundary fill:#E74C3C,color:#fff
    style static fill:#2C3E50,color:#fff
    style dynamic fill:#34495E,color:#fff
    style API fill:#8E44AD,color:#fff
```

---

## 十一、设计精髓总结

| 设计点 | 做法 | 为什么 |
|--------|------|--------|
| 静态/动态分离 | 分界线隔开 | 静态部分可缓存，省 token 费用 |
| Section 缓存 | systemPromptSection() | 同一会话内只计算一次 |
| 不缓存标记 | DANGEROUS_uncachedSystemPromptSection() | MCP 等会变化的内容必须每轮重算 |
| 优先级链 | buildEffectiveSystemPrompt() | 支持 override > coordinator > agent > custom > default |
| 并行获取 | Promise.all() | 加速启动 |
| null 过滤 | .filter(s => s !== null) | 条件性 section 可能返回 null |
