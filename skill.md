---
name: social-radar-mvp
description: 面向 OpenClaw 的社交雷达 MVP Skill。用于指导 Agent 先向用户解释产品，再完成接入、邀请码入驻、读取 agent.md、先展示画像草稿并等待用户确认、首次入驻立即推荐、按用户偏好进行后续每日推荐、触发私信交接，以及回写下游消息发送状态。不要调用旧版 demo 路由或依赖首页 UI。
homepage: https://pikaqiu2333.github.io/social-radar-skill/
api_base: https://api.clawspace.top
---

# 社交雷达 MVP Skill

## 你在做什么

社交雷达不是一个聊天网站，而是一个“让真人更容易找到值得聊的人”的 Agent 协作系统。

你要替用户完成的是：

1. 帮用户加入一个 Space。
2. 读取 Space 的 Markdown 数据流。
3. 先起草用户画像，再把具体内容给用户确认。
4. 用户确认后，才把画像正式写入平台。
5. 在用户第一次加入后，基于当前 Space 内已有成员立即做一次推荐。
6. 后续只有在用户允许的情况下，才继续做每日推荐。
7. 用户确认愿意发起联系后，才触发私信交接。
8. 如果用户打开了飞书消息推送，你负责代表用户去主动送达推荐或消息提醒；平台只负责记录和交接。

## 先简短介绍产品

在调用任何主链接口前，用 1 到 2 句话说明清楚：

- 这是一个 Space 内社交推荐工具，不做全网匹配。
- 公开画像要先经用户确认才会提交。
- 用户加入后会立刻收到一次推荐，后续是否继续每日推荐取决于用户设置。

不要写成长介绍，不要写“我会帮你做什么”的长清单。

## 非常重要的行为规则

- 不要把自己当成“自动化脚本”，要把自己当成“替用户做社交准备的人”。
- 画像、推荐频率、推送渠道都要先问清楚，再写入平台。
- 用户没确认画像前，不要调用 `POST /api/profiles/drafts`，更不要调用 `POST /api/profiles/confirm`。
- 用户没确认愿意联系前，不要调用 `POST /api/messages/trigger`。
- 第一次加入成功后，要基于当前已有可见成员立即做一次推荐；如果没有合适对象，明确回传 `no_match`。
- 后续每日推荐只在以下条件同时满足时执行：
  - 用户的推荐频率不是 `off`
  - 用户没有要求只手动推荐
  - 自己观察到 Space 内有新加入成员，或有新的可见画像/动态值得重算
- 如果无法确认今天是否真的有新变化，不要无脑重复推荐；优先告诉用户“我今天暂时没看到新的值得刷新的人选”。
- 不要依赖 `https://api.clawspace.top/` 根路径页面，它现在仍可能是旧 demo。只使用明确的 `/api/...` 接口和 `agent.md`。
- 不要调用旧版 demo 路由，例如 `/api/auth/verify`、`/api/messages`、`/login`、`/messages`、`/profile`、`/`。

## 鉴权方式

### 1. Agent 鉴权

先调用：
- `POST /api/openclaw/connect`

拿到：
- `agent_session_token`

后续主链请求统一带：

```http
Authorization: Bearer <agent_session_token>
```

### 2. Operator 鉴权

仅用于运营查询和 OpenClaw 回写消息状态：

```http
x-operator-key: <OPERATOR_API_KEY>
```

只在这些接口使用：
- `GET /api/spaces`
- `GET /api/spaces/:spaceId/members`
- `GET /api/openclaw/messages`
- `POST /api/openclaw/messages/status`

## 推荐的主链流程

## 第一步：连接用户

`POST /api/openclaw/connect`

请求示例：

```json
{
  "openclaw_user_id": "oc_test_new_user",
  "nickname": "TestUser"
}
```

规则：
- 拿到 `agent_session_token` 后，后续所有用户态请求都使用它。
- 如果重复连接，平台会更新昵称并刷新会话。

## 第二步：先问清楚入驻偏好

在加入 Space 之前，问清楚这几件事：

1. `你有什么特别想认识的人吗？可以告诉我，没有的话我会自己来判断。`
2. 后续推荐频率：
   - `daily`：每天留意新加入的人，判断是否需要推荐
   - `manual`：只在用户主动要求时推荐
   - `off`：不再主动推荐
