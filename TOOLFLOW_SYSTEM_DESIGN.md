# 基于工具流的智能编排系统 - 实战方案

> 以工具调用为核心，对话生成为辅助，打造现代化工作流平台

## 📋 目录

1. [核心理念](#核心理念)
2. [系统架构](#系统架构)
3. [工具流设计](#工具流设计)
4. [现代化 UI 设计](#现代化ui设计)
5. [完整代码实现](#完整代码实现)
6. [部署方案](#部署方案)

---

## 💡 核心理念

### 设计原则

```
🎯 核心：工具流编排（80%）
  ├─ 工具是第一公民
  ├─ 可视化拖拽连接
  ├─ 数据流自动映射
  └─ 丰富的内置工具库

💬 辅助：对话生成（20%）
  ├─ 自然语言描述需求
  ├─ AI 自动生成工作流
  ├─ 智能推荐工具组合
  └─ 工作流优化建议

✨ 用户体验
  ├─ 现代化 UI（Tailwind + shadcn/ui）
  ├─ 流畅动画（Framer Motion）
  ├─ 实时反馈（WebSocket）
  └─ 响应式设计
```

---

## 🏗️ 系统架构

### 整体架构图

```
┌──────────────────────────────────────────────────────────────┐
│                    现代化前端 UI                              │
│  ┌───────────────────────────────────────────────────────┐   │
│  │          工具流可视化编辑器 (React Flow)               │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │   │
│  │  │文件工具   │→ │代码工具   │→ │Shell工具 │            │   │
│  │  └──────────┘  └──────────┘  └──────────┘            │   │
│  └───────────────────────────────────────────────────────┘   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ 工具库面板    │  │ 实时监控面板 │  │ 对话助手面板 │      │
│  │ (拖拽添加)    │  │ (执行日志)   │  │ (生成工作流) │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└──────────────────────┬───────────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────────┐
│                  工具流编排引擎                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │           ToolFlowOrchestrator                      │     │
│  │  ① 解析工具流 DSL                                    │     │
│  │  ② 构建执行计划（DAG）                               │     │
│  │  ③ 数据流自动传递                                    │     │
│  │  ④ 并发执行控制                                      │     │
│  │  ⑤ 错误处理与重试                                    │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ToolExecutor  │  │DataFlowEngine│  │ErrorHandler  │       │
│  │(工具执行器)   │  │(数据映射)     │  │(错误重试)     │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└──────────────────────┬───────────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────────┐
│              工具层 (复用 core + 扩展)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │Read File │  │Write File│  │Bash Shell│  │Grep      │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │HTTP Call │  │Database  │  │Transform │  │Custom... │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
└───────────────────────────────────────────────────────────────┘

                    [对话助手 - 可选辅助功能]
                    ┌──────────────────────┐
                    │ ConversationAgent    │
                    │ "我想读取文件并分析" │
                    │        ↓             │
                    │  生成工具流 JSON      │
                    └──────────────────────┘
```

---

## 🔧 工具流设计

### 工具流 DSL（JSON 格式）

```typescript
// 工具流定义
interface ToolFlow {
  id: string;
  name: string;
  description: string;

  // 工具节点（核心）
  tools: ToolNode[];

  // 连接关系
  connections: Connection[];

  // 全局配置
  config: {
    timeout: number;
    retryPolicy: RetryPolicy;
    concurrency: number;
  };
}

// 工具节点
interface ToolNode {
  id: string;
  type: string;           // 工具类型：read_file, write_file, bash, http, etc.
  name: string;
  position: Position;     // UI 位置

  // 工具参数（支持模板变量）
  inputs: Record<string, any>;

  // 输出映射
  outputs: {
    [key: string]: {
      type: string;       // string, number, object, array
      description: string;
    };
  };

  // 节点配置
  config?: {
    timeout?: number;
    retries?: number;
    condition?: string;   // 执行条件
  };
}

// 连接定义
interface Connection {
  id: string;
  source: string;         // 源节点 ID
  sourceOutput: string;   // 源节点输出字段
  target: string;         // 目标节点 ID
  targetInput: string;    // 目标节点输入字段

  // 数据转换（可选）
  transform?: {
    type: 'jsonPath' | 'template' | 'script';
    expression: string;
  };
}
```

### 示例：文件分析工具流

```json
{
  "id": "file-analysis-flow",
  "name": "文件内容分析",
  "description": "读取文件 → 分析内容 → 生成报告",

  "tools": [
    {
      "id": "read-1",
      "type": "read_file",
      "name": "读取源文件",
      "position": { "x": 100, "y": 100 },
      "inputs": {
        "path": "${flow.input.filePath}"
      },
      "outputs": {
        "content": {
          "type": "string",
          "description": "文件内容"
        }
      }
    },
    {
      "id": "analyze-1",
      "type": "llm_analyze",
      "name": "AI 分析内容",
      "position": { "x": 300, "y": 100 },
      "inputs": {
        "text": "${read-1.content}",
        "prompt": "分析这段代码的质量和问题"
      },
      "outputs": {
        "analysis": {
          "type": "object",
          "description": "分析结果"
        }
      }
    },
    {
      "id": "write-1",
      "type": "write_file",
      "name": "保存报告",
      "position": { "x": 500, "y": 100 },
      "inputs": {
        "path": "./analysis-report.md",
        "content": "${analyze-1.analysis}"
      },
      "outputs": {
        "success": {
          "type": "boolean",
          "description": "是否成功"
        }
      }
    }
  ],

  "connections": [
    {
      "id": "conn-1",
      "source": "read-1",
      "sourceOutput": "content",
      "target": "analyze-1",
      "targetInput": "text"
    },
    {
      "id": "conn-2",
      "source": "analyze-1",
      "sourceOutput": "analysis",
      "target": "write-1",
      "targetInput": "content",
      "transform": {
        "type": "template",
        "expression": "# 分析报告\n\n${value}"
      }
    }
  ],

  "config": {
    "timeout": 300,
    "retryPolicy": {
      "maxRetries": 3,
      "backoff": "exponential"
    },
    "concurrency": 5
  }
}
```

---

## 🎨 现代化 UI 设计

### 技术栈

```yaml
核心框架:
  - React 18 + TypeScript
  - Vite (构建工具)

UI 组件库:
  - shadcn/ui (推荐)           # 现代化组件库
  - Radix UI                  # 无障碍组件
  - Tailwind CSS              # 原子化 CSS

工作流可视化:
  - React Flow                # 可视化编辑器
  - XYFlow (React Flow v12)   # 最新版本

图表可视化:
  - Recharts                  # React 图表库
  - Tremor                    # 现代数据可视化

动画:
  - Framer Motion             # 流畅动画

表单:
  - React Hook Form           # 表单管理
  - Zod                       # Schema 验证

状态管理:
  - Zustand                   # 轻量级状态管理

图标:
  - Lucide React              # 现代图标库
```

### UI 布局设计

```
┌─────────────────────────────────────────────────────────────┐
│  Header                                           [用户菜单] │
├──────────┬──────────────────────────────────────────────────┤
│          │                                                   │
│  工具库   │            工作流画布 (React Flow)                │
│          │  ┌─────────┐          ┌─────────┐                │
│ 🔧 文件   │  │ 读文件   │ ───────→ │ 分析内容 │                │
│ 🔧 代码   │  └─────────┘          └─────────┘                │
│ 🔧 网络   │                            │                     │
│ 🔧 数据库 │                            ↓                     │
│ 🔧 AI     │                       ┌─────────┐                │
│ ➕ 自定义 │                       │ 保存报告 │                │
│          │                       └─────────┘                │
│          │                                                   │
│  [搜索]   │                                  [运行] [保存]   │
├──────────┴──────────────────────────────────────────────────┤
│  底部面板 (可折叠)                                           │
│  ┌──────────┬──────────┬──────────┬──────────┐             │
│  │ 执行日志 │ 节点详情 │ 数据预览 │ 对话助手 │             │
│  └──────────┴──────────┴──────────┴──────────┘             │
└─────────────────────────────────────────────────────────────┘
```

### 关键 UI 组件

#### 1. 工具节点卡片

```tsx
// ToolNodeCard.tsx

import { Handle, Position } from 'reactflow';
import { Card } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { FileText, AlertCircle, CheckCircle2, Loader2 } from 'lucide-react';

interface ToolNodeCardProps {
  data: {
    type: string;
    name: string;
    status?: 'idle' | 'running' | 'success' | 'error';
    icon?: React.ReactNode;
  };
}

export function ToolNodeCard({ data }: ToolNodeCardProps) {
  const statusConfig = {
    idle: { color: 'bg-gray-500', icon: null },
    running: { color: 'bg-blue-500', icon: <Loader2 className="animate-spin" /> },
    success: { color: 'bg-green-500', icon: <CheckCircle2 /> },
    error: { color: 'bg-red-500', icon: <AlertCircle /> },
  };

  const status = statusConfig[data.status || 'idle'];

  return (
    <Card className="min-w-[200px] shadow-lg hover:shadow-xl transition-shadow">
      {/* 输入句柄 */}
      <Handle type="target" position={Position.Left} className="w-3 h-3" />

      <div className="p-4">
        {/* 工具图标和名称 */}
        <div className="flex items-center gap-2 mb-2">
          {data.icon || <FileText className="w-5 h-5 text-primary" />}
          <span className="font-semibold">{data.name}</span>
        </div>

        {/* 工具类型标签 */}
        <Badge variant="secondary" className="text-xs">
          {data.type}
        </Badge>

        {/* 状态指示器 */}
        <div className="flex items-center gap-2 mt-3">
          <div className={`w-2 h-2 rounded-full ${status.color}`} />
          {status.icon && <span className="text-xs text-muted-foreground">{status.icon}</span>}
        </div>
      </div>

      {/* 输出句柄 */}
      <Handle type="source" position={Position.Right} className="w-3 h-3" />
    </Card>
  );
}
```

#### 2. 工具库侧边栏

```tsx
// ToolLibrarySidebar.tsx

import { useState } from 'react';
import { Input } from '@/components/ui/input';
import { ScrollArea } from '@/components/ui/scroll-area';
import { Separator } from '@/components/ui/separator';
import {
  FileText,
  Code,
  Globe,
  Database,
  Cpu,
  Search,
} from 'lucide-react';

const toolCategories = [
  {
    name: '文件操作',
    icon: FileText,
    tools: [
      { id: 'read_file', name: '读取文件', description: '读取文本文件内容' },
      { id: 'write_file', name: '写入文件', description: '写入内容到文件' },
      { id: 'glob', name: '查找文件', description: '使用模式匹配查找文件' },
    ],
  },
  {
    name: '代码工具',
    icon: Code,
    tools: [
      { id: 'grep', name: '搜索代码', description: '在代码中搜索文本' },
      { id: 'edit', name: '编辑文件', description: '精确替换文件内容' },
      { id: 'bash', name: '执行命令', description: '运行 Shell 命令' },
    ],
  },
  {
    name: '网络请求',
    icon: Globe,
    tools: [
      { id: 'http_get', name: 'HTTP GET', description: '发送 GET 请求' },
      { id: 'http_post', name: 'HTTP POST', description: '发送 POST 请求' },
      { id: 'websocket', name: 'WebSocket', description: 'WebSocket 连接' },
    ],
  },
  {
    name: 'AI 工具',
    icon: Cpu,
    tools: [
      { id: 'llm_analyze', name: 'AI 分析', description: '使用 LLM 分析文本' },
      { id: 'llm_generate', name: 'AI 生成', description: '生成代码或文本' },
      { id: 'llm_translate', name: 'AI 翻译', description: '翻译文本' },
    ],
  },
];

export function ToolLibrarySidebar() {
  const [searchQuery, setSearchQuery] = useState('');

  const onDragStart = (event: React.DragEvent, toolId: string) => {
    event.dataTransfer.setData('application/reactflow', toolId);
    event.dataTransfer.effectAllowed = 'move';
  };

  return (
    <div className="w-64 border-r bg-background">
      {/* 搜索框 */}
      <div className="p-4">
        <div className="relative">
          <Search className="absolute left-2 top-2.5 h-4 w-4 text-muted-foreground" />
          <Input
            placeholder="搜索工具..."
            value={searchQuery}
            onChange={(e) => setSearchQuery(e.target.value)}
            className="pl-8"
          />
        </div>
      </div>

      <Separator />

      {/* 工具分类 */}
      <ScrollArea className="h-[calc(100vh-120px)]">
        <div className="p-4 space-y-4">
          {toolCategories.map((category) => (
            <div key={category.name}>
              {/* 分类标题 */}
              <div className="flex items-center gap-2 mb-2 text-sm font-semibold text-foreground">
                <category.icon className="w-4 h-4" />
                {category.name}
              </div>

              {/* 工具列表 */}
              <div className="space-y-1">
                {category.tools.map((tool) => (
                  <div
                    key={tool.id}
                    draggable
                    onDragStart={(e) => onDragStart(e, tool.id)}
                    className="p-2 rounded-md border border-border hover:bg-accent hover:border-primary cursor-grab active:cursor-grabbing transition-colors"
                  >
                    <div className="font-medium text-sm">{tool.name}</div>
                    <div className="text-xs text-muted-foreground">
                      {tool.description}
                    </div>
                  </div>
                ))}
              </div>
            </div>
          ))}
        </div>
      </ScrollArea>
    </div>
  );
}
```

#### 3. 执行监控面板

```tsx
// ExecutionMonitorPanel.tsx

import { useState, useEffect } from 'react';
import { Card } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { ScrollArea } from '@/components/ui/scroll-area';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { PlayCircle, CheckCircle2, XCircle, Clock } from 'lucide-react';

interface ExecutionLog {
  id: string;
  nodeId: string;
  nodeName: string;
  status: 'running' | 'success' | 'error';
  timestamp: number;
  duration?: number;
  output?: any;
  error?: string;
}

export function ExecutionMonitorPanel({ executionId }: { executionId: string }) {
  const [logs, setLogs] = useState<ExecutionLog[]>([]);

  // WebSocket 连接
  useEffect(() => {
    const ws = new WebSocket(`ws://localhost:8080/executions/${executionId}`);

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);

      if (message.type === 'node:started') {
        setLogs((prev) => [
          ...prev,
          {
            id: message.data.nodeId,
            nodeId: message.data.nodeId,
            nodeName: message.data.nodeName,
            status: 'running',
            timestamp: message.data.timestamp,
          },
        ]);
      } else if (message.type === 'node:completed') {
        setLogs((prev) =>
          prev.map((log) =>
            log.nodeId === message.data.nodeId
              ? {
                  ...log,
                  status: 'success',
                  duration: Date.now() - log.timestamp,
                  output: message.data.output,
                }
              : log
          )
        );
      } else if (message.type === 'node:failed') {
        setLogs((prev) =>
          prev.map((log) =>
            log.nodeId === message.data.nodeId
              ? {
                  ...log,
                  status: 'error',
                  duration: Date.now() - log.timestamp,
                  error: message.data.error,
                }
              : log
          )
        );
      }
    };

    return () => ws.close();
  }, [executionId]);

  const getStatusIcon = (status: string) => {
    switch (status) {
      case 'running':
        return <PlayCircle className="w-4 h-4 text-blue-500 animate-spin" />;
      case 'success':
        return <CheckCircle2 className="w-4 h-4 text-green-500" />;
      case 'error':
        return <XCircle className="w-4 h-4 text-red-500" />;
      default:
        return <Clock className="w-4 h-4 text-gray-500" />;
    }
  };

  return (
    <Card className="h-64">
      <Tabs defaultValue="logs" className="h-full">
        <TabsList className="w-full justify-start rounded-none border-b">
          <TabsTrigger value="logs">执行日志</TabsTrigger>
          <TabsTrigger value="data">数据流</TabsTrigger>
          <TabsTrigger value="timeline">时间轴</TabsTrigger>
        </TabsList>

        <TabsContent value="logs" className="h-[calc(100%-48px)] p-0">
          <ScrollArea className="h-full">
            <div className="p-4 space-y-2">
              {logs.map((log) => (
                <div
                  key={log.id}
                  className="flex items-start gap-3 p-3 rounded-lg border hover:bg-accent transition-colors"
                >
                  {/* 状态图标 */}
                  <div className="mt-0.5">{getStatusIcon(log.status)}</div>

                  {/* 日志内容 */}
                  <div className="flex-1 min-w-0">
                    <div className="flex items-center gap-2 mb-1">
                      <span className="font-medium">{log.nodeName}</span>
                      <Badge
                        variant={
                          log.status === 'success'
                            ? 'default'
                            : log.status === 'error'
                            ? 'destructive'
                            : 'secondary'
                        }
                      >
                        {log.status}
                      </Badge>
                      {log.duration && (
                        <span className="text-xs text-muted-foreground">
                          {log.duration}ms
                        </span>
                      )}
                    </div>

                    {/* 时间戳 */}
                    <div className="text-xs text-muted-foreground">
                      {new Date(log.timestamp).toLocaleTimeString()}
                    </div>

                    {/* 错误信息 */}
                    {log.error && (
                      <div className="mt-2 p-2 bg-destructive/10 rounded text-xs text-destructive">
                        {log.error}
                      </div>
                    )}

                    {/* 输出预览 */}
                    {log.output && (
                      <div className="mt-2 p-2 bg-muted rounded text-xs font-mono">
                        {typeof log.output === 'string'
                          ? log.output.slice(0, 100)
                          : JSON.stringify(log.output, null, 2).slice(0, 100)}
                        ...
                      </div>
                    )}
                  </div>
                </div>
              ))}

              {logs.length === 0 && (
                <div className="text-center text-muted-foreground py-8">
                  暂无执行日志
                </div>
              )}
            </div>
          </ScrollArea>
        </TabsContent>

        <TabsContent value="data" className="h-[calc(100%-48px)]">
          <div className="p-4">数据流可视化...</div>
        </TabsContent>

        <TabsContent value="timeline" className="h-[calc(100%-48px)]">
          <div className="p-4">时间轴视图...</div>
        </TabsContent>
      </Tabs>
    </Card>
  );
}
```

#### 4. 对话助手面板

```tsx
// ConversationAssistantPanel.tsx

import { useState } from 'react';
import { Card } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { ScrollArea } from '@/components/ui/scroll-area';
import { Sparkles, Send } from 'lucide-react';

interface Message {
  role: 'user' | 'assistant';
  content: string;
}

export function ConversationAssistantPanel({
  onWorkflowGenerated,
}: {
  onWorkflowGenerated: (workflow: any) => void;
}) {
  const [messages, setMessages] = useState<Message[]>([
    {
      role: 'assistant',
      content: '你好！我可以帮你生成工作流。请描述你想要实现的功能。',
    },
  ]);
  const [input, setInput] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSend = async () => {
    if (!input.trim()) return;

    // 添加用户消息
    const userMessage: Message = { role: 'user', content: input };
    setMessages((prev) => [...prev, userMessage]);
    setInput('');
    setLoading(true);

    try {
      // 调用 API 生成工作流
      const response = await fetch('/api/conversation/generate-workflow', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message: input, history: messages }),
      });

      const data = await response.json();

      // 添加助手回复
      const assistantMessage: Message = {
        role: 'assistant',
        content: data.message,
      };
      setMessages((prev) => [...prev, assistantMessage]);

      // 如果生成了工作流，通知父组件
      if (data.workflow) {
        onWorkflowGenerated(data.workflow);
      }
    } catch (error) {
      console.error('Failed to generate workflow:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <Card className="flex flex-col h-[400px]">
      {/* 标题 */}
      <div className="flex items-center gap-2 p-4 border-b">
        <Sparkles className="w-5 h-5 text-primary" />
        <span className="font-semibold">AI 工作流助手</span>
      </div>

      {/* 对话历史 */}
      <ScrollArea className="flex-1 p-4">
        <div className="space-y-4">
          {messages.map((message, index) => (
            <div
              key={index}
              className={`flex ${
                message.role === 'user' ? 'justify-end' : 'justify-start'
              }`}
            >
              <div
                className={`max-w-[80%] p-3 rounded-lg ${
                  message.role === 'user'
                    ? 'bg-primary text-primary-foreground'
                    : 'bg-muted'
                }`}
              >
                {message.content}
              </div>
            </div>
          ))}

          {loading && (
            <div className="flex justify-start">
              <div className="max-w-[80%] p-3 rounded-lg bg-muted">
                <div className="flex gap-1">
                  <div className="w-2 h-2 bg-current rounded-full animate-bounce" />
                  <div className="w-2 h-2 bg-current rounded-full animate-bounce [animation-delay:0.2s]" />
                  <div className="w-2 h-2 bg-current rounded-full animate-bounce [animation-delay:0.4s]" />
                </div>
              </div>
            </div>
          )}
        </div>
      </ScrollArea>

      {/* 输入框 */}
      <div className="p-4 border-t">
        <div className="flex gap-2">
          <Textarea
            placeholder="描述你想要的工作流..."
            value={input}
            onChange={(e) => setInput(e.target.value)}
            onKeyDown={(e) => {
              if (e.key === 'Enter' && !e.shiftKey) {
                e.preventDefault();
                handleSend();
              }
            }}
            className="min-h-[60px] resize-none"
          />
          <Button onClick={handleSend} disabled={loading || !input.trim()}>
            <Send className="w-4 h-4" />
          </Button>
        </div>
      </div>
    </Card>
  );
}
```

---

## 💻 完整代码实现

### 1. 工具流编排引擎

```typescript
// packages/toolflow/src/orchestrator.ts

