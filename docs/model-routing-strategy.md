# 多 Provider/模型 任务分配策略

## 一、策略目标

在多个 LLM Provider 和模型之间智能路由任务，平衡以下维度：

- **任务复杂度** → 匹配合适参数规模的模型
- **上下文长度** → 选择足够 context window 的模型
- **限流配额** → 优先消耗宽松配额，保护稀缺配额
- **成本** → 简单任务走低成本/免费通道
- **推理模式** → 需要深度思考的任务启用 thinking/budgetTokens

---

## 二、模型能力矩阵

### 2.1 模型规格表

| Provider | 模型 | 参数量 | 上下文 | Thinking | 多模态 | 限流/成本 |
|----------|------|--------|--------|----------|--------|-----------|
| 本地 omlx | Copaw-9B | 9B | 取决于部署 | 否 | 否 | 无限制/免费 |
| 星辰 | astron-code-latest (Qwen3.5-35B-A3B) | 35B(A3B) | 128K | 否 | ✅ | 未知 |
| 小米MiMo | mimo-v2-flash | 轻量级 | ≤256K | 否 | 否 | ¥0.07/M input(命中) |
| 小米MiMo | mimo-v2.5 | 中量级 | ≤1M | 否 | 否 | ¥0.56/M input(命中) |
| 小米MiMo | mimo-v2-pro / v2.5-pro | 重量级 | ≤1M | 否 | 否 | ¥1.40/M input(命中) |
| 阿里云 | qwen3.6-plus | 未知(大) | 未知 | ✅ budget:1024 | 否 | 配额制 |
| 阿里云 | MiniMax-M2.5 | 未知 | 未知 | 否 | 否 | 配额制 |
| 阿里云 | glm-5 | 未知(大) | 未知 | ✅ budget:1024 | 否 | 配额制 |
| 阿里云 | glm-4.7 | 未知 | 未知 | ✅ budget:1024 | 否 | 配额制 |
| 阿里云 | kimi-k2.5 | 未知 | 未知 | ✅ budget:1024 | 否 | 配额制 |
| GLM | GLM-5.1 | 未知(大) | 未知 | 否 | 否 | ~80次/5h |
| GLM | GLM-5-Turbo | 未知 | 未知 | 否 | 否 | ~80次/5h |
| GLM | GLM-4.7 | 未知 | 未知 | 否 | 否 | ~80次/5h |
| GLM | GLM-4.5-Air | 轻量级 | 未知 | 否 | 否 | ~80次/5h |

### 2.2 参数规模与适用场景

| 参数量级 | 代表模型 | 适用场景 | 不适用场景 |
|----------|----------|----------|-----------|
| 9B | Copaw-9B | 代码格式化、语法检查、简单重构、正则生成、JSON/YAML 转换 | 架构设计、复杂推理、长上下文分析 |
| 35B-A3B (MoE) | astron-code-latest | 中等复杂度代码生成、Bug 修复、代码审查、单元测试编写 | 超长文档分析(>128K)、多模态理解 |
| 220B-A10B (MoE) | 大型模型(待确认provider) | 复杂架构设计、系统设计、深度代码审查、跨文件重构 | 高频小额请求(浪费配额) |
| 1T-A42B (MoE) | 超大模型(待确认provider) | 战略规划、复杂推理、多领域知识整合、疑难问题诊断 | 日常编码(过度杀伤) |

---

## 三、配额管理系统

### 3.1 配额分级

将 Provider 按配额充裕度分为三级：

```
Tier 1 — 无限/按量付费（优先用于高频小额任务）
├── 本地 omlx (Copaw-9B): 无限制
├── 小米MiMo: 按量付费，无硬限流
└── 星辰: 限流规则未知，暂按 Tier 1 处理

Tier 2 — 宽松配额（用于中等复杂度任务）
└── 阿里云百炼: 6000次/5h, 45000次/周, 90000次/月

Tier 3 — 严格配额（仅用于高质量需求场景）
└── GLM: ~80次/5h, ~400次/周
```

### 3.2 配额消耗预算

按月 90,000 次（阿里云）和 400 次（GLM）计算：

| Provider | 月配额 | 日均预算 | 5h 预算 | 用途建议 |
|----------|--------|----------|---------|---------|
| 阿里云 | 90,000 | 3,000 | 6,000 | 日常编码主力 |
| GLM | 400 | ~13 | ~80 | 仅高质量需求 |

