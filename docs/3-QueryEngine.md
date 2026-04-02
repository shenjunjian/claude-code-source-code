我需要先查看 QueryEngine.ts 文件的内容，了解其实现细节，然后查看 main.ts 文件，分析如何初始化和调用 QueryEngine 类。
        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\QueryEngine.ts
          
现在我需要查看 main.ts 文件，分析如何初始化和调用 QueryEngine 类。
        
            
toolName: view_files
            
status: failed
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.ts
          
main.ts 文件不存在，我需要查看 src 目录下的文件来找到主入口点。
        
            
toolName: view_folder
            
status: success
          
c:\Users\shenj\Desktop\claude-code-source-code\src
          
我看到 src 目录下有 main.tsx 文件，这应该是主入口点。我需要查看这个文件来分析如何初始化和调用 QueryEngine 类。
        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          
main.tsx 文件内容被截断了，我需要继续查看更多内容来找到 QueryEngine 类的初始化和调用部分。
        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          
我需要继续查看 main.tsx 文件，寻找 QueryEngine 类的初始化和调用部分。
        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\main.tsx
          
我需要继续查看 main.tsx 文件，寻找 QueryEngine 类的初始化和调用部分。从之前的代码中，我看到在非交互式模式下，程序会调用 runHeadless 函数，这个函数可能会初始化和使用 QueryEngine 类。
        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\cli\print.ts
          
我需要继续查看 print.ts 文件，找到 ask 函数的调用位置，因为这个函数是从 QueryEngine.ts 导入的，它会初始化和使用 QueryEngine 类。
        
            
toolName: view_files
            
status: success
          
            
filePath: c:\Users\shenj\Desktop\claude-code-source-code\src\cli\print.ts
          
# QueryEngine 类的初始化和调用分析

## 核心调用流程

在 Claude Code 项目中，QueryEngine 类是处理查询生命周期和会话状态的核心组件。通过分析代码，我发现其初始化和调用主要通过以下流程实现：

### 1. 非交互式模式调用路径

当使用 `--print` 或 `--p` 标志运行时，程序会进入非交互式模式，核心调用路径如下：

1. **main.tsx** 中检测到非交互式模式，调用 `runHeadless` 函数
2. **print.ts** 中的 `runHeadless` 函数初始化必要的状态和配置
3. **print.ts** 中的 `runHeadlessStreaming` 函数创建 `run` 函数
4. **print.ts** 中的 `run` 函数调用 `ask` 函数
5. **QueryEngine.ts** 中的 `ask` 函数创建 QueryEngine 实例并调用其 `submitMessage` 方法

### 2. QueryEngine 初始化

在 `QueryEngine.ts` 的 `ask` 函数中，QueryEngine 类的初始化参数非常丰富：

```typescript
const engine = new QueryEngine({
  cwd,
  tools,
  commands,
  mcpClients,
  agents,
  canUseTool,
  getAppState,
  setAppState,
  initialMessages: mutableMessages,
  readFileCache: cloneFileStateCache(getReadFileCache()),
  customSystemPrompt,
  appendSystemPrompt,
  userSpecifiedModel,
  fallbackModel,
  thinkingConfig,
  maxTurns,
  maxBudgetUsd,
  taskBudget,
  jsonSchema,
  verbose,
  handleElicitation,
  replayUserMessages,
  includePartialMessages,
  setSDKStatus,
  abortController,
  orphanedPermission,
  ...(feature('HISTORY_SNIP')
    ? {
        snipReplay: (yielded: Message, store: Message[]) => {
          if (!snipProjection!.isSnipBoundaryMessage(yielded))
            return undefined
          return snipModule!.snipCompactIfNeeded(store, { force: true })
        },
      }
    : {}),
})
```
<mcfile name="QueryEngine.ts" path="c:\Users\shenj\Desktop\claude-code-source-code\src\QueryEngine.ts"></mcfile>

### 3. 核心参数说明

