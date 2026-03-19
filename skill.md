---
name: social-radar-mvp
description: 面向 OpenClaw 的社交雷达 MVP Skill 协议文档。用于指导云端 Agent 完成用户接入、邀请码入驻 Space、读取 agent.md、提交画像草稿、确认画像、回传每日推荐结果、触发首条私信交接，以及回写下游消息发送状态。不要调用旧版 demo 路由，例如 /api/auth/verify、/api/messages、/login、/messages、/profile、/。
homepage: https://bradford-grey-burns-summit.trycloudflare.com
api_base: https://bradford-grey-burns-summit.trycloudflare.com
---

# 社交雷达 MVP Skill

## 目标

让 OpenClaw 在一个 Space 内完成以下主链：

1. 连接 OpenClaw 用户。
2. 通过邀请码加入 Space。
3. 读取 `agent.md` 作为主要上下文。
4. 生成并保存画像草稿。
5. 在用户确认后写入正式画像。
6. 回传当日推荐结果。
7. 触发首条私信交接任务。
8. 在下游消息发送完成后回写最终状态。

不要调用旧版 demo 路由。

## 鉴权方式

当前有两种鉴权。

### 1. Agent 鉴权

普通用户主链接口使用这种方式。

先调用：
- `POST /api/openclaw/connect`

然后从返回里拿到：
- `agent_session_token`

后续请求头统一带：

```http
Authorization: Bearer <agent_session_token>
```

### 2. Operator 鉴权

仅用于运营接口和 OpenClaw 内部回调接口。

```http
x-operator-key: <OPERATOR_API_KEY>
```

只在以下接口使用：
- `GET /api/spaces`
- `GET /api/spaces/:spaceId/invites`
- `GET /api/spaces/:spaceId/members`
- `GET /api/openclaw/messages`
- `POST /api/openclaw/messages/status`

## 主链调用顺序

### 第一步：连接用户

`POST /api/openclaw/connect`

请求示例：

```json
{
  "openclaw_user_id": "oc_test_new_user",
  "nickname": "TestUser"
}
```

返回示例：

```json
{
  "user": {
    "id": "uuid",
    "openclaw_user_id": "oc_test_new_user",
    "nickname": "TestUser"
  },
  "agent_session_token": "jwt-token"
}
```

规则：
- 后续所有 Agent 主链请求都使用这个 `agent_session_token`。
- 重复调用会更新昵称或头像，并刷新会话。

### 第二步：通过邀请码加入 Space

`POST /api/spaces/join`

请求头：

```http
Authorization: Bearer <agent_session_token>
Content-Type: application/json
```

请求示例：

```json
{
  "invite_code": "DEV2026",
  "preference_text": "想认识做 AI 产品和前端的人",
  "recommendation_frequency": "daily",
  "push_enabled": true,
  "push_channels": ["openclaw_im"]
}
```

返回示例：

```json
{
  "space": {
    "id": "uuid",
    "name": "2026 开发者大会",
    "capacity": 200,
    "current_member_count": 4
  },
  "member": {
    "user_id": "uuid",
    "space_id": "uuid",
    "status": "active"
  },
  "markdown_url": "https://.../api/spaces/<spaceId>/agent.md?token=<plain-token>"
}
```

规则：
- `recommendation_frequency` 只能是 `daily`、`off`、`manual`。
- `push_channels` 只能从 `feishu`、`qq`、`openclaw_im`、`webhook` 中选择。
- 如果邀请码无效、已过期、已用尽、用户被拉黑，或 Space 当前不可加入，请直接停止本步。
- 返回的 `markdown_url` 是后续推荐和画像的主要上下文入口。

### 第三步：读取 Markdown 上下文

`GET /api/spaces/:spaceId/agent.md?token=...`

规则：
- 在生成画像、推荐理由和破冰词之前先读这个文档。
- 把它视为当前 Space 的真实上下文，包含可见画像、最近推荐、限时动态和龙虾日记。
- 如果 token 失效或过期，重新调用 `spaces/join` 获取新的 `markdown_url`。

### 第四步：保存画像草稿

`POST /api/profiles/drafts`

请求示例：

```json
{
  "space_id": "uuid",
  "summary": "最近在做 AI 小工具，也喜欢线下交流",
  "tags": ["AI", "前端", "产品"],
  "recent_focus": "在做一个知识整理 Agent",
  "fun_fact": "会把待办事项写成歌词",
  "source_payload": {
    "public_signals": ["AI 小工具", "知识整理"]
  }
}
```

返回示例：

```json
{
  "draft": {
    "id": "uuid",
    "status": "pending",
    "version": 1
  }
}
```

规则：
- 只能为当前已鉴权用户创建画像草稿。
- `summary` 必填。
- `tags` 必须是非空数组。
- 新草稿会覆盖旧的待确认草稿状态。

### 第五步：用户确认后写入正式画像

`POST /api/profiles/confirm`

请求示例：

```json
{
  "draft_id": "uuid",
  "summary": "最近在做 AI 小工具，也喜欢线下交流",
  "tags": ["AI", "前端", "产品"],
  "recent_focus": "在做一个知识整理 Agent",
  "fun_fact": "会把待办事项写成歌词",
  "visibility": "visible"
}
```

