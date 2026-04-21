# Nervus v2：以 Personal Model 为中心的个人神经系统
## 完整系统开发设计文档

**版本**：v2.0-alpha
**日期**：2026-04-21
**对比参考**：Nervus v1（https://github.com/wangqioo/nervus）

---

## 核心哲学转变

| | Nervus v1 | Nervus v2 |
|--|-----------|-----------|
| **中心是什么** | Arbor Core（路由器） | Personal Model（推断引擎） |
| **App 的角色** | 被路由的终点，各自处理数据 | Personal Model 的投影界面 |
| **Memory Graph** | App 写入的日志库 | Personal Model 的历史快照 |
| **智能来源** | 每个 App 各自做推断 | 统一的模型维护，跨域理解 |
| **App 卸载后** | 该 App 的数据消失 | 对应维度历史保留，理解不丢失 |
| **新 App 接入** | 从零开始积累数据 | 立刻继承完整的模型历史 |

**一句话定义 v2**：Nervus v2 是一个持续存在的个人智能体，App 只是它表达自己的界面。

---

## 第一章：系统架构总览

### 1.1 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                      感知层 Perception Layer                  │
│  [相机] [麦克风] [日历] [RSS] [位置] [传感器] [剪贴板]          │
└──────────────────────────┬──────────────────────────────────┘
                           │ 原始事件流
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    突触总线 Synapse Bus                        │
│              NATS + JetStream（事件持久化队列）                 │
│                                                             │
│  topics: perception.raw.*                                   │
│          model.dimension.updated.*                          │
│          app.action.request.*                               │
└────────────┬───────────────────────────┬────────────────────┘
             │                           │
             ▼                           ▼
┌────────────────────┐    ┌──────────────────────────────────┐
│  快循环 Fast Loop   │    │      慢循环 Slow Loop              │
│  Arbor Core v2     │    │      Model Updater               │
│  （事件路由，实时）  │    │      （模型维护，后台静默）          │
│  延迟 <2s           │    │      每 5 分钟运行一次              │
└────────────────────┘    └──────────────┬───────────────────┘
                                         │ 更新维度
                                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   Personal Model Layer                       │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              维度状态（Redis，毫秒级读取）               │  │
│  │  nutrition_24h | sleep_quality | cognitive_load       │  │
│  │  active_topics | location_pattern | social_graph ...  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           维度历史（pgvector，语义检索）               │  │
│  │    每次维度更新的快照 + embedding，支持时序查询          │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           跨维度推断（Cross-Dimension Insights）       │  │
│  │    规律发现 | 异常检测 | 预判生成 | 主动建议             │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────┘
                          │ model.dimension.updated.*
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                       App Layer                              │
│                                                             │
│  [Calorie] [Knowledge] [Meeting] [Memory] [Calendar] ...    │
│                                                             │
│  每个 App 是 Personal Model 某些维度的 UI 投影               │
│  App 不做 AI 推断，只负责展示和交互                           │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 与 v1 的核心架构差异

**v1 数据流向**：
```
感知 → Arbor Core 路由 → App 接收 → App 做推断 → App 写 Memory Graph
```

**v2 数据流向**：
```
感知 → Synapse Bus → Model Updater 更新 Personal Model → App 订阅维度变化 → App 展示
                  ↘ Arbor Core 处理实时响应（保留，缩减职责）
```

---

## 第二章：Personal Model 设计

Personal Model 是 v2 的核心创新，是整个系统智能的来源。

### 2.1 维度体系

Personal Model 由若干**维度（Dimension）**组成，每个维度代表对用户某一方面的持续推断。

#### 核心维度清单（v2.0 初始版本）

**身体健康类**

| 维度 ID | 含义 | 更新频率 | 数据来源 |
|---------|------|----------|----------|
| `nutrition_24h` | 过去24小时营养摄入状态 | 每次饮食感知后 | 相机、食物识别 |
| `nutrition_pattern_7d` | 近7天饮食规律 | 每日 | nutrition_24h 历史 |
| `sleep_last_night` | 昨晚睡眠质量 | 每日晨间 | 手机使用时间、加速计 |
| `sleep_pattern_14d` | 近两周睡眠规律 | 每日 | sleep_last_night 历史 |
| `activity_today` | 今日活动水平 | 每小时 | 位置变化、步数感知 |

