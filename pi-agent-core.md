# @earendil-works/pi-agent-core

## 概述

`@earendil-works/pi-agent-core` 是 Pi Agent Harness 的核心运行时包，提供通用的 Agent 运行时、工具调用、状态管理和会话持久化能力。它是构建在 `@earendil-works/pi-ai` 之上的高层抽象，为上层应用（如编码代理）提供完整的 Agent 生命周期管理。

**版本**: 0.78.0  
**许可证**: MIT  
**作者**: Mario Zechner

## 核心架构

```
┌─────────────────────────────────────────────────────────────┐
│                    AgentHarness (高层 API)                    │
│  - 会话管理  - 工具管理  - 资源管理  - 压缩  - 树导航        │
├─────────────────────────────────────────────────────────────┤
│                      Agent (状态包装器)                       │
│  - 消息队列  - 事件订阅  - 状态管理  - 流控制                │
├─────────────────────────────────────────────────────────────┤
│                  Agent Loop (核心循环)                        │
│  - LLM 调用  - 工具执行  - 事件发射  - 错误处理              │
├─────────────────────────────────────────────────────────────┤
│                  @earendil-works/pi-ai                        │
│  - 多提供商 LLM API  - 流式处理  - 模型管理                  │
└─────────────────────────────────────────────────────────────┘
```

## 目录结构

```
packages/agent/
├── src/
│   ├── agent.ts              # Agent 类 - 状态包装器
│   ├── agent-loop.ts         # 核心 Agent 循环
│   ├── types.ts              # 类型定义
│   ├── proxy.ts              # 代理流函数
│   ├── index.ts              # 包入口
│   ├── node.ts               # Node.js 特定导出
│   └── harness/              # Agent Harness 框架
│       ├── agent-harness.ts  # AgentHarness 主类
│       ├── types.ts          # Harness 类型定义
│       ├── messages.ts       # 消息转换工具
│       ├── skills.ts         # 技能系统
│       ├── prompt-templates.ts # 提示词模板
│       ├── system-prompt.ts  # 系统提示词生成
│       ├── session/          # 会话管理
│       │   ├── session.ts    # Session 类
│       │   ├── memory-repo.ts # 内存会话存储
│       │   ├── jsonl-repo.ts # JSONL 文件存储
│       │   ├── repo-utils.ts # 存储工具
│       │   └── uuid.ts       # UUID 生成
│       ├── compaction/       # 会话压缩
│       │   ├── compaction.ts # 压缩逻辑
│       │   ├── branch-summarization.ts # 分支摘要
│       │   └── utils.ts      # 压缩工具
│       ├── env/              # 执行环境抽象
│       └── utils/            # 工具函数
│           ├── shell-output.ts # Shell 输出处理
│           └── truncate.ts   # 截断工具
├── test/                     # 测试文件
├── docs/                     # 文档
└── package.json
```

## 核心模块详解

### 1. Agent 类 (`agent.ts`)

Agent 类是底层 Agent 循环的状态包装器，提供以下功能：

#### 主要职责
- **状态管理**: 维护系统提示词、模型、思维级别、工具和消息
- **事件订阅**: 支持生命周期事件的订阅和监听
- **消息队列**: 实现 steering 和 follow-up 消息队列
- **流控制**: 管理 prompt、continue、abort 等操作

#### 关键接口

```typescript
interface AgentOptions {
  initialState?: Partial<AgentState>;
  convertToLlm?: (messages: AgentMessage[]) => Message[];
  transformContext?: (messages: AgentMessage[]) => Promise<AgentMessage[]>;
  streamFn?: StreamFn;
  getApiKey?: (provider: string) => Promise<string | undefined>;
  beforeToolCall?: (context: BeforeToolCallContext) => Promise<BeforeToolCallResult>;
  afterToolCall?: (context: AfterToolCallContext) => Promise<AfterToolCallResult>;
  prepareNextTurn?: () => Promise<AgentLoopTurnUpdate | undefined>;
  steeringMode?: QueueMode;
  followUpMode?: QueueMode;
  toolExecution?: ToolExecutionMode;
}
```

