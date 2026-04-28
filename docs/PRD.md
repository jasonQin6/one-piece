# 企业级 Agent 协作系统 PRD

## 一、产品概述

构建一个 Human-in-the-Loop 的企业级 Agent 协作系统，核心理念是：**让 Agent 在合适的时机找到正确的员工验证结果，避免错误在任务流中扩散。**

---

## 二、核心痛点

### 2.1 "传话游戏"问题

当前多 Agent 协作模式中，任务在多个 Agent 之间流转时，信息会逐步失真：

- **上下文丢失**：每个 Agent 只接收到上一节点的输出摘要，缺乏完整上下文
- **理解偏差累积**：每个节点的解读偏差在流转中被放大
- **责任边界模糊**：多个 Agent 参与同一任务链时，出问题难以追溯

### 2.2 错误扩散

- 前置节点的错误被后续节点默认接受并放大
- 缺乏有效的验证点来阻断错误传播
- 错误在任务流末端才被发现，修复成本极高

### 2.3 协作效率低下

- 任务拆分后跨 Agent 流转需要大量协调
- 员工无法追踪任务在哪一环、谁在处理
- 重复性工作模式无法沉淀为可复用流程

---

## 三、核心解决方案

### 3.1 原则：同一上下文 → 同一个 Agent

**核心策略：不拆分上下文，而是委托 sub-agent 处理。**

```
错误模式（传话游戏）：
Agent-A → 输出摘要 → Agent-B → 输出摘要 → Agent-C
（上下文逐层丢失，偏差累积）

正确模式（委托 sub-agent）：
Agent-A (主 Agent，持有完整上下文)
├── Sub-Agent-A1 (处理子任务1)
├── Sub-Agent-A2 (处理子任务2)
└── Sub-Agent-A3 (处理子任务3)
（上下文保持一致，由主 Agent 整合）
```

**设计原则：**
- 主 Agent 持有任务完整上下文，负责统筹协调
- sub-agent 仅处理原子化子任务，完成后返回结果给主 Agent
- 主 Agent 在关键节点请求人类验证，而非每个 sub-agent 都找人类

### 3.2 Human-in-the-Loop：智能路由验证

**核心策略：Agent 在合适的时机找到正确的员工验证，而非固定绑定。**

```
任务执行 → 到达验证点 → 系统推荐验证者 → 员工审核 → 继续/退回
                                  ↓
                      基于：任务类型 + 历史数据 + 角色能力
```

**验证点设计：**
- **前置验证**：任务开始前，确认执行方向正确
- **中间验证**：关键决策点或高风险操作前请求确认
- **最终验证**：输出交付前的人工审核

**路由策略：**
- 基于任务类型自动匹配最适合的验证者
- 考虑员工的专业领域、当前负载、历史反馈质量
- 记录每次路由决策，持续优化推荐算法

### 3.3 错误阻断机制

- 每个验证点都是潜在的"断路器"，员工可选择：
  - **通过**：继续流转
  - **退回**：要求修改，错误不会扩散到下游
  - **转派**：发现验证者不合适，重新路由

- 关键操作前强制验证：
  - 文件修改（尤其生产环境）
  - 命令执行
  - 外部系统调用
  - 数据删除

### 3.4 多 Agent 协作模式

**模式一：主从架构（推荐）**
```
Lead (主 Agent) 持有完整上下文
├── Sub-Agent-1: 前端开发
├── Sub-Agent-2: 后端开发
├── Sub-Agent-3: 测试
└── 人类验证者: 各关键节点审核
（Lead 整合结果，统一向人类汇报）
```

**模式二：并行验证（竞争假设）**
```
主 Agent 分发调查任务
── Sub-Agent-1: 调查数据库性能
├── Sub-Agent-2: 检查认证中间件
├── Sub-Agent-3: 分析网络延迟
└── 人类验证者: 综合发现，决策修复方案
（各 sub-agent 独立验证，结果汇总后人工决策）
```

**模式三：多维度审核**
```
主 Agent 完成初稿
├── Sub-Agent-1: 安全审核
├── Sub-Agent-2: 性能审核
├── Sub-Agent-3: 架构审核
└── 人类验证者: 汇总报告，优先级排序
（多维度并行审核，避免单一视角盲区）
```

---

## 四、三层 Skill 结构

> 核心原则：**层级越高，Agent 判断空间越大，同时也越难确定**

### 4.1 结构总览

```
atoms 原子层（砖块） → molecules 分子层（工件） → compounds 复合层（工程）
```

