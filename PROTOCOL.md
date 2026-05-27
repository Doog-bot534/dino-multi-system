# Dino Agent Protocol v0.2 · A2A over Telegram

> Telegram 上的多 agent 调度协议。**v0.2 起重定位为 [A2A (Agent-to-Agent) protocol](https://a2a-protocol.org) 的 Telegram transport binding + emergent role extension**。
>
> 你不再需要学一套新协议 — 跟 A2A 兼容,只是把 transport 从 JSON-RPC over HTTPS 换成 TG 消息。任何遵守 A2A 语义的 agent 都能接入 Dino 生态。

## 1. Why A2A over Telegram

A2A protocol(Google / Linux Foundation,2025-04 发布,2026 已 150+ 组织参与)定义了:
- **Agent Card**:agent 能力声明
- **Task**:工作单元(uuid + status + history)
- **Message**:agent 间消息(role + parts + taskId)
- **状态机**:`SUBMITTED → WORKING → COMPLETED / FAILED / CANCELED / INPUT_REQUIRED`

A2A 推荐 transport 是 JSON-RPC over HTTPS。Dino v0.2 定义了 **A2A 在 Telegram 上的 transport binding** — 用 TG 群消息当 wire format。语义 100% 跟 A2A 兼容,但你只需 Telegram bot 而非 HTTP server。

### 与 raw A2A 的区别

| A2A 标准 | Dino TG binding |
|---|---|
| `/.well-known/agent-card.json` 自动发现 | TG `/refresh` 命令 + `/addagent @bot purpose="..."` 手动注册 |
| JSON-RPC over HTTPS | TG 群消息(纯文本 wire format) |
| `Authorization: Bearer ...` | TG bot token + BotFather Bot-to-Bot Mode |
| Webhook push notification | TG message 自然推送(bot 收 update) |
| `messageId` / `contextId` UUID | TG `message_id` + workgroup `chat_id` |

## 2. Agent 准入要求

### 2.1 BotFather 配置

```
/mybots → 选择你的 bot → Bot Settings
  → Group Privacy → Turn off       (读群里所有消息)
  → Bot-to-Bot Mode → Enable       (能收/发其它 bot 消息)
  → Allow Groups → Enable
```

### 2.2 加入 Workgroup

- 加入用户建的 TG 群
- **必须管理员权限**
- 自己 DM `/start` 一次,允许 Dispatcher 在任务完成时 DM 用户

### 2.3 注册到 Dispatcher

用户 DM Dino助手:
```
/addagent @your_bot purpose="<能力描述>" [priority=100]
```

`purpose` 字段 ≈ A2A Agent Card 的 `skills[].description` — Dispatcher 据此 routing。

## 3. Text wire format(A2A 在 TG 上的表示)

### 3.1 入站 · `SendMessage` (Dispatcher → Agent)

```
[TASK#<uuid>] @<your-bot-username> <task description text>
```

A2A 等价:
```json
{
  "method": "SendMessage",
  "params": {
    "message": {
      "messageId": "<TG message_id>",
      "taskId": "<uuid>",
      "contextId": "<workgroup chat_id>",
      "role": "ROLE_USER",
      "parts": [{"text": "<task description text>"}]
    }
  }
}
```

### 3.2 出站 DONE · Task completed

在同一 workgroup 回:
```
[TASK#<uuid> DONE]
<result body, 任意 markdown>
```

A2A 等价:
```json
{
  "method": "SendMessage",
  "params": {
    "message": {
      "taskId": "<uuid>",
      "role": "ROLE_AGENT",
      "parts": [{"text": "<result>"}]
    },
    "taskStatus": "COMPLETED"
  }
}
```

### 3.3 出站 FAILED · Task failed

```
[TASK#<uuid> FAILED] <reason>
```

A2A `taskStatus: FAILED`。

### 3.4 出站 PROGRESS · Streaming update(optional)

```
[TASK#<uuid> PROGRESS] <短描述>
```

A2A `SendStreamingMessage` event,`taskStatus: WORKING`。Dispatcher 收到重置 timeout 倒计时。

### 3.5 出站 CLARIFY · Input required(V0.3 规划)

```
[TASK#<uuid> CLARIFY] <question to user>
```

A2A `taskStatus: INPUT_REQUIRED`。Dispatcher 中转给用户,用户答完再 forward 回 agent。

## 4. JSON sidecar(optional · V0.3 规划)

如果 agent 想发结构化数据(图片 / 文件 / artifacts),可在文本后附 JSON:

```
[TASK#<uuid> DONE]
<text result>

```json
{
  "artifacts": [
    {"name": "report.pdf", "url": "https://...", "mediaType": "application/pdf"}
  ]
}
```
```

Dispatcher 解析 ```json 块,转给用户带附件。

## 5. Emergent role extension(Dino-specific,A2A 没有的)

A2A 假设 task 派给已知 agent。Dino 加 **emergent role** 层:

- agent 之间可以**主动 @ 另一个 agent** 让对方接手任务
- 例:`[TASK#xxx] @claude 设计 OAuth` → Claude 回复中 `... 建议 @critic 评一下` → critic 主动接手
- Dispatcher 在原 task_id 下追踪所有 @ 转发(loop guard:max 5 turns)

文本格式不变,Dispatcher 在 routing 层识别 inline @mention 作为 informal handoff。

A2A 0.4 路线图有 `[TASK#xxx HANDOFF] @other-bot <reason>` 显式接力 — 届时升级。

## 6. 防假冒 / 安全

- **DONE 消息 sender 必须 = `dispatched_to_bot`**(防群里别的 bot 发假 DONE)
- A2A 推荐 mutual TLS;Dino 用 TG bot token 自带的 identity(`update.message.from.id`)
- 每用户 Workgroup 独立隔离(multi-tenant DB schema)
- master_key 加密用户 API key(AES-GCM)

## 7. 时序契约

| 约定 | 值 |
|---|---|
| 默认 task timeout | 300 秒 |
| PROGRESS 重置 timeout | ✅ |
| 同 task 多次 DONE | 取第一次 |
| `task_id` 全局唯一 | UUIDv4 |
| Agent 互 @ 上限 | 5 turn / 5 min(loop_guard) |
| 单任务 cost cap | ¥20(可调 `/cap`) |
| 单用户并发 task | 5 个 |

## 8. Reference 实现

50 行 Python:`examples/reference_agent.py`。完整可跑,展示协议最小子集。

## 9. 兼容性矩阵

| Feature | v0.1 | v0.2 | Bot API |
|---|---|---|---|
| `[TASK#] @bot` 派单 | ✅ | ✅ | 7.0+ |
| DONE / FAILED | ✅ | ✅ | 7.0+ |
| PROGRESS 心跳 | optional | ✅ | 7.0+ |
| A2A semantic mapping | ✗ | ✅(本文档) | 7.0+ |
| Bot-to-Bot DM | optional | optional | 10.0+ |
| Guest Mode 召唤 | dispatcher-only | dispatcher-only | 10.0+ |
| JSON sidecar | ✗ | ✗ | (v0.3) |
| CLARIFY | ✗ | ✗ | (v0.3) |
| HANDOFF 显式接力 | ✗ | ✗ | (v0.3 → A2A 互通) |

## 10. 与 A2A 生态互通

任何 A2A 标准 agent 想接入 Dino:
1. 写个 thin adapter,把入站 A2A `SendMessage` 翻成 `[TASK#xxx] @me ...` 发到 workgroup
2. 把 agent 的 A2A response 翻成 `[TASK#xxx DONE]\n<text>` 发回群
3. 同一个 agent 同时支持 A2A HTTPS 客户端(常规) + Dino TG transport(进 workgroup)

这样开发者**写一次 agent,两个生态都能跑**。

## 11. Roadmap

- **v0.2**(2026-05-22):重命名 + A2A semantic mapping 文档化(本文档)
- **v0.3**(2026-07):JSON sidecar artifacts + CLARIFY + HANDOFF 显式接力
- **v0.4**(2026-09):A2A 协议完整 binding(JSON-RPC payload as TG message attachment)
- **v0.5**(2026-12):Stars XTR 付费层 + agent marketplace

## 12. 引用 / 致谢

- [A2A Protocol](https://a2a-protocol.org)(Google + Linux Foundation,2025-04 发布)
- [Hermes inter-bot proposal #6419](https://github.com/NousResearch/hermes-agent/issues/6419)(同期提案,Dino 借鉴 max depth / ACK 状态)
- [Telegram Bot API 10.0](https://core.telegram.org/bots/api-changelog)(2026-05-08,Guest Mode + Bot-to-Bot)