import { ToolRegistry } from '@google/gemini-cli-core';
import type { ToolFlow, ToolNode, Connection, ExecutionContext } from './types';

/**
 * 工具流编排引擎
 * 专注于工具的串联执行和数据流传递
 */
export class ToolFlowOrchestrator {
  constructor(
    private toolRegistry: ToolRegistry,
    private eventBus: EventBus
  ) {}

  /**
   * 执行工具流
   */
  async execute(
    toolFlow: ToolFlow,
    inputs: Record<string, any>,
    signal: AbortSignal
  ): Promise<ExecutionResult> {

    // 1. 创建执行上下文
    const context: ExecutionContext = {
      flowInputs: inputs,
      toolOutputs: new Map(),
      variables: new Map(),
    };

    try {
      // 2. 构建执行图（DAG）
      const graph = this.buildGraph(toolFlow);

      // 3. 拓扑排序
      const executionOrder = this.topologicalSort(graph);

      // 4. 逐层执行工具
      for (const level of executionOrder) {
        await this.executeToolsInParallel(
          level,
          toolFlow,
          context,
          signal
        );
      }

      return {
        status: 'success',
        outputs: Object.fromEntries(context.toolOutputs),
      };

    } catch (error) {
      return {
        status: 'error',
        error: error.message,
        outputs: Object.fromEntries(context.toolOutputs),
      };
    }
  }

