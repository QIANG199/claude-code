# Claude Code 四大模块深入解析：Tool / Prompt / Skill / MCP

> 阅读本文档后，你将理解：工具系统怎么设计的、Prompt 怎么拼装的、Skill 和 Tool 的本质区别、MCP 外部工具怎么接入的。

---

## 一、Tool 系统 — agent 的手和脚

### 1.1 Tool 是什么

Tool 就是 agent 能做的"动作"。Claude 说"我想执行 `ls`"，harness 就调用 BashTool；Claude 说"我想读文件"，harness 就调用 FileReadTool。

**类比 Java**：Tool ≈ `Command` 模式。每个 Tool 就是一个 Command 对象。

### 1.2 Tool 接口定义（Tool.ts）

```mermaid
classDiagram
    class Tool {
        <<interface>>
        +name: string
        +aliases?: string[]
        +searchHint?: string
        +inputSchema: ZodSchema
        +inputJSONSchema?: JSONSchema
        +description(input, options) string
        +call(args, context, ...) ToolResult
        +isEnabled() boolean
        +isReadOnly(input) boolean
        +isDestructive?(input) boolean
        +isConcurrencySafe(input) boolean
        +needsPermissions: boolean
        +validateInput?(input) ValidationResult
    }
    class ToolResult {
        +data: T
        +newMessages?: Message[]
        +contextModifier?: Function
    }
    Tool --> ToolResult : call() 返回
```

**类比 Java**：

```java
// 如果用 Java 写，大概是这样：
public interface Tool<I, O> {
    String name();
    ZodSchema<I> inputSchema();
    String description(I input, Options options);
    ToolResult<O> call(I input, ExecutionContext context);  // 核心
    boolean isEnabled();
    boolean isReadOnly(I input);
    boolean needsPermissions();
}
```

### 1.3 ToolUseContext — 执行上下文

每次调用 `tool.call()` 时，会传入一个 `ToolUseContext`，它包含：

```mermaid
classDiagram
    class ToolUseContext {
        +options: ToolOptions
        +abortController: AbortController
        +readFileState: FileStateCache
        +getAppState() AppState
        +setAppState(fn) void
        +messages: Message[]
        +agentId?: string
    }
    class ToolOptions {
        +tools: Tool[]
        +mainLoopModel: string
        +mcpClients: MCPConnection[]
        +agentDefinitions: AgentDefs
    }
    ToolUseContext --> ToolOptions
```

**类比 Java**：这就是 `ApplicationContext` + `RequestContext` 的合体——既有全局状态，也有本次请求的上下文。

### 1.4 Tool 注册与过滤（tools.ts）

```mermaid
flowchart TB
    GetAll["getAllBaseTools() — 获取所有工具"]

    subgraph conditional["条件加载 (feature flag / 环境变量)"]
        direction LR
        T1["BashTool 永远加载"]
        T2["FileReadTool 永远加载"]
        T3["AgentTool 永远加载"]
        T4["REPLTool 仅 ant 用户"]
        T5["SleepTool 仅 PROACTIVE"]
        T6["... 50+ 个工具"]
    end

    Filter["getTools(permissionContext) — 按权限过滤"]

    subgraph filter_logic["过滤逻辑"]
        direction LR
        F1["SIMPLE 模式 → 只保留 Bash, Read, Edit"]
        F2["Coordinator 模式 → 只保留协调器工具"]
        F3["Agent 模式 → 按定义过滤"]
        F4["deny rules → 移除被禁止的工具"]
    end

    Result["最终工具列表 → 传给 Claude API"]

    GetAll --> conditional --> Filter --> filter_logic --> Result

    style GetAll fill:#4A90D9,color:#fff
    style Filter fill:#E67E22,color:#fff
    style Result fill:#27AE60,color:#fff
```

**类比 Java**：类似 Spring 的 `@ConditionalOnProperty`——根据配置决定是否加载某个 Bean。

### 1.5 BashTool 实现分析（最典型的 Tool）

```mermaid
flowchart TB
    Call["BashTool.call(command)"] --> V["validateInput(command)"]
    V --> P["checkPermissions(command)"]
    P --> S["parseForSecurity(command)\n安全检查: 解析命令 AST"]
    S --> E["exec(command, timeout)\n在子 shell 中执行"]
    E --> R["处理输出"]

    subgraph output["输出处理"]
        direction LR
        O1["截断过长结果"]
        O2["检测图片输出"]
        O3["记录 git 操作"]
    end

    R --> output --> Result["返回 ToolResult"]

    style Call fill:#4A90D9,color:#fff
    style S fill:#E74C3C,color:#fff
    style Result fill:#27AE60,color:#fff
```