#### 主要方法

| 方法 | 描述 |
|------|------|
| `prompt(message)` | 发送新消息启动对话 |
| `continue()` | 从当前转录继续 |
| `steer(message)` | 注入转向消息（当前轮次后处理） |
| `followUp(message)` | 注入后续消息（Agent 停止后处理） |
| `abort()` | 中止当前运行 |
| `reset()` | 重置所有状态 |
| `subscribe(listener)` | 订阅生命周期事件 |
| `waitForIdle()` | 等待当前运行完成 |

#### 事件类型

```typescript
type AgentEvent =
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
  | { type: "tool_execution_update"; toolCallId: string; toolName: string; partialResult: any }
  | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

---

### 2. Agent Loop (`agent-loop.ts`)

Agent Loop 是核心执行引擎，负责协调 LLM 调用和工具执行。

#### 主要函数

```typescript
// 启动新的 Agent 循环
function runAgentLoop(
  prompts: AgentMessage[],
  context: AgentContext,
  config: AgentLoopConfig,
  emit: AgentEventSink,
  signal?: AbortSignal,
  streamFn?: StreamFn
): Promise<AgentMessage[]>;

// 从当前上下文继续（用于重试）
function runAgentLoopContinue(
  context: AgentContext,
  config: AgentLoopConfig,
  emit: AgentEventSink,
  signal?: AbortSignal,
  streamFn?: StreamFn
): Promise<AgentMessage[]>;
```

#### 执行流程

```
1. 发射 agent_start 事件
2. 注入初始消息
3. 进入主循环:
   a. 发射 turn_start
   b. 注入待处理消息（steering/next-turn）
   c. 流式获取 LLM 响应
   d. 如果有工具调用:
      - 准备工具调用（验证参数、检查权限）
      - 执行工具（顺序或并行）
      - 收集工具结果
   e. 发射 turn_end
   f. 检查是否应该停止
   g. 获取 steering 消息
   h. 如果没有更多消息，检查 follow-up
4. 发射 agent_end
```

#### 工具执行模式

```typescript
type ToolExecutionMode = "sequential" | "parallel";

// 顺序执行：每个工具依次执行
// 并行执行：预检顺序，执行并发，结果按完成顺序发射
```

#### 消息队列模式

```typescript
type QueueMode = "all" | "one-at-a-time";

// all: 一次排空所有队列消息
// one-at-a-time: 每次只处理一条消息
```

---

### 3. Agent Harness (`harness/agent-harness.ts`)

AgentHarness 是高层 API，封装了完整的 Agent 生命周期管理。

#### 主要职责

- **会话管理**: 持久化对话历史
- **工具管理**: 动态添加/移除工具
- **资源管理**: 管理技能和提示词模板
- **压缩管理**: 自动/手动压缩对话历史
- **树导航**: 支持对话分支和导航
- **事件系统**: 提供丰富的生命周期事件

#### 核心类

```typescript
class AgentHarness<TSkill, TPromptTemplate, TTool> {
  readonly env: ExecutionEnv;
  
  // 构造函数
  constructor(options: AgentHarnessOptions);
  
  // 核心方法
  prompt(text: string, options?: { images?: ImageContent[] }): Promise<AssistantMessage>;
  skill(name: string, additionalInstructions?: string): Promise<AssistantMessage>;
  promptFromTemplate(name: string, args?: string[]): Promise<AssistantMessage>;
  steer(text: string): Promise<void>;
  followUp(text: string): Promise<void>;
  nextTurn(text: string): Promise<void>;
  
  // 状态管理
  getModel(): Model<any>;
  setModel(model: Model<any>): Promise<void>;
  getThinkingLevel(): ThinkingLevel;
  setThinkingLevel(level: ThinkingLevel): Promise<void>;
  
