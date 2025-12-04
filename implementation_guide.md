# 基于 Gemini CLI 架构实现自主任务编排系统指南

## 📋 目录
1. [核心架构设计](#核心架构设计)
2. [实现步骤](#实现步骤)
3. [代码示例](#代码示例)
4. [关键设计模式](#关键设计模式)

---

## 核心架构设计

### 三层架构模型

```
┌──────────────────────────────────────────────────┐
│          会话管理层 (Session Manager)              │
│  - 对话历史管理                                    │
│  - 循环检测与压缩                                  │
│  - 错误恢复与降级                                  │
└────────────────┬─────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────┐
│          工具编排层 (Tool Orchestrator)           │
│  - 工具调用状态管理                                │
│  - 确认流程协调                                    │
│  - 并发控制与超时                                  │
└────────────────┬─────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────────┐
│          执行引擎层 (Execution Engine)            │
│  - Agent 轮次循环                                 │
│  - LLM 调用与响应处理                             │
│  - 工具实例化与执行                                │
└──────────────────────────────────────────────────┘
```

---

## 实现步骤

### Step 1: 定义核心类型系统

```typescript
// types.ts

/** Agent 终止模式 */
export enum AgentTerminateMode {
  ERROR = 'ERROR',
  TIMEOUT = 'TIMEOUT',
  GOAL = 'GOAL',              // 成功完成任务
  MAX_TURNS = 'MAX_TURNS',
  ABORTED = 'ABORTED',
}

/** Agent 输出结构 */
export interface OutputObject {
  result: string;
  terminate_reason: AgentTerminateMode;
  metadata?: Record<string, any>;
}

/** Agent 定义 */
export interface AgentDefinition {
  name: string;
  description: string;

  // 提示配置
  promptConfig: {
    systemPrompt?: string;
    initialMessages?: Message[];
    query?: string;
  };

  // 模型配置
  modelConfig: {
    model: string;
    temperature: number;
    maxTokens?: number;
  };

  // 运行配置
  runConfig: {
    maxTimeMinutes: number;
    maxTurns?: number;
  };

  // 工具配置
  toolConfig?: {
    tools: Array<string | ToolDefinition>;
  };

  // 输出配置（用于结构化输出）
  outputConfig?: {
    schema: any;  // 使用 Zod 或 JSON Schema
  };
}

/** 工具调用请求 */
export interface ToolCallRequest {
  id: string;
  name: string;
  arguments: Record<string, any>;
}

/** 工具调用响应 */
export interface ToolCallResponse {
  id: string;
  name: string;
  result: string | object;
  error?: string;
}
```

---

### Step 2: 实现工具系统

```typescript
// tool-system.ts

/** 工具接口 */
export interface Tool<TParams = any, TResult = any> {
  name: string;
  description: string;

  // 工具声明（供 LLM 使用）
  getDeclaration(): ToolDeclaration;

  // 构建工具调用实例
  build(params: TParams): ToolInvocation<TParams, TResult>;
}

/** 工具调用实例 */
export interface ToolInvocation<TParams, TResult> {
  params: TParams;

  // 获取描述
  getDescription(): string;

  // 是否需要确认
  shouldConfirm(): boolean;

  // 执行工具
  execute(signal: AbortSignal): Promise<TResult>;
}

/** 工具注册表 */
export class ToolRegistry {
  private tools = new Map<string, Tool>();

  register(tool: Tool) {
    this.tools.set(tool.name, tool);
  }

  get(name: string): Tool | undefined {
    return this.tools.get(name);
  }

  getDeclarations(): ToolDeclaration[] {
    return Array.from(this.tools.values()).map(t => t.getDeclaration());
  }
}

/** 示例：文件读取工具 */
export class ReadFileTool implements Tool<{ path: string }, string> {
  name = 'read_file';
  description = '读取文件内容';

  getDeclaration(): ToolDeclaration {
    return {
      name: this.name,
      description: this.description,
      parameters: {
        type: 'object',
        properties: {
          path: {
            type: 'string',
            description: '文件路径',
          },
        },
        required: ['path'],
      },
    };
  }

  build(params: { path: string }): ToolInvocation<{ path: string }, string> {
    return new ReadFileInvocation(params);
  }
}

class ReadFileInvocation implements ToolInvocation<{ path: string }, string> {
  constructor(public params: { path: string }) {}

  getDescription(): string {
    return `读取文件: ${this.params.path}`;
  }

  shouldConfirm(): boolean {
    return false;  // 读取操作无需确认
  }

  async execute(signal: AbortSignal): Promise<string> {
    // 实现文件读取
    const fs = await import('fs/promises');
    return await fs.readFile(this.params.path, 'utf-8');
  }
}
```

---

### Step 3: 实现工具调度器

```typescript
// tool-scheduler.ts

/** 工具调用状态 */
type ToolCallState =
  | { status: 'validating'; request: ToolCallRequest }
  | { status: 'scheduled'; request: ToolCallRequest; invocation: ToolInvocation }
  | { status: 'executing'; request: ToolCallRequest; invocation: ToolInvocation }
  | { status: 'success'; request: ToolCallRequest; response: ToolCallResponse }
  | { status: 'error'; request: ToolCallRequest; error: string };

/** 工具调度器 */
export class ToolScheduler {
  private toolCalls = new Map<string, ToolCallState>();

  constructor(
    private registry: ToolRegistry,
    private confirmHandler?: (invocation: ToolInvocation) => Promise<boolean>
  ) {}

  /** 验证并调度工具调用 */
  async scheduleToolCall(request: ToolCallRequest): Promise<void> {
    // 1. 设置为验证状态
    this.toolCalls.set(request.id, { status: 'validating', request });

    try {
      // 2. 获取工具
      const tool = this.registry.get(request.name);
      if (!tool) {
        throw new Error(`Tool not found: ${request.name}`);
      }

      // 3. 构建工具调用实例
      const invocation = tool.build(request.arguments);

      // 4. 设置为已调度
      this.toolCalls.set(request.id, {
        status: 'scheduled',
        request,
        invocation,
      });
    } catch (error) {
      // 验证失败
      this.toolCalls.set(request.id, {
        status: 'error',
        request,
        error: error.message,
      });
    }
  }

  /** 执行工具调用 */
  async executeToolCall(
    id: string,
    signal: AbortSignal
  ): Promise<ToolCallResponse> {
    const state = this.toolCalls.get(id);
    if (!state || state.status !== 'scheduled') {
      throw new Error(`Tool call not scheduled: ${id}`);
    }

    const { request, invocation } = state;

    try {
      // 1. 检查是否需要确认
      if (invocation.shouldConfirm() && this.confirmHandler) {
        const approved = await this.confirmHandler(invocation);
        if (!approved) {
          throw new Error('Tool execution denied by user');
        }
      }

      // 2. 设置为执行中
      this.toolCalls.set(id, { status: 'executing', request, invocation });

      // 3. 执行工具
      const result = await invocation.execute(signal);

      // 4. 构建响应
      const response: ToolCallResponse = {
        id,
        name: request.name,
        result,
      };

      // 5. 设置为成功
      this.toolCalls.set(id, { status: 'success', request, response });

      return response;
    } catch (error) {
      // 执行失败
      this.toolCalls.set(id, {
        status: 'error',
        request,
        error: error.message,
      });

      return {
        id,
        name: request.name,
        result: '',
        error: error.message,
      };
    }
  }

  /** 批量执行所有已调度的工具 */
  async executeAllScheduled(signal: AbortSignal): Promise<ToolCallResponse[]> {
    const scheduled = Array.from(this.toolCalls.entries())
      .filter(([_, state]) => state.status === 'scheduled')
      .map(([id]) => id);

    return await Promise.all(
      scheduled.map(id => this.executeToolCall(id, signal))
    );
  }
}
```

---

### Step 4: 实现 Agent 执行器

```typescript
// agent-executor.ts

/** Agent 执行器 */
export class AgentExecutor {
  private turnCounter = 0;
  private chatHistory: Message[] = [];

  constructor(
    private definition: AgentDefinition,
    private llmClient: LLMClient,
    private toolScheduler: ToolScheduler,
  ) {}

  /** 主执行循环 */
  async run(inputs: Record<string, any>, signal: AbortSignal): Promise<OutputObject> {
    const startTime = Date.now();
    const { maxTimeMinutes, maxTurns = 100 } = this.definition.runConfig;

    // 设置超时
    const timeoutController = new AbortController();
    const timeoutId = setTimeout(
      () => timeoutController.abort(new Error('Agent timeout')),
      maxTimeMinutes * 60 * 1000
    );

    const combinedSignal = AbortSignal.any([signal, timeoutController.signal]);

    try {
      // 初始化聊天历史
      await this.initializeChat(inputs);

      // 发送初始查询
      let currentMessage = this.buildInitialMessage(inputs);

      // 主循环
      while (true) {
        // 检查终止条件
        if (this.turnCounter >= maxTurns) {
          return {
            result: 'Agent exceeded maximum turns',
            terminate_reason: AgentTerminateMode.MAX_TURNS,
          };
        }

        if (combinedSignal.aborted) {
          const reason = timeoutController.signal.aborted
            ? AgentTerminateMode.TIMEOUT
            : AgentTerminateMode.ABORTED;
          return {
            result: 'Agent execution aborted',
            terminate_reason: reason,
          };
        }

        // 执行单轮
        const turnResult = await this.executeTurn(currentMessage, combinedSignal);

        // 检查是否完成
        if (turnResult.status === 'completed') {
          return {
            result: turnResult.result,
            terminate_reason: AgentTerminateMode.GOAL,
          };
        }

        if (turnResult.status === 'error') {
          return {
            result: turnResult.error,
            terminate_reason: AgentTerminateMode.ERROR,
          };
        }

        // 继续下一轮
        currentMessage = turnResult.nextMessage;
        this.turnCounter++;
      }
    } finally {
      clearTimeout(timeoutId);
    }
  }

  /** 执行单轮对话 */
  private async executeTurn(
    userMessage: Message,
    signal: AbortSignal
  ): Promise<TurnResult> {
    // 1. 添加用户消息到历史
    this.chatHistory.push(userMessage);

    // 2. 调用 LLM
    const response = await this.llmClient.generateContent({
      messages: this.chatHistory,
      tools: this.toolScheduler.registry.getDeclarations(),
      signal,
    });

    // 3. 添加助手响应到历史
    this.chatHistory.push(response.message);

    // 4. 检查是否有工具调用
    if (!response.toolCalls || response.toolCalls.length === 0) {
      // 没有工具调用 = 任务未完成（协议违规）
      return {
        status: 'error',
        error: 'Agent stopped calling tools without completing task',
      };
    }

    // 5. 处理工具调用
    const toolResult = await this.processToolCalls(response.toolCalls, signal);

    // 6. 检查是否调用了 complete_task
    if (toolResult.completed) {
      return {
        status: 'completed',
        result: toolResult.finalResult,
      };
    }

    // 7. 构建下一轮消息（包含工具响应）
    const nextMessage: Message = {
      role: 'user',
      content: this.formatToolResponses(toolResult.responses),
    };

    return {
      status: 'continue',
      nextMessage,
    };
  }

  /** 处理工具调用 */
  private async processToolCalls(
    toolCalls: ToolCallRequest[],
    signal: AbortSignal
  ): Promise<{
    completed: boolean;
    finalResult?: string;
    responses: ToolCallResponse[];
  }> {
    // 1. 调度所有工具
    await Promise.all(
      toolCalls.map(call => this.toolScheduler.scheduleToolCall(call))
    );

    // 2. 执行所有工具
    const responses = await this.toolScheduler.executeAllScheduled(signal);

    // 3. 检查是否有 complete_task 调用
    const completeTaskCall = toolCalls.find(call => call.name === 'complete_task');
    if (completeTaskCall) {
      const completeTaskResponse = responses.find(r => r.id === completeTaskCall.id);
      return {
        completed: true,
        finalResult: completeTaskResponse?.result as string,
        responses,
      };
    }

    return {
      completed: false,
      responses,
    };
  }

  /** 初始化聊天历史 */
  private async initializeChat(inputs: Record<string, any>): Promise<void> {
    // 添加系统提示
    if (this.definition.promptConfig.systemPrompt) {
      const systemPrompt = this.templateString(
        this.definition.promptConfig.systemPrompt,
        inputs
      );
      this.chatHistory.push({
        role: 'system',
        content: systemPrompt,
      });
    }

    // 添加初始消息（few-shot 示例）
    if (this.definition.promptConfig.initialMessages) {
      this.chatHistory.push(...this.definition.promptConfig.initialMessages);
    }
  }

  /** 构建初始消息 */
  private buildInitialMessage(inputs: Record<string, any>): Message {
    const query = this.definition.promptConfig.query
      ? this.templateString(this.definition.promptConfig.query, inputs)
      : 'Get Started!';

    return {
      role: 'user',
      content: query,
    };
  }

  /** 模板字符串替换 */
  private templateString(template: string, inputs: Record<string, any>): string {
    return template.replace(/\$\{(\w+)\}/g, (_, key) => {
      return inputs[key]?.toString() || '';
    });
  }

  /** 格式化工具响应 */
  private formatToolResponses(responses: ToolCallResponse[]): string {
    return responses.map(r => {
      if (r.error) {
        return `Tool ${r.name} (${r.id}) failed: ${r.error}`;
      }
      return `Tool ${r.name} (${r.id}) result: ${JSON.stringify(r.result)}`;
    }).join('\n\n');
  }
}

/** 单轮结果 */
type TurnResult =
  | { status: 'completed'; result: string }
  | { status: 'error'; error: string }
  | { status: 'continue'; nextMessage: Message };
```

---

### Step 5: 实现 LLM 客户端抽象

```typescript
// llm-client.ts

/** LLM 客户端接口 */
export interface LLMClient {
  generateContent(config: GenerateConfig): Promise<GenerateResponse>;
}

export interface GenerateConfig {
  messages: Message[];
  tools?: ToolDeclaration[];
  signal: AbortSignal;
}

export interface GenerateResponse {
  message: Message;
  toolCalls?: ToolCallRequest[];
}

/** OpenAI 兼容客户端 */
export class OpenAIClient implements LLMClient {
  constructor(
    private apiKey: string,
    private model: string,
    private baseURL?: string
  ) {}

  async generateContent(config: GenerateConfig): Promise<GenerateResponse> {
    const response = await fetch(`${this.baseURL || 'https://api.openai.com/v1'}/chat/completions`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: this.model,
        messages: config.messages,
        tools: config.tools?.map(t => ({ type: 'function', function: t })),
      }),
      signal: config.signal,
    });

    const data = await response.json();
    const choice = data.choices[0];

    // 解析响应
    const message: Message = {
      role: 'assistant',
      content: choice.message.content || '',
    };

    // 解析工具调用
    const toolCalls = choice.message.tool_calls?.map((tc: any) => ({
      id: tc.id,
      name: tc.function.name,
      arguments: JSON.parse(tc.function.arguments),
    }));

    return { message, toolCalls };
  }
}
```

---

### Step 6: 使用示例

```typescript
// example.ts

async function main() {
  // 1. 创建工具注册表
  const toolRegistry = new ToolRegistry();
  toolRegistry.register(new ReadFileTool());
  toolRegistry.register(new WriteFileTool());
  toolRegistry.register(new ShellTool());

  // 注册特殊的 complete_task 工具
  toolRegistry.register({
    name: 'complete_task',
    description: '完成任务并返回最终结果',
    getDeclaration: () => ({
      name: 'complete_task',
      description: '完成任务并返回最终结果',
      parameters: {
        type: 'object',
        properties: {
          result: {
            type: 'string',
            description: '任务的最终结果',
          },
        },
        required: ['result'],
      },
    }),
    build: (params: { result: string }) => ({
      params,
      getDescription: () => `完成任务: ${params.result}`,
      shouldConfirm: () => false,
      execute: async () => params.result,
    }),
  });

  // 2. 创建工具调度器
  const toolScheduler = new ToolScheduler(
    toolRegistry,
    async (invocation) => {
      // 确认处理器：这里可以实现用户交互
      console.log(`执行工具: ${invocation.getDescription()}`);
      return true;  // 自动批准
    }
  );

  // 3. 创建 LLM 客户端
  const llmClient = new OpenAIClient(
    process.env.OPENAI_API_KEY!,
    'gpt-4'
  );

  // 4. 定义 Agent
  const agentDef: AgentDefinition = {
    name: 'file-analyzer',
    description: '分析文件内容的 Agent',

    promptConfig: {
      systemPrompt: `你是一个文件分析助手。
你可以使用以下工具：
- read_file: 读取文件内容
- write_file: 写入文件内容
- shell: 执行 shell 命令

当完成任务后，使用 complete_task 工具返回最终结果。`,

      query: '请分析 ${filePath} 文件的内容，并给出总结。',
    },

    modelConfig: {
      model: 'gpt-4',
      temperature: 0.7,
    },

    runConfig: {
      maxTimeMinutes: 5,
      maxTurns: 20,
    },

    toolConfig: {
      tools: ['read_file', 'write_file', 'shell', 'complete_task'],
    },
  };

  // 5. 创建执行器
  const executor = new AgentExecutor(agentDef, llmClient, toolScheduler);

  // 6. 运行 Agent
  const controller = new AbortController();
  const result = await executor.run(
    { filePath: './README.md' },
    controller.signal
  );

  console.log('Agent 执行结果:');
  console.log('- 终止原因:', result.terminate_reason);
  console.log('- 结果:', result.result);
}

main().catch(console.error);
```

---

## 关键设计模式

### 1. **状态机模式**（工具调用生命周期）

```
validating → scheduled → executing → success/error
                ↓
            waiting (需要确认)
                ↓
            scheduled
```

### 2. **责任链模式**（Hook 系统）

```typescript
// 在工具执行前后插入 Hook
async executeWithHooks(invocation: ToolInvocation) {
  // Before hooks
  await this.runBeforeHooks(invocation);

  // Execute
  const result = await invocation.execute();

  // After hooks
  await this.runAfterHooks(invocation, result);

  return result;
}
```

### 3. **策略模式**（LLM 客户端）

```typescript
// 支持多种 LLM 提供商
interface LLMClient {
  generateContent(config: GenerateConfig): Promise<GenerateResponse>;
}

// OpenAI 实现
class OpenAIClient implements LLMClient { ... }

// Anthropic 实现
class AnthropicClient implements LLMClient { ... }

// Google Gemini 实现
class GeminiClient implements LLMClient { ... }
```

### 4. **观察者模式**（事件系统）

```typescript
// 事件发射器
class AgentExecutor extends EventEmitter {
  async executeTurn(...) {
    this.emit('turn:start', { turnNumber: this.turnCounter });

    const result = await this.callLLM();
    this.emit('llm:response', result);

    const toolResponses = await this.executeTools();
    this.emit('tools:completed', toolResponses);

    this.emit('turn:end', { turnNumber: this.turnCounter });
  }
}

// 监听器
executor.on('tools:completed', (responses) => {
  console.log('工具执行完成:', responses);
});
```

---

## 高级特性

### 1. **消息压缩**（处理长对话历史）

```typescript
async tryCompressChat(chat: ChatHistory): Promise<void> {
  const tokenCount = await this.estimateTokens(chat.messages);
  const maxTokens = 128000;  // 模型上下文窗口

  if (tokenCount > maxTokens * 0.8) {
    // 压缩策略：保留系统提示 + 最近 N 条消息 + 总结旧消息
    const summary = await this.summarizeOldMessages(
      chat.messages.slice(1, -10)
    );

    chat.messages = [
      chat.messages[0],  // 系统提示
      { role: 'user', content: `之前的对话总结:\n${summary}` },
      ...chat.messages.slice(-10),  // 最近 10 条消息
    ];

    this.emit('chat:compressed', {
      oldTokenCount: tokenCount,
      newTokenCount: await this.estimateTokens(chat.messages),
    });
  }
}
```

### 2. **循环检测**

```typescript
class LoopDetector {
  private recentActions: string[] = [];

  detectLoop(toolCalls: ToolCallRequest[]): boolean {
    const signature = this.computeSignature(toolCalls);
    this.recentActions.push(signature);

    // 保留最近 20 个动作
    if (this.recentActions.length > 20) {
      this.recentActions.shift();
    }

    // 检测是否有重复模式
    const pattern = this.findRepeatingPattern(this.recentActions);
    if (pattern && pattern.length >= 3) {
      return true;  // 检测到循环
    }

    return false;
  }

  private findRepeatingPattern(actions: string[]): string[] | null {
    // 实现重复模式检测算法
    // 例如：[A, B, C, A, B, C, A, B, C] → 检测到 [A, B, C] 重复
    // ...
  }
}
```

### 3. **错误恢复与重试**

```typescript
async callModelWithRetry(
  config: GenerateConfig,
  maxRetries: number = 3
): Promise<GenerateResponse> {
  let lastError: Error;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await this.llmClient.generateContent(config);
    } catch (error) {
      lastError = error;

      if (this.isRateLimitError(error)) {
        // 速率限制：指数退避
        const delay = Math.pow(2, attempt) * 1000;
        await this.sleep(delay);
      } else if (this.isContextOverflowError(error)) {
        // 上下文溢出：压缩消息
        await this.tryCompressChat(config.messages);
      } else {
        // 其他错误：不重试
        throw error;
      }
    }
  }

  throw lastError!;
}
```

### 4. **子 Agent 系统**

```typescript
// 工具可以调用其他 Agent
class SubagentTool implements Tool {
  constructor(
    private agentDef: AgentDefinition,
    private llmClient: LLMClient,
    private toolRegistry: ToolRegistry
  ) {}

  build(params: { task: string }): ToolInvocation {
    return {
      params,
      getDescription: () => `调用子 Agent: ${params.task}`,
      shouldConfirm: () => false,
      execute: async (signal) => {
        // 创建子 Agent 执行器
        const scheduler = new ToolScheduler(this.toolRegistry);
        const executor = new AgentExecutor(
          this.agentDef,
          this.llmClient,
          scheduler
        );

        // 运行子 Agent
        const result = await executor.run(
          { query: params.task },
          signal
        );

        return result.result;
      },
    };
  }
}
```

---

## 部署建议

### 1. **本地 CLI 工具**
```bash
# 打包为可执行文件
npm run build
chmod +x dist/cli.js
ln -s $(pwd)/dist/cli.js /usr/local/bin/myagent
```

### 2. **Web 服务**
```typescript
// Express.js 示例
app.post('/api/agents/:name/execute', async (req, res) => {
  const { name } = req.params;
  const { inputs } = req.body;

  const agentDef = await loadAgentDefinition(name);
  const executor = new AgentExecutor(agentDef, llmClient, toolScheduler);

  const result = await executor.run(inputs, req.signal);
  res.json(result);
});
```

### 3. **VSCode 扩展**
```typescript
// extension.ts
const command = vscode.commands.registerCommand('myagent.run', async () => {
  const agentDef = await loadAgentFromWorkspace();
  const executor = new AgentExecutor(agentDef, llmClient, toolScheduler);

  const result = await executor.run({}, new AbortController().signal);
  vscode.window.showInformationMessage(result.result);
});
```

---

## 总结

这个架构的核心优势：

1. **模块化设计**：工具、Agent、LLM 客户端都可以独立开发和测试
2. **状态管理清晰**：工具调用的每个阶段都有明确的状态
3. **可扩展性强**：通过 Hook 系统和插件机制可以轻松扩展功能
4. **容错性好**：支持重试、降级、错误恢复等机制
5. **类型安全**：使用 TypeScript 提供完整的类型定义

你可以根据自己的需求选择性实现这些功能，从最小的原型开始，逐步添加高级特性。