### 1.6 AgentTool — 子 Agent 调度

```mermaid
flowchart TB
    Call["AgentTool.call(input)"] --> CTX["创建子 agent 的 ToolUseContext\n(克隆父上下文)"]
    CTX --> Filter["过滤子 agent 可用的工具"]
    Filter --> Prompt["构建子 agent 的 system prompt"]
    Prompt --> Query["调用 query()\n(就是 Day 1 学的那个循环)"]
    Query --> Return["返回子 agent 的最终回复"]

    style Call fill:#4A90D9,color:#fff
    style Query fill:#8E44AD,color:#fff
    style Return fill:#27AE60,color:#fff
```

**类比 Java**：就像在一个线程里启动另一个线程——子 agent 有自己的上下文、工具、对话历史。

---

## 二、Prompt 工程 — agent 的大脑指令

### 2.1 Prompt 构建流程

```mermaid
flowchart TB
    GP["getSystemPrompt()"]

    subgraph sections["prompt 组成部分"]
        direction TB
        S1["核心身份指令\nYou are Claude, an AI assistant..."]
        S2["工具使用指南\nYou have access to: Bash, Read, Edit..."]
        S3["Git 指令\nWhen committing, use HEREDOC..."]
        S4["MCP 服务器指令\nYou have access to MCP servers..."]
        S5["Agent 定义\nAvailable agents: Explore, Plan..."]
        S6["Memory 提示\nYou have a persistent memory system..."]
        S7["输出风格配置\nAlways respond in 简体中文..."]
        S8["模型特定指令\nYou are powered by the model..."]
    end

    GP --> sections --> Final["完整 system prompt 数组"]

    style GP fill:#4A90D9,color:#fff
    style Final fill:#27AE60,color:#fff
```

### 2.2 上下文注入（context.ts）

```mermaid
flowchart LR
    subgraph system["systemContext (缓存)"]
        direction TB
        SC1["gitStatus:\nCurrent branch: main\nStatus: (clean)"]
        SC2["cacheBreaker:\n[调试用]"]
    end

    subgraph user["userContext (缓存)"]
        direction TB
        UC1["claudeMd:\nContents of CLAUDE.md..."]
        UC2["currentDate:\nToday's date is 2026/05/02."]
    end

    system -- "附加到 system prompt 末尾" --> API["Claude API"]
    user -- "注入到用户消息前面" --> API

    style system fill:#3498DB,color:#fff
    style user fill:#2ECC71,color:#fff
    style API fill:#8E44AD,color:#fff
```

### 2.3 System Prompt 优先级链（systemPrompt.ts）

```mermaid
flowchart TB
    Start["buildEffectiveSystemPrompt()"]

    Start --> Check1{"override\nsystem prompt?"}
    Check1 -- "是" --> Override["替换一切\n(用于 loop 模式)"]

    Check1 -- "否" --> Check2{"coordinator\n模式?"}
    Check2 -- "是" --> Coord["coordinator prompt"]

    Check2 -- "否" --> Check3{"自定义\nagent?"}
    Check3 -- "是" --> Agent["agent prompt\n替换默认"]

    Check3 -- "否" --> Check4{"custom\nsystem prompt?"}
    Check4 -- "是" --> Custom["用户 --system-prompt"]

    Check4 -- "否" --> Default["默认 Claude Code prompt"]

    Override --> Append["+ appendSystemPrompt\n(总是附加在最后)"]
    Coord --> Append
    Agent --> Append
    Custom --> Append
    Default --> Append

    Append --> Final["最终 system prompt"]

    style Override fill:#E74C3C,color:#fff
    style Final fill:#27AE60,color:#fff
```

**类比 Java**：就像 Spring 的 `@Order` 注解——优先级高的配置覆盖优先级低的。

---

## 三、Skill 系统 — 预编排的指令集

### 3.1 Skill 是什么

**关键区分**：Skill ≠ Tool。

