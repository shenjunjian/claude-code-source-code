## CLI 参数解析逻辑分析

### 1. 整体架构

Claude Code 的 CLI 参数解析采用了**分层解析策略**，结合了 **Commander.js** 框架和自定义的**预解析逻辑**。

### 2. 核心组件

#### 2.1 预解析工具 (`src/utils/cliArgs.ts`)

```typescript
// 两个核心函数：

// 1. eagerParseCliFlag - 在 Commander 之前提前解析特定标志
export function eagerParseCliFlag(
  flagName: string,
  argv: string[] = process.argv,
): string | undefined

// 2. extractArgsAfterDoubleDash - 处理 -- 分隔符约定
export function extractArgsAfterDoubleDash(
  commandOrValue: string,
  args: string[] = [],
): { command: string; args: string[] }
```

#### 2.2 主解析流程 (`src/main.tsx`)

**第一阶段：早期标志检测（在 init() 之前）**

```typescript
// 第 797-813 行：检测非交互模式标志
const hasPrintFlag = cliArgs.includes('-p') || cliArgs.includes('--print');
const hasInitOnlyFlag = cliArgs.includes('--init-only');
const hasSdkUrl = cliArgs.some(arg => arg.startsWith('--sdk-url'));
const isNonInteractive = hasPrintFlag || hasInitOnlyFlag || hasSdkUrl || !process.stdout.isTTY;
```

**第二阶段：eagerLoadSettings()（第 852 行）**

```typescript
function eagerLoadSettings(): void {
  // 使用 eagerParseCliFlag 提前解析 --settings
  const settingsFile = eagerParseCliFlag('--settings');
  if (settingsFile) {
    loadSettingsFromFlag(settingsFile);  // 支持 JSON 字符串或文件路径
  }
  
  // 提前解析 --setting-sources 控制配置源
  const settingSourcesArg = eagerParseCliFlag('--setting-sources');
  if (settingSourcesArg !== undefined) {
    loadSettingSourcesFromFlag(settingSourcesArg);
  }
}
```

**第三阶段：Commander.js 主解析（第 902 行起）**

```typescript
const program = new CommanderCommand()
  .configureHelp(createSortedHelpConfig())
  .enablePositionalOptions();

// 使用 preAction hook 在命令执行前初始化
program.hook('preAction', async thisCommand => {
  await init();  // 初始化在参数解析之后，命令执行之前
  // ...
});
```

### 3. 参数分类

#### 3.1 全局选项（第 968-1006 行）

| 类别 | 选项示例 |
|------|----------|
| 调试 | `-d, --debug`, `--debug-file`, `--verbose` |
| 模式 | `-p, --print`, `--bare`, `--init`, `--init-only` |
| 输出格式 | `--output-format`, `--json-schema` |
| 权限 | `--dangerously-skip-permissions`, `--permission-mode` |
| 会话控制 | `-c, --continue`, `-r, --resume`, `--session-id` |
| 模型 | `--model`, `--effort`, `--agent` |
| 配置 | `--settings`, `--setting-sources`, `--plugin-dir` |

#### 3.2 子命令结构

```
claude
├── mcp (配置管理)
│   ├── serve
│   ├── add
│   ├── remove
│   └── ...
├── auth (认证)
│   ├── login
│   └── status
├── plugin (插件)
│   ├── list
│   ├── install
│   └── ...
├── config (配置)
├── doctor (诊断)
└── ...
```

### 4. 特殊处理逻辑

#### 4.1 SSH 远程模式预解析（第 702-795 行）

```typescript
// 在 Commander 运行前提取 SSH 相关标志
if (feature('SSH_REMOTE') && _pendingSSH) {
  // 手动提取 --local, --dangerously-skip-permissions, --permission-mode 等
  // 重写 process.argv 以隐藏 SSH 子命令，让主命令处理剩余参数
}
```

#### 4.2 Print 模式优化（第 3875-3890 行）

```typescript
// -p/--print 模式下跳过子命令注册（节省 ~65ms）
const isPrintMode = process.argv.includes('-p') || process.argv.includes('--print');
if (isPrintMode && !isCcUrl) {
  await program.parseAsync(process.argv);
  return program;
}
```

#### 4.3 特性标志控制的条件选项（第 3816-3872 行）

```typescript
if ("external" === 'ant') {
  // ANT-ONLY 选项
  program.addOption(new Option('--delegate-permissions', ...));
}

if (feature('KAIROS')) {
  program.addOption(new Option('--assistant', ...));
}
```

### 5. 参数处理流程图

```
process.argv
    │
    ▼
┌─────────────────┐
│  早期标志检测    │  ← 检测 -p/--print, --init-only 等
│  (第 797-813 行) │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ eagerLoadSettings│  ← 使用 eagerParseCliFlag 解析
│  (第 852 行)     │     --settings, --setting-sources
└─────────────────┘
    │
    ▼
┌─────────────────┐
│   SSH 预处理    │  ← 提取 SSH 相关标志，重写 argv
│  (第 702-795 行) │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ Commander.js    │  ← 主解析逻辑
│  初始化和配置   │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│  preAction Hook │  ← 执行 init(), 加载配置
│  (第 907-966 行) │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│   命令执行      │  ← 调用对应 action 处理函数
│   (action)      │
└─────────────────┘
```

### 6. 关键设计特点

1. **分层解析**：先手动预解析关键标志，再使用 Commander 完整解析
2. **性能优化**：Print 模式下跳过不必要的子命令注册
3. **扩展性**：使用特性标志控制条件编译和功能开关
4. **类型安全**：使用 `@commander-js/extra-typings` 提供 TypeScript 类型支持
5. **延迟初始化**：通过 `preAction` hook 确保解析完成后再执行耗时初始化

