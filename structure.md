# Pi Agent Harness - 项目结构文档

## 项目概述

Pi Agent Harness 是一个开源的 AI 代理框架，包含一个可自扩展的编码代理。项目采用 TypeScript 编写，基于 Node.js 运行时，使用 monorepo 架构管理多个相关包。

## 目录结构

```
pi_agent/
└── pi/                          # 项目根目录
    ├── .git/                    # Git 版本控制
    ├── .github/                 # GitHub 配置（CI/CD 等）
    ├── .husky/                  # Git hooks（pre-commit 等）
    ├── .pi/                     # Pi 项目配置
    │   ├── extensions/          # 扩展配置
    │   ├── git/                 # Git 相关配置
    │   ├── npm/                 # npm 相关配置
    │   ├── prompts/             # 提示词模板
    │   └── skills/              # 技能配置
    ├── packages/                # 核心包目录
    │   ├── agent/               # Agent 核心运行时
    │   ├── ai/                  # AI/LLM 提供商抽象层
    │   ├── coding-agent/        # 编码代理 CLI
    │   └── tui/                 # 终端 UI 库
    ├── scripts/                 # 构建、发布、测试脚本
    ├── package.json             # 根 package.json（monorepo 配置）
    ├── tsconfig.base.json       # TypeScript 基础配置
    ├── tsconfig.json            # TypeScript 配置
    └── biome.json               # Biome 代码格式化配置
```

## 核心包详解

### 1. @earendil-works/pi-agent-core (`packages/agent/`)

**功能描述**: Agent 核心运行时，提供工具调用和状态管理能力。

**目录结构**:
```
agent/
├── src/
│   ├── agent.ts          # Agent 核心实现
│   ├── agent-loop.ts     # Agent 主循环逻辑
│   ├── proxy.ts          # 代理相关功能
│   ├── types.ts          # 类型定义
│   ├── harness/          # Agent harness 框架
│   └── index.ts          # 入口文件
├── test/                 # 测试文件
├── docs/                 # 文档
└── package.json
```

**主要职责**:
- Agent 生命周期管理
- 工具调用执行
- 状态管理和持久化
- 提供可扩展的 harness 框架

---

### 2. @earendil-works/pi-ai (`packages/ai/`)

**功能描述**: 统一的多提供商 LLM API 抽象层，支持 OpenAI、Anthropic、Google 等。

**目录结构**:
```
ai/
├── src/
│   ├── providers/              # 各 LLM 提供商实现
│   │   ├── anthropic.ts
│   │   ├── openai.ts
│   │   ├── google.ts
│   │   └── ... (其他提供商)
│   ├── models.ts               # 模型定义
│   ├── models.generated.ts     # 自动生成的模型配置
│   ├── image-models.ts         # 图像模型
│   ├── types.ts                # 类型定义
│   ├── stream.ts               # 流式处理
│   ├── api-registry.ts         # API 注册表
│   └── index.ts                # 入口文件
├── test/                       # 测试文件
└── package.json
```

**主要职责**:
- 统一不同 LLM 提供商的 API 接口
- 模型管理和配置
- 流式响应处理
- API 密钥管理

---

### 3. @earendil-works/pi-coding-agent (`packages/coding-agent/`)

**功能描述**: 交互式编码代理 CLI，是用户直接使用的主要产品。

