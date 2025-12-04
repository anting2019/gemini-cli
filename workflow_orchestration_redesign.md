# Gemini CLI 工作流编排系统二开方案

## 📋 目录

1. [可行性分析](#可行性分析)
2. [系统架构设计](#系统架构设计)
3. [核心功能实现](#核心功能实现)
4. [技术栈选型](#技术栈选型)
5. [实施路线图](#实施路线图)
6. [代码示例](#代码示例)

---

## ✅ 可行性分析

### 现有项目基础

该项目**非常适合**进行二次开发，原因如下：

#### 1. 架构优势

```
✅ 模块化设计
  ├─ packages/core        # 核心引擎（独立可复用）
  ├─ packages/a2a-server  # HTTP API 服务器（已有 Express）
  ├─ packages/cli         # CLI 工具
  └─ packages/vscode-ide-companion  # IDE 集成

✅ 已有的关键组件
  ├─ AgentExecutor        # Agent 执行引擎（可复用）
  ├─ ToolScheduler        # 工具调度系统（可复用）
  ├─ GeminiClient         # LLM 客户端（可复用）
  ├─ ToolRegistry         # 工具注册表（可扩展）
  ├─ TaskStore            # 任务持久化（支持 GCS/内存）
  └─ Event System         # 事件系统（支持观察）

✅ 扩展点
  ├─ Hook System          # 工具执行前后的钩子
  ├─ MCP Support          # Model Context Protocol 支持
  ├─ Extension System     # 扩展机制
  └─ Command Registry     # 命令注册表
```

#### 2. 已有功能

| 功能 | 现状 | 可复用性 |
|------|------|---------|
| Agent 执行 | ✅ 完整实现 | ⭐⭐⭐⭐⭐ |
| 工具调用 | ✅ 完整实现 | ⭐⭐⭐⭐⭐ |
| HTTP API | ✅ Express 服务器 | ⭐⭐⭐⭐⭐ |
| 任务持久化 | ✅ TaskStore | ⭐⭐⭐⭐ |
| 事件系统 | ✅ GeminiEventType | ⭐⭐⭐⭐ |

#### 3. 需要新增的功能

| 需求 | 现状 | 开发难度 |
|------|------|---------|
| 页面配置界面 | ❌ 无 | 🟡 中等（前端开发） |
| 可视化工作流编排 | ❌ 无 | 🟠 较高（需设计 DSL） |
| 定时任务调度 | ❌ 无 | 🟢 简单（使用 node-cron） |
| 对话生成工作流 | ✅ 部分（Agent） | 🟢 简单（扩展现有 Agent） |
| 节点可观察性 | ✅ 部分（事件系统） | 🟢 简单（扩展事件） |

### 结论

**强烈推荐进行二次开发**，项目架构优秀，核心组件完备，只需添加：
1. Web UI 前端（React/Vue）
2. 工作流 DSL 定义
3. 定时调度器
4. 可视化引擎

---

## 🏗️ 系统架构设计

### 整体架构

```
┌────────────────────────────────────────────────────────────────┐
│                       前端层 (Frontend)                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ 工作流设计器     │  │ 监控仪表盘       │  │ 对话生成器       │ │
│  │ (React Flow)    │  │ (Echarts/D3)    │  │ (Chat UI)       │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└───────────────────────────┬────────────────────────────────────┘
                            │ HTTP/WebSocket
┌───────────────────────────▼────────────────────────────────────┐
│                      API 层 (Backend)                           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ 工作流 API       │  │ 调度 API         │  │ 执行 API         │ │
│  │ /workflows       │  │ /schedules       │  │ /executions      │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└───────────────────────────┬────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────┐
│                    业务逻辑层 (Services)                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │             WorkflowOrchestrator                        │   │
│  │  - 工作流解析与编排                                      │   │
│  │  - 节点依赖管理                                          │   │
│  │  - 条件分支路由                                          │   │
│  │  - 并发控制                                             │   │
│  └─────────┬──────────────┬──────────────┬─────────────────┘   │
│            │              │              │                     │
│  ┌─────────▼─────┐ ┌─────▼─────┐ ┌─────▼──────────────────┐   │
│  │ SchedulerService│ ExecutionSvc│ ConversationService    │   │
│  │ (node-cron)    │ (监控管理)   │ (对话生成工作流)         │   │
│  └────────────────┘ └───────────┘ └────────────────────────┘   │
└───────────────────────────┬────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────┐
│                  核心引擎层 (Core Engine)                        │
│                    [复用现有 core 包]                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │AgentExecutor │  │ToolScheduler │  │ GeminiClient         │ │
│  │(多轮执行)     │  │(工具调度)     │  │ (LLM 交互)           │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │ToolRegistry  │  │ Event System │  │ Hook System          │ │
│  │(工具管理)     │  │(事件发布)     │  │ (扩展点)             │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
└───────────────────────────┬────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────┐
│                     持久化层 (Persistence)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │ WorkflowDB   │  │ ExecutionDB  │  │ ScheduleDB           │ │
│  │ (工作流定义)  │  │ (执行记录)    │  │ (调度配置)           │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
│              PostgreSQL / MongoDB / SQLite                     │
└────────────────────────────────────────────────────────────────┘
```

### 工作流 DSL 设计

```typescript
// 工作流定义格式
interface WorkflowDefinition {
  id: string;
  name: string;
  description: string;
  version: string;

  // 全局配置
  config: {
    timeout: number;          // 总超时时间（秒）
    maxRetries: number;       // 最大重试次数
    concurrency: number;      // 最大并发节点数
  };

  // 节点定义
  nodes: WorkflowNode[];

  // 边定义（节点之间的连接）
  edges: WorkflowEdge[];

  // 触发器
  triggers: WorkflowTrigger[];

  // 输入输出
  inputs?: Record<string, ParameterSchema>;
  outputs?: Record<string, ParameterSchema>;
}

// 节点类型
type WorkflowNode =
  | AgentNode       // Agent 执行节点
  | ToolNode        // 单一工具调用节点
  | ConditionNode   // 条件分支节点
  | LoopNode        // 循环节点
  | SubWorkflowNode // 子工作流节点
  | StartNode       // 开始节点
  | EndNode;        // 结束节点

// Agent 节点（使用现有 AgentExecutor）
interface AgentNode {
  id: string;
  type: 'agent';
  name: string;

  // 复用现有 AgentDefinition
  agentConfig: {
    promptConfig: PromptConfig;
    modelConfig: ModelConfig;
    runConfig: RunConfig;
    toolConfig?: ToolConfig;
  };

  // 节点级配置
  position: { x: number; y: number };  // UI 位置
  timeout?: number;                     // 节点超时
  retryPolicy?: RetryPolicy;
}

// 工具节点（单一工具调用）
interface ToolNode {
  id: string;
  type: 'tool';
  name: string;

  // 工具配置
  toolName: string;                     // 工具名称
  parameters: Record<string, any>;      // 参数（支持模板）

  position: { x: number; y: number };
  timeout?: number;
  retryPolicy?: RetryPolicy;
}

// 条件分支节点
interface ConditionNode {
  id: string;
  type: 'condition';
  name: string;

  // 条件表达式（支持 JSONLogic 或自定义 DSL）
  condition: string | object;

  position: { x: number; y: number };
}

// 循环节点
interface LoopNode {
  id: string;
  type: 'loop';
  name: string;

  // 循环配置
  loopConfig: {
    type: 'for' | 'while' | 'foreach';
    condition?: string;                 // while 条件
    iterable?: string;                  // foreach 迭代对象
    maxIterations?: number;             // 最大迭代次数
  };

  // 循环体节点 ID
  bodyNodeIds: string[];

  position: { x: number; y: number };
}

// 边定义
interface WorkflowEdge {
  id: string;
  source: string;           // 源节点 ID
  target: string;           // 目标节点 ID

  // 条件边（用于条件节点）
  condition?: {
    type: 'success' | 'failure' | 'custom';
    expression?: string;    // 自定义表达式
  };

  // 数据映射（源节点输出 → 目标节点输入）
  dataMapping?: Record<string, string>;
}

// 触发器
type WorkflowTrigger =
  | ManualTrigger      // 手动触发
  | CronTrigger        // 定时触发
  | EventTrigger       // 事件触发
  | WebhookTrigger;    // Webhook 触发

interface CronTrigger {
  type: 'cron';
  schedule: string;    // Cron 表达式
  timezone?: string;
}

interface EventTrigger {
  type: 'event';
  eventType: string;   // 事件类型
  filter?: object;     // 事件过滤条件
}

interface WebhookTrigger {
  type: 'webhook';
  path: string;        // Webhook 路径
  method: 'GET' | 'POST' | 'PUT' | 'DELETE';
  authentication?: {
    type: 'apiKey' | 'bearer' | 'basic';
    config: object;
  };
}
```

---

## 🔧 核心功能实现

### 1. 工作流编排引擎

```typescript
// packages/workflow/src/orchestrator.ts

import { AgentExecutor } from '@google/gemini-cli-core';
import type { WorkflowDefinition, WorkflowNode, WorkflowExecution } from './types';

/**
 * 工作流编排器
 * 负责解析工作流定义并协调节点执行
 */
export class WorkflowOrchestrator {
  private nodeExecutors = new Map<string, NodeExecutor>();

  constructor(
    private eventBus: EventBus,
    private executionStore: ExecutionStore
  ) {
    // 注册节点执行器
    this.registerNodeExecutors();
  }

  /**
   * 执行工作流
   */
  async execute(
    workflowDef: WorkflowDefinition,
    inputs: Record<string, any>,
    signal: AbortSignal
  ): Promise<WorkflowExecution> {

    // 1. 创建执行上下文
    const execution = await this.createExecution(workflowDef, inputs);

    try {
      // 2. 构建执行图（DAG）
      const executionGraph = this.buildExecutionGraph(workflowDef);

      // 3. 拓扑排序（确定执行顺序）
      const executionOrder = this.topologicalSort(executionGraph);

      // 4. 执行节点
      for (const level of executionOrder) {
        // 同一层级的节点并发执行
        await this.executeLevelNodes(
          level,
          execution,
          workflowDef,
          signal
        );

        // 检查是否中止
        if (signal.aborted) {
          execution.status = 'aborted';
          break;
        }
      }

      // 5. 更新执行状态
      execution.status = 'completed';
      execution.endTime = Date.now();

    } catch (error) {
      execution.status = 'failed';
      execution.error = error.message;

    } finally {
      // 6. 持久化执行记录
      await this.executionStore.save(execution);

      // 7. 发布完成事件
      this.eventBus.emit('workflow:completed', execution);
    }

    return execution;
  }

  /**
   * 执行单个节点
   */
  private async executeNode(
    node: WorkflowNode,
    execution: WorkflowExecution,
    context: ExecutionContext,
    signal: AbortSignal
  ): Promise<NodeExecutionResult> {

    // 1. 获取节点执行器
    const executor = this.nodeExecutors.get(node.type);
    if (!executor) {
      throw new Error(`Unknown node type: ${node.type}`);
    }

    // 2. 发布开始事件（用于 UI 观察）
    this.eventBus.emit('node:started', {
      executionId: execution.id,
      nodeId: node.id,
      nodeName: node.name,
      timestamp: Date.now(),
    });

    const startTime = Date.now();

    try {
      // 3. 执行节点
      const result = await executor.execute(node, context, signal);

      // 4. 保存结果到上下文
      context.setNodeOutput(node.id, result.output);

      // 5. 发布成功事件
      this.eventBus.emit('node:completed', {
        executionId: execution.id,
        nodeId: node.id,
        nodeName: node.name,
        duration: Date.now() - startTime,
        output: result.output,
      });

      return result;

    } catch (error) {
      // 6. 发布失败事件
      this.eventBus.emit('node:failed', {
        executionId: execution.id,
        nodeId: node.id,
        nodeName: node.name,
        error: error.message,
      });

      throw error;
    }
  }

  /**
   * 并发执行同级节点
   */
  private async executeLevelNodes(
    nodeIds: string[],
    execution: WorkflowExecution,
    workflowDef: WorkflowDefinition,
    signal: AbortSignal
  ): Promise<void> {

    const context = execution.context;
    const nodes = nodeIds.map(id =>
      workflowDef.nodes.find(n => n.id === id)!
    );

    // 并发执行（受 concurrency 限制）
    const concurrency = workflowDef.config.concurrency || 5;
    const results = await pLimit(concurrency, nodes.map(node =>
      () => this.executeNode(node, execution, context, signal)
    ));

    // 检查是否有失败
    const failures = results.filter(r => r.status === 'failed');
    if (failures.length > 0) {
      throw new Error(`${failures.length} nodes failed in level`);
    }
  }

  /**
   * 构建执行图（DAG）
   */
  private buildExecutionGraph(
    workflowDef: WorkflowDefinition
  ): ExecutionGraph {

    const graph = new Map<string, Set<string>>();

    // 初始化节点
    for (const node of workflowDef.nodes) {
      graph.set(node.id, new Set());
    }

    // 添加边（依赖关系）
    for (const edge of workflowDef.edges) {
      graph.get(edge.target)!.add(edge.source);
    }

    return graph;
  }

  /**
   * 拓扑排序（Kahn 算法）
   */
  private topologicalSort(graph: ExecutionGraph): string[][] {
    const inDegree = new Map<string, number>();
    const levels: string[][] = [];

    // 计算入度
    for (const [node, deps] of graph) {
      inDegree.set(node, deps.size);
    }

    // 找到所有入度为 0 的节点
    let currentLevel = Array.from(inDegree.keys())
      .filter(node => inDegree.get(node) === 0);

    while (currentLevel.length > 0) {
      levels.push(currentLevel);

      const nextLevel: string[] = [];

      // 删除当前层节点，更新入度
      for (const node of currentLevel) {
        for (const [target, deps] of graph) {
          if (deps.has(node)) {
            deps.delete(node);
            if (deps.size === 0) {
              nextLevel.push(target);
            }
          }
        }
      }

      currentLevel = nextLevel;
    }

    // 检查是否有环
    if (levels.flat().length !== graph.size) {
      throw new Error('Workflow contains cycles');
    }

    return levels;
  }

  /**
   * 注册节点执行器
   */
  private registerNodeExecutors() {
    // Agent 节点执行器
    this.nodeExecutors.set('agent', new AgentNodeExecutor());

    // 工具节点执行器
    this.nodeExecutors.set('tool', new ToolNodeExecutor());

    // 条件节点执行器
    this.nodeExecutors.set('condition', new ConditionNodeExecutor());

    // 循环节点执行器
    this.nodeExecutors.set('loop', new LoopNodeExecutor());

    // 子工作流执行器
    this.nodeExecutors.set('subworkflow', new SubWorkflowNodeExecutor(this));
  }
}

/**
 * Agent 节点执行器（复用现有 AgentExecutor）
 */
class AgentNodeExecutor implements NodeExecutor {
  async execute(
    node: AgentNode,
    context: ExecutionContext,
    signal: AbortSignal
  ): Promise<NodeExecutionResult> {

    // 1. 准备 Agent 定义
    const agentDef: AgentDefinition = {
      name: node.name,
      description: node.name,
      ...node.agentConfig,
    };

    // 2. 解析输入（支持模板变量）
    const inputs = this.resolveInputs(node, context);

    // 3. 创建 Agent 执行器（复用现有组件）
    const executor = await AgentExecutor.create(
      agentDef,
      context.config,
      context.runtimeContext
    );

    // 4. 执行 Agent
    const result = await executor.run(inputs, signal);

    return {
      status: result.terminate_reason === 'GOAL' ? 'success' : 'failed',
      output: result.result,
    };
  }

  private resolveInputs(
    node: AgentNode,
    context: ExecutionContext
  ): Record<string, any> {
    // 模板变量替换
    // 例如: ${node1.output} → context.getNodeOutput('node1')
    // ...
  }
}

/**
 * 工具节点执行器
 */
class ToolNodeExecutor implements NodeExecutor {
  async execute(
    node: ToolNode,
    context: ExecutionContext,
    signal: AbortSignal
  ): Promise<NodeExecutionResult> {

    // 1. 获取工具
    const tool = context.toolRegistry.get(node.toolName);
    if (!tool) {
      throw new Error(`Tool not found: ${node.toolName}`);
    }

    // 2. 解析参数（支持模板）
    const params = this.resolveParameters(node.parameters, context);

    // 3. 构建工具调用
    const invocation = tool.build(params);

    // 4. 执行工具
    const result = await invocation.execute(signal);

    return {
      status: 'success',
      output: result,
    };
  }

  private resolveParameters(
    params: Record<string, any>,
    context: ExecutionContext
  ): Record<string, any> {
    // 递归解析参数中的模板变量
    // ...
  }
}

/**
 * 条件节点执行器
 */
class ConditionNodeExecutor implements NodeExecutor {
  async execute(
    node: ConditionNode,
    context: ExecutionContext,
    signal: AbortSignal
  ): Promise<NodeExecutionResult> {

    // 使用 JSONLogic 或自定义表达式引擎
    const condition = this.evaluateCondition(node.condition, context);

    return {
      status: 'success',
      output: { condition },
    };
  }

  private evaluateCondition(
    condition: string | object,
    context: ExecutionContext
  ): boolean {
    // 表达式求值
    // 支持：${node1.output.success} === true
    // ...
  }
}
```

### 2. 定时调度器

```typescript
// packages/workflow/src/scheduler.ts

import cron from 'node-cron';
import type { WorkflowDefinition, CronTrigger } from './types';
import { WorkflowOrchestrator } from './orchestrator';

/**
 * 工作流调度器
 * 支持定时触发工作流
 */
export class WorkflowScheduler {
  private scheduledTasks = new Map<string, cron.ScheduledTask>();

  constructor(
    private orchestrator: WorkflowOrchestrator,
    private workflowStore: WorkflowStore
  ) {}

  /**
   * 调度工作流
   */
  schedule(workflowId: string, trigger: CronTrigger): void {
    // 1. 验证 Cron 表达式
    if (!cron.validate(trigger.schedule)) {
      throw new Error(`Invalid cron expression: ${trigger.schedule}`);
    }

    // 2. 取消现有调度
    this.unschedule(workflowId);

    // 3. 创建新调度
    const task = cron.schedule(
      trigger.schedule,
      async () => {
        try {
          // 加载工作流定义
          const workflowDef = await this.workflowStore.get(workflowId);

          // 执行工作流
          await this.orchestrator.execute(
            workflowDef,
            {},  // 定时触发无输入
            new AbortController().signal
          );

        } catch (error) {
          console.error(`Scheduled workflow ${workflowId} failed:`, error);
        }
      },
      {
        timezone: trigger.timezone || 'UTC',
      }
    );

    this.scheduledTasks.set(workflowId, task);

    console.log(`Scheduled workflow ${workflowId} with cron: ${trigger.schedule}`);
  }

  /**
   * 取消调度
   */
  unschedule(workflowId: string): void {
    const task = this.scheduledTasks.get(workflowId);
    if (task) {
      task.stop();
      this.scheduledTasks.delete(workflowId);
    }
  }

  /**
   * 从数据库加载所有调度
   */
  async loadSchedules(): Promise<void> {
    const workflows = await this.workflowStore.getAllWithCronTriggers();

    for (const workflow of workflows) {
      const cronTriggers = workflow.triggers.filter(
        t => t.type === 'cron'
      ) as CronTrigger[];

      for (const trigger of cronTriggers) {
        this.schedule(workflow.id, trigger);
      }
    }
  }
}
```

### 3. 对话生成工作流

```typescript
// packages/workflow/src/conversation-workflow-generator.ts

import { AgentExecutor } from '@google/gemini-cli-core';
import type { WorkflowDefinition } from './types';

/**
 * 对话生成工作流
 * 使用 Agent 与用户对话，自动生成工作流定义
 */
export class ConversationWorkflowGenerator {
  private agentDef: AgentDefinition = {
    name: 'workflow-generator',
    description: '工作流生成助手',

    promptConfig: {
      systemPrompt: `你是一个工作流设计助手。你的任务是与用户对话，理解他们的需求，并生成工作流定义。

工作流由以下类型的节点组成：
1. agent: Agent 执行节点，可以执行复杂的多步骤任务
2. tool: 单一工具调用节点，执行特定操作（如读取文件、执行命令）
3. condition: 条件分支节点，根据条件选择路径
4. loop: 循环节点，重复执行某些操作

你需要：
1. 询问用户的需求（目标是什么）
2. 询问需要哪些步骤
3. 询问步骤之间的依赖关系
4. 询问是否需要条件分支或循环
5. 生成完整的工作流定义 JSON

当你准备好生成工作流时，使用 complete_task 工具返回工作流定义 JSON。`,
    },

    modelConfig: {
      model: 'gemini-2.0-flash-exp',
      temperature: 0.7,
    },

    runConfig: {
      maxTimeMinutes: 10,
      maxTurns: 50,
    },

    toolConfig: {
      tools: ['complete_task'],  // 只需要 complete_task 工具
    },

    outputConfig: {
      outputName: 'workflow',
      description: '生成的工作流定义 JSON',
      schema: z.object({
        workflow: z.string(),  // JSON 字符串
      }),
    },
  };

  /**
   * 通过对话生成工作流
   */
  async generateFromConversation(
    userId: string,
    initialMessage: string,
    signal: AbortSignal
  ): Promise<WorkflowDefinition> {

    // 1. 创建对话式 Agent
    const executor = await AgentExecutor.create(
      this.agentDef,
      config,
      runtimeContext
    );

    // 2. 运行 Agent（与用户对话）
    const result = await executor.run(
      { query: initialMessage },
      signal
    );

    // 3. 解析生成的工作流 JSON
    const workflowJson = JSON.parse(result.result);

    // 4. 验证工作流定义
    const workflowDef = this.validateWorkflow(workflowJson);

    return workflowDef;
  }

  /**
   * 验证工作流定义
   */
  private validateWorkflow(json: any): WorkflowDefinition {
    // 使用 Zod 或其他验证库
    // ...
    return json as WorkflowDefinition;
  }
}
```

### 4. 节点可观察性（实时监控）

```typescript
// packages/workflow/src/observability.ts

import type { WorkflowExecution, NodeEvent } from './types';

/**
 * 工作流可观察性服务
 * 提供实时监控和历史查询
 */
export class WorkflowObservabilityService {
  constructor(
    private eventBus: EventBus,
    private wsServer: WebSocketServer
  ) {
    this.setupEventListeners();
  }

  /**
   * 订阅工作流执行事件
   */
  private setupEventListeners() {
    // 节点开始
    this.eventBus.on('node:started', (event: NodeEvent) => {
      this.broadcastToClients(event.executionId, {
        type: 'node:started',
        data: event,
      });
    });

    // 节点完成
    this.eventBus.on('node:completed', (event: NodeEvent) => {
      this.broadcastToClients(event.executionId, {
        type: 'node:completed',
        data: event,
      });
    });

    // 节点失败
    this.eventBus.on('node:failed', (event: NodeEvent) => {
      this.broadcastToClients(event.executionId, {
        type: 'node:failed',
        data: event,
      });
    });

    // 工作流完成
    this.eventBus.on('workflow:completed', (execution: WorkflowExecution) => {
      this.broadcastToClients(execution.id, {
        type: 'workflow:completed',
        data: execution,
      });
    });
  }

  /**
   * 广播到 WebSocket 客户端
   */
  private broadcastToClients(executionId: string, message: any) {
    // 只发送给订阅了该执行的客户端
    this.wsServer.clients.forEach(client => {
      if (client.subscriptions?.includes(executionId)) {
        client.send(JSON.stringify(message));
      }
    });
  }

  /**
   * 获取执行历史
   */
  async getExecutionHistory(
    workflowId: string,
    limit: number = 50
  ): Promise<WorkflowExecution[]> {
    // 从数据库查询
    return await this.executionStore.getByWorkflowId(workflowId, limit);
  }

  /**
   * 获取节点执行详情
   */
  async getNodeExecutionDetails(
    executionId: string,
    nodeId: string
  ): Promise<NodeExecutionDetails> {
    // 查询节点执行日志、输入输出等
    return await this.executionStore.getNodeDetails(executionId, nodeId);
  }
}
```

---

## 🎨 技术栈选型

### 后端

```yaml
核心框架:
  - Express.js (已有)          # HTTP 服务器
  - TypeScript (已有)          # 类型安全

工作流引擎:
  - 自研 WorkflowOrchestrator  # 基于现有 core

定时调度:
  - node-cron                  # Cron 调度

实时通信:
  - ws (WebSocket)             # 实时监控

数据库:
  - PostgreSQL (推荐)          # 工作流定义、执行记录
  - OR MongoDB (备选)          # 文档型存储
  - TypeORM / Prisma           # ORM

队列 (可选):
  - Bull / BullMQ              # Redis 队列（处理长时间工作流）

日志:
  - Winston (已有)             # 日志记录
  - Loki / Elasticsearch       # 日志聚合（生产环境）
```

### 前端

```yaml
框架:
  - React 18                   # UI 框架
  - TypeScript                 # 类型安全

工作流设计器:
  - React Flow                 # 可视化工作流编辑器（推荐）
  - OR X6 (AntV)               # 备选方案

UI 组件库:
  - Ant Design                 # 组件库
  - OR Material-UI             # 备选方案

可视化:
  - ECharts                    # 图表
  - D3.js                      # 自定义可视化

状态管理:
  - Zustand / Jotai            # 轻量级状态管理

网络请求:
  - axios                      # HTTP 客户端
  - native WebSocket           # 实时通信

表单:
  - React Hook Form            # 表单管理
  - Zod                        # Schema 验证
```

### 部署

```yaml
容器化:
  - Docker                     # 容器
  - Docker Compose             # 本地开发
  - Kubernetes (可选)          # 生产环境

CI/CD:
  - GitHub Actions             # 已有

监控:
  - Prometheus + Grafana       # 指标监控
  - Sentry                     # 错误追踪
```

---

## 📅 实施路线图

### Phase 1: 基础设施（2-3 周）

```
Week 1-2: 后端基础
├─ 扩展 a2a-server 包
│  ├─ 添加工作流 CRUD API
│  ├─ 添加执行 API
│  └─ 添加 WebSocket 支持
├─ 实现 WorkflowOrchestrator
│  ├─ DAG 构建与拓扑排序
│  ├─ 节点执行器接口
│  └─ 事件系统集成
└─ 数据库设计与迁移
   ├─ workflows 表
   ├─ executions 表
   └─ schedules 表

Week 3: 前端基础
├─ 创建 React 项目
├─ 集成 React Flow
├─ 实现基础布局
│  ├─ 工作流列表页
│  ├─ 工作流编辑器页
│  └─ 执行监控页
└─ API 客户端封装
```

### Phase 2: 核心功能（3-4 周）

```
Week 4-5: 工作流编辑器
├─ 节点库（拖拽）
│  ├─ Agent 节点
│  ├─ Tool 节点
│  ├─ Condition 节点
│  └─ Loop 节点
├─ 节点配置面板
│  ├─ 参数表单
│  ├─ 模板变量支持
│  └─ 验证逻辑
├─ 边连接与配置
│  └─ 数据映射配置
└─ 工作流保存/加载

Week 6-7: 执行与监控
├─ 实现节点执行器
│  ├─ AgentNodeExecutor
│  ├─ ToolNodeExecutor
│  ├─ ConditionNodeExecutor
│  └─ LoopNodeExecutor
├─ 实时监控 UI
│  ├─ WebSocket 连接
│  ├─ 节点状态更新
│  └─ 日志流式显示
└─ 执行历史查询
   └─ 详情查看
```

### Phase 3: 高级特性（2-3 周）

```
Week 8-9: 定时调度与对话生成
├─ WorkflowScheduler 实现
│  ├─ Cron 调度
│  ├─ 调度管理 UI
│  └─ 调度历史
├─ ConversationWorkflowGenerator
│  ├─ 对话式 Agent
│  ├─ 工作流生成逻辑
│  └─ UI 集成
└─ Webhook 触发器
   └─ Webhook 管理 UI

Week 10: 子工作流与高级特性
├─ 子工作流节点
├─ 错误处理与重试
├─ 变量与表达式系统
└─ 工作流模板库
```

### Phase 4: 优化与发布（1-2 周）

```
Week 11-12: 打磨与发布
├─ 性能优化
│  ├─ 大规模工作流优化
│  └─ 并发控制优化
├─ 测试
│  ├─ 单元测试
│  ├─ 集成测试
│  └─ E2E 测试
├─ 文档
│  ├─ 用户文档
│  ├─ API 文档
│  └─ 开发文档
└─ 部署与发布
   ├─ Docker 镜像
   └─ 部署脚本
```

---

## 💻 代码示例

### 1. 项目结构

```
gemini-cli/
├── packages/
│   ├── core/                    # 现有核心引擎（复用）
│   ├── a2a-server/              # 现有 HTTP 服务器（扩展）
│   ├── cli/                     # 现有 CLI（保持）
│   │
│   ├── workflow/                # 【新增】工作流引擎包
│   │   ├── src/
│   │   │   ├── orchestrator.ts          # 工作流编排器
│   │   │   ├── scheduler.ts             # 定时调度器
│   │   │   ├── conversation-generator.ts # 对话生成
│   │   │   ├── observability.ts         # 可观察性
│   │   │   ├── executors/               # 节点执行器
│   │   │   │   ├── agent-executor.ts
│   │   │   │   ├── tool-executor.ts
│   │   │   │   ├── condition-executor.ts
│   │   │   │   └── loop-executor.ts
│   │   │   ├── types.ts                 # 类型定义
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   ├── workflow-api/            # 【新增】工作流 API 服务
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   │   ├── workflows.ts         # 工作流 CRUD
│   │   │   │   ├── executions.ts        # 执行管理
│   │   │   │   ├── schedules.ts         # 调度管理
│   │   │   │   └── conversation.ts      # 对话生成
│   │   │   ├── services/
│   │   │   │   ├── workflow-service.ts
│   │   │   │   └── execution-service.ts
│   │   │   ├── database/
│   │   │   │   ├── models/              # ORM 模型
│   │   │   │   └── migrations/          # 数据库迁移
│   │   │   ├── websocket/
│   │   │   │   └── execution-ws.ts      # WebSocket 服务
│   │   │   ├── server.ts                # Express 服务器
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   └── workflow-ui/             # 【新增】前端 UI
│       ├── src/
│       │   ├── pages/
│       │   │   ├── WorkflowList.tsx     # 工作流列表
│       │   │   ├── WorkflowEditor.tsx   # 工作流编辑器
│       │   │   ├── ExecutionMonitor.tsx # 执行监控
│       │   │   └── ConversationCreate.tsx # 对话创建
│       │   ├── components/
│       │   │   ├── workflow/
│       │   │   │   ├── FlowCanvas.tsx   # React Flow 画布
│       │   │   │   ├── NodeLibrary.tsx  # 节点库
│       │   │   │   ├── NodeConfig.tsx   # 节点配置面板
│       │   │   │   └── nodes/           # 自定义节点组件
│       │   │   └── execution/
│       │   │       ├── ExecutionGraph.tsx # 执行可视化
│       │   │       └── NodeStatus.tsx    # 节点状态
│       │   ├── api/
│       │   │   └── client.ts            # API 客户端
│       │   ├── hooks/
│       │   │   ├── useWorkflow.ts
│       │   │   ├── useExecution.ts
│       │   │   └── useWebSocket.ts
│       │   ├── stores/
│       │   │   └── workflow-store.ts    # 状态管理
│       │   ├── App.tsx
│       │   └── main.tsx
│       ├── package.json
│       └── vite.config.ts
│
├── docker-compose.yml           # Docker Compose 配置
└── README.md
```

### 2. API 路由实现

```typescript
// packages/workflow-api/src/routes/workflows.ts

import { Router } from 'express';
import { WorkflowService } from '../services/workflow-service';
import { WorkflowOrchestrator } from '@google/gemini-cli-workflow';

const router = Router();
const workflowService = new WorkflowService();
const orchestrator = new WorkflowOrchestrator(eventBus, executionStore);

/**
 * GET /api/workflows
 * 获取工作流列表
 */
router.get('/', async (req, res) => {
  try {
    const { page = 1, limit = 20 } = req.query;
    const workflows = await workflowService.list(
      Number(page),
      Number(limit)
    );
    res.json(workflows);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

/**
 * GET /api/workflows/:id
 * 获取工作流详情
 */
router.get('/:id', async (req, res) => {
  try {
    const workflow = await workflowService.getById(req.params.id);
    if (!workflow) {
      return res.status(404).json({ error: 'Workflow not found' });
    }
    res.json(workflow);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

/**
 * POST /api/workflows
 * 创建工作流
 */
router.post('/', async (req, res) => {
  try {
    const workflowDef = req.body;
    const workflow = await workflowService.create(workflowDef);
    res.status(201).json(workflow);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

/**
 * PUT /api/workflows/:id
 * 更新工作流
 */
router.put('/:id', async (req, res) => {
  try {
    const workflowDef = req.body;
    const workflow = await workflowService.update(req.params.id, workflowDef);
    res.json(workflow);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

/**
 * DELETE /api/workflows/:id
 * 删除工作流
 */
router.delete('/:id', async (req, res) => {
  try {
    await workflowService.delete(req.params.id);
    res.status(204).send();
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

/**
 * POST /api/workflows/:id/execute
 * 执行工作流
 */
router.post('/:id/execute', async (req, res) => {
  try {
    const workflow = await workflowService.getById(req.params.id);
    if (!workflow) {
      return res.status(404).json({ error: 'Workflow not found' });
    }

    const inputs = req.body.inputs || {};
    const controller = new AbortController();

    // 异步执行工作流
    const executionPromise = orchestrator.execute(
      workflow,
      inputs,
      controller.signal
    );

    // 立即返回执行 ID
    const executionId = executionPromise.id;
    res.status(202).json({ executionId });

  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

export default router;
```

### 3. React Flow 工作流编辑器

```typescript
// packages/workflow-ui/src/pages/WorkflowEditor.tsx

import React, { useCallback } from 'react';
import ReactFlow, {
  Background,
  Controls,
  MiniMap,
  addEdge,
  useNodesState,
  useEdgesState,
  type Connection,
} from 'reactflow';
import 'reactflow/dist/style.css';

import { AgentNode } from '../components/workflow/nodes/AgentNode';
import { ToolNode } from '../components/workflow/nodes/ToolNode';
import { ConditionNode } from '../components/workflow/nodes/ConditionNode';
import { NodeLibrary } from '../components/workflow/NodeLibrary';
import { NodeConfig } from '../components/workflow/NodeConfig';
import { useWorkflow } from '../hooks/useWorkflow';

// 注册自定义节点类型
const nodeTypes = {
  agent: AgentNode,
  tool: ToolNode,
  condition: ConditionNode,
};

export const WorkflowEditor: React.FC = () => {
  const { workflow, saveWorkflow } = useWorkflow();

  const [nodes, setNodes, onNodesChange] = useNodesState(workflow?.nodes || []);
  const [edges, setEdges, onEdgesChange] = useEdgesState(workflow?.edges || []);
  const [selectedNode, setSelectedNode] = React.useState(null);

  // 连接节点
  const onConnect = useCallback(
    (connection: Connection) => {
      setEdges((eds) => addEdge(connection, eds));
    },
    [setEdges]
  );

  // 从节点库拖拽添加节点
  const onDrop = useCallback(
    (event: React.DragEvent) => {
      event.preventDefault();

      const nodeType = event.dataTransfer.getData('nodeType');
      const position = {
        x: event.clientX,
        y: event.clientY,
      };

      const newNode = {
        id: `${nodeType}-${Date.now()}`,
        type: nodeType,
        position,
        data: {
          label: `${nodeType} Node`,
          // 默认配置
        },
      };

      setNodes((nds) => [...nds, newNode]);
    },
    [setNodes]
  );

  // 选择节点
  const onNodeClick = useCallback((event, node) => {
    setSelectedNode(node);
  }, []);

  // 保存工作流
  const handleSave = useCallback(async () => {
    await saveWorkflow({
      ...workflow,
      nodes,
      edges,
    });
  }, [workflow, nodes, edges, saveWorkflow]);

  return (
    <div className="workflow-editor">
      {/* 工具栏 */}
      <div className="toolbar">
        <button onClick={handleSave}>保存</button>
        <button>运行</button>
      </div>

      {/* 主编辑区 */}
      <div className="editor-container">
        {/* 节点库 */}
        <NodeLibrary />

        {/* React Flow 画布 */}
        <div
          className="flow-canvas"
          onDrop={onDrop}
          onDragOver={(e) => e.preventDefault()}
        >
          <ReactFlow
            nodes={nodes}
            edges={edges}
            onNodesChange={onNodesChange}
            onEdgesChange={onEdgesChange}
            onConnect={onConnect}
            onNodeClick={onNodeClick}
            nodeTypes={nodeTypes}
            fitView
          >
            <Background />
            <Controls />
            <MiniMap />
          </ReactFlow>
        </div>

        {/* 节点配置面板 */}
        {selectedNode && (
          <NodeConfig
            node={selectedNode}
            onUpdate={(updatedNode) => {
              setNodes((nds) =>
                nds.map((n) => (n.id === updatedNode.id ? updatedNode : n))
              );
            }}
            onClose={() => setSelectedNode(null)}
          />
        )}
      </div>
    </div>
  );
};
```

### 4. 执行监控组件

```typescript
// packages/workflow-ui/src/pages/ExecutionMonitor.tsx

import React, { useEffect, useState } from 'react';
import { useParams } from 'react-router-dom';
import ReactFlow, { Background, Controls } from 'reactflow';
import { useWebSocket } from '../hooks/useWebSocket';
import { useExecution } from '../hooks/useExecution';

export const ExecutionMonitor: React.FC = () => {
  const { executionId } = useParams();
  const { execution, loading } = useExecution(executionId);
  const [nodeStatuses, setNodeStatuses] = useState<Map<string, NodeStatus>>(new Map());

  // WebSocket 连接
  const { connected, lastMessage } = useWebSocket(`/executions/${executionId}`);

  // 处理 WebSocket 消息
  useEffect(() => {
    if (!lastMessage) return;

    const message = JSON.parse(lastMessage.data);

    switch (message.type) {
      case 'node:started':
        setNodeStatuses((prev) => new Map(prev).set(message.data.nodeId, {
          status: 'running',
          startTime: message.data.timestamp,
        }));
        break;

      case 'node:completed':
        setNodeStatuses((prev) => new Map(prev).set(message.data.nodeId, {
          status: 'success',
          startTime: prev.get(message.data.nodeId)?.startTime,
          endTime: Date.now(),
          output: message.data.output,
        }));
        break;

      case 'node:failed':
        setNodeStatuses((prev) => new Map(prev).set(message.data.nodeId, {
          status: 'failed',
          startTime: prev.get(message.data.nodeId)?.startTime,
          endTime: Date.now(),
          error: message.data.error,
        }));
        break;
    }
  }, [lastMessage]);

  if (loading) return <div>加载中...</div>;
  if (!execution) return <div>执行不存在</div>;

  // 给节点添加状态样式
  const nodesWithStatus = execution.workflow.nodes.map((node) => {
    const status = nodeStatuses.get(node.id);
    return {
      ...node,
      data: {
        ...node.data,
        status: status?.status || 'pending',
      },
      style: {
        ...node.style,
        backgroundColor: getNodeColor(status?.status),
      },
    };
  });

  return (
    <div className="execution-monitor">
      {/* 头部信息 */}
      <div className="header">
        <h2>{execution.workflow.name}</h2>
        <div className="status">
          状态: {execution.status}
          {connected && <span className="ws-connected">● 实时监控</span>}
        </div>
      </div>

      {/* 工作流可视化 */}
      <div className="flow-visualization">
        <ReactFlow
          nodes={nodesWithStatus}
          edges={execution.workflow.edges}
          fitView
          nodesDraggable={false}
          nodesConnectable={false}
        >
          <Background />
          <Controls showInteractive={false} />
        </ReactFlow>
      </div>

      {/* 执行日志 */}
      <div className="execution-logs">
        <h3>执行日志</h3>
        <div className="log-list">
          {Array.from(nodeStatuses.entries()).map(([nodeId, status]) => (
            <div key={nodeId} className={`log-entry ${status.status}`}>
              <span className="time">{new Date(status.startTime).toLocaleTimeString()}</span>
              <span className="node">{nodeId}</span>
              <span className="status">{status.status}</span>
              {status.error && <span className="error">{status.error}</span>}
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

function getNodeColor(status?: string) {
  switch (status) {
    case 'running': return '#1890ff';
    case 'success': return '#52c41a';
    case 'failed': return '#ff4d4f';
    default: return '#d9d9d9';
  }
}
```

---

## 🎯 总结

### ✅ 可行性结论

**强烈推荐二开**，理由：
1. ✅ 项目架构优秀，模块化程度高
2. ✅ 核心组件完备（Agent、Tool、Scheduler）
3. ✅ 已有 HTTP API 服务器基础
4. ✅ 事件系统支持可观察性
5. ✅ 扩展性强（Hook、Extension、MCP）

### 🚀 关键优势

1. **复用核心引擎**：无需重新实现 Agent 执行、工具调度等核心功能
2. **快速迭代**：基于成熟代码库，降低开发风险
3. **类型安全**：TypeScript 提供完整类型支持
4. **灵活扩展**：可以渐进式添加功能

### 📝 下一步行动

1. **Fork/Clone** 项目
2. **创建新分支** `feature/workflow-orchestration`
3. **添加新包** `packages/workflow`
4. **扩展 API** `packages/workflow-api`
5. **开发前端** `packages/workflow-ui`
6. **迭代发布**

需要我提供任何部分的详细代码实现吗？