  /**
   * 执行单个工具
   */
  private async executeTool(
    toolNode: ToolNode,
    toolFlow: ToolFlow,
    context: ExecutionContext,
    signal: AbortSignal
  ): Promise<void> {

    // 1. 发布开始事件
    this.eventBus.emit('tool:started', {
      toolId: toolNode.id,
      toolName: toolNode.name,
      timestamp: Date.now(),
    });

    const startTime = Date.now();

    try {
      // 2. 解析输入参数（支持模板变量）
      const resolvedInputs = this.resolveInputs(toolNode.inputs, context, toolFlow);

      // 3. 获取工具
      const tool = this.toolRegistry.get(toolNode.type);
      if (!tool) {
        throw new Error(`Tool not found: ${toolNode.type}`);
      }

      // 4. 构建工具调用
      const invocation = tool.build(resolvedInputs);

      // 5. 执行工具
      const result = await invocation.execute(signal);

      // 6. 保存输出到上下文
      context.toolOutputs.set(toolNode.id, result);

      // 7. 发布成功事件
      this.eventBus.emit('tool:completed', {
        toolId: toolNode.id,
        toolName: toolNode.name,
        duration: Date.now() - startTime,
        output: result,
      });

    } catch (error) {
      // 8. 发布失败事件
      this.eventBus.emit('tool:failed', {
        toolId: toolNode.id,
        toolName: toolNode.name,
        duration: Date.now() - startTime,
        error: error.message,
      });

      throw error;
    }
  }