**认知状态类**

| 维度 ID | 含义 | 更新频率 | 数据来源 |
|---------|------|----------|----------|
| `cognitive_load_now` | 当前认知负荷 | 每15分钟 | 日历密度、会议记录、使用频率 |
| `focus_quality_today` | 今日专注质量 | 每小时 | App 使用模式、任务完成情况 |
| `stress_indicator` | 压力信号 | 每小时 | 综合多维度推断 |

**知识与兴趣类**

| 维度 ID | 含义 | 更新频率 | 数据来源 |
|---------|------|----------|----------|
| `active_topics` | 当前活跃学习/关注话题 | 每次内容消费后 | 阅读、保存、搜索记录 |
| `knowledge_graph` | 知识连接图谱（粗粒度） | 每日 | 所有知识类事件 |
| `reading_velocity` | 阅读/学习节奏 | 每日 | 内容消费量和类型 |

**时间与习惯类**

| 维度 ID | 含义 | 更新频率 | 数据来源 |
|---------|------|----------|----------|
| `daily_routine` | 一天的时间分配模式 | 每日 | 位置、日历、App 使用 |
| `weekly_pattern` | 周节律（工作日 vs 周末） | 每周 | daily_routine 汇总 |
| `upcoming_context` | 未来24小时的预期状态 | 每日晨间 + 日历变更时 | 日历、历史规律 |

**社交与沟通类**

| 维度 ID | 含义 | 更新频率 | 数据来源 |
|---------|------|----------|----------|
| `social_rhythm` | 社交互动密度和模式 | 每日 | 通话、会议、消息 |
| `key_relationships` | 高频互动对象的上下文 | 每次互动后 | 会议记录、通讯 |

### 2.2 维度数据结构

```python
from dataclasses import dataclass
from typing import Any, List, Optional
from datetime import datetime

@dataclass
class DimensionSnapshot:
    """单次维度快照，存入 pgvector 作为历史"""
    snapshot_id: str
    dimension_id: str
    value: dict              # 推断结果，结构因维度而异
    confidence: float        # 0.0 - 1.0，模型的置信度
    captured_at: datetime
    derived_from_events: List[str]  # 来源事件 ID，可追溯
    embedding: List[float]   # 语义向量，用于相似性检索
    version: int             # 用于冲突解决


@dataclass
class DimensionState:
    """维度当前状态，存入 Redis 供实时读取"""
    dimension_id: str
    current_value: dict
    confidence: float
    last_updated: datetime
    snapshot_id: str         # 指向 pgvector 中最新快照
    staleness_ttl: int       # 秒，超时视为过期需要重新推断
```

**示例：`nutrition_24h` 维度的 value 结构**

```json
{
  "total_calories_estimate": 1850,
  "meal_count": 3,
  "last_meal_at": "13:45",
  "quality_score": 0.6,
  "flags": ["high_fat_lunch", "skipped_breakfast"],
  "dominant_food_types": ["rice", "vegetables", "fried"],
  "hydration_signals": "low",
  "model_note": "午饭检测到炸鸡+汽水，今日蔬菜摄入不足"
}
```

### 2.3 维度更新流程

```
1. Model Updater 每5分钟醒来
2. 从 JetStream 读取过去5分钟的 perception.raw.* 事件
3. 从 Redis 读取受影响维度的当前值
4. 构建推断 Prompt，调用本地模型
5. 本地模型输出新的维度值 + 置信度
6. 与旧值对比，计算变化幅度
7. 如果变化幅度 > threshold：
   a. 更新 Redis（当前状态）
   b. 写入 pgvector（历史快照 + embedding）
   c. 发布 model.dimension.updated.{dimension_id} 到 NATS
8. 如果变化幅度 < threshold：不发布，节省 App 唤醒成本
```

**推断 Prompt 模板示例（nutrition_24h）**：

```
你是一个营养状态分析模型。

当前维度状态：
{current_nutrition_24h_value}

过去5分钟新增事件：
{recent_events}

用户历史规律（近7天 nutrition_24h 摘要）：
{nutrition_pattern_summary}

请更新 nutrition_24h 维度，输出 JSON 格式：
{
  "value": {...},
  "confidence": 0.0-1.0,
  "change_magnitude": "none|minor|significant|major",
  "model_note": "简短推断说明"
}
```