**目录结构**:
```
coding-agent/
├── src/
│   ├── core/                   # 核心功能模块
│   │   ├── agent-session.ts    # Agent 会话管理（核心文件）
│   │   ├── agent-session-runtime.ts  # 会话运行时
│   │   ├── agent-session-services.ts # 会话服务
│   │   ├── model-registry.ts   # 模型注册
│   │   ├── model-resolver.ts   # 模型解析
│   │   ├── resource-loader.ts  # 资源加载
│   │   ├── package-manager.ts  # 包管理器
│   │   ├── extensions/         # 扩展系统
│   │   ├── compaction/         # 会话压缩
│   │   └── ...
│   ├── cli/                    # CLI 命令处理
│   │   ├── args.ts             # 参数解析
│   │   ├── file-processor.ts   # 文件处理
│   │   └── ...
│   ├── modes/                  # 运行模式
│   │   ├── interactive/        # 交互模式
│   │   ├── rpc/                # RPC 模式
│   │   └── print-mode.ts       # 打印模式
│   ├── utils/                  # 工具函数
│   │   ├── git.ts              # Git 操作
│   │   ├── clipboard.ts        # 剪贴板操作
│   │   ├── image-resize.ts     # 图像处理
│   │   └── ...
│   ├── config.ts               # 配置管理
│   ├── main.ts                 # 主入口
│   ├── index.ts                # 包入口
│   └── migrations.ts           # 数据迁移
├── docs/                       # 文档目录
├── examples/                   # 示例代码
│   └── extensions/             # 扩展示例
├── test/                       # 测试文件
├── scripts/                    # 脚本工具
└── package.json
```

**主要职责**:
- 用户交互界面（CLI）
- 会话管理和历史记录
- 文件读写和代码编辑
- 执行 bash 命令
- 扩展系统支持
- 多种运行模式（交互、RPC、打印）

---

### 4. @earendil-works/pi-tui (`packages/tui/`)

**功能描述**: 终端 UI 库，支持差分渲染。

**目录结构**:
```
tui/
├── src/
│   ├── components/             # UI 组件
│   ├── tui.ts                  # TUI 核心实现
│   ├── terminal.ts             # 终端抽象
│   ├── keys.ts                 # 按键处理
│   ├── keybindings.ts          # 快捷键绑定
│   ├── autocomplete.ts         # 自动补全
│   ├── terminal-image.ts       # 终端图像显示
│   ├── stdin-buffer.ts         # stdin 缓冲处理
│   ├── utils.ts                # 工具函数
│   └── index.ts                # 入口文件
├── native/                     # 原生模块
├── test/                       # 测试文件
└── package.json
```

**主要职责**:
- 终端渲染引擎
- 差分更新优化
- 按键和事件处理
- 自动补全
- 终端图像支持

## 配置文件说明

| 文件 | 用途 |
|------|------|
| `package.json` | monorepo 配置，定义工作空间和脚本 |
| `tsconfig.base.json` | TypeScript 基础配置 |
| `biome.json` | 代码格式化和检查规则 |
| `.npmrc` | npm 配置（精确版本、最小发布年龄） |
| `.husky/pre-commit` | Git pre-commit hook |

## 构建和开发

```bash
# 安装依赖
npm install --ignore-scripts

# 构建所有包
npm run build

# 代码检查
npm run check

# 运行测试
./test.sh

# 从源码运行 pi
./pi-test.sh
```

## 测试结构

每个包都有独立的测试目录：
- `packages/agent/test/` - Agent 核心测试
- `packages/ai/test/` - AI 提供商测试
- `packages/coding-agent/test/` - 编码代理测试（最全面）
- `packages/tui/test/` - TUI 组件测试

测试文件使用 `*.test.ts` 命名约定，使用 Vitest 作为测试框架。

## 文档

- `packages/coding-agent/docs/` - 主要文档目录
  - `tui.md` - TUI 相关文档
  - `extensions.md` - 扩展系统文档
  - `skills.md` - 技能系统文档
  - `models.md` - 模型配置文档
  - `sessions.md` - 会话管理文档
  - `keybindings.md` - 快捷键文档
  - `sdk.md` - SDK 集成文档

## 技术栈

- **语言**: TypeScript 5.9+
- **运行时**: Node.js >= 22.19.0
- **包管理**: npm (workspaces)
- **构建**: esbuild
- **测试**: Vitest
- **代码检查**: Biome
- **类型检查**: TypeScript (tsgo)