| 参数 | 类型 | 说明 |
|------|------|------|
| cwd | string | 当前工作目录 |
| tools | Tools | 可用的工具列表 |
| commands | Command[] | 可用的命令列表 |
| mcpClients | MCPServerConnection[] | MCP 客户端连接列表 |
| agents | AgentDefinition[] | 代理定义列表 |
| canUseTool | CanUseToolFn | 判断是否可以使用工具的函数 |
| getAppState | () => AppState | 获取应用状态的函数 |
| setAppState | (f: (prev: AppState) => AppState) => void | 设置应用状态的函数 |
| initialMessages | Message[] | 初始消息列表 |
| readFileCache | FileStateCache | 文件状态缓存 |
| customSystemPrompt | string | 自定义系统提示 |
| appendSystemPrompt | string | 附加系统提示 |
| userSpecifiedModel | string | 用户指定的模型 |
| fallbackModel | string | 备用模型 |
| thinkingConfig | ThinkingConfig | 思考配置 |
| maxTurns | number | 最大轮数 |
| maxBudgetUsd | number | 最大预算（美元） |
| taskBudget | { total: number } | 任务预算 |
| jsonSchema | Record<string, unknown> | JSON 模式 |
| verbose | boolean | 是否详细输出 |
| handleElicitation | ToolUseContext['handleElicitation'] | 处理引出的函数 |
| replayUserMessages | boolean | 是否重放用户消息 |
| includePartialMessages | boolean | 是否包含部分消息 |
| setSDKStatus | (status: SDKStatus) => void | 设置 SDK 状态的函数 |
| abortController | AbortController | 中止控制器 |
| orphanedPermission | OrphanedPermission | 孤立权限 |
| snipReplay | (yielded: Message, store: Message[]) => { messages: Message[]; executed: boolean } | 剪枝重放函数（HISTORY_SNIP 特性） |

### 4. 调用 submitMessage 方法

初始化完成后，`ask` 函数会调用 QueryEngine 实例的 `submitMessage` 方法来处理用户输入：

```typescript
try {
  yield* engine.submitMessage(prompt, {
    uuid: promptUuid,
    isMeta,
  })
} finally {
  setReadFileCache(engine.getReadFileState())
}
```
<mcfile name="QueryEngine.ts" path="c:\Users\shenj\Desktop\claude-code-source-code\src\QueryEngine.ts"></mcfile>

### 5. submitMessage 方法的核心功能

`submitMessage` 方法是 QueryEngine 类的核心方法，它：

1. 处理用户输入
2. 构建系统提示
3. 处理权限检查
4. 执行查询循环
5. 生成响应消息
6. 处理工具使用
7. 管理会话状态
8. 跟踪使用情况和成本

## 初始化和调用的关键步骤

1. **配置准备**：在 `main.tsx` 中，程序会解析命令行参数，准备各种配置项
2. **工具和命令加载**：加载可用的工具和命令
3. **MCP 客户端初始化**：初始化 MCP 客户端连接
4. **状态管理**：创建应用状态存储
5. **QueryEngine 实例化**：通过 `ask` 函数创建 QueryEngine 实例
6. **消息处理**：调用 `submitMessage` 方法处理用户输入
7. **响应生成**：生成并返回响应消息
8. **状态更新**：更新文件状态缓存等状态

## 代码优化建议

1. **参数简化**：QueryEngine 构造函数参数过多，建议使用配置对象模式，将相关参数分组
2. **依赖注入**：考虑使用依赖注入容器来管理 QueryEngine 的依赖
3. **模块化**：将 submitMessage 方法拆分为更小的、职责单一的方法
4. **错误处理**：增强错误处理机制，提供更详细的错误信息
5. **性能优化**：对于大型会话，考虑优化内存使用，特别是消息历史的管理

## 总结

QueryEngine 类是 Claude Code 项目的核心组件，负责处理查询生命周期和会话状态。它通过 `ask` 函数进行初始化和调用，接收丰富的参数来配置其行为。在 `main.tsx` 中，程序会根据运行模式选择不同的执行路径，非交互式模式会通过 `runHeadless` 函数最终调用 `ask` 函数来使用 QueryEngine。

这种设计使得 QueryEngine 能够灵活适应不同的使用场景，无论是交互式的 REPL 环境还是非交互式的命令行环境，都能提供一致的查询处理能力。