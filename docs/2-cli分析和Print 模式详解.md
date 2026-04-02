让我搜索 Print 模式的相关代码来详细了解它的工作原理：
        
            
toolName: search_by_regex
            
status: success
          
query: print.*mode|isPrintMode|--print|-p|Print response and exit
          
让我搜索更具体的 Print 模式实现：
        
            
toolName: search_by_regex
            
status: success
          
query: isPrintMode|printMode|output-format.*stream-json|--print
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: failed
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\cli\print.ts
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\cli\print.ts
          

        
            
toolName: search_by_regex
            
status: success
          
query: isPrintMode|printMode|Print response and exit|useful for pipes
          

        
            
toolName: search_by_regex
            
status: success
          
query: print\s*===\s*true|if.*print.*\{|options\.print|printMode
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          

        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          
现在我已经获取了足够的信息。让我为您详细解释 Print 模式：

---

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