### 2.4 跨维度推断（Cross-Dimension Insights）

这是 v2 最有价值的功能，也是 v1 根本做不到的。

跨维度推断由独立的 **Insight Engine** 负责，每小时运行一次，读取多个维度的当前值和历史，发现规律，生成主动建议。

```python
class InsightEngine:
    """跨维度规律发现与预判生成"""

    INSIGHT_RULES = [
        # 规律发现型
        {
            "id": "meeting_diet_correlation",
            "dimensions": ["cognitive_load_now", "nutrition_24h"],
            "prompt": "分析认知负荷和饮食质量的相关性，是否存在高压力日→差饮食的规律",
            "min_history_days": 14,
            "output_topic": "insight.pattern.discovered"
        },
        # 预判型
        {
            "id": "upcoming_stress_prediction",
            "dimensions": ["upcoming_context", "sleep_last_night", "stress_indicator"],
            "prompt": "基于明天日历密度和今日睡眠，预测明天的压力水平，生成建议",
            "trigger": "daily_morning",
            "output_topic": "insight.prediction.generated"
        },
        # 异常检测型
        {
            "id": "routine_deviation_alert",
            "dimensions": ["daily_routine", "weekly_pattern"],
            "prompt": "今日行为是否明显偏离历史规律？如果是，分析可能原因",
            "output_topic": "insight.anomaly.detected"
        }
    ]
```

---

## 第三章：服务架构设计

### 3.1 服务清单

v2 新增两个核心服务，Arbor Core 职责缩减：

| 服务名 | v1 是否有 | 职责 | 技术 |
|--------|-----------|------|------|
| `synapse-bus` | 有 | 事件总线 | NATS + JetStream（不变） |
| `arbor-core-v2` | 有→缩减 | 实时路由（仅快循环） | Python FastAPI（精简） |
| `model-updater` | **新增** | 慢循环，维护 Personal Model | Python |
| `insight-engine` | **新增** | 跨维度推断，主动建议 | Python |
| `personal-model-api` | **新增** | Personal Model 的统一查询接口 | Python FastAPI |
| `llama-server` | 有 | 本地模型服务 | llama.cpp（不变） |
| `redis` | 有 | 维度当前状态 | Redis（扩展用途） |
| `postgres` | 有 | 维度历史快照 + pgvector | PostgreSQL（不变） |
| `nsi-gateway` | 有 | App 接入网关 | Fastify（不变） |

### 3.2 内存预算（8GB Jetson Orin Nano）

| 组件 | v1 内存 | v2 内存 | 变化说明 |
|------|---------|---------|----------|
| 系统/JetPack | 1.5GB | 1.5GB | 不变 |
| Qwen3.5-4B（llama.cpp） | 2.8GB | 2.8GB | 不变 |
| faster-whisper | 0.5GB（按需） | 0.5GB（按需） | 不变 |
| Redis | ~0.1GB | ~0.2GB | 维度状态增加 |
| PostgreSQL | ~0.3GB | ~0.4GB | 快照数量增加 |
| Arbor Core | ~0.2GB | ~0.15GB | 职责缩减 |
| Model Updater（新） | - | ~0.15GB | 新增 |
| Insight Engine（新） | - | ~0.1GB | 新增，按需唤醒 |
| Personal Model API（新） | - | ~0.1GB | 新增 |
| **常驻总计** | ~6.3GB | ~6.4GB | 安全（<8GB） |

### 3.3 NATS Topic 体系

**v1 Topics（保留）**：
```
perception.raw.photo
perception.raw.audio
perception.raw.calendar
perception.raw.clipboard
```

**v2 新增 Topics**：
```
# 维度更新通知
model.dimension.updated.nutrition_24h
model.dimension.updated.cognitive_load_now
model.dimension.updated.{dimension_id}

# 推断结果
insight.pattern.discovered
insight.prediction.generated
insight.anomaly.detected

# 模型查询（请求-响应模式）
model.query.request
model.query.response.{correlation_id}
```

---

## 第四章：Nervus App Interface v2（NAI v2）

v2 的 App 接入协议升级，从"事件订阅"扩展为"模型订阅"。

