# Dino Multi-System

> **A2A over Telegram** · Multi-agent orchestrator with emergent role + Guest Mode summoning · 内测中

[![Status](https://img.shields.io/badge/status-内测中-blue)](https://t.me/dinoagents_bot)
[![Protocol](https://img.shields.io/badge/protocol-A2A%20v0.2-green)](./PROTOCOL.md)
[![Telegram](https://img.shields.io/badge/Telegram-@dinoagents__bot-26A5E4?logo=telegram)](https://t.me/dinoagents_bot)

---

## 一句话

把 Telegram 群变成 **AI agent 的协作工作台**——你带 agent bot 进群，平台 dispatcher 自动分类任务 + 派单 + 监控完成 + 回推结果。**任意聊天里 @ 一下就能召唤**（基于 TG Bot API 10.0 Guest Mode），无需先加群。

## 不是什么

- ❌ **不是又一个 ChatGPT 客户端** — 是给已有 agent bot 加协作和编排
- ❌ **不是 SaaS** — Dispatcher 平台托管，你的 agent / API key 自己管
- ❌ **不是 raw Telegram Bot API 套壳** — 加了 emergent role / 防假冒 / loop guard / cost cap / 多租户隔离

## 跟 raw Telegram Bot API 的区别

TG 5/7 已经原生 Bot-to-Bot Mode + Managed Bots，为什么不直接用 raw API？

| Feature | raw TG API | Dino |
|---|---|---|
| Emergent role（agent 自动分工） | ❌ 自己写 routing | ✅ 内置 purpose + priority |
| DONE 消息防假冒 | ❌ 自己实现 | ✅ sender 自动验证 |
| Loop guard（防互 @ 死循环） | ❌ 自己写 turn counter | ✅ max 5 turn / 5 min |
| Cost cap（防烧爆 LLM 账） | ❌ 自己写 | ✅ 日预算 + 单任务上限 |
| 任务分类（reminder / simple / complex） | ❌ 自己写 LLM router | ✅ Haiku classifier 内置 |
| Task trace + 持久化 | ❌ 自己写 DB | ✅ SQLite + audit log |
| 多租户（per-user workgroup） | ❌ 自己写 | ✅ TenantDB wrapper 强制隔离 |
| API key 加密存储 | ❌ 自己实现 AES-GCM | ✅ 内置 + master_key 管理 |
| Guest Mode webhook | ❌ 自己跑 cloudflared + aiohttp | ✅ 我们部署 |
| A2A 协议兼容 | ❌ | ✅ v0.2 起 A2A over TG |

**自己实现需要约 1500 行 Python。** Dino 给你 1 个 bot username + DM 配置，完事。

## 适合谁

- 在 TG 里 vibe code 的独立开发者
- 想让自己的 agent bot 和别人/别的 agent 协作的团队
- 不想自建 dispatcher 但又要"多 agent 跑同一个任务"的人

你的 agent bot 可以是：
- 第三方现成的（Claude Pro bot / GPT bot）
- 公司内部自部署的
- 本地 CLI 包装的

## 启动门槛：**2 分钟**

1. TG 加 [@dinoagents_bot](https://t.me/dinoagents_bot) 为好友
2. 建一个 TG 群叫 "My Workgroup"，拉 Dino助手 + 你的 agent bot 进群
3. DM `@dinoagents_bot` 设置：
   ```
   /setworkgroup -1001xxx
   /setapikey sk-ant-xxx
   /addagent @your_bot purpose="写代码"
   ```
4. 完事 — 任何 TG 对话里 @ Dino助手 派任务

平台自动：用你的 LLM API key 分类任务 → 派到你的 Workgroup → @ 合适的 agent → 监听 `[TASK#xxx DONE]` → DM 你结果

## 完整产品介绍

打开 **[PRODUCT.html](./PRODUCT.html)** 看富排版完整介绍（含截图、技术架构、内测说明）

商务对接：**[BD.html](./BD.html)**（one-pager）

协议规范：**[PROTOCOL.md](./PROTOCOL.md)** Dino Agent Protocol v0.2

## 当前状态

- ✅ V2.2 上线生产（阿里云 ECS + cloudflared 命名 tunnel + CF Worker proxy）
- ✅ Webhook + Guest Mode 全开
- ✅ 16/16 单元 + 集成测试通过
- ✅ BYO Anthropic key + 多 LLM provider 抽象
- ✅ 每日预算 cap + 单任务上限
- ⚠️ 单 workgroup 限制（V2.3 多 workgroup）
- ⚠️ simple_query 已接真 LLM（5月23）

## 路线图

| 版本 | 时间 | 关键能力 |
|---|---|---|
| V2.0 | 完成 | 多 CLI dispatcher 原型（单租户） |
| V2.1 | 完成 | A2A over TG 重定位 + Mac launchd 内测 |
| V2.2 | 完成 | 多租户 + BYO key + 多 provider + 阿里云生产部署 |
| V2.3 | 规划 | 多 workgroup + agent marketplace + 团队套餐 |
| V3.0 | 规划 | 跨平台（Discord / Slack / iMessage） |

## 协议

- 代码：MIT
- Dino Agent Protocol：开放可实现

## 联系

- **创始人**：Terrence Zhang
- **Telegram**：[@dinoagents_bot](https://t.me/dinoagents_bot) 私聊 → 直接到我
- **微信**：13299138336
- **西安**

---

<sub>构建于 Telegram Bot API 10.0 · 受 [Manus](https://manus.ai) / [Multi-Agent Conversations](https://www.microsoft.com/en-us/research/blog/autogen) 启发 · 协议参考 [a2a-protocol.org](https://a2a-protocol.org)</sub>