  // 工具管理
  getTools(): TTool[];
  setTools(tools: TTool[], activeToolNames?: string[]): Promise<void>;
  getActiveTools(): TTool[];
  setActiveTools(toolNames: string[]): Promise<void>;
  
  // 资源管理
  getResources(): AgentHarnessResources;
  setResources(resources: AgentHarnessResources): Promise<void>;
  
  // 会话管理
  compact(customInstructions?: string): Promise<CompactResult>;
  navigateTree(targetId: string, options?: NavigateTreeOptions): Promise<NavigateTreeResult>;
  appendMessage(message: AgentMessage): Promise<void>;
  
  // 生命周期
  abort(): Promise<AbortResult>;
  waitForIdle(): Promise<void>;
  
  // 事件订阅
  subscribe(listener: (event: AgentHarnessEvent) => Promise<void>): () => void;
  on<TType>(type: TType, handler: (event: any) => Promise<any>): () => void;
}
```

#### 配置选项

```typescript
interface AgentHarnessOptions<TSkill, TPromptTemplate, TTool> {
  env: ExecutionEnv;                    // 执行环境（文件系统 + Shell）
  session: Session;                     // 会话存储
  tools?: TTool[];                      // 可用工具
  resources?: AgentHarnessResources;    // 技能和提示词模板
  systemPrompt?: string | ((context) => string | Promise<string>);
  getApiKeyAndHeaders?: (model: Model) => Promise<{ apiKey: string; headers?: Record<string, string> }>;
  streamOptions?: AgentHarnessStreamOptions;
  model: Model<any>;                    // 当前模型
  thinkingLevel?: ThinkingLevel;        // 思维级别
  activeToolNames?: string[];           // 活跃工具名称
  steeringMode?: QueueMode;             // 转向队列模式
  followUpMode?: QueueMode;             // 后续队列模式
}
```

#### 生命周期事件

```typescript
type AgentHarnessEvent = AgentEvent | AgentHarnessOwnEvent;

type AgentHarnessOwnEvent =
  | QueueUpdateEvent           // 队列更新
  | SavePointEvent             // 保存点
  | AbortEvent                 // 中止
  | SettledEvent               // 结算完成
  | BeforeAgentStartEvent      // Agent 启动前
  | ContextEvent               // 上下文事件
  | BeforeProviderRequestEvent // 提供商请求前
  | BeforeProviderPayloadEvent // 提供商负载前
  | AfterProviderResponseEvent // 提供商响应后
  | ToolCallEvent              // 工具调用
  | ToolResultEvent            // 工具结果
  | SessionBeforeCompactEvent  // 会话压缩前
  | SessionCompactEvent        // 会话压缩
  | SessionBeforeTreeEvent     // 树导航前
  | SessionTreeEvent           // 树导航
  | ModelUpdateEvent           // 模型更新
  | ThinkingLevelUpdateEvent   // 思维级别更新
  | ToolsUpdateEvent           // 工具更新
  | ResourcesUpdateEvent;      // 资源更新
```

---

### 4. 会话管理 (`harness/session/`)

#### Session 类

Session 类提供会话的持久化和树状结构管理。

```typescript
class Session<TMetadata extends SessionMetadata> {
  constructor(storage: SessionStorage<TMetadata>);
  
  // 元数据
  getMetadata(): Promise<TMetadata>;
  getSessionName(): Promise<string | undefined>;
  
  // 条目管理
  getLeafId(): Promise<string | null>;
  getEntry(id: string): Promise<SessionTreeEntry | undefined>;
  getEntries(): Promise<SessionTreeEntry[]>;
  getBranch(fromId?: string): Promise<SessionTreeEntry[]>;
  buildContext(): Promise<SessionContext>;
  