### 4.1 manifest.json v2 格式

```json
{
  "app_id": "calorie-tracker",
  "version": "2.0",
  "display_name": "卡路里追踪",

  "identity": {
    "lifecycle_status": "active",
    "data_tier": "presentation",
    "description": "营养摄入的可视化和记录界面"
  },

  "model_subscriptions": [
    {
      "dimension_id": "nutrition_24h",
      "notify_on_change": true,
      "min_change_magnitude": "minor"
    },
    {
      "dimension_id": "nutrition_pattern_7d",
      "notify_on_change": true,
      "min_change_magnitude": "significant"
    }
  ],

  "model_reads": [
    "nutrition_24h",
    "nutrition_pattern_7d",
    "cognitive_load_now",
    "activity_today"
  ],

  "model_writes": [],

  "perception_subscriptions": [
    "perception.raw.photo"
  ],

  "actions": [
    {
      "name": "manual_log_meal",
      "description": "用户手动记录一次饮食",
      "risk_level": "low",
      "requires_confirmation": false
    }
  ],

  "lifecycle": {
    "auto_suspend_after_idle_days": 30,
    "priority": "normal"
  }
}
```

**关键变化**：v1 的 App 声明订阅事件（`perception.raw.photo`），v2 的 App 主要声明订阅维度（`nutrition_24h`）。App 拿到的不是原始事件，而是已经推断好的模型状态。

### 4.2 NAI v2 端点

每个 App 实现以下端点：

```
GET  /manifest              → 能力声明（v2 格式）
POST /intake/dimension      → 接收维度更新通知
POST /intake/insight        → 接收推断结果通知
POST /intake/perception     → 接收原始感知（可选，仅需要时声明）
POST /emit                  → 发布事件（via SDK）
GET  /query/:dimension_id   → 查询 App 对某维度的解读
POST /action/:name          → 执行能力
GET  /state                 → 当前状态快照
POST /correction            → 向 Personal Model 提交修正（新增）
```

**`/correction` 端点说明**：

当 App 发现 Personal Model 的推断有误时（比如用户手动修正了卡路里记录），App 通过这个端点把修正推回给 Model Updater：

```json
{
  "dimension_id": "nutrition_24h",
  "correction_type": "value_override",
  "field_path": "total_calories_estimate",
  "corrected_value": 2100,
  "reason": "用户手动修正",
  "confidence_boost": 0.3
}
```

这是 v2 的反馈闭环机制，让模型持续从用户行为中校准。

### 4.3 SDK 核心方法（nervus-sdk v2）

```typescript
// App 开发者使用的 SDK

class NervusClient {

  // 读取维度当前值
  async getDimension(dimensionId: string): Promise<DimensionState>

  // 查询维度历史
  async queryDimensionHistory(
    dimensionId: string,
    options: { from: Date, to: Date, semantic_query?: string }
  ): Promise<DimensionSnapshot[]>

  // 订阅维度变化
  onDimensionUpdate(
    dimensionId: string,
    handler: (state: DimensionState) => void
  ): Unsubscribe

  // 订阅推断结果
  onInsight(
    insightType: string,
    handler: (insight: Insight) => void
  ): Unsubscribe

  // 提交修正
  async submitCorrection(correction: Correction): Promise<void>

  // 发布感知事件（感知 App 使用）
  async emitPerception(event: PerceptionEvent): Promise<void>

  // 自然语言查询 Personal Model（调用 Personal Model API）
  async query(question: string): Promise<QueryResult>
}
```

---

## 第五章：Personal Model API

提供统一的查询接口，让 App 和未来的语音助手都能用自然语言访问 Personal Model。

### 5.1 端点设计

```
GET  /dimensions                    → 列出所有维度及当前状态
GET  /dimensions/:id                → 获取某维度详情
GET  /dimensions/:id/history        → 获取维度历史（支持时间范围）
POST /query                         → 自然语言查询
GET  /insights                      → 获取最新推断结果
POST /corrections                   → 提交修正
GET  /patterns                      → 获取跨维度规律摘要
```

### 5.2 自然语言查询

`POST /query` 是最重要的端点，支持任意自然语言问题：

