# 前端 UI 项目搭建指南

> 使用 shadcn/ui + React Flow 构建现代化工具流编辑器

## 🚀 快速开始

### 1. 创建 Vite + React + TypeScript 项目

```bash
# 进入 packages 目录
cd packages

# 创建前端项目
npm create vite@latest workflow-ui -- --template react-ts

# 进入项目目录
cd workflow-ui

# 安装依赖
npm install
```

### 2. 安装核心依赖

```bash
# Tailwind CSS
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p

# shadcn/ui 依赖
npm install class-variance-authority clsx tailwind-merge

# React Flow
npm install reactflow

# 图标库
npm install lucide-react

# 状态管理
npm install zustand

# 表单
npm install react-hook-form zod @hookform/resolvers

# 其他 UI 库
npm install framer-motion
npm install react-router-dom
```

### 3. 初始化 shadcn/ui

```bash
# 初始化 shadcn/ui
npx shadcn-ui@latest init

# 选择配置：
# - Style: Default
# - Base color: Slate
# - CSS variables: Yes

# 添加需要的组件
npx shadcn-ui@latest add button
npx shadcn-ui@latest add card
npx shadcn-ui@latest add input
npx shadcn-ui@latest add label
npx shadcn-ui@latest add textarea
npx shadcn-ui@latest add select
npx shadcn-ui@latest add dialog
npx shadcn-ui@latest add dropdown-menu
npx shadcn-ui@latest add tabs
npx shadcn-ui@latest add scroll-area
npx shadcn-ui@latest add separator
npx shadcn-ui@latest add badge
npx shadcn-ui@latest add tooltip
npx shadcn-ui@latest add toast
```

---

## 📁 项目结构

```
packages/workflow-ui/
├── public/
├── src/
│   ├── components/
│   │   ├── ui/                    # shadcn/ui 组件
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   └── ...
│   │   ├── layout/
│   │   │   ├── AppLayout.tsx      # 主布局
│   │   │   ├── Header.tsx         # 顶部导航
│   │   │   └── Sidebar.tsx        # 侧边栏
│   │   ├── workflow/
│   │   │   ├── FlowCanvas.tsx     # React Flow 画布
│   │   │   ├── ToolLibrary.tsx    # 工具库面板
│   │   │   ├── NodeConfig.tsx     # 节点配置面板
│   │   │   └── nodes/             # 自定义节点组件
│   │   │       ├── ToolNode.tsx
│   │   │       ├── StartNode.tsx
│   │   │       └── EndNode.tsx
│   │   ├── execution/
│   │   │   ├── ExecutionMonitor.tsx  # 执行监控
│   │   │   ├── LogPanel.tsx          # 日志面板
│   │   │   └── DataFlowView.tsx      # 数据流视图
│   │   └── conversation/
│   │       └── AssistantPanel.tsx    # 对话助手
│   ├── pages/
│   │   ├── HomePage.tsx           # 首页
│   │   ├── WorkflowListPage.tsx   # 工作流列表
│   │   ├── WorkflowEditorPage.tsx # 工作流编辑器
│   │   └── ExecutionPage.tsx      # 执行详情
│   ├── hooks/
│   │   ├── useWorkflow.ts         # 工作流 Hook
│   │   ├── useExecution.ts        # 执行 Hook
│   │   └── useWebSocket.ts        # WebSocket Hook
│   ├── stores/
│   │   ├── workflowStore.ts       # 工作流状态
│   │   └── executionStore.ts      # 执行状态
│   ├── api/
│   │   └── client.ts              # API 客户端
│   ├── types/
│   │   └── index.ts               # 类型定义
│   ├── lib/
│   │   └── utils.ts               # 工具函数
│   ├── App.tsx
│   ├── main.tsx
│   └── index.css
├── index.html
├── package.json
├── tsconfig.json
├── tailwind.config.js
├── postcss.config.js
├── vite.config.ts
└── components.json              # shadcn/ui 配置
```

---