**关键规则：**
- 阿里云 5h 限额 6,000 次 = 月限额 90,000 / (30天 × 4.8个5h窗口)
- 实际瓶颈在 5h 窗口，不是月度总量
- GLM 的 5h 限额和周限额一致（80×5 ≈ 400），说明是均匀分布的

---

## 四、路由决策引擎

### 4.1 决策流程图

```
任务进入
  │
  ├─ 是否需要多模态？
  │   ├─ 是 → 星辰 astron-code-latest (唯一多模态)
  │   └─ 否 ↓
  │
  ├─ 是否需要 thinking/reasoning？
  │   ├─ 是 → 阿里云 (qwen3.6-plus / glm-5 / kimi-k2.5, budgetTokens:1024)
  │   └─ 否 ↓
  │
  ├─ 上下文是否 > 128K？
  │   ├─ 是 → 小米MiMo (支持到 1M) 或 阿里云(若未知但可能支持)
  │   └─ 否 ↓
  │
  ├─ 任务复杂度评估
  │   │
  │   ├─ 简单 (格式化/语法/转换/简单补全)
  │   │   → 本地 Copaw-9B (免费/无限制)
  │   │   → fallback: 小米 mimo-v2-flash
  │   │
  │   ├─ 中等 (函数级代码生成/修复/审查/单元测试)
  │   │   → 星辰 astron-code-latest (35B-A3B, 128K)
  │   │   → fallback: 阿里云 qwen3.6-plus
  │   │   → fallback: 小米 mimo-v2.5
  │   │
  │   ├─ 复杂 (模块级/架构设计/跨文件重构/系统设计)
  │   │   → 阿里云 qwen3.6-plus 或 kimi-k2.5
  │   │   → fallback: 小米 mimo-v2-pro
  │   │   → fallback: GLM GLM-5.1 (配额紧张时跳过)
  │   │
  │   └─ 极复杂 (战略规划/疑难诊断/多领域整合)
  │       → 最大模型 (1T-A42B / 220B-A10B, 待确认 provider)
  │       → fallback: 阿里云 glm-5 (消耗 GLM 配额时谨慎)
  │
  └─ 配额检查
      ├─ 当前 Provider 配额不足 → 按 fallback 链降级
      └─ 配额充足 → 直接路由
```

### 4.2 任务分类与路由表

| 任务类型 | Skill 层级 | 首选模型 | 备选模型 | 成本预估 |
|----------|-----------|---------|---------|---------|
| 代码格式化 | atom | Copaw-9B | mimo-v2-flash | 免费/¥0.07 |
| 语法检查 | atom | Copaw-9B | mimo-v2-flash | 免费/¥0.07 |
| 正则/脚本生成 | atom | Copaw-9B | 星辰 35B-A3B | 免费 |
| 文件读写/转换 | atom | Copaw-9B | mimo-v2-flash | 免费/¥0.07 |
| 函数级代码生成 | molecule | 星辰 35B-A3B | qwen3.6-plus | 配额 |
| Bug 修复 | molecule | 星辰 35B-A3B | qwen3.6-plus | 配额 |
| 代码审查 | molecule | 星辰 35B-A3B | qwen3.6-plus | 配额 |
| 单元测试编写 | molecule | 星辰 35B-A3B | Copaw-9B(简单) | 免费/配额 |
| 模块级重构 | compound | qwen3.6-plus | mimo-v2-pro | 配额/¥1.40 |
| 架构设计 | compound | 最大可用模型 | qwen3.6-plus | 配额/¥1.40 |
| 系统设计 | compound | 最大可用模型 | GLM-5.1 | 配额(紧张) |
| 事故排查 | compound | kimi-k2.5/thinking | qwen3.6-plus | 配额 |
| 多模态分析 | compound | 星辰 35B-A3B | 无备选 | 未知 |
| 长文档分析(>128K) | compound | 小米 mimo-v2.5-pro | 阿里云(确认上限) | ¥1.40 |
| 战略规划 | compound | 最大可用模型 | GLM-5.1 | 配额(紧张) |

### 4.3 配额感知的动态路由