**请求**：
```json
{
  "question": "我上个月有没有高压力日多吃垃圾食品的规律？",
  "time_range": "last_30_days",
  "dimensions_hint": ["stress_indicator", "nutrition_24h"]
}
```

**处理流程**：
1. 向 pgvector 做语义检索，找到相关维度历史
2. 加载相关维度的时序数据
3. 调用本地/云端模型做跨维度分析
4. 返回结构化回答 + 引用的维度快照

**响应**：
```json
{
  "answer": "有明显规律。过去30天中，你有8天cognitive_load_now超过0.8（高压力），其中6天的nutrition_24h quality_score低于0.5，相关系数约0.75。具体看：每周三和周四会议最密集，这两天的饮食质量评分平均比其他工作日低0.3。",
  "confidence": 0.82,
  "supporting_snapshots": [...],
  "visualization_hint": "time_series_dual_axis"
}
```

---

## 第六章：Model Updater 实现

### 6.1 运行机制

```python
class ModelUpdater:
    """Personal Model 的维护引擎，每5分钟运行"""

    DIMENSION_CONFIGS = {
        "nutrition_24h": {
            "trigger_events": ["perception.raw.photo"],
            "requires_dims": [],
            "staleness_ttl": 3600,  # 1小时不更新则标记为过期
            "min_events_to_trigger": 1,
            "cloud_threshold": 0.3,  # 置信度低于此值时转云端推断
        },
        "cognitive_load_now": {
            "trigger_events": ["perception.raw.calendar", "perception.raw.usage"],
            "requires_dims": [],
            "staleness_ttl": 900,   # 15分钟
            "schedule": "every_15min",
        },
        # ... 其他维度
    }

    async def run_cycle(self):
        """一次更新循环"""
        # 1. 读取过去5分钟的新事件
        events = await self.nats.fetch_recent("perception.raw.*", minutes=5)

        # 2. 确定需要更新的维度
        affected_dims = self.identify_affected_dimensions(events)

        # 3. 并行更新维度（非依赖型）
        results = await asyncio.gather(*[
            self.update_dimension(dim_id, events)
            for dim_id in affected_dims
        ])

        # 4. 处理维度间依赖（如 nutrition_pattern_7d 依赖 nutrition_24h）
        await self.update_derived_dimensions(results)

    async def update_dimension(self, dimension_id: str, events: List[Event]):
        """更新单个维度"""
        config = self.DIMENSION_CONFIGS[dimension_id]

        # 读取当前值
        current = await self.redis.get_dimension(dimension_id)

        # 读取相关历史（给模型作上下文）
        history = await self.pgvector.get_recent_snapshots(
            dimension_id, days=7, limit=10
        )

        # 构建推断 Prompt
        prompt = self.build_prompt(dimension_id, current, events, history)

        # 选择模型（本地 or 云端）
        model = self.select_model(current.confidence, config)
        result = await model.infer(prompt)

        # 计算变化幅度
        magnitude = self.calc_change_magnitude(current.value, result.value)

        if magnitude > "none":
            # 写入 Redis（当前状态）
            await self.redis.set_dimension(dimension_id, result)

            # 写入 pgvector（历史快照）
            snapshot_id = await self.pgvector.save_snapshot(
                dimension_id, result, events
            )

            # 通知订阅的 App
            await self.nats.publish(
                f"model.dimension.updated.{dimension_id}",
                {"dimension_id": dimension_id, "magnitude": magnitude, "snapshot_id": snapshot_id}
            )
```

### 6.2 推断模型选择策略

```python
def select_model(self, current_confidence: float, config: dict) -> Model:
    """
    本地模型：简单更新、数据充足、近期有类似推断
    云端模型：首次推断、置信度低、复杂跨维度、用户发出查询
    """
    if current_confidence < config.get("cloud_threshold", 0.3):
        return self.cloud_model  # GPT-4o / Claude
    if self.is_novel_pattern(config):
        return self.cloud_model
    return self.local_model  # Qwen3.5-4B
```

---

## 第七章：数据分层设计

v2 采用三层数据模型，解决 v1 混存导致的质量问题。

### 7.1 三层数据层