```mermaid
flowchart LR
    subgraph tool["Tool — 原子操作"]
        direction TB
        T1["BashTool: 执行命令"]
        T2["FileReadTool: 读文件"]
        T3["EditTool: 编辑文件"]
    end

    subgraph skill["Skill — 预编排指令"]
        direction TB
        S1["/simplify: 审查代码质量"]
        S2["/debug: 调试流程"]
        S3["/verify: 验证代码"]
    end

    skill -- "告诉 Claude 按流程做事" --> Claude["Claude 按指令调用 tools"]
    Claude --> tool

    style tool fill:#E67E22,color:#fff
    style skill fill:#9B59B6,color:#fff
```

**类比 Java**：
- Tool ≈ Service 方法
- Skill ≈ 一段 Spring Batch Job 定义——它不是代码，而是一组指令

### 3.2 BundledSkillDefinition 结构

```mermaid
classDiagram
    class BundledSkillDefinition {
        +name: string
        +description: string
        +aliases?: string[]
        +whenToUse?: string
        +allowedTools?: string[]
        +model?: string
        +hooks?: HooksSettings
        +context: "inline" | "fork"
        +files?: Record~string, string~
        +getPromptForCommand(args, ctx) ContentBlock[]
    }
    class Command {
        +type: "prompt"
        +name: string
        +description: string
        +allowedTools: string[]
        +source: "bundled"
        +getPromptForCommand Function
    }
    BundledSkillDefinition --> Command : registerBundledSkill() 转换
```

### 3.3 Skill 的加载流程

```mermaid
flowchart TB
    subgraph startup["启动时"]
        direction TB
        I1["initBundledSkills()\n注册内置技能"]
        I2["loadSkillsDir()\n扫描 .claude/skills/ 目录\n解析 SKILL.md 文件"]
        I3["mcpSkills.ts\n从 MCP 加载 (当前已 stub)"]
    end

    subgraph builtin["内置技能 (15+)"]
        direction LR
        B1["/simplify"]
        B2["/verify"]
        B3["/debug"]
        B4["/loop"]
        B5["/remember"]
    end

    subgraph invoke["用户输入 /simplify 时"]
        direction TB
        V1["SkillTool.call()"]
        V2["查找匹配的 Skill"]
        V3["调用 skill.getPromptForCommand()"]
        V4["获取 Skill 的 allowedTools"]
        V5["把 prompt + 工具权限注入到对话中"]
        V1 --> V2 --> V3 --> V4 --> V5
    end

    startup --> builtin
    I2 --> invoke

    style I1 fill:#4A90D9,color:#fff
    style V5 fill:#27AE60,color:#fff
```

### 3.4 SKILL.md 文件格式（磁盘技能）

```markdown
---
name: my-skill
description: 做某件事的技能
allowedTools:
  - Bash
  - Read
  - Edit
hooks:
  pre: echo "starting"
  post: echo "done"
---

你是一个专业的代码审查员。请按以下步骤工作：

1. 使用 GrepTool 搜索所有 TODO 注释
2. 使用 FileReadTool 读取相关文件
3. 使用 EditTool 修复发现的问题

完成后给出总结报告。
```

---

## 四、MCP 协议 — 外部工具接入

### 4.1 MCP 是什么

MCP（Model Context Protocol）是一个标准协议，让外部程序能向 Claude 暴露工具、资源和提示词。

**类比 Java**：MCP ≈ SPI（Service Provider Interface）。

```mermaid
flowchart LR
    CC["Claude Code\n(Client)"] <-->|"MCP 协议\nstdio / SSE / HTTP"| Server["MCP Server\n(Provider)"]

    Server --> Cap1["tools:\nListTools, CallTool"]
    Server --> Cap2["resources:\nListResources, ReadResource"]
    Server --> Cap3["prompts:\nListPrompts, GetPrompt"]

    style CC fill:#4A90D9,color:#fff
    style Server fill:#27AE60,color:#fff
```

### 4.2 MCP 客户端连接流程（client.ts）