  /**
   * 解析输入参数（支持模板变量）
   * 例如：${read-1.content} → context.toolOutputs.get('read-1').content
   */
  private resolveInputs(
    inputs: Record<string, any>,
    context: ExecutionContext,
    toolFlow: ToolFlow
  ): Record<string, any> {

    const resolved: Record<string, any> = {};

    for (const [key, value] of Object.entries(inputs)) {
      if (typeof value === 'string' && value.startsWith('${') && value.endsWith('}')) {
        // 模板变量
        const path = value.slice(2, -1); // 去掉 ${ 和 }

        if (path.startsWith('flow.input.')) {
          // 工作流输入：${flow.input.xxx}
          const inputKey = path.replace('flow.input.', '');
          resolved[key] = context.flowInputs[inputKey];

        } else {
          // 工具输出：${toolId.outputKey}
          const [toolId, ...outputPath] = path.split('.');
          const toolOutput = context.toolOutputs.get(toolId);

          if (toolOutput) {
            // 支持嵌套路径：${tool1.result.data.value}
            resolved[key] = this.getNestedValue(toolOutput, outputPath);
          }
        }
      } else {
        // 字面值
        resolved[key] = value;
      }
    }

    return resolved;
  }

  /**
   * 获取嵌套对象的值
   */
  private getNestedValue(obj: any, path: string[]): any {
    return path.reduce((current, key) => current?.[key], obj);
  }