- **atoms 原子层**：单一目的可执行块，高确定性、无调度
- **molecules 分子层**：2-10 个 atoms 链式拼接，完成有界任务
- **compounds 复合层**：多 molecules 协作，真正交出判断，需要人在驾驶座

### 4.2 层级对比表

| 层级 | 对应 Skill 类型 | Agent 判断空间特征 |
|------|-----------------|-------------------|
| atoms 原子层 | 单一目的 | 几乎确定性，不调别的 Skill |
| molecules 分子层 | 链式 / Orchestrator | 有判断，但被显式约束 |
| compounds 复合层 | 高层编排 | 真正交出判断，需要人在驾驶座 |

### 4.3 各层详解

#### atoms 原子层（砖块）
- **定位**：最小可执行块
- **特征**：范围窄，越接近确定性越好
- **作用**：提供稳定、可靠的基础执行单元，是上层的"砖块"
- **示例**：读取文件、执行命令、调用 API、格式转换

#### molecules 分子层（工件）
- **定位**：2-10 个 atoms 拼出的有界任务
- **特征**：把组合逻辑写在 Skill 里，不留给 Agent 现场决策
- **作用**：将原子层的能力组合成标准化的"工件"，减少上层的不确定性
- **示例**：代码审查（读取代码→静态分析→生成报告）、安全扫描、测试执行

#### compounds 复合层（工程）
- **定位**：多个 molecules 协作完成的工作流
- **特征**：判断和不确定都集中在这里，需要人工介入或强编排
- **作用**：处理复杂、不确定的业务流程，是 Agent 真正发挥决策能力的层级
- **示例**：功能开发（需求分析→代码编写→测试→部署）、事故排查

### 4.4 核心设计原则

> **原子要稳 · 分子要可靠 · 复合层才能放心交出判断**

- 原子层：追求**高确定性、低抽象**，避免底层执行出错
- 分子层：追求**标准化、强约束**，将原子组合为可复用的任务单元
- 复合层：聚焦**决策编排、人工兜底**，让 Agent 在复杂场景中发挥价值

### 4.5 Skill 与验证点映射

| Skill 层级 | 验证点密度 | 验证策略 |
|-----------|-----------|---------|
| atoms 原子层 | 无验证点 | 执行结果自动返回，由上层判断 |
| molecules 分子层 | 低密度验证点 | 完成后请求人工确认（可选） |
| compounds 复合层 | 高密度验证点 | 关键节点强制人工验证（Human-in-the-Loop） |

---

## 五、核心数据模型

### 5.1 Agent 与 Sub-Agent

```typescript
interface Agent {
  id: string;
  name: string;
  runtime: string; // claude, hermes, codex, etc.
  departmentId: string | null;
  status: 'online' | 'offline' | 'busy';
  
  // 上下文持有情况
  contextSnapshot: ContextSnapshot;
  
  // 委托关系
  subAgents: SubAgent[];
  
  createdAt: Date;
}

interface SubAgent {
  id: string;
  parentAgentId: string;
  name: string;
  task: string; // 子任务描述
  status: 'pending' | 'in_progress' | 'completed' | 'failed';
  
  // 只返回结果，不持有完整上下文
  result: TaskResult | null;
  
  createdAt: Date;
}
```

### 5.2 任务与验证点

```typescript
interface Task {
  id: string;
  title: string;
  description: string;
  status: TaskStatus;
  
  // 当前持有上下文的 Agent
  agentId: string;
  
  // 验证点列表
  verificationPoints: VerificationPoint[];
  
  // 审批记录（复用 Agent 内置能力，只记录结果）
  approvals: Approval[];
  
  // 推送决策（人工路由记录）
  routingDecisions: RoutingDecision[];
  
  // 使用的 Skill（复合层 Skill）
  skillId: string | null;
  skillLevel: 'atom' | 'molecule' | 'compound' | null;
  
  output: Json | null;
  feedback: string | null;
  
  createdAt: Date;
  completedAt: Date | null;
}

interface VerificationPoint {
  id: string;
  taskId: string;
  type: 'pre' | 'intermediate' | 'post'; // 前置/中间/最终
  
  // 当前请求
  request: {
    description: string;
    riskLevel: 'high' | 'medium' | 'low';
    context: string; // 验证所需的上下文摘要
  };
  
  // 路由
  recommendedVerifiers: VerifierCandidate[];
  selectedVerifierId: string | null;
  
  // 结果
  status: 'pending' | 'approved' | 'rejected' | 'rerouted';
  feedback: string | null;
  
  createdAt: Date;
  resolvedAt: Date | null;
}

interface VerifierCandidate {
  userId: string;
  userName: string;
  expertise: string[];
  confidence: number; // 0-1，匹配置信度
  reason: string;
}
```