  // 追加条目
  appendMessage(message: AgentMessage): Promise<string>;
  appendThinkingLevelChange(thinkingLevel: string): Promise<string>;
  appendModelChange(provider: string, modelId: string): Promise<string>;
  appendActiveToolsChange(activeToolNames: string[]): Promise<string>;
  appendCompaction(summary: string, firstKeptEntryId: string, tokensBefore: number, details?: unknown): Promise<string>;
  appendCustomEntry(customType: string, data?: unknown): Promise<string>;
  appendCustomMessageEntry(customType: string, content: string | Content[], display: boolean, details?: unknown): Promise<string>;
  appendLabel(targetId: string, label: string | undefined): Promise<string>;
  appendSessionName(name: string): Promise<string>;
  
  // 树导航
  moveTo(entryId: string | null, summary?: { summary: string; details?: unknown }): Promise<string | undefined>;
}
```

#### 会话树条目类型

```typescript
type SessionTreeEntry =
  | MessageEntry              // 消息
  | ThinkingLevelChangeEntry  // 思维级别变更
  | ModelChangeEntry          // 模型变更
  | ActiveToolsChangeEntry    // 活跃工具变更
  | CompactionEntry           // 压缩条目
  | BranchSummaryEntry        // 分支摘要
  | CustomEntry               // 自定义条目
  | CustomMessageEntry        // 自定义消息
  | LabelEntry                // 标签
  | SessionInfoEntry          // 会话信息
  | LeafEntry;                // 叶子节点
```

#### 会话上下文构建

```typescript
function buildSessionContext(pathEntries: SessionTreeEntry[]): SessionContext {
  // 遍历条目路径，构建:
  // - messages: 消息列表（包含压缩摘要和分支摘要）
  // - thinkingLevel: 当前思维级别
  // - model: 当前模型
  // - activeToolNames: 当前活跃工具
}
```

#### 存储实现

**内存存储** (`memory-repo.ts`):
- 用于测试和临时会话
- 所有数据存储在内存中

**JSONL 存储** (`jsonl-repo.ts`):
- 持久化到文件系统
- 每个会话一个目录
- 条目存储为 JSONL 格式

---

### 5. 会话压缩 (`harness/compaction/`)

#### 压缩逻辑

```typescript
// 压缩设置
interface CompactionSettings {
  enabled: boolean;          // 启用自动压缩
  reserveTokens: number;     // 为摘要预留的 token 数
  keepRecentTokens: number;  // 保留的近期 token 数
}

// 默认设置
const DEFAULT_COMPACTION_SETTINGS = {
  enabled: true,
  reserveTokens: 16384,
  keepRecentTokens: 20000,
};

// 准备压缩
function prepareCompaction(
  pathEntries: SessionTreeEntry[],
  settings: CompactionSettings
): Result<CompactionPreparation | undefined, CompactionError>;

// 执行压缩
async function compact(
  preparation: CompactionPreparation,
  model: Model<any>,
  apiKey: string,
  headers?: Record<string, string>,
  customInstructions?: string,
  signal?: AbortSignal,
  thinkingLevel?: ThinkingLevel
): Promise<Result<CompactionResult, CompactionError>>;
```

#### 压缩流程

1. **评估上下文**: 计算当前 token 使用量
2. **查找切点**: 找到合适的压缩切点
3. **提取历史**: 提取需要压缩的消息
4. **生成摘要**: 使用 LLM 生成结构化摘要
5. **追加条目**: 将压缩条目追加到会话
6. **更新上下文**: 使用摘要和保留消息重建上下文

#### 摘要格式

```markdown
## Goal
[用户目标]

## Constraints & Preferences
- [约束和偏好]

## Progress
### Done
- [x] [已完成任务]

### In Progress
- [ ] [进行中任务]

### Blocked
- [阻塞问题]

## Key Decisions
- **[决策]**: [理由]

## Next Steps
1. [下一步]

