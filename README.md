# Nervus · 从未停止存在的系统

> 连接所有 App 的神经系统。本地 AI 24 小时常驻，跨应用智能联动，把你的一生存进一个盒子里。

---

## 版本说明

| 版本 | 状态 | 核心理念 | 文档 |
|------|------|----------|------|
| **v2（当前设计方向）** | 设计中，与 v1 并行运行对比 | Personal Model 为中心：系统持续推断"你是谁"，App 只是 Personal Model 的投影界面 | [`Nervus_v2_系统设计文档.md`](./Nervus_v2_系统设计文档.md) |
| v1（已部署运行） | 运行中 | Arbor Core 为中心：事件路由，App 各自处理数据 | [`Nervus_完整开发文档.md`](./Nervus_完整开发文档.md) |

**v1 vs v2 最核心的区别**：

```
v1：感知 → Arbor Core 路由 → App 接收事件 → App 自己做推断 → App 写 Memory Graph

v2：感知 → Model Updater 更新 Personal Model → App 订阅维度变化 → App 展示
```

v1 是"用 AI 连接 20 个 App"，v2 是"一个持续存在的个人智能体，App 只是它表达自己的界面"。

---

## 这是什么

Nervus 不是一个 App，不是一个平台，不是一个助手。

**它是连接所有 App 的神经系统。**

它运行在你的边缘设备上，永不停止。所有子 App 共享同一套感知、同一套记忆、同一套行动能力。用户不需要主动管理信息——信息自己找到它该去的地方。

```
你拍了一张餐厅的照片 → 热量 App 自动记录，你什么都没做
你开完了一个会 → 录音 + 白板照片自动整合成完整报告
你最近压力很大 → 系统感知到，主动建议调整明天的日程
```

**它从未停止存在过。**

---

## v2 核心架构

```
┌──────────────────────────────────────────────────┐
│              感知层 Perception Layer               │
│  [相机] [麦克风] [日历] [RSS] [位置] [传感器]       │
└─────────────────────┬────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────┐
│           突触总线 Synapse Bus（NATS）              │
└──────────┬───────────────────────┬───────────────┘
           │                       │
           ▼                       ▼
┌─────────────────┐    ┌───────────────────────────┐
│  快循环          │    │  慢循环                     │
│  Arbor Core v2  │    │  Model Updater             │
│  实时路由 <2s    │    │  每5分钟，后台静默           │
└─────────────────┘    └──────────────┬────────────┘
                                      │ 更新维度
                                      ▼
┌──────────────────────────────────────────────────┐
│              Personal Model Layer                 │
│                                                  │
│  维度状态（Redis）：                               │
│  nutrition_24h | cognitive_load | sleep_quality  │
│  active_topics | daily_routine | ...             │
│                                                  │
│  维度历史（pgvector）：时序快照 + 语义检索           │
│                                                  │
│  跨维度推断：规律发现 | 异常检测 | 主动预判          │
└─────────────────────┬────────────────────────────┘
                      │ model.dimension.updated.*
                      ▼
┌──────────────────────────────────────────────────┐
│                   App Layer                       │
│  每个 App 是 Personal Model 某些维度的 UI 投影     │
│  App 不做 AI 推断，只负责展示和交互                 │
└──────────────────────────────────────────────────┘
```

---

## v1 架构（保留参考）

```
感知层（照片/录音/日历/RSS）
    ↓
Synapse Bus（NATS 事件总线）
    ↓
Arbor Core（本地 AI 神经中枢 · Qwen3.5-4B · 24h 常驻）
    ↓
App Layer（20个 App · 通过 NSI 接口互联）
    ↓
Memory Graph（长期记忆 · PostgreSQL + pgvector）
Context Graph（当下状态 · Redis）
```

---

## 双大脑架构