```mermaid
flowchart TB
    Config["读取 MCP 配置\n(.claude/settings.json, 命令行参数)"]

    Config --> Loop["遍历每个 MCP 服务器配置"]

    Loop --> T1{"stdio?"}
    Loop --> T2{"sse?"}
    Loop --> T3{"streamable-http?"}
    Loop --> T4{"websocket?"}

    T1 -- "是" --> Stdio["StdioClientTransport\n启动子进程通信"]
    T2 -- "是" --> SSE["SSEClientTransport\nHTTP 长连接"]
    T3 -- "是" --> HTTP["StreamableHTTPClientTransport"]
    T4 -- "是" --> WS["WebSocketTransport"]

    Stdio --> Connect["client.connect(transport)"]
    SSE --> Connect
    HTTP --> Connect
    WS --> Connect

    Connect --> Discover["发现能力"]
    Discover --> D1["client.listTools() → 工具列表"]
    Discover --> D2["client.listResources() → 资源列表"]
    Discover --> D3["client.listPrompts() → 提示词列表"]

    D1 --> Register["注册到 harness"]
    D2 --> Register
    D3 --> Register

    Register --> R1["MCP 工具 → MCPTool 对象 → 加入工具列表"]
    Register --> R2["MCP 资源 → ListMcpResourcesTool"]
    Register --> R3["MCP 提示词 → 加入命令列表"]

    style Config fill:#4A90D9,color:#fff
    style Connect fill:#E67E22,color:#fff
    style Register fill:#27AE60,color:#fff
```

### 4.3 MCPTool — MCP 工具的包装

```mermaid
flowchart TB
    Input["Claude 调用 mcp__my-server__search(query)"]
    Input --> Find["找到对应的 MCP 客户端连接"]
    Find --> Call["client.callTool(toolName, args)"]

    Call --> Success{"成功?"}
    Success -- "文本内容" --> Text["直接返回"]
    Success -- "图片内容" --> Image["裁剪/压缩后返回"]
    Success -- "二进制内容" --> Binary["保存到文件，返回路径"]
    Success -- "失败" --> Error["返回错误信息"]

    Text --> Result["包装成 ToolResult"]
    Image --> Result
    Binary --> Result
    Error --> Result

    style Input fill:#9B59B6,color:#fff
    style Result fill:#27AE60,color:#fff
```

---

## 五、四大模块协作关系图

```mermaid
sequenceDiagram
    participant U as 用户
    participant SS as SkillSystem
    participant PS as PromptSystem
    participant QL as Query Loop
    participant TS as ToolSystem
    participant MCP as MCP Server
    participant API as Claude API

    U->>SS: /simplify
    SS->>SS: 查找 simplify skill
    SS->>PS: prompt + allowedTools

    PS->>PS: 注入到 system prompt
    PS->>PS: 注入 systemContext + userContext
    PS->>QL: 完整 prompt

    QL->>API: 发送 prompt
    API-->>QL: tool_use: Read("src/main.ts")

    QL->>TS: 查找 Read tool
    TS->>TS: FileReadTool.call()
    TS-->>QL: 文件内容

    QL->>API: [消息 + tool_result]
    API-->>QL: tool_use: mcp__search__(query)

    QL->>TS: 查找 MCP tool
    TS->>MCP: client.callTool()
    MCP-->>TS: 搜索结果
    TS-->>QL: ToolResult

    QL->>API: [消息 + tool_result]
    API-->>QL: 纯文本: "以下是代码审查结果..."

    QL-->>U: 最终结果
```

---

## 六、设计模式总结

| 模式 | 在哪里用 | Java 等价 |
|------|---------|-----------|
| Command | Tool 接口 | `Command.execute()` |
| Strategy | Tool 过滤策略 | `Strategy` 接口 |
| Observer | store.ts 状态订阅 | `EventListener` |
| Template Method | Skill.getPromptForCommand | `AbstractMethod` |
| Adapter | MCPTool 包装 MCP 工具 | `Adapter` 模式 |
| Chain of Responsibility | 消息预处理管道 | `Filter Chain` |
| Composite | system prompt 拼装 | `Builder` 模式 |
| Singleton | context.ts 的 memoize | `@Scope("singleton")` |

---

## 七、一句话总结每个模块

- **Tool**：agent 能做的原子操作，定义了"能做什么"
- **Prompt**：agent 的行为指令，定义了"怎么做"
- **Skill**：预编排的指令集，定义了"按什么流程做"
- **MCP**：外部工具接入协议，扩展了"能做什么"的边界

---

## 八、下一步

Day 3 我们将学习：
- **状态管理**：store.ts 的极简设计
- **QueryEngine**：非交互式模式（SDK 调用）
- **上下文压缩**：对话太长怎么办
- **任务系统**：多步骤任务怎么管理