```python
# 伪代码：配额感知路由
def route_task(task):
    provider = select_by_complexity(task)
    
    # 检查配额
    if provider == "aliyun":
        remaining_5h = get_aliyun_5h_remaining()
        remaining_week = get_aliyun_week_remaining()
        remaining_month = get_aliyun_month_remaining()
        
        if remaining_5h < 500:  # 5h窗口紧张
            if remaining_week < 5000:
                provider = fallback_to_xiaomi(task.complexity)
            else:
                # 5h快用完但周/月充足 → 等窗口刷新或走小米
                provider = fallback_to_xiaomi(task.complexity)
                
    elif provider == "glm":
        remaining_5h = get_glm_5h_remaining()
        remaining_week = get_glm_week_remaining()
        
        if remaining_5h < 10 or remaining_week < 50:
            # GLM 配额紧张 → 降级到阿里云
            provider = "aliyun"
    
    # 成本优化：简单任务优先本地
    if task.complexity == "simple" and local_model_available():
        provider = "local"
    
    return provider
```

---

## 五、成本优化策略

### 5.1 缓存命中优化

小米 MiMo 的缓存命中价格大幅降低：

| 模型 | 缓存命中 | 缓存未命中 | 节省 |
|------|---------|-----------|------|
| mimo-v2.5-pro | ¥1.40 | ¥7.00 | 80% |
| mimo-v2.5 | ¥0.56 | ¥2.80 | 80% |
| mimo-v2-flash | ¥0.07 | ¥0.70 | 90% |

**策略：**
- 对同一代码库的连续请求，尽量保持上下文复用
- 批量处理同一项目的任务，提高缓存命中率
- 将频繁访问的上下文（如项目结构、编码规范）放在 prompt 前部

### 5.2 分级成本预算

| 月度预算档位 | 适用场景 | 推荐模型 |
|-------------|---------|---------|
| 免费 | 日常简单任务 | Copaw-9B 本地 |
| 低 (<¥50/月) | 中等任务补充 | 星辰(若免费) + 阿里云配额 |
| 中 (¥50-200/月) | 常规开发 | 小米 mimo-v2-flash + v2.5 |
| 高 (¥200+/月) | 高强度开发 | 小米 mimo-v2-pro + 阿里云满配额 |

### 5.3 Thinking 模式成本控制

阿里云 thinking 模型的 budgetTokens: 1024 限制了思考深度：

- 1024 tokens ≈ 700-800 中文字的思考空间
- 适合：中等复杂度的推理、代码逻辑分析
- 不适合：长篇架构设计、多步骤规划
- **策略：** 对需要深度思考的任务，优先用 kimi-k2.5 或 glm-5（可能有不同的 budget 行为），或在小米大模型上不使用 thinking 模式（减少 token 消耗）

---

## 六、Fallback 链

每个任务类型定义完整的降级路径：

```
极复杂任务:
  1T-A42B → 220B-A10B → 阿里云 qwen3.6-plus → 小米 mimo-v2-pro → GLM-5.1

复杂任务:
  阿里云 qwen3.6-plus → 阿里云 kimi-k2.5 → 小米 mimo-v2-pro → 星辰 35B-A3B

中等任务:
  星辰 35B-A3B → 阿里云 qwen3.6-plus → 小米 mimo-v2.5 → Copaw-9B

简单任务:
  Copaw-9B → 小米 mimo-v2-flash → 星辰 35B-A3B
```

---

## 七、实现建议

### 7.1 配置文件结构