## ⚙️ 配置文件

### tailwind.config.js

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  darkMode: ['class'],
  content: [
    './pages/**/*.{ts,tsx}',
    './components/**/*.{ts,tsx}',
    './app/**/*.{ts,tsx}',
    './src/**/*.{ts,tsx}',
  ],
  theme: {
    container: {
      center: true,
      padding: '2rem',
      screens: {
        '2xl': '1400px',
      },
    },
    extend: {
      colors: {
        border: 'hsl(var(--border))',
        input: 'hsl(var(--input))',
        ring: 'hsl(var(--ring))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
        },
        secondary: {
          DEFAULT: 'hsl(var(--secondary))',
          foreground: 'hsl(var(--secondary-foreground))',
        },
        destructive: {
          DEFAULT: 'hsl(var(--destructive))',
          foreground: 'hsl(var(--destructive-foreground))',
        },
        muted: {
          DEFAULT: 'hsl(var(--muted))',
          foreground: 'hsl(var(--muted-foreground))',
        },
        accent: {
          DEFAULT: 'hsl(var(--accent))',
          foreground: 'hsl(var(--accent-foreground))',
        },
        popover: {
          DEFAULT: 'hsl(var(--popover))',
          foreground: 'hsl(var(--popover-foreground))',
        },
        card: {
          DEFAULT: 'hsl(var(--card))',
          foreground: 'hsl(var(--card-foreground))',
        },
      },
      borderRadius: {
        lg: 'var(--radius)',
        md: 'calc(var(--radius) - 2px)',
        sm: 'calc(var(--radius) - 4px)',
      },
      keyframes: {
        'accordion-down': {
          from: { height: 0 },
          to: { height: 'var(--radix-accordion-content-height)' },
        },
        'accordion-up': {
          from: { height: 'var(--radix-accordion-content-height)' },
          to: { height: 0 },
        },
      },
      animation: {
        'accordion-down': 'accordion-down 0.2s ease-out',
        'accordion-up': 'accordion-up 0.2s ease-out',
      },
    },
  },
  plugins: [require('tailwindcss-animate')],
}
```

### vite.config.ts

```typescript
import path from 'path'
import react from '@vitejs/plugin-react'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:3000',
        changeOrigin: true,
      },
    },
  },
})
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,

    /* Path alias */
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

---

## 🎨 核心组件实现

### 1. 主布局

```tsx
// src/components/layout/AppLayout.tsx

import { Outlet } from 'react-router-dom';
import { Header } from './Header';

export function AppLayout() {
  return (
    <div className="h-screen flex flex-col">
      <Header />
      <main className="flex-1 overflow-hidden">
        <Outlet />
      </main>
    </div>
  );
}
```

### 2. 工作流编辑器页面