## Critical Context
- [关键上下文]
```

#### 分支摘要

```typescript
async function generateBranchSummary(
  entries: SessionTreeEntry[],
  options: GenerateBranchSummaryOptions
): Promise<Result<BranchSummaryResult, BranchSummaryError>>;
```

用于在对话树导航时生成分支摘要，保留关键上下文信息。

---

### 6. 技能系统 (`harness/skills.ts`)

#### 技能定义

```typescript
interface Skill {
  name: string;                    // 技能名称
  description: string;             // 技能描述
  content: string;                 // 技能内容
  filePath: string;                // 技能文件路径
  disableModelInvocation?: boolean; // 禁止模型调用
}
```

#### 技能加载

```typescript
async function loadSkills(
  env: ExecutionEnv,
  dirs: string | string[]
): Promise<{ skills: Skill[]; diagnostics: SkillDiagnostic[] }>;

async function loadSourcedSkills<TSource, TSkill extends Skill>(
  env: ExecutionEnv,
  inputs: Array<{ path: string; source: TSource }>,
  mapSkill?: (skill: Skill, source: TSource) => TSkill
): Promise<{ skills: Array<{ skill: TSkill; source: TSource }>; diagnostics: Array<SkillDiagnostic & { source: TSource }> }>;
```

#### 技能文件格式

技能文件使用 `SKILL.md` 格式，包含 YAML frontmatter：

```markdown
---
name: my-skill
description: A description of the skill
disable-model-invocation: false
---

# Skill Content

Instructions for the skill...
```

#### 技能调用格式

```typescript
function formatSkillInvocation(skill: Skill, additionalInstructions?: string): string;
// 返回:
// <skill name="skill-name" location="/path/to/SKILL.md">
// References are relative to /path/to/.
//
// [skill content]
// </skill>
```

---

### 7. 提示词模板 (`harness/prompt-templates.ts`)

#### 模板定义

```typescript
interface PromptTemplate {
  name: string;           // 模板名称
  description?: string;   // 模板描述
  content: string;        // 模板内容
}
```

#### 模板加载

```typescript
async function loadPromptTemplates(
  env: ExecutionEnv,
  paths: string | string[]
): Promise<{ promptTemplates: PromptTemplate[]; diagnostics: PromptTemplateDiagnostic[] }>;
```

#### 参数替换

```typescript
function substituteArgs(content: string, args: string[]): string;
// 支持:
// $1, $2, ... - 位置参数
// $@ - 所有参数
// $ARGUMENTS - 所有参数
// ${@:N} - 从第 N 个参数开始
// ${@:N:L} - 从第 N 个参数开始，取 L 个
```

#### 调用格式

```typescript
function formatPromptTemplateInvocation(template: PromptTemplate, args: string[]): string;
```

---

### 8. 执行环境 (`harness/types.ts`)

#### 文件系统接口

```typescript
interface FileSystem {
  cwd: string;
  
  absolutePath(path: string): Promise<Result<string, FileError>>;
  joinPath(parts: string[]): Promise<Result<string, FileError>>;
  readTextFile(path: string): Promise<Result<string, FileError>>;
  readTextLines(path: string, options?: { maxLines?: number }): Promise<Result<string[], FileError>>;
  readBinaryFile(path: string): Promise<Result<Uint8Array, FileError>>;
  writeFile(path: string, content: string | Uint8Array): Promise<Result<void, FileError>>;
  appendFile(path: string, content: string | Uint8Array): Promise<Result<void, FileError>>;
  fileInfo(path: string): Promise<Result<FileInfo, FileError>>;
  listDir(path: string): Promise<Result<FileInfo[], FileError>>;
  canonicalPath(path: string): Promise<Result<string, FileError>>;
  exists(path: string): Promise<Result<boolean, FileError>>;
  createDir(path: string, options?: { recursive?: boolean }): Promise<Result<void, FileError>>;
  remove(path: string, options?: { recursive?: boolean; force?: boolean }): Promise<Result<void, FileError>>;
  createTempDir(prefix?: string): Promise<Result<string, FileError>>;
  createTempFile(options?: { prefix?: string; suffix?: string }): Promise<Result<string, FileError>>;
  cleanup(): Promise<void>;
}
```

#### Shell 接口

```typescript
interface Shell {
  exec(command: string, options?: ExecutionEnvExecOptions): Promise<Result<{ stdout: string; stderr: string; exitCode: number }, ExecutionError>>;
  cleanup(): Promise<void>;
}