  /**
   * 并发执行同级工具
   */
  private async executeToolsInParallel(
    toolIds: string[],
    toolFlow: ToolFlow,
    context: ExecutionContext,
    signal: AbortSignal
  ): Promise<void> {

    const tools = toolIds.map(id =>
      toolFlow.tools.find(t => t.id === id)!
    );

    // 并发执行（受 concurrency 限制）
    const concurrency = toolFlow.config.concurrency || 5;
    await pLimit(concurrency, tools.map(tool =>
      () => this.executeTool(tool, toolFlow, context, signal)
    ));
  }

  /**
   * 构建执行图（DAG）
   */
  private buildGraph(toolFlow: ToolFlow): Map<string, Set<string>> {
    const graph = new Map<string, Set<string>>();

    // 初始化所有工具节点
    for (const tool of toolFlow.tools) {
      graph.set(tool.id, new Set());
    }

    // 添加依赖关系（基于连接）
    for (const conn of toolFlow.connections) {
      graph.get(conn.target)!.add(conn.source);
    }

    return graph;
  }

  /**
   * 拓扑排序（Kahn 算法）
   */
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

    // 检查循环依赖
    if (levels.flat().length !== graph.size) {
      throw new Error('Tool flow contains circular dependencies');
    }

    return levels;
  }
}
```

### 2. 对话生成工具流 API

```typescript
// packages/workflow-api/src/routes/conversation.ts