```tsx
// src/pages/WorkflowEditorPage.tsx

import { useState, useCallback } from 'react';
import ReactFlow, {
  Background,
  Controls,
  MiniMap,
  addEdge,
  useNodesState,
  useEdgesState,
  type Connection,
  type Node,
  type Edge,
} from 'reactflow';
import 'reactflow/dist/style.css';

import { ToolLibrary } from '@/components/workflow/ToolLibrary';
import { NodeConfig } from '@/components/workflow/NodeConfig';
import { ExecutionMonitor } from '@/components/execution/ExecutionMonitor';
import { AssistantPanel } from '@/components/conversation/AssistantPanel';
import { Button } from '@/components/ui/button';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { Play, Save } from 'lucide-react';

import { ToolNode } from '@/components/workflow/nodes/ToolNode';

const nodeTypes = {
  tool: ToolNode,
};

export function WorkflowEditorPage() {
  const [nodes, setNodes, onNodesChange] = useNodesState([]);
  const [edges, setEdges, onEdgesChange] = useEdgesState([]);
  const [selectedNode, setSelectedNode] = useState<Node | null>(null);
  const [executionId, setExecutionId] = useState<string | null>(null);

  const onConnect = useCallback(
    (connection: Connection) => {
      setEdges((eds) => addEdge(connection, eds));
    },
    [setEdges]
  );

  const onDrop = useCallback(
    (event: React.DragEvent) => {
      event.preventDefault();

      const toolType = event.dataTransfer.getData('application/reactflow');
      const position = {
        x: event.clientX - 250,
        y: event.clientY - 100,
      };

      const newNode: Node = {
        id: `${toolType}-${Date.now()}`,
        type: 'tool',
        position,
        data: {
          type: toolType,
          name: toolType,
          status: 'idle',
        },
      };

      setNodes((nds) => [...nds, newNode]);
    },
    [setNodes]
  );

  const onNodeClick = useCallback((event: React.MouseEvent, node: Node) => {
    setSelectedNode(node);
  }, []);

  const handleRun = async () => {
    // TODO: 调用 API 执行工作流
    console.log('Running workflow...', { nodes, edges });
  };

  const handleSave = async () => {
    // TODO: 调用 API 保存工作流
    console.log('Saving workflow...', { nodes, edges });
  };

  return (
    <div className="h-full flex">
      {/* 左侧工具库 */}
      <ToolLibrary />

      {/* 中间画布 */}
      <div className="flex-1 flex flex-col">
        {/* 顶部工具栏 */}
        <div className="h-14 border-b flex items-center justify-between px-4">
          <h2 className="text-lg font-semibold">工作流编辑器</h2>
          <div className="flex gap-2">
            <Button variant="outline" onClick={handleSave}>
              <Save className="w-4 h-4 mr-2" />
              保存
            </Button>
            <Button onClick={handleRun}>
              <Play className="w-4 h-4 mr-2" />
              运行
            </Button>
          </div>
        </div>

        {/* React Flow 画布 */}
        <div
          className="flex-1"
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
      </div>

      {/* 右侧面板 */}
      <div className="w-96 border-l">
        <Tabs defaultValue="config" className="h-full">
          <TabsList className="w-full justify-start rounded-none border-b">
            <TabsTrigger value="config">配置</TabsTrigger>
            <TabsTrigger value="execution">执行</TabsTrigger>
            <TabsTrigger value="assistant">助手</TabsTrigger>
          </TabsList>

          <TabsContent value="config" className="h-[calc(100%-48px)] p-4">
            {selectedNode ? (
              <NodeConfig
                node={selectedNode}
                onUpdate={(updatedNode) => {
                  setNodes((nds) =>
                    nds.map((n) => (n.id === updatedNode.id ? updatedNode : n))
                  );
                }}
                onClose={() => setSelectedNode(null)}
              />
            ) : (
              <div className="text-center text-muted-foreground py-8">
                选择一个节点查看配置
              </div>
            )}
          </TabsContent>

          <TabsContent value="execution" className="h-[calc(100%-48px)] p-4">
            {executionId ? (
              <ExecutionMonitor executionId={executionId} />
            ) : (
              <div className="text-center text-muted-foreground py-8">
                运行工作流后查看执行状态
              </div>
            )}
          </TabsContent>

          <TabsContent value="assistant" className="h-[calc(100%-48px)] p-4">
            <AssistantPanel
              onWorkflowGenerated={(workflow) => {
                setNodes(workflow.nodes);
                setEdges(workflow.edges);
              }}
            />
          </TabsContent>
        </Tabs>
      </div>
    </div>
  );
}
```

### 3. Zustand Store