| | 本地小模型（常驻） | 云端大模型（按需） |
|--|--|--|
| 模型 | Qwen3.5-4B 多模态 | GPT-4o / Claude |
| 角色 | 慢性记忆 · 潜意识 | 急性思维 · 意识 |
| 工作时机 | 24h 不间断 | 需要深度推理时 |
| 成本 | 零 · 完全本地 | 按需付费 |

---

## 硬件

**NVIDIA Jetson Orin Nano 8GB**

- 专用边缘计算设备，永远在线
- Qwen3.5-4B 多模态 GGUF 模型 via llama.cpp
- 所有数据本地存储，不上云

---

## 技术栈

| 职责 | 技术 |
|------|------|
| 容器编排 | Docker Compose |
| 事件总线 | NATS + JetStream |
| 当下状态 | Redis（Context Graph / 维度状态） |
| 长期记忆 | PostgreSQL + pgvector（Memory Graph / 维度历史） |
| 本地 AI | llama.cpp server + Qwen3.5-4B 多模态 |
| 语音转写 | faster-whisper |
| AI 中枢 | Python + FastAPI（Arbor Core v2） |
| 模型维护 | Python（Model Updater · v2 新增） |
| 推断引擎 | Python（Insight Engine · v2 新增） |
| App 后端 | FastAPI / Fastify |
| 移动端壳 | Capacitor.js（iOS 优先） |
| 内部 SDK | nervus-sdk v2（自研） |

---

## 仓库文件

| 文件 | 版本 | 说明 |
|------|------|------|
| [`Nervus_v2_系统设计文档.md`](./Nervus_v2_系统设计文档.md) | **v2** | Personal Model 为中心的完整系统设计：架构·维度体系·服务设计·Sprint 计划 |
| [`Nervus_完整开发文档.md`](./Nervus_完整开发文档.md) | v1 | v1 完整开发文档：架构·设计·契约·计划 |
| [`app-prototype.html`](./app-prototype.html) | v1 | 前端交互原型（可直接浏览器打开） |

---

## v2 开发计划

| Sprint | 内容 | 关键产出 |
|--------|------|----------|
| Sprint 0（第1周） | 基础设施扩展 | Model Updater 服务骨架，dimension_snapshots 表，5个核心维度定义 |
| Sprint 1（第2周） | 第一个完整维度 | nutrition_24h 端到端：拍照→维度更新→App 收到通知 |
| Sprint 2（第3周） | 扩展维度 + API | 5个维度全部运行，Personal Model API，自然语言查询基础版 |
| Sprint 3（第4周） | Insight Engine | 第一个跨维度规律自动发现（高压力日→差饮食） |
| Sprint 4-5（第5-6周） | App 迁移 + 对比 | 主要 App 迁移 v2 协议，v1 vs v2 正式对比测试 |

完整设计见 [v2 系统设计文档](./Nervus_v2_系统设计文档.md)。

---

## v1 开发计划概览

| Sprint | 内容 | 目标 |
|--------|------|------|
| Sprint 0 | 基础设施 | Docker + NATS + Redis + PostgreSQL + llama.cpp |
| Sprint 1 | nervus-sdk | 5 行代码接入生态 |
| Sprint 2 | Arbor Core | 神经中枢，三种路由模式 |
| Sprint 3 | 第一条数据流 | 照片 → 热量自动记录，端到端跑通 |
| Sprint 4 | Memory Graph | 长期记忆 + Sense 页数据化 |
| Sprint 5 | App NSI 接入 | 20个 App 打通，旗舰体验可用 |
| Sprint 6 | Capacitor iOS | 完整移动端 MVP |

---

## 两个旗舰体验

**人生记忆库** · 把你的一生，存进一个盒子里

> 自动收集照片、视频、行程、笔记。自动生成年度回忆录、旅行日志、孩子成长瞬间。

**私人知识大脑** · 你看过的所有东西，都变成你的第二大脑

> 自动收录文章、PDF、视频字幕、会议录音。语义检索，随时问答。

---

*Nervus · 从未停止存在的系统*