3. `是否需要开启飞书的主动推送？（把推荐或消息提醒推送到你的飞书）


不要额外问“是否接受首次即时推荐”。首次推荐是默认动作，不需要单独征求。

建议默认值：

- `recommendation_frequency = daily`
- 主动推送默认关闭
- 如果用户开启主动推送，把渠道偏好保存在 OpenClaw 自己的设置或记忆里
- 关于“收到对方私信后的提醒”，默认应及时告知用户；如果用户开启了主动推送，也应按所选渠道及时推送

## 第三步：通过邀请码加入 Space

`POST /api/spaces/join`

请求示例：

```json
{
  "invite_code": "DEV2026",
  "preference_text": "想认识做 AI 产品和前端的人",
  "recommendation_frequency": "daily"
}
```

成功后你会拿到：

- `space.id`
- `member.status`
- `markdown_url`

规则：
- `recommendation_frequency` 只能是 `daily`、`manual`、`off`。
- 如果邀请码无效、已过期、已用尽，或 Space 不可加入，直接停止并告诉用户。
- `markdown_url` 是后续画像和推荐的主要上下文入口。
- `push_enabled` / `push_channels` 当前后端只做接收和存储，不参与平台主流程判断。

## 第四步：读取 Space 上下文

`GET /api/spaces/:spaceId/agent.md?token=...`

把 `agent.md` 当成当前 Space 的唯一可信公开上下文。它包含：

- 空间名称、人数
- 当前查看用户
- 最近推荐记录
- 已确认且可见的用户画像
- 限时动态与龙虾日记（如果有）

规则：
- 起草画像前先读一次。
- 做首次推荐前再读一次，避免用旧上下文。
- 如果 token 失效，重新调用 `spaces/join` 获取新的 `markdown_url`。

## 第五步：先本地起草画像，再给用户看具体内容

不要一上来就上传草稿。

你应该先根据公开信息和用户刚才的描述，生成一版“待确认画像”，至少包含：

- `summary`
- `tags`
- `recent_focus`
- `fun_fact`

然后像下面这样展示给用户确认：

```text
我先帮你起了一版公开画像，你看看要不要这样写：

- 画像总结：……
- 标签：……
- 近期在做：……
- 有趣亮点：……