```typescript
// src/stores/workflowStore.ts

import { create } from 'zustand';
import type { Node, Edge } from 'reactflow';

interface WorkflowState {
  nodes: Node[];
  edges: Edge[];
  selectedNode: Node | null;

  setNodes: (nodes: Node[]) => void;
  setEdges: (edges: Edge[]) => void;
  setSelectedNode: (node: Node | null) => void;

  addNode: (node: Node) => void;
  updateNode: (id: string, data: any) => void;
  removeNode: (id: string) => void;

  addEdge: (edge: Edge) => void;
  removeEdge: (id: string) => void;
}

export const useWorkflowStore = create<WorkflowState>((set) => ({
  nodes: [],
  edges: [],
  selectedNode: null,

  setNodes: (nodes) => set({ nodes }),
  setEdges: (edges) => set({ edges }),
  setSelectedNode: (selectedNode) => set({ selectedNode }),

  addNode: (node) => set((state) => ({ nodes: [...state.nodes, node] })),

  updateNode: (id, data) =>
    set((state) => ({
      nodes: state.nodes.map((node) =>
        node.id === id ? { ...node, data: { ...node.data, ...data } } : node
      ),
    })),

  removeNode: (id) =>
    set((state) => ({
      nodes: state.nodes.filter((node) => node.id !== id),
      edges: state.edges.filter((edge) => edge.source !== id && edge.target !== id),
    })),

  addEdge: (edge) => set((state) => ({ edges: [...state.edges, edge] })),

  removeEdge: (id) =>
    set((state) => ({
      edges: state.edges.filter((edge) => edge.id !== id),
    })),
}));
```

### 4. API 客户端

```typescript
// src/api/client.ts

const API_BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:3000/api';

export class APIClient {
  // 工作流 API
  async listWorkflows() {
    const response = await fetch(`${API_BASE_URL}/workflows`);
    return response.json();
  }

  async getWorkflow(id: string) {
    const response = await fetch(`${API_BASE_URL}/workflows/${id}`);
    return response.json();
  }

  async createWorkflow(workflow: any) {
    const response = await fetch(`${API_BASE_URL}/workflows`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(workflow),
    });
    return response.json();
  }

  async updateWorkflow(id: string, workflow: any) {
    const response = await fetch(`${API_BASE_URL}/workflows/${id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(workflow),
    });
    return response.json();
  }

  async executeWorkflow(id: string, inputs: any) {
    const response = await fetch(`${API_BASE_URL}/workflows/${id}/execute`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ inputs }),
    });
    return response.json();
  }

  // 执行 API
  async getExecution(id: string) {
    const response = await fetch(`${API_BASE_URL}/executions/${id}`);
    return response.json();
  }

  // 对话 API
  async generateWorkflowFromConversation(message: string, history: any[]) {
    const response = await fetch(`${API_BASE_URL}/conversation/generate-workflow`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ message, history }),
    });
    return response.json();
  }
}

export const apiClient = new APIClient();
```

---

## 🚀 运行项目

```bash
# 开发模式
npm run dev

# 构建
npm run build

# 预览构建产物
npm run preview
```

访问：`http://localhost:5173`

---

## 🎨 主题切换

```tsx
// src/components/ThemeToggle.tsx

import { Moon, Sun } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { useTheme } from '@/hooks/useTheme';

export function ThemeToggle() {
  const { theme, setTheme } = useTheme();

  return (
    <Button
      variant="ghost"
      size="icon"
      onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}
    >
      <Sun className="h-5 w-5 rotate-0 scale-100 transition-all dark:-rotate-90 dark:scale-0" />
      <Moon className="absolute h-5 w-5 rotate-90 scale-0 transition-all dark:rotate-0 dark:scale-100" />
      <span className="sr-only">切换主题</span>
    </Button>
  );
}
```

---

## 📦 打包部署

### Dockerfile

```dockerfile
# Dockerfile

FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### nginx.conf

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://api:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## ✨ 总结

这个前端项目使用了：

- ✅ **Vite** - 快速构建工具
- ✅ **React 18** - 最新版本
- ✅ **TypeScript** - 类型安全
- ✅ **shadcn/ui** - 现代组件库
- ✅ **React Flow** - 工作流可视化
- ✅ **Tailwind CSS** - 原子化 CSS
- ✅ **Zustand** - 轻量状态管理

开始开发：
```bash
cd packages/workflow-ui
npm install
npm run dev
```

访问 `http://localhost:5173` 开始体验！🚀