interface ExecutionEnvExecOptions {
  cwd?: string;
  env?: Record<string, string>;
  timeout?: number;
  abortSignal?: AbortSignal;
  onStdout?: (chunk: string) => void;
  onStderr?: (chunk: string) => void;
}
```

#### 执行环境

```typescript
interface ExecutionEnv extends FileSystem, Shell {}
```

---

### 9. 消息类型 (`harness/messages.ts`)

#### 自定义消息类型

```typescript
// Bash 执行消息
interface BashExecutionMessage {
  role: "bashExecution";
  command: string;
  output: string;
  exitCode: number | undefined;
  cancelled: boolean;
  truncated: boolean;
  fullOutputPath?: string;
  timestamp: number;
  excludeFromContext?: boolean;
}

// 自定义消息
interface CustomMessage<T = unknown> {
  role: "custom";
  customType: string;
  content: string | (TextContent | ImageContent)[];
  display: boolean;
  details?: T;
  timestamp: number;
}

// 分支摘要消息
interface BranchSummaryMessage {
  role: "branchSummary";
  summary: string;
  fromId: string;
  timestamp: number;
}

// 压缩摘要消息
interface CompactionSummaryMessage {
  role: "compactionSummary";
  summary: string;
  tokensBefore: number;
  timestamp: number;
}
```

#### 消息转换

```typescript
function convertToLlm(messages: AgentMessage[]): Message[];
// 将自定义消息转换为 LLM 兼容格式:
// - bashExecution → user 消息（格式化输出）
// - custom → user 消息
// - branchSummary → user 消息（包含摘要标签）
// - compactionSummary → user 消息（包含摘要标签）
```

---

### 10. 代理流函数 (`proxy.ts`)

用于通过代理服务器路由 LLM 调用。

```typescript
function streamProxy(
  model: Model<any>,
  context: Context,
  options: ProxyStreamOptions
): ProxyMessageEventStream;

interface ProxyStreamOptions extends SimpleStreamOptions {
  signal?: AbortSignal;
  authToken: string;
  proxyUrl: string;
}
```

#### 代理事件类型

```typescript
type ProxyAssistantMessageEvent =
  | { type: "start" }
  | { type: "text_start"; contentIndex: number }
  | { type: "text_delta"; contentIndex: number; delta: string }
  | { type: "text_end"; contentIndex: number; contentSignature?: string }
  | { type: "thinking_start"; contentIndex: number }
  | { type: "thinking_delta"; contentIndex: number; delta: string }
  | { type: "thinking_end"; contentIndex: number; contentSignature?: string }
  | { type: "toolcall_start"; contentIndex: number; id: string; toolName: string }
  | { type: "toolcall_delta"; contentIndex: number; delta: string }
  | { type: "toolcall_end"; contentIndex: number }
  | { type: "done"; reason: StopReason; usage: Usage }
  | { type: "error"; reason: StopReason; errorMessage?: string; usage: Usage };
```

## 错误处理

### 错误类型层次

```typescript
// 基础错误
class FileError extends Error { code: FileErrorCode; path?: string; }
class ExecutionError extends Error { code: ExecutionErrorCode; }
class CompactionError extends Error { code: CompactionErrorCode; }
class BranchSummaryError extends Error { code: BranchSummaryErrorCode; }
class SessionError extends Error { code: SessionErrorCode; }
class AgentHarnessError extends Error { code: AgentHarnessErrorCode; }
```

### Result 类型

```typescript
type Result<TValue, TError> = 
  | { ok: true; value: TValue }
  | { ok: false; error: TError };