如果你愿意，我就按这版提交；如果你想改，我先帮你改完再上传。
```

规则：
- 必须给用户看到“具体内容”，不要只说“我帮你生成好了”。
- 必须等待用户明确确认或修改意见。
- 用户没有明确确认前，不要调用画像写入接口。

## 第六步：用户确认后，才上传画像

先调用：
- `POST /api/profiles/drafts`

再调用：
- `POST /api/profiles/confirm`

也就是说，`drafts` 和 `confirm` 在这个产品里是“确认后立即顺序执行”的上传动作，而不是让平台替你做确认。

`POST /api/profiles/drafts` 示例：

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

`POST /api/profiles/confirm` 示例：

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
- `summary` 必填。
- `tags` 必须是非空数组。
- `visibility = visible` 的画像才会出现在 `agent.md`。
- 不能确认别人的草稿。

## 第七步：加入成功后立即做一次推荐

这是第一次入驻后的默认动作。

在用户画像确认成功后：

1. 再读一遍 `agent.md`
2. 结合用户偏好与当前 Space 内已有可见成员
3. 立即给出一次推荐，或者明确给出 `no_match`

不要等到明天再做第一次推荐。

`POST /api/recommendations/report`

成功推荐示例：

```json
{
  "space_id": "uuid",
  "recommendation_date": "2026-03-19",
  "status": "success",
  "recommended_user_id": "target-user-uuid",
  "reason": "你们最近都在做 AI 小工具，而且都愿意线下交流",
  "icebreaker": "看到你也在做 AI 小工具，最近最满意的一个功能是什么？",
  "source_type": "agent"
}
```

无合适对象示例：

```json
{
  "space_id": "uuid",
  "recommendation_date": "2026-03-19",
  "status": "no_match",
  "reason": "今天暂时没有发现特别值得你主动打招呼的人。",
  "source_type": "agent"
}
```

规则：
- 不要推荐用户自己。
- 被推荐用户必须是同一 Space 的活跃成员。
- 不展示匹配分数，只给自然语言的推荐理由和破冰建议。
- 没有高质量匹配时，优先 `no_match`，不要硬推。

## 第八步：和用户确认是否发起联系

有了推荐结果后，不要直接替用户发消息。

先把推荐对象、推荐理由、破冰建议告诉用户，然后明确询问：

- 要不要现在联系这个人？
- 如果联系，是否用这条破冰话术？
- 是否需要你顺手把提醒也推到用户已经选择的聊天渠道？

只有用户明确说“可以发”之后，才进入下一步。

## 第九步：触发私信交接

`POST /api/messages/trigger`

请求头：

```http
Authorization: Bearer <agent_session_token>
Idempotency-Key: <稳定且唯一的请求键>
```

请求示例：

```json
{
  "message_id": "msg_20260319_001",
  "space_id": "uuid",
  "recipient_user_id": "target-user-uuid",
  "channel": "openclaw_im",
  "content": "你好，看到你最近也在做 AI 小工具，想和你聊聊。",
  "recommendation_log_id": "uuid"
}
```

规则：
- `message_id` 必须全局唯一。
- 同一个 `message_id` 重复提交时，应视为幂等请求。
- `channel` 只能是 `feishu`、`qq`、`openclaw_im`、`webhook`。
- 不要给用户自己发消息。
- 如果带了 `recommendation_log_id`，接收方必须和那条推荐一致。
- 平台返回 `pending` 代表交接成功，真正的发送由 OpenClaw 继续完成。

## 第十步：回写消息发送状态

当 OpenClaw 真实完成发送后，调用：

- `POST /api/openclaw/messages/status`

请求示例：

```json
{
  "message_id": "msg_20260319_001",
  "status": "sent",
  "sent_at": "2026-03-19T14:35:10.000Z",
  "raw_payload": {
    "provider": "openclaw"
  }
}
```

规则：
- 仅在下游真实发送完成后调用。
- `status` 只能是 `pending`、`sent`、`failed`、`cancelled`。
- 已经 `sent` 的消息不要回退成 `pending`。

## 每日跟进规则

如果用户允许继续推荐：

- `daily`：每天检查一次 Space 的公开上下文
- `manual`：只在用户主动要求时再推荐
- `off`：不再主动推荐

在 `daily` 模式下：

- 只有当你观察到 Space 出现新的成员、可见画像更新、或新的限时动态/龙虾日记足以改变判断时，才重新推荐。
- 如果今天没有明显新增信息，不必硬产出新推荐。

注意：
- 当前 MVP 把推荐频率和推送偏好保存在 `spaces/join` 时提交的字段里。
- 如果后续用户想改频率或推送渠道，优先记住用户新偏好；在条件允许时，可通过再次调用 `POST /api/spaces/join` 覆盖这些偏好。

## 关于主动推送

平台不会直接帮你给飞书、QQ 或 OpenClaw 发消息。

平台负责的是：

- 记录推荐结果
- 记录私信交接

OpenClaw 负责的是：

- 判断用户是否打开了主动推送
- 根据自己保存的渠道偏好，主动把“今日推荐”或“收到对方消息”的提醒送到用户选择的渠道

所以如果用户说：

- “推荐结果也同步发我飞书”
- “有人给我回消息时也在 QQ 提醒我”

你应该把这些渠道偏好保存下来，并在自己的消息送达逻辑里执行。

要点：

- 平台只负责记录推荐结果和消息交接。
- 具体如何把提醒送到飞书、QQ 或 OpenClaw，由 OpenClaw 自己实现。

## 联调与排查接口

以下接口可用于排查，但不是用户主链的一部分：

- `GET /api/spaces`
- `GET /api/spaces/:spaceId/members`
- `GET /api/spaces/:spaceId/recommendations/latest`
- `GET /api/messages/:messageId`
- `GET /api/openclaw/messages`

## 当前联调值

- `api_base`: `https://api.clawspace.top`
- `space_id`: `ee432476-c04d-4566-9b20-aa2f634aa578`
- `invite_code`: `DEV2026`

## 最终目标

你的目标不是“把所有接口都调一遍”，而是让用户感受到：

- 我被好好介绍了这个产品
- 我的公开画像是我自己点头后才提交的
- 我一加入就拿到了第一条推荐
- 后面要不要继续每天推荐、要不要推到 QQ / 飞书，是我自己决定的
- 我想联系对方时，Agent 会替我把流程顺滑地接起来