### 5.3 Skill 三层结构

```typescript
// Skill 基类
interface Skill {
  id: string;
  name: string;
  level: 'atom' | 'molecule' | 'compound'; // 原子/分子/复合层
  departmentId: string | null;
  
  description: string;
  
  // 层级特定字段
  atomConfig?: AtomConfig;
  moleculeConfig?: MoleculeConfig;
  compoundConfig?: CompoundConfig;
  
  // 使用统计
  usageCount: number;
  successRate: number;
  
  // 反思来源（由历史任务总结）
  sourceTasks: string[];
  
  createdAt: Date;
  updatedAt: Date;
}

// 原子层配置
interface AtomConfig {
  // 原子 Skill 不可调用其他 Skill
  allowCallOtherSkills: false;
  
  // 执行动作
  action: {
    type: string; // read_file, exec_command, call_api, etc.
    parameters: Record<string, any>;
  };
  
  // 输出格式（强约束）
  outputSchema: JsonSchema;
}

// 分子层配置
interface MoleculeConfig {
  // 链式调用 2-10 个 atoms
  atomChain: {
    skillId: string;
    order: number;
    condition?: string; // 可选执行条件
  }[];
  
  // 组合逻辑写在 Skill 里，不给 Agent 现场决策
  orchestration: 'chain' | 'parallel' | 'conditional';
  
  // 最大原子数约束
  maxAtoms: 10;
}

// 复合层配置
interface CompoundConfig {
  // 调用多个 molecules
  moleculeChain: {
    skillId: string;
    order: number;
    verificationPoints: VerificationPoint[]; // 验证点集中在这里
  }[];
  
  // 真正的判断和不确定在这里
  requiresHumanInLoop: boolean;
  
  // 可选的编排逻辑
  orchestration?: Json;
}
```

### 5.4 路由决策

```typescript
interface RoutingDecision {
  id: string;
  taskId: string;
  
  // 路由来源
  fromAgentId: string;
  fromType: 'human' | 'system' | 'agent';
  
  // 路由目标
  toAgentId: string | null; // null 表示任务完成
  toType: 'agent' | 'sub-agent' | 'verifier' | 'completed';
  
  // 决策信息
  reason: string | null;
  decisionMakerId: string; // 做决策的人/系统
  
  // 决策质量（用于优化推荐）
  wasSuccessful: boolean | null;
  
  createdAt: Date;
}
```

---

## 六、核心流程

### 6.1 任务执行流程（Human-in-the-Loop）

```
1. 任务创建 → 分配给主 Agent
2. 主 Agent 分析任务，识别验证点
   - 前置验证：确认执行方向
   - 中间验证：关键决策点
   - 最终验证：输出审核
3. 主 Agent 委托 sub-agent 处理子任务
   - sub-agent 只处理原子任务，返回结果
   - 主 Agent 保持完整上下文，整合 sub-agent 结果
4. 到达验证点 → 系统推荐验证者 → 员工审核
   - 通过：继续执行
   - 退回：要求修改，错误不扩散
   - 转派：重新路由
5. 所有验证点通过 → 任务完成
```

### 6.2 路由优化流程

```
1. 记录每次路由决策（谁选的、选的谁、结果如何）
2. 反思引擎定期分析：
   - 某类任务最常路由给谁？
   - 某员工在哪类任务上反馈质量最高？
   - 哪些路由决策导致了问题？
3. 生成路由优化建议
4. 新任务自动应用优化后的路由策略
```

### 6.3 Skill 沉淀与发现流程

```
1. 历史任务积累
2. 反思引擎定期分析
   - 识别重复的 atoms/molecules 调用模式
   - 计算成功率和平均耗时
   - 提取高价值的 Skill 组合
3. 生成 Skill 候选
   - 原子层：发现可复用的基础操作
   - 分子层：将频繁出现的 atom 链封装为 molecule
   - 复合层：将 molecule 序列提炼为 compound
4. 员工审核后启用
5. 新任务匹配时推荐对应层级的 Skill
```

---

## 七、技术架构