import { Router } from 'express';
import { AgentExecutor } from '@google/gemini-cli-core';

const router = Router();

/**
 * POST /api/conversation/generate-workflow
 * 使用对话生成工作流
 */
router.post('/generate-workflow', async (req, res) => {
  try {
    const { message, history } = req.body;

    // 1. 创建对话 Agent
    const agentDef = {
      name: 'workflow-generator',
      description: '工具流生成助手',

      promptConfig: {
        systemPrompt: `你是一个工具流设计助手。用户会描述他们想要实现的功能，你需要生成对应的工具流 JSON。

可用的工具类型：
- read_file: 读取文件
- write_file: 写入文件
- bash: 执行 Shell 命令
- grep: 搜索代码
- http_get: HTTP GET 请求
- http_post: HTTP POST 请求
- llm_analyze: AI 分析文本
- llm_generate: AI 生成内容

工具流 JSON 格式：
{
  "id": "unique-id",
  "name": "工作流名称",
  "description": "描述",
  "tools": [
    {
      "id": "tool-1",
      "type": "read_file",
      "name": "读取文件",
      "position": { "x": 100, "y": 100 },
      "inputs": { "path": "file.txt" },
      "outputs": { "content": { "type": "string", "description": "文件内容" } }
    }
  ],
  "connections": [
    {
      "id": "conn-1",
      "source": "tool-1",
      "sourceOutput": "content",
      "target": "tool-2",
      "targetInput": "text"
    }
  ]
}

根据用户的描述，生成合适的工具流。使用 complete_task 工具返回 JSON。`,
      },

      modelConfig: {
        model: 'gemini-2.0-flash-exp',
        temperature: 0.7,
      },

      runConfig: {
        maxTimeMinutes: 5,
        maxTurns: 20,
      },

      toolConfig: {
        tools: ['complete_task'],
      },
    };

    // 2. 运行 Agent
    const executor = await AgentExecutor.create(agentDef, config, runtimeContext);
    const result = await executor.run(
      { query: message },
      new AbortController().signal
    );

    // 3. 解析生成的工作流
    let workflow;
    try {
      workflow = JSON.parse(result.result);
    } catch {
      // 如果不是 JSON，说明还在对话中
      workflow = null;
    }

    res.json({
      message: result.result,
      workflow,
    });

  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

export default router;
```

---

## 🚀 部署方案

### Docker Compose 配置

```yaml
# docker-compose.yml

version: '3.8'

services:
  # 后端 API
  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:pass@db:5432/toolflow
      - GEMINI_API_KEY=${GEMINI_API_KEY}
    depends_on:
      - db
      - redis

  # 前端 UI
  ui:
    build:
      context: ./packages/workflow-ui
      dockerfile: Dockerfile
    ports:
      - "5173:80"
    depends_on:
      - api

  # PostgreSQL 数据库
  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=toolflow
    volumes:
      - postgres_data:/var/lib/postgresql/data

  # Redis（可选，用于队列）
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

---

## 🎯 总结

这个方案强调：

1. **工具流优先** - 工具是第一公民，可视化拖拽连接
2. **数据流自动传递** - 模板变量 `${tool1.output}` 自动解析
3. **对话生成辅助** - AI 助手帮助生成工具流 JSON
4. **现代化 UI** - shadcn/ui + React Flow + Tailwind CSS
5. **实时监控** - WebSocket 实时推送执行状态
6. **易于扩展** - 轻松添加新工具类型

开发工期：**2-3 个月**
技术难度：**中等**
用户体验：**⭐⭐⭐⭐⭐**
