# 🚀 工作流编排系统快速开始指南

> 基于 Gemini CLI 的工作流编排系统二次开发快速指南

## 📋 目录

1. [环境准备](#环境准备)
2. [快速开始](#快速开始)
3. [创建第一个工作流](#创建第一个工作流)
4. [API 使用示例](#api-使用示例)
5. [常见问题](#常见问题)

---

## ✅ 环境准备

### 系统要求

```bash
# Node.js 版本
node >= 20

# 包管理器
npm >= 9 或 pnpm >= 8

# 数据库（可选，开发时使用 SQLite）
PostgreSQL >= 14 或 MongoDB >= 6
```

### 克隆项目

```bash
# Clone 项目
git clone https://github.com/anting2019/gemini-cli.git
cd gemini-cli

# 安装依赖
npm install
# 或
pnpm install
```

---

## 🚀 快速开始

### Step 1: 创建工作流包

```bash
# 创建新的工作流包目录
mkdir -p packages/workflow/src/{executors,types}

# 初始化 package.json
cd packages/workflow
cat > package.json <<EOF
{
  "name": "@google/gemini-cli-workflow",
  "version": "0.1.0",
  "description": "Workflow orchestration engine",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch"
  },
  "dependencies": {
    "@google/gemini-cli-core": "workspace:*",
    "p-limit": "^5.0.0"
  },
  "devDependencies": {
    "typescript": "^5.3.3"
  }
}
EOF

# 创建 tsconfig.json
cat > tsconfig.json <<EOF
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}
EOF
```

### Step 2: 实现核心工作流引擎

创建 `packages/workflow/src/types.ts`：

```typescript
import type { AgentDefinition } from '@google/gemini-cli-core';

export interface WorkflowDefinition {
  id: string;
  name: string;
  description: string;
  nodes: WorkflowNode[];
  edges: WorkflowEdge[];
}

export type WorkflowNode = {
  id: string;
  type: 'agent' | 'tool' | 'start' | 'end';
  name: string;
  position: { x: number; y: number };
  config?: any;
};

export interface WorkflowEdge {
  id: string;
  source: string;
  target: string;
}

export interface WorkflowExecution {
  id: string;
  workflowId: string;
  status: 'pending' | 'running' | 'completed' | 'failed';
  startTime: number;
  endTime?: number;
  context: ExecutionContext;
}

export class ExecutionContext {
  private outputs = new Map<string, any>();

  setNodeOutput(nodeId: string, output: any) {
    this.outputs.set(nodeId, output);
  }

  getNodeOutput(nodeId: string): any {
    return this.outputs.get(nodeId);
  }
}
```

创建 `packages/workflow/src/orchestrator.ts`：

```typescript
import { AgentExecutor } from '@google/gemini-cli-core';
import type {
  WorkflowDefinition,
  WorkflowExecution,
  WorkflowNode,
  ExecutionContext,
} from './types';

export class WorkflowOrchestrator {
  async execute(
    workflow: WorkflowDefinition,
    inputs: Record<string, any>,
    signal: AbortSignal
  ): Promise<WorkflowExecution> {

    const execution: WorkflowExecution = {
      id: `exec-${Date.now()}`,
      workflowId: workflow.id,
      status: 'running',
      startTime: Date.now(),
      context: new ExecutionContext(),
    };

    try {
      // 构建执行图
      const graph = this.buildGraph(workflow);

      // 拓扑排序
      const order = this.topologicalSort(graph);

      // 逐层执行
      for (const level of order) {
        await this.executeLevelNodes(level, workflow, execution, signal);
      }

      execution.status = 'completed';
      execution.endTime = Date.now();

    } catch (error) {
      execution.status = 'failed';
      execution.endTime = Date.now();
      throw error;
    }

    return execution;
  }

  private buildGraph(workflow: WorkflowDefinition): Map<string, Set<string>> {
    const graph = new Map<string, Set<string>>();

    // 初始化所有节点
    for (const node of workflow.nodes) {
      graph.set(node.id, new Set());
    }

    // 添加依赖关系
    for (const edge of workflow.edges) {
      graph.get(edge.target)!.add(edge.source);
    }

    return graph;
  }

  private topologicalSort(graph: Map<string, Set<string>>): string[][] {
    const inDegree = new Map<string, number>();
    const levels: string[][] = [];

    // 计算入度
    for (const [node, deps] of graph) {
      inDegree.set(node, deps.size);
    }

    // Kahn 算法
    let currentLevel = Array.from(inDegree.keys())
      .filter(node => inDegree.get(node) === 0);

    while (currentLevel.length > 0) {
      levels.push(currentLevel);

      const nextLevel: string[] = [];
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

    return levels;
  }

  private async executeLevelNodes(
    nodeIds: string[],
    workflow: WorkflowDefinition,
    execution: WorkflowExecution,
    signal: AbortSignal
  ): Promise<void> {

    const nodes = nodeIds.map(id =>
      workflow.nodes.find(n => n.id === id)!
    );

    // 并发执行
    await Promise.all(
      nodes.map(node => this.executeNode(node, execution, signal))
    );
  }

  private async executeNode(
    node: WorkflowNode,
    execution: WorkflowExecution,
    signal: AbortSignal
  ): Promise<void> {

    console.log(`Executing node: ${node.name}`);

    // 根据节点类型执行
    if (node.type === 'agent') {
      // 使用 AgentExecutor 执行
      const agentDef = node.config as AgentDefinition;
      // TODO: 调用 AgentExecutor
    } else if (node.type === 'tool') {
      // 执行单一工具
      // TODO: 调用工具
    }

    // 保存输出
    execution.context.setNodeOutput(node.id, { success: true });
  }
}
```

创建 `packages/workflow/src/index.ts`：

```typescript
export * from './types';
export * from './orchestrator';
```

### Step 3: 构建工作流包

```bash
cd packages/workflow
npm run build
```

### Step 4: 扩展 API 服务器

编辑 `packages/a2a-server/src/http/app.ts`，添加工作流 API：

```typescript
import { WorkflowOrchestrator } from '@google/gemini-cli-workflow';
import type { WorkflowDefinition } from '@google/gemini-cli-workflow';

// 在 createApp 函数中添加以下路由

// 创建编排器实例
const orchestrator = new WorkflowOrchestrator();

// 内存存储（生产环境应使用数据库）
const workflows = new Map<string, WorkflowDefinition>();
const executions = new Map<string, any>();

/**
 * POST /api/workflows
 * 创建工作流
 */
expressApp.post('/api/workflows', async (req, res) => {
  try {
    const workflow: WorkflowDefinition = req.body;
    workflows.set(workflow.id, workflow);
    res.status(201).json(workflow);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

/**
 * GET /api/workflows
 * 获取工作流列表
 */
expressApp.get('/api/workflows', async (req, res) => {
  try {
    const list = Array.from(workflows.values());
    res.json(list);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

/**
 * GET /api/workflows/:id
 * 获取工作流详情
 */
expressApp.get('/api/workflows/:id', async (req, res) => {
  try {
    const workflow = workflows.get(req.params.id);
    if (!workflow) {
      return res.status(404).json({ error: 'Workflow not found' });
    }
    res.json(workflow);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

/**
 * POST /api/workflows/:id/execute
 * 执行工作流
 */
expressApp.post('/api/workflows/:id/execute', async (req, res) => {
  try {
    const workflow = workflows.get(req.params.id);
    if (!workflow) {
      return res.status(404).json({ error: 'Workflow not found' });
    }

    const inputs = req.body.inputs || {};
    const controller = new AbortController();

    // 异步执行
    orchestrator.execute(workflow, inputs, controller.signal)
      .then(execution => {
        executions.set(execution.id, execution);
      })
      .catch(error => {
        console.error('Workflow execution failed:', error);
      });

    // 立即返回执行 ID
    const executionId = `exec-${Date.now()}`;
    res.status(202).json({ executionId });

  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

/**
 * GET /api/executions/:id
 * 获取执行状态
 */
expressApp.get('/api/executions/:id', async (req, res) => {
  try {
    const execution = executions.get(req.params.id);
    if (!execution) {
      return res.status(404).json({ error: 'Execution not found' });
    }
    res.json(execution);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### Step 5: 启动服务器

```bash
# 构建所有包
npm run build

# 启动 API 服务器
cd packages/a2a-server
npm start
```

服务器默认运行在 `http://localhost:41242`

---

## 📝 创建第一个工作流

### 使用 API 创建工作流

```bash
# 创建一个简单的工作流
curl -X POST http://localhost:41242/api/workflows \
  -H "Content-Type: application/json" \
  -d '{
    "id": "workflow-1",
    "name": "My First Workflow",
    "description": "A simple workflow example",
    "nodes": [
      {
        "id": "start",
        "type": "start",
        "name": "Start",
        "position": { "x": 100, "y": 100 }
      },
      {
        "id": "agent-1",
        "type": "agent",
        "name": "Code Generator",
        "position": { "x": 300, "y": 100 },
        "config": {
          "name": "code-generator",
          "description": "Generate code based on user request",
          "promptConfig": {
            "systemPrompt": "You are a code generator. Generate code based on user requests.",
            "query": "Generate a hello world function"
          },
          "modelConfig": {
            "model": "gemini-2.0-flash-exp",
            "temperature": 0.7
          },
          "runConfig": {
            "maxTimeMinutes": 5,
            "maxTurns": 10
          }
        }
      },
      {
        "id": "end",
        "type": "end",
        "name": "End",
        "position": { "x": 500, "y": 100 }
      }
    ],
    "edges": [
      {
        "id": "edge-1",
        "source": "start",
        "target": "agent-1"
      },
      {
        "id": "edge-2",
        "source": "agent-1",
        "target": "end"
      }
    ]
  }'
```

### 执行工作流

```bash
# 执行工作流
curl -X POST http://localhost:41242/api/workflows/workflow-1/execute \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": {}
  }'

# 返回
{
  "executionId": "exec-1234567890"
}
```

### 查询执行状态

```bash
# 查询执行状态
curl http://localhost:41242/api/executions/exec-1234567890

# 返回
{
  "id": "exec-1234567890",
  "workflowId": "workflow-1",
  "status": "completed",
  "startTime": 1234567890000,
  "endTime": 1234567900000
}
```

---

## 💻 API 使用示例

### JavaScript/TypeScript 客户端

```typescript
// workflow-client.ts

class WorkflowClient {
  constructor(private baseURL: string = 'http://localhost:41242') {}

  // 创建工作流
  async createWorkflow(workflow: WorkflowDefinition): Promise<WorkflowDefinition> {
    const response = await fetch(`${this.baseURL}/api/workflows`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(workflow),
    });
    return response.json();
  }

  // 获取工作流列表
  async listWorkflows(): Promise<WorkflowDefinition[]> {
    const response = await fetch(`${this.baseURL}/api/workflows`);
    return response.json();
  }

  // 获取工作流详情
  async getWorkflow(id: string): Promise<WorkflowDefinition> {
    const response = await fetch(`${this.baseURL}/api/workflows/${id}`);
    return response.json();
  }

  // 执行工作流
  async executeWorkflow(id: string, inputs: Record<string, any>): Promise<string> {
    const response = await fetch(`${this.baseURL}/api/workflows/${id}/execute`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ inputs }),
    });
    const result = await response.json();
    return result.executionId;
  }

  // 获取执行状态
  async getExecution(id: string): Promise<WorkflowExecution> {
    const response = await fetch(`${this.baseURL}/api/executions/${id}`);
    return response.json();
  }

  // 轮询执行状态直到完成
  async waitForCompletion(executionId: string, timeout: number = 300000): Promise<WorkflowExecution> {
    const startTime = Date.now();

    while (Date.now() - startTime < timeout) {
      const execution = await this.getExecution(executionId);

      if (execution.status === 'completed' || execution.status === 'failed') {
        return execution;
      }

      // 等待 1 秒后重试
      await new Promise(resolve => setTimeout(resolve, 1000));
    }

    throw new Error('Execution timeout');
  }
}

// 使用示例
async function main() {
  const client = new WorkflowClient();

  // 创建工作流
  const workflow = await client.createWorkflow({
    id: 'test-workflow',
    name: 'Test Workflow',
    description: 'Test workflow',
    nodes: [
      // ...节点定义
    ],
    edges: [
      // ...边定义
    ],
  });

  console.log('Workflow created:', workflow);

  // 执行工作流
  const executionId = await client.executeWorkflow(workflow.id, {});
  console.log('Execution started:', executionId);

  // 等待完成
  const execution = await client.waitForCompletion(executionId);
  console.log('Execution completed:', execution);
}

main().catch(console.error);
```

### Python 客户端

```python
# workflow_client.py

import requests
import time
from typing import Dict, List, Any

class WorkflowClient:
    def __init__(self, base_url: str = "http://localhost:41242"):
        self.base_url = base_url

    def create_workflow(self, workflow: Dict[str, Any]) -> Dict[str, Any]:
        """创建工作流"""
        response = requests.post(
            f"{self.base_url}/api/workflows",
            json=workflow
        )
        response.raise_for_status()
        return response.json()

    def list_workflows(self) -> List[Dict[str, Any]]:
        """获取工作流列表"""
        response = requests.get(f"{self.base_url}/api/workflows")
        response.raise_for_status()
        return response.json()

    def get_workflow(self, workflow_id: str) -> Dict[str, Any]:
        """获取工作流详情"""
        response = requests.get(f"{self.base_url}/api/workflows/{workflow_id}")
        response.raise_for_status()
        return response.json()

    def execute_workflow(self, workflow_id: str, inputs: Dict[str, Any] = None) -> str:
        """执行工作流"""
        response = requests.post(
            f"{self.base_url}/api/workflows/{workflow_id}/execute",
            json={"inputs": inputs or {}}
        )
        response.raise_for_status()
        return response.json()["executionId"]

    def get_execution(self, execution_id: str) -> Dict[str, Any]:
        """获取执行状态"""
        response = requests.get(f"{self.base_url}/api/executions/{execution_id}")
        response.raise_for_status()
        return response.json()

    def wait_for_completion(self, execution_id: str, timeout: int = 300) -> Dict[str, Any]:
        """轮询直到执行完成"""
        start_time = time.time()

        while time.time() - start_time < timeout:
            execution = self.get_execution(execution_id)

            if execution["status"] in ["completed", "failed"]:
                return execution

            time.sleep(1)

        raise TimeoutError("Execution timeout")

# 使用示例
if __name__ == "__main__":
    client = WorkflowClient()

    # 创建工作流
    workflow = client.create_workflow({
        "id": "test-workflow",
        "name": "Test Workflow",
        "description": "Test workflow",
        "nodes": [
            # ...节点定义
        ],
        "edges": [
            # ...边定义
        ]
    })

    print("Workflow created:", workflow)

    # 执行工作流
    execution_id = client.execute_workflow(workflow["id"], {})
    print("Execution started:", execution_id)

    # 等待完成
    execution = client.wait_for_completion(execution_id)
    print("Execution completed:", execution)
```

---

## ❓ 常见问题

### Q1: 如何添加自定义节点类型？

在 `WorkflowOrchestrator` 中添加新的节点执行器：

```typescript
private async executeNode(node: WorkflowNode, ...): Promise<void> {
  if (node.type === 'custom') {
    // 自定义节点逻辑
    const result = await this.executeCustomNode(node);
    execution.context.setNodeOutput(node.id, result);
  }
}

private async executeCustomNode(node: WorkflowNode): Promise<any> {
  // 实现自定义逻辑
  return { success: true };
}
```

### Q2: 如何实现条件分支？

添加条件节点类型，并在执行时评估条件：

```typescript
if (node.type === 'condition') {
  const condition = this.evaluateCondition(node.config.expression, execution.context);
  execution.context.setNodeOutput(node.id, { condition });

  // 根据条件选择下一个节点
  // ...
}
```

### Q3: 如何实现定时触发？

使用 `node-cron`：

```typescript
import cron from 'node-cron';

// 调度工作流
cron.schedule('*/5 * * * *', async () => {
  // 每 5 分钟执行一次
  await orchestrator.execute(workflow, {}, new AbortController().signal);
});
```

### Q4: 如何实现 WebSocket 实时监控？

```typescript
import { WebSocketServer } from 'ws';

const wss = new WebSocketServer({ port: 8080 });

// 在节点执行时发送事件
wss.clients.forEach(client => {
  client.send(JSON.stringify({
    type: 'node:started',
    data: { nodeId: node.id, timestamp: Date.now() }
  }));
});
```

### Q5: 如何持久化工作流定义？

使用数据库（PostgreSQL 示例）：

```typescript
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

// 保存工作流
await pool.query(
  'INSERT INTO workflows (id, name, definition) VALUES ($1, $2, $3)',
  [workflow.id, workflow.name, JSON.stringify(workflow)]
);

// 查询工作流
const result = await pool.query(
  'SELECT definition FROM workflows WHERE id = $1',
  [workflowId]
);
const workflow = JSON.parse(result.rows[0].definition);
```

---

## 📚 下一步

1. ✅ 阅读完整的二开方案：`workflow_orchestration_redesign.md`
2. ✅ 查看架构分析：`implementation_guide.md`
3. ✅ 开发前端 UI（React Flow）
4. ✅ 添加更多节点类型（循环、子工作流）
5. ✅ 实现定时调度和对话生成
6. ✅ 集成数据库和 WebSocket

## 🎯 资源链接

- [React Flow 文档](https://reactflow.dev/)
- [node-cron 文档](https://www.npmjs.com/package/node-cron)
- [Express.js 文档](https://expressjs.com/)
- [PostgreSQL 文档](https://www.postgresql.org/docs/)

祝你二开顺利！🚀