```yaml
model_routing:
  providers:
    local:
      name: "omlx"
      models:
        - id: copaw-9b
          params: 9B
          context: 32K
          cost: 0
          tier: 1
    aliyun:
      name: "阿里云百炼"
      base_url: "https://coding.dashscope.aliyuncs.com/v1"
      quota:
        per_5h: 6000
        per_week: 45000
        per_month: 90000
      models:
        - id: qwen3.6-plus
          thinking: true
          budget_tokens: 1024
        - id: MiniMax-M2.5
        - id: glm-5
          thinking: true
          budget_tokens: 1024
        - id: glm-4.7
          thinking: true
          budget_tokens: 1024
        - id: kimi-k2.5
          thinking: true
          budget_tokens: 1024
    glm:
      name: "GLM"
      base_url: "https://open.bigmodel.cn/api/coding/paas/v4"
      quota:
        per_5h: 80
        per_week: 400
      models:
        - id: GLM-5.1
        - id: GLM-5-Turbo
        - id: GLM-4.7
        - id: GLM-4.5-Air
    xingchen:
      name: "星辰"
      base_url: "https://maas-coding-api.cn-huabei-1.xf-yun.com/v2"
      models:
        - id: astron-code-latest
          params: 35B-A3B
          context: 128K
          multimodal: true
    xiaomi:
      name: "小米MiMo"
      base_url: "https://token-plan-cn.xiaomimimo.com/v1"
      tier: 1
      models:
        - id: mimo-v2.5-pro
          cost_input_cached: 1.40
          cost_input_uncached: 7.00
          cost_output: 21.00
          context: 1M
        - id: mimo-v2.5
          cost_input_cached: 0.56
          cost_input_uncached: 2.80
          cost_output: 14.00
          context: 1M
        - id: mimo-v2-flash
          cost_input_cached: 0.07
          cost_input_uncached: 0.70
          cost_output: 2.10
          context: 256K

  routing_rules:
    - complexity: simple
      primary: local/copaw-9b
      fallback: [xiaomi/mimo-v2-flash, xingchen/astron-code-latest]
    - complexity: medium
      primary: xingchen/astron-code-latest
      fallback: [aliyun/qwen3.6-plus, xiaomi/mimo-v2.5, local/copaw-9b]
    - complexity: complex
      primary: aliyun/qwen3.6-plus
      fallback: [aliyun/kimi-k2.5, xiaomi/mimo-v2-pro, glm/GLM-5.1]
    - complexity: extreme
      primary: aliyun/glm-5
      fallback: [aliyun/qwen3.6-plus, xiaomi/mimo-v2-pro]

  special_rules:
    multimodal: xingchen/astron-code-latest
    thinking_required: aliyun/qwen3.6-plus
    long_context_128k_plus: xiaomi/mimo-v2.5-pro
    long_context_256k_to_1m: xiaomi/mimo-v2.5-pro
    quota_emergency: xiaomi/mimo-v2-flash
```

### 7.2 配额监控

建议实现以下监控指标：

```
# 配额使用率仪表盘
- 阿里云: 5h使用率 / 周使用率 / 月使用率
- GLM: 5h使用率 / 周使用率
- 小米: 月度成本累计
- 本地: 可用状态

# 告警阈值
- 阿里云 5h 使用率 > 80% → 切换到小米
- GLM 周使用率 > 70% → 暂停自动使用 GLM
- 小米月度成本 > 预算 → 降级到 flash 模型
```

---

## 八、待确认信息

以下信息需要进一步确认以完善策略：

1. **星辰限流规则** — 目前未知，需确认是否有请求频率限制
2. **阿里云模型上下文长度** — qwen3.6-plus、kimi-k2.5 等的具体 context window
3. **220B-A10B 和 1T-A42B 的归属 Provider** — 需确认哪个 Provider 提供这些模型
4. **阿里云 Anthropic 兼容端点** — `https://coding.dashscope.aliyuncs.com/apps/anthropic` 的限流是否与 OpenAI 端点共享
5. **小米 mimo-v2.5-tts 限时免费截止日期** — 需关注是否到期
6. **各模型的实际能力排名** — 参数量 ≠ 实际表现，需通过 benchmark 验证

---

## 九、与 PRD 路由引擎的集成

本策略对应 PRD 中 **7.1 架构 → 路由引擎** 组件的实现：

```
┌──────────────────────────────────────────────────────┐
│                  路由引擎 (Routing Engine)              │
│                                                      │
│  ┌───────────┐  ┌───────────┐  ┌──────────────────┐  │
│  │ 任务分类器 │→│ 配额管理器 │→│ 模型选择器        │  │
│  │(复杂度评估)│  │(配额检查)  │  │(主选+Fallback链) │  │
│  └───────────┘  └───────────┘  └──────────────────┘  │
│       ↓              ↓               ↓               │
│  任务类型标签    配额余量/告警    最终模型选择         │
└──────────────────────────────────────────────────────┘
```

路由引擎作为 Agent Provider 层的一部分，在 Agent 发起 LLM 调用时自动选择最优模型，无需人工干预。