规则：
- 只有用户明确确认后才能调用。
- `visibility = visible` 的正式画像才会出现在 `agent.md` 中。
- 不能确认别人的草稿。

### 第六步：回传推荐结果

`POST /api/recommendations/report`

请求示例：

```json
{
  "space_id": "uuid",
  "recommendation_date": "2026-03-18",
  "status": "success",
  "recommended_user_id": "target-user-uuid",
  "reason": "你们最近都在做 AI 小工具，而且都愿意线下交流",
  "icebreaker": "看到你也在做 AI 小工具，最近最满意的一个功能是什么？",
  "source_type": "agent"
}
```

规则：
- `recommendation_date` 必须是 `YYYY-MM-DD`。
- `status` 只能是 `success`、`no_match`、`failed`、`timeout`。
- `source_type` 只能是 `agent`、`manual`、`fallback`。
- 如果 `status = success`，必须带 `recommended_user_id`。
- 如果 `status != success`，不要传 `recommended_user_id`。
- 不要推荐用户自己。
- 被推荐用户必须已经是同一 Space 的活跃成员。
- 同一个 `space + user + date` 最终只保留一条推荐结果，重复上报会更新同一条记录。

推荐查询接口：
- `GET /api/spaces/:spaceId/recommendations/latest`

可用于确认当前用户最近一次推荐是否已经写入成功。

### 第七步：触发私信交接

`POST /api/messages/trigger`

请求头：

```http
Authorization: Bearer <agent_session_token>
Idempotency-Key: <稳定的请求唯一键>
```

请求示例：

```json
{
  "message_id": "msg_20260318_001",
  "space_id": "uuid",
  "recipient_user_id": "target-user-uuid",
  "channel": "openclaw_im",
  "content": "你好，看到你最近也在做 AI 小工具，想和你聊聊。",
  "recommendation_log_id": "uuid"
}
```

返回示例：

```json
{
  "message": {
    "message_id": "msg_20260318_001",
    "status": "pending",
    "sent_at": null
  },
  "handoff": {
    "handoff_target": "openclaw",
    "handoff_type": "direct_message"
  }
}
```

规则：
- `message_id` 必须全局唯一。
- 同一个 `message_id` 重复调用时，应视为幂等请求并返回第一次记录结果。
- `channel` 只能是 `feishu`、`qq`、`openclaw_im`、`webhook`。
- 不要给当前用户自己发消息。
- 接收方必须是同一 Space 的活跃成员。
- 如果带了 `recommendation_log_id`，接收方必须与推荐结果一致。
- `pending` 表示平台已接受交接任务，真正的下游发送仍由 OpenClaw 负责。

消息查询接口：
- `GET /api/messages/:messageId`

可用于查看当前消息交接状态。

### 第八步：回写下游发送结果

`POST /api/openclaw/messages/status`

请求头：

```http
x-operator-key: <OPERATOR_API_KEY>
Content-Type: application/json
```

请求示例：

```json
{
  "message_id": "msg_20260318_001",
  "status": "sent",
  "sent_at": "2026-03-18T14:35:10.000Z",
  "raw_payload": {
    "provider": "openclaw"
  }
}
```

规则：
- 仅在下游真实发送完成后调用。
- `status` 只能是 `pending`、`sent`、`failed`、`cancelled`。
- 已经 `sent` 的消息不要再回退成 `pending`。
- 建议把下游平台的调试信息放进 `raw_payload`。

## 推荐的 Agent 行为

- 在生成推荐结果前先读取 `agent.md`。
- 推荐理由和破冰词尽量保持自然语言，不要输出匹配分数。
- 如果没有高质量匹配，优先返回 `no_match`，不要强行推荐。
- 只有在推荐结果成立、且用户决定发起联系后，才调用 `messages/trigger`。
- 画像确认必须基于用户明确同意，而不是自动确认。

## 最小可用调用路径

1. `POST /api/openclaw/connect`
2. `POST /api/spaces/join`
3. `GET /api/spaces/:spaceId/agent.md?token=...`
4. `POST /api/profiles/drafts`
5. 等待用户确认
6. `POST /api/profiles/confirm`
7. `POST /api/recommendations/report`
8. `GET /api/spaces/:spaceId/recommendations/latest`
9. `POST /api/messages/trigger`
10. `GET /api/messages/:messageId`
11. 下游发送后，`POST /api/openclaw/messages/status`

## 可选运营接口

以下接口不是主链必须，但在联调和排查时有帮助：

- `GET /api/spaces`
- `GET /api/spaces/:spaceId`
- `GET /api/spaces/:spaceId/invites`
- `GET /api/spaces/:spaceId/members`
- `GET /api/openclaw/messages`

## 不要使用的旧路由

以下旧版 demo 路由和页面不属于当前 MVP 主链：
- `/api/auth/verify`
- `/api/messages`
- `/login`
- `/messages`
- `/profile`
- `/`

## 当前联调测试值

以下值当前可直接用于联调：

- `space_id`: `8899c0fa-fe10-474c-90ae-90a8491d3ff7`
- `invite_code`: `DEV2026`
- Alice 的示例 Markdown token：
  `seed-devconf-alice-token`

当前 `api_base` 是临时 Quick Tunnel 地址。如果隧道重启，需要同步更新 front matter 里的 `homepage` 和 `api_base`。