function ok<TValue, TError>(value: TValue): Result<TValue, TError>;
function err<TValue, TError>(error: TError): Result<TValue, TError>;
function getOrThrow<TValue, TError>(result: Result<TValue, TError>): TValue;
function getOrUndefined<TValue extends object, TError>(result: Result<TValue, TError>): TValue | undefined;
function toError(error: unknown): Error;
```

## 依赖关系

```json
{
  "dependencies": {
    "@earendil-works/pi-ai": "^0.78.0",
    "ignore": "7.0.5",
    "typebox": "1.1.38",
    "yaml": "2.9.0"
  }
}
```

- **@earendil-works/pi-ai**: 底层 LLM API 抽象
- **ignore**: .gitignore 风格的文件忽略规则
- **typebox**: JSON Schema 类型验证
- **yaml**: YAML 解析（用于 frontmatter）

## 使用示例

### 基本 Agent 使用

```typescript
import { Agent } from "@earendil-works/pi-agent-core";

const agent = new Agent({
  initialState: {
    systemPrompt: "You are a helpful assistant.",
    model: someModel,
    tools: [myTool],
  },
  streamFn: myStreamFn,
  getApiKey: async (provider) => await getApiKey(provider),
});

// 订阅事件
agent.subscribe(async (event) => {
  if (event.type === "message_end") {
    console.log("Assistant:", event.message);
  }
});

// 发送消息
await agent.prompt("Hello, how are you?");

// 注入转向消息
agent.steer("Actually, I meant something else...");

// 等待完成
await agent.waitForIdle();
```

### AgentHarness 使用

```typescript
import { AgentHarness } from "@earendil-works/pi-agent-core";
import { nodeEnv } from "@earendil-works/pi-agent-core/node";

const harness = new AgentHarness({
  env: nodeEnv(),
  session: mySession,
  tools: [readFileTool, writeFileTool, bashTool],
  model: someModel,
  systemPrompt: "You are a coding assistant.",
});

// 订阅事件
harness.on("tool_call", async (event) => {
  console.log(`Tool called: ${event.toolName}`);
});

// 发送消息
const response = await harness.prompt("Read the main.ts file");

// 使用技能
await harness.skill("code-review");

// 使用提示词模板
await harness.promptFromTemplate("explain-code", ["src/main.ts"]);

// 压缩会话
await harness.compaction("Focus on recent changes");

// 导航对话树
await harness.navigateTree(someEntryId, { summarize: true });
```

### 自定义工具

```typescript
import { type AgentTool, type AgentToolResult } from "@earendil-works/pi-agent-core";
import { Type } from "typebox";

const readFileTool: AgentTool<typeof ReadFileSchema> = {
  name: "read_file",
  label: "Read File",
  description: "Read the contents of a file",
  parameters: Type.Object({
    path: Type.String({ description: "File path to read" }),
  }),
  execute: async (toolCallId, params, signal, onUpdate): Promise<AgentToolResult<any>> => {
    const content = await fs.readFile(params.path, "utf-8");
    return {
      content: [{ type: "text", text: content }],
      details: { path: params.path, size: content.length },
    };
  },
};
```

## 测试

```bash
# 运行所有测试
npm run test

# 运行 harness 测试
npm run test:harness

# 运行带覆盖率的 harness 测试
npm run coverage:harness
```

## 最佳实践

1. **错误处理**: 始终使用 Result 类型处理可能失败的操作
2. **资源清理**: 使用 cleanup() 方法释放资源
3. **事件订阅**: 在不需要时取消订阅以避免内存泄漏
4. **会话管理**: 定期压缩长会话以控制 token 使用
5. **工具设计**: 工具应该原子化，避免副作用
6. **类型安全**: 使用 TypeScript 类型确保类型安全

## 相关文档

- [@earendil-works/pi-ai](../ai/README.md) - 底层 LLM API
- [@earendil-works/pi-coding-agent](../coding-agent/README.md) - 编码代理实现
- [@earendil-works/pi-tui](../tui/README.md) - 终端 UI 库