```
Layer 0: Raw Perception
  → 原始感知事件，存入 JetStream（保留7天），不可修改
  → 例：相机拍到炸鸡的原始图像事件

Layer 1: Dimension State
  → Model Updater 的推断结果，存入 Redis（当前）+ pgvector（历史）
  → 例：nutrition_24h = {quality: 0.6, flags: ["high_fat"]}
  → 可被修正（通过 /correction 接口）

Layer 2: Insights
  → Insight Engine 的跨维度分析结果
  → 例："高压力日 → 坏饮食"规律
  → 存入独立表，附带置信度和有效期
```

查询时按层选择：
- App 实时展示 → 查 Redis（Layer 1 当前值）
- 历史回溯 → 查 pgvector（Layer 1 历史）
- 规律展示 → 查 Insights 表（Layer 2）
- 原始数据审计 → 查 JetStream（Layer 0）

### 7.2 数据库 Schema

**pgvector（维度历史）**：

```sql
CREATE TABLE dimension_snapshots (
    snapshot_id     UUID PRIMARY KEY,
    dimension_id    VARCHAR(64) NOT NULL,
    captured_at     TIMESTAMPTZ NOT NULL,
    value           JSONB NOT NULL,
    confidence      FLOAT NOT NULL,
    derived_from    TEXT[],          -- 来源事件 ID
    embedding       vector(1536),    -- 语义向量
    version         INT DEFAULT 1,
    is_corrected    BOOLEAN DEFAULT FALSE,
    correction_note TEXT
);

CREATE INDEX ON dimension_snapshots (dimension_id, captured_at DESC);
CREATE INDEX ON dimension_snapshots USING ivfflat (embedding vector_cosine_ops);
```

**Redis（维度当前状态）**：

```
KEY: pm:dim:{dimension_id}
TYPE: Hash
FIELDS:
  value          → JSON string
  confidence     → float string
  last_updated   → ISO timestamp
  snapshot_id    → UUID
  staleness_ttl  → int (seconds)
TTL: {staleness_ttl} seconds (自动过期)
```

---

## 第八章：冷启动策略

新用户或重置后，Personal Model 为空，需要分阶段建立。

### 8.1 三阶段启动

**Phase 0（第1-3天）：感知积累**
- 所有维度标记为 `initializing`
- Model Updater 运行但不发布维度更新通知
- App 使用 fallback 模式（显示"正在了解你"）
- 目标：积累足够的原始事件

**Phase 1（第4-14天）：模式发现**
- 维度开始产生置信度 < 0.6 的初始推断
- App 开始收到维度更新，但标记为 `low_confidence`
- Insight Engine 不运行（数据不足）
- 用户看到基础功能，无跨维度规律

**Phase 2（第15天+）：稳定运行**
- 大多数维度置信度 > 0.7
- Insight Engine 启动
- 跨维度规律开始出现
- 全功能模式

### 8.2 快速启动（可选）

用户可以回答一组引导问题（10-15题）来提前填充部分维度：

```json
{
  "onboarding_questions": [
    {"dimension": "daily_routine", "question": "你通常几点起床/睡觉？"},
    {"dimension": "nutrition_pattern_7d", "question": "你平时饮食规律吗？"},
    {"dimension": "active_topics", "question": "你现在在学习或关注什么？"}
  ]
}
```

回答后直接初始化对应维度（低置信度），缩短冷启动时间约5天。

---

## 第九章：与 v1 并行运行的过渡方案

用户希望对比两套系统，以下是并行运行策略。

### 9.1 数据共享

v1 和 v2 共享同一个 NATS 总线和 PostgreSQL 实例：
- v1 的 Memory Graph 表保持不变（v2 只读，不写入）
- v2 新增 `dimension_snapshots` 和 `insights` 表
- Redis 用不同 key prefix 隔离：`v1:` 和 `pm:`（personal model）

### 9.2 流量分流

Synapse Bus 同时路由给 v1 的 Arbor Core 和 v2 的 Model Updater：

```yaml
# NATS Consumer 配置
v1-arbor-core:
  filter_subjects: ["perception.raw.*"]
  deliver_policy: all

v2-model-updater:
  filter_subjects: ["perception.raw.*"]
  deliver_policy: all
```

同一个感知事件两套系统都收到，完全独立处理。

### 9.3 对比指标

建议追踪以下对比数据：