### 7.1 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│                    云端服务 (Next.js + Go)                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ 任务管理  │  │ 路由引擎  │  │ Skill    │  │  反思引擎     │   │
│  │          │  │          │  │ 管理层    │  │              │   │
│  └──────────┘  ──────────┘  └──────────┘  └──────────────┘   │
────────────────────────┬─────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────────┐
│                     Agent 运行时层                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ Approval │  │ Agent    │  │ Sub-     │  │  实时通信     │   │
│  │ Gateway  │  │ Provider │  │ Agent    │  │ (WebSocket)  │   │
│  ──────────┘  └──────────┘  │ Manager  │  └──────────────┘   │
│                               └──────────┘                    │
└──────────────────────────────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────────┐
│                        数据层                                  │
│  ┌──────────┐  ┌──────────┐  ──────────┐                    │
│  │PostgreSQL│  │ Redis    │  │ 对象存储  │                    │
│  │          │  │(缓存/队列)│  │          │                    │
│  └──────────┘  └──────────┘  ──────────┘                    │
└──────────────────────────────────────────────────────────────┘
```

### 7.2 技术选型

| 层级 | 技术 | 说明 |
|------|------|------|
| 业务服务 | Next.js (App Router) + TypeScript | 任务管理、审核、Skill 管理 |
| 核心服务 | Go | Agent 调度、Approval Gateway |
| 数据库 | PostgreSQL | 关系型数据存储 |
| 缓存/队列 | Redis | 缓存、会话、任务队列 |
| 实时通信 | WebSocket | 任务状态同步、审批通知 |
| Agent 运行时 | 可插拔 Provider | Claude Code、Hermes、Codex 等 |

### 7.3 关键文件结构

```
agent-collab-system/
├── server/                     # 核心服务 (Go)
│   ├── internal/
│   │   ├── approval/           # 审批网关
│   │   ├── routing/            # 路由引擎
│   │   ├── skill/              # Skill 三层管理
│   │   ├── reflection/         # Skill 沉淀与发现引擎
│   └── pkg/
│       ├── db/                 # 数据库模型
│       └── realtime/           # WebSocket
── web/                        # 扩展服务 (Next.js)
│   ├── src/
│   │   ├── app/                # App Router
│   │   ├── components/         # React 组件
│   │   └── services/           # 业务逻辑
└── prisma/
    └── schema.prisma           # 数据模型
```

---

## 八、API 设计

### 8.1 核心 REST API

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/tasks` | GET | 获取任务列表 |
| `/api/tasks` | POST | 创建任务 |
| `/api/tasks/:id` | GET | 获取任务详情 |
| `/api/tasks/:id/verify` | POST | 提交验证结果 |
| `/api/tasks/:id/route` | POST | 路由到下一节点 |
| `/api/tasks/:id/recommend-verifier` | POST | 推荐验证者 |
| `/api/agents` | GET | 获取 Agent 列表 |
| `/api/agents/:id/subagents` | GET | 获取 sub-agent 列表 |
| `/api/skills` | GET | 获取 Skill 列表 |
| `/api/skills` | POST | 创建 Skill |
| `/api/skills/:id` | GET | 获取 Skill 详情 |
| `/api/skills/:id/level` | GET | 获取 Skill 层级详情 |
| `/api/reflection/analyze` | POST | 触发 Skill 沉淀分析 |

### 8.2 WebSocket 事件

| 事件 | 方向 | 说明 |
|------|------|------|
| `task:created` | server → client | 任务创建 |
| `task:verification_requested` | server → client | 验证请求 |
| `task:verification_resolved` | server → client | 验证完成 |
| `task:routing_recommended` | server → client | 路由推荐 |
| `subagent:completed` | server → client | sub-agent 完成 |
| `agent:status` | server ↔ client | Agent 状态同步 |

---

## 九、安全与审计

### 9.1 安全机制

- **验证点强制拦截**：关键操作前必须经过人工验证
- **权限隔离**：部门级别数据隔离，员工只看到相关任务
- **操作审计**：完整的验证、路由、审批日志
- **API 限流**：防止滥用

### 9.2 审计日志

所有以下操作均记录审计日志：
- 任务创建/修改/完成
- 验证请求与结果
- 路由决策（谁选的、选的谁、为什么）
- sub-agent 委托与结果
- Skill 启用/修改

---

## 十、MVP 功能清单

### Phase 1: 基础能力
1. 任务创建与 Agent 分配
2. 主 Agent 委托 sub-agent 执行子任务
3. 验证点机制：前置/中间/最终验证
4. 员工验证：通过/退回/转派
5. 基础路由：手动选择验证者

### Phase 2: 智能路由
6. 验证者推荐引擎（基于任务类型匹配）
7. 路由决策记录与反馈
8. 验证历史与质量追踪
9. 审计日志完整记录

### Phase 3: Skill 三层体系
10. 反思引擎：分析历史任务，识别 atoms/molecules 模式
11. atoms 沉淀：高频基础操作封装为原子 Skill
12. molecules 沉淀：atom 链封装为分子 Skill
13. compounds 沉淀：molecule 序列提炼为复合 Skill（含验证点）
14. Skill 推荐与半自动执行
15. 路由策略自动优化