## Print 模式详解

### 什么是 Print 模式？

**Print 模式**（`-p` 或 `--print`）是 Claude Code 的**非交互式/无头模式（Headless Mode）**，设计用于**脚本化、管道化和自动化场景**。

### 核心特点

根据代码中的定义（[main.tsx:976](file:///c:/Users/shenj/Desktop/claude-code-source-code/src/main.tsx#L976)）：

```typescript
.option('-p, --print', 'Print response and exit (useful for pipes). Note: The workspace trust dialog is skipped when Claude is run with the -p mode. Only use this flag in directories you trust.', () => true)
```

| 特性 | 说明 |
|------|------|
| **非交互式** | 没有 TUI（终端用户界面），直接输出结果后退出 |
| **跳过信任对话框** | 自动信任当前目录（注意安全风险） |
| **管道友好** | 支持标准输入输出重定向 |
| **单次执行** | 处理完提示词后立即退出，不进入对话循环 |

### Print 模式能否进行对话？

**不能直接进行多轮对话**，但支持以下变通方式：

#### 1. **单轮对话**（默认）
```bash
claude -p "解释这段代码"
```

#### 2. **多轮 Agentic 执行**（`--max-turns`）
```typescript
// main.tsx:988
.addOption(new Option('--max-turns <turns>', 'Maximum number of agentic turns in non-interactive mode. This will early exit the conversation after the specified number of turns. (only works with --print)').argParser(Number).hideHelp())
```

允许 Claude 在单条提示词内执行多步工具调用（如读取文件、编辑代码等），但**用户无法在中间介入**。

```bash
claude -p --max-turns 10 "重构这个项目的所有日志调用"
```

#### 3. **流式 JSON 输入输出**（`--input-format=stream-json --output-format=stream-json`）
这是 SDK 集成模式，支持程序化多轮交互：

```typescript
// main.tsx:1000
.addOption(new Option('--input-format <format>', 'Input format (only works with --print): "text" (default), or "stream-json" (realtime streaming input)').choices(['text', 'stream-json']))
.addOption(new Option('--output-format <format>', 'Output format (only works with --print): "text" (default), "json" (single result), or "stream-json" (realtime streaming)').choices(['text', 'json', 'stream-json']))
```

```bash
claude -p --input-format=stream-json --output-format=stream-json --sdk-url=ws://...
```

### Print 模式 vs 交互模式对比

| 功能 | Print 模式 | 交互模式 |
|------|-----------|----------|
| TUI 界面 | ❌ 无 | ✅ 有（Ink/React） |
| 多轮对话 | ❌ 不支持（除 `--max-turns`） | ✅ 支持 |
| 用户实时输入 | ❌ 不支持 | ✅ 支持 |
| 管道支持 | ✅ 完美支持 | ❌ 不支持 |
| 会话恢复 (`--continue`) | ✅ 支持 | ✅ 支持 |
| MCP 服务器 | ✅ 支持 | ✅ 支持 |
| 工具执行 | ✅ 支持 | ✅ 支持 |
| 信任对话框 | ❌ 自动跳过 | ✅ 显示 |

### Print 模式执行流程

```
┌─────────────────────────────────────────────────────────────┐
│  1. 解析 CLI 参数（--print 被识别）                            │
│  2. 设置 isNonInteractiveSession = true                       │
│  3. 跳过子命令注册（性能优化）                                  │
│  4. 跳过信任对话框（自动信任）                                  │
│  5. 初始化无头存储（headlessStore）                            │
│  6. 连接 MCP 服务器                                           │
│  7. 调用 runHeadless()（src/cli/print.ts）                   │
│  8. 处理提示词 → 执行工具 → 输出结果 → 退出                    │
└─────────────────────────────────────────────────────────────┘
```

### 关键代码路径

```typescript
// main.tsx:2584-2860
if (isNonInteractiveSession) {
  // 初始化无头状态存储
  const headlessStore = createStore(headlessInitialState, onChangeAppState);
  
  // 连接 MCP 服务器
  await connectMcpBatch(regularMcpConfigs, 'regular');
  
  // 运行无头模式主逻辑
  const { runHeadless } = await import('src/cli/print.js');
  void runHeadless(inputPrompt, () => headlessStore.getState(), ...);
  return;  // 直接返回，不进入 REPL
}
```

### 使用场景

1. **CI/CD 管道**：自动化代码审查、生成文档
2. **脚本集成**：Shell 脚本中调用 Claude
3. **批处理**：批量处理文件
4. **SDK 后端**：作为其他应用程序的后端服务（通过 `--sdk-url`）

### 示例用法

```bash
# 基础用法 - 单轮问答
claude -p "总结 README.md 的内容"

# JSON 输出
claude -p --output-format=json "分析 package.json 的依赖"

# 多轮工具调用（最大 5 轮）
claude -p --max-turns 5 "找出项目中所有未使用的函数"

# 管道输入
cat file.ts | claude -p "解释这段代码"

# 禁用会话持久化（纯临时会话）
claude -p --no-session-persistence "临时任务"

# 指定模型
claude -p --model sonnet "使用 Sonnet 模型回答"
```

### 总结

**Print 模式本质上是"一问一答"的非交互模式**，虽然不能像交互模式那样进行自由的多轮对话，但通过 `--max-turns` 可以让 Claude 自主执行多步操作，或者通过 `stream-json` 格式实现程序化的多轮交互。