| 指标 | v1 数据源 | v2 数据源 |
|------|-----------|-----------|
| 跨域问题回答质量 | 手动测试 | 手动测试 |
| App 数据准确率 | App 内部日志 | correction 提交次数 |
| 新 App 接入时间 | 开发计时 | 开发计时 |
| 删除 App 后数据保留 | 不保留 | 维度历史完整保留 |
| 系统主动建议数量 | 0（v1 无此功能） | Insight Engine 输出计数 |

---

## 第十章：Sprint 计划

### Sprint 0（第1周）：基础设施扩展

**目标**：在 v1 基础上搭建 v2 新增组件，不破坏 v1

- [ ] 新增 `dimension_snapshots` 表（PostgreSQL）
- [ ] 配置 Redis v2 key 空间（`pm:dim:*`）
- [ ] 搭建 Model Updater 服务骨架
- [ ] 定义5个核心维度的 Prompt 模板
- [ ] 新增 NATS topics（`model.dimension.updated.*`）

**产出**：Model Updater 能跑起来，能写入第一条维度快照

### Sprint 1（第2周）：第一个完整维度

**目标**：`nutrition_24h` 维度端到端跑通

- [ ] Model Updater 实现 nutrition_24h 更新逻辑
- [ ] Calorie App 迁移到订阅 `model.dimension.updated.nutrition_24h`
- [ ] `/correction` 端点实现
- [ ] 基础 Personal Model API（`GET /dimensions/nutrition_24h`）

**产出**：拍食物照片 → 维度更新 → Calorie App 收到通知 → 展示结果

### Sprint 2（第3周）：扩展维度 + Personal Model API

**目标**：5个维度全部运行，API 可查询

- [ ] 实现 `cognitive_load_now`、`sleep_last_night`、`active_topics`、`daily_routine`
- [ ] Personal Model API 完整端点
- [ ] 自然语言查询（`POST /query`）基础版
- [ ] nervus-sdk v2 核心方法

**产出**：可以用自然语言问"我今天状态怎么样"

### Sprint 3（第4周）：Insight Engine

**目标**：第一个跨维度规律发现

- [ ] Insight Engine 基础框架
- [ ] 实现 2-3 个 Insight Rule
- [ ] insight.pattern.discovered 发布和 App 订阅
- [ ] 冷启动引导问卷

**产出**：系统自动发现并推送一条跨维度规律

### Sprint 4（第5-6周）：App 迁移 + 对比测试

**目标**：主要 App 迁移到 v2 协议，开始对比

- [ ] Meeting App 迁移
- [ ] Knowledge App 迁移
- [ ] v1 vs v2 对比指标采集
- [ ] 全维度 lifecycle 管理

**产出**：可以正式进行 v1 vs v2 的日常使用对比

---

## 附录：关键设计决策与取舍

**决策1：为什么保留 Arbor Core 而不是全部用 Model Updater？**

实时响应（<2s）和后台推断（分钟级）的需求不同。语音播放、即时通知仍然需要 Arbor Core 的快速路由。两套循环并存，不是冗余而是职责分离。

**决策2：维度置信度低时是否应该对 App 隐藏？**

推荐做法：不隐藏，但在 App 侧暴露 `confidence` 字段，由 App 决定展示方式。低置信度不等于错误，只是不确定。App 可以显示"根据有限数据推断"的提示。

**决策3：多快触发维度更新比较合理？**

5分钟是当前的平衡点。太频繁（<1分钟）：模型调用成本高，Jetson 发热；太稀疏（>15分钟）：实时感较差。5分钟内用户的状态变化通常不需要立即反映在维度上。

**决策4：Personal Model 数据是否加密？**

所有数据本地存储，与 v1 安全策略一致。敏感维度（`key_relationships`、`stress_indicator`）在 Redis 中使用额外加密层。pgvector 的 embedding 本身不含原始文本，降低泄漏风险。

**决策5：App 能不能直接写入维度？**

不能。维度只能由 Model Updater 写入（通过推断）或通过 `/correction` 接口提交修正建议，最终由 Model Updater 决定是否采纳。这保证了 Personal Model 的一致性，防止多个 App 互相覆盖导致模型紊乱。

---

*文档版本 v2.0-alpha | 对比参考 Nervus v1 (github.com/wangqioo/nervus)*
