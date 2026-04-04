
# QueryEngine 多 Agent 调度机制分析

## 1. Agent 定义机制

### AgentDefinition 结构
Agent 在系统中通过 `AgentDefinition` 类型定义，它是一个联合类型，包含三种类型的 agent：

- **BuiltInAgentDefinition**：内置 agent，如 Explore、Plan 等
- **CustomAgentDefinition**：用户自定义 agent，从 markdown 文件或 JSON 配置加载
- **PluginAgentDefinition**：插件提供的 agent

每个 agent 定义包含以下核心属性：
- `agentType`：agent 的唯一标识符
- `whenToUse`：agent 的描述和使用场景
- `tools`：agent 可以使用的工具列表
- `getSystemPrompt`：获取 agent 系统提示的函数
- `model`：可选的模型覆盖
- `effort`：工作努力程度
- `permissionMode`：权限模式
- `memory`：内存作用域（user、project、local）
- `isolation`：隔离模式（worktree、remote）

### Agent 加载过程
1. **内置 Agent**：通过 `getBuiltInAgents()` 加载
2. **自定义 Agent**：从 markdown 文件中解析
3. **插件 Agent**：从插件中加载

加载过程由 `getAgentDefinitionsWithOverrides` 函数处理，它会：
- 加载内置 agent
- 解析用户和项目级别的 agent 定义文件
- 加载插件提供的 agent
- 过滤出激活的 agent

## 2. Agent 调度机制

### 1. QueryEngine 初始化
QueryEngine 在构造时接收 `agents` 数组作为配置参数：

```typescript
export type QueryEngineConfig = {
  // ...其他配置
  agents: AgentDefinition[]
  // ...其他配置
}
```

### 2. Agent 工具调用
当需要调用 agent 时，系统通过 `AgentTool` 工具进行调度：

1. **AgentTool.prompt**：生成 agent 工具的提示，包含可用的 agent 列表
2. **AgentTool.call**：执行 agent 调用，处理以下逻辑：
   - 解析输入参数（description、prompt、subagent_type 等）
   - 选择要使用的 agent 定义
   - 检查 agent 的 MCP 服务器要求
   - 决定同步或异步运行 agent
   - 设置 agent 的工作环境（如工作树隔离）
   - 调用 `runAgent` 执行 agent

### 3. Agent 执行过程
`runAgent` 函数负责 agent 的实际执行：

1. **环境准备**：
   - 解析 agent 模型
   - 设置 agent ID
   - 准备初始消息
   - 克隆文件状态缓存
   - 解析用户和系统上下文

2. **工具和权限设置**：
   - 解析 agent 可用的工具
   - 设置 agent 的权限模式
   - 初始化 agent 特定的 MCP 服务器

3. **执行查询**：
   - 构建 agent 系统提示
   - 创建 agent 工具使用上下文
   - 调用 `query` 函数执行 agent 对话
   - 记录 agent 执行过程中的消息

4. **清理资源**：
   - 清理 agent 特定的 MCP 服务器
   - 清理会话钩子
   - 释放文件状态缓存
   - 终止 agent 产生的后台任务

## 3. 多 Agent 协作模式

### 1. Agent 团队（Multi-Agent）
系统支持通过 `team_name` 和 `name` 参数创建 agent 团队：

```typescript
if (teamName && name) {
  // 生成队友 agent
  const result = await spawnTeammate({
    name,
    prompt,
    description,
    team_name: teamName,
    use_splitpane: true,
    plan_mode_required: spawnMode === 'plan',
    model: model ?? agentDef?.model,
    agent_type: subagent_type,
    invokingRequestId: assistantMessage?.requestId
  }, toolUseContext);
  // ...
}
```

### 2. 异步 Agent
Agent 可以异步运行，在后台执行任务：

```typescript
const shouldRunAsync = (run_in_background === true || selectedAgent.background === true || isCoordinator || forceAsync || assistantForceAsync || (proactiveModule?.isProactiveActive() ?? false)) && !isBackgroundTasksDisabled;

if (shouldRunAsync) {
  // 注册异步 agent 任务
  const asyncAgentId = earlyAgentId;
  const agentBackgroundTask = registerAsyncAgent({
    agentId: asyncAgentId,
    description,
    prompt,
    selectedAgent,
    setAppState: rootSetAppState,
    toolUseId: toolUseContext.toolUseId
  });
  // ...
}
```

### 3. Agent 隔离
Agent 可以在隔离的环境中运行，如 git worktree：

```typescript
if (effectiveIsolation === 'worktree') {
  const slug = `agent-${earlyAgentId.slice(0, 8)}`;
  worktreeInfo = await createAgentWorktree(slug);
}
```

## 4. Agent 结果返回机制

### 1. 同步 Agent 返回
同步 agent 直接返回执行结果：

```typescript
return runWithAgentContext(syncAgentContext, () => wrapWithCwd(async () => {
  const agentMessages: MessageType[] = [];
  // ...执行 agent
  // 收集结果并返回
  return {
    data: {
      status: 'completed' as const,
      prompt,
      // ...其他结果数据
    }
  };
}));
```

### 2. 异步 Agent 返回
异步 agent 返回一个包含 agent ID 和输出文件路径的结果：

```typescript
return {
  data: {
    isAsync: true as const,
    status: 'async_launched' as const,
    agentId: agentBackgroundTask.agentId,
    description: description,
    prompt: prompt,
    outputFile: getTaskOutputPath(agentBackgroundTask.agentId),
    canReadOutputFile
  }
};
```

### 3. 结果处理
QueryEngine 通过以下方式处理 agent 结果：
- 记录 agent 执行过程中的消息
- 处理 agent 的工具使用结果
- 管理 agent 的生命周期
- 提供结果给用户或其他 agent

## 5. 代码优化建议

1. **Agent 资源管理优化**
   - 目前 agent 执行完成后会清理资源，但对于长时间运行的 agent 可能会占用过多资源
   - 建议添加资源使用监控，当资源使用超过阈值时自动调整或终止 agent

2. **Agent 通信机制增强**
   - 目前 agent 之间的通信主要通过消息传递，缺乏更直接的通信机制
   - 建议实现 agent 之间的直接调用接口，支持更复杂的协作模式

3. **Agent 错误处理改进**
   - 当 agent 执行失败时，错误信息可能不够详细
   - 建议增强错误处理，提供更详细的错误信息和恢复策略

4. **Agent 性能优化**
   - 对于频繁使用的 agent，可以考虑缓存其初始化状态
   - 建议实现 agent 池机制，避免重复初始化 agent

## 6. 总结

QueryEngine 通过以下机制调度多个 agents 进行共同工作：

1. **Agent 定义**：通过 `AgentDefinition` 类型定义不同类型的 agent，支持内置、自定义和插件 agent
2. **Agent 调度**：通过 `AgentTool` 工具进行 agent 调用，支持同步和异步执行
3. **Agent 执行**：通过 `runAgent` 函数执行 agent，处理环境准备、工具设置、查询执行和资源清理
4. **多 Agent 协作**：支持 agent 团队、异步执行和环境隔离
5. **结果返回**：根据 agent 执行模式返回不同形式的结果

这种设计使得系统能够灵活地调度多个 agents 完成复杂任务，支持 agent 之间的协作和资源管理，为用户提供更强大的工具使用体验。