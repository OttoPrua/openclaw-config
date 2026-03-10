# 微信群聊 AI 接入配置指南 / WeChat Group AI Integration Guide

> 将 OpenClaw Agent 接入微信群聊，通过 macOS 原生通知监控实现消息收发。  
> Integrate an OpenClaw Agent into WeChat group chats using macOS native notification monitoring.

---

## 🇨🇳 中文版

### 参考项目

本方案基于以下开源项目：

- **主要参考**：[ahaduoduoduo/openclaw-wechat-plugin](https://github.com/ahaduoduoduo/openclaw-wechat-plugin)
  - 原始方案：AppleScript 轮询 macOS 通知中心获取微信消息，再通过 UI 自动化发送回复
  - 原版针对较早版本 macOS，本指南记录了在 **macOS Tahoe（26.x）** 上运行所需的全部修改

---

### 前置条件

| 条件 | 说明 |
|------|------|
| macOS Tahoe 或以上 | 通知结构与旧版本有差异 |
| 微信桌面版 | 需在后台运行（**不可置于最前台**） |
| 微信通知权限 | 系统设置 → 通知 → 微信 → **允许通知**（必须开启） |
| OpenClaw | 已安装并运行，目标 agent 已配置 |

> ⚠️ 微信在前台时不会弹出通知，消息将无法被捕获。

---

### 实际操作所需修改（相对原插件共 7 项）

#### 修改 1 · macOS Tahoe 通知结构变更

原插件的 AppleScript 访问路径：
```
group 1 > scroll area 1 > group 1 > static text
```

macOS Tahoe 实际路径（多一层 group）：
```
group 1 > scroll area 1 > group 1 > group N > {static text 1, static text 2}
```

**影响**：需要遍历每个 `group N` 才能取到发送者和消息内容。

---

#### 修改 2 · WeChat 激活方式

原插件：
```applescript
tell application "WeChat" to activate  -- 无效
```

修复：
```applescript
tell application "System Events" to set frontmost of process "WeChat" to true
```

---

#### 修改 3 · 群聊识别

macOS 通知本身无法区分群聊和私聊。  
解决方案：解析通知 body 格式（群消息格式为 `发送者: 消息内容`）→ 判断为群聊（`chatType=group`）。

---

#### 修改 4 · 触发符替代 @mention

`@mention` 通知内容被截断为 `发送者在群聊中@了你`，丢失实际消息。  
解决方案：改用自定义全角字符作为触发符（配置项 `botTrigger`）。

> 注：由于同样原因，`botName`（本地微信账号名）对消息路由无实质影响，可留空。

---

#### 修改 5 · 输入框聚焦

原插件用 `Cmd+↓×2` 跳转，会跳到第三个聊天窗口。  
修复：改为 `Cmd+↓ → Cmd+↑` 聚焦第一个输入框。

---

#### 修改 6 · Session 绑定 Agent

原始 sessionKey 格式 `wechat:group:群名` 不含 agentId，会 fallback 到默认 agent。  
修复：sessionKey 改为 `agent:{agent-id}:wechat:group:群名`，正确路由到目标 agent。

---

#### 修改 7 · 群名白名单

新增 `allowedGroups` 配置项，限定只响应指定群聊，避免误触发。

---

### 完整配置步骤

#### Step 1 · 安装插件

```bash
cd ~/.openclaw/workspace/plugins
git clone https://github.com/ahaduoduoduo/openclaw-wechat-plugin
# 按上述 7 项修改内容修改 index.ts
```

---

#### Step 2 · 确认触发词 ⚠️

在写入配置之前，先确定你的两个触发词：

| 触发词 | 配置项 | 说明 | 示例 |
|--------|--------|------|------|
| **群绑定触发词** | `bindTrigger` | 在目标群里发这个词，将该群设为 agent 的激活群 | `！助手`、`！bind` |
| **消息触发词** | `botTrigger` | 群成员在消息里加上这个词，agent 才会回复 | `＆助手`、`＆ai` |

**选词建议：**
- 使用全角字符（如 `！` `＆`），减少误触发
- 两个触发词要能明显区分
- 触发词会出现在每条需要 agent 回复的消息里，建议简短易输入

**确认好触发词后**，将它们填入下方配置，再继续后续步骤。

---

#### Step 3 · 配置 openclaw.json

备份：
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak_$(date +%Y%m%d_%H%M%S)
```

在 `channels` 中添加（替换 `{...}` 占位符）：
```json
"wechat": {
  "enabled": true,
  "blockStreaming": true,
  "groupOnly": true,
  "botName": "",
  "botTrigger": "{你的消息触发词}",
  "bindTrigger": "{你的绑定触发词}",
  "agent": "{your-agent-id}",
  "allowedGroups": [],
  "allowedSenders": [],
  "rateLimitPerMinute": 20,
  "dailyTokenBudget": 50000
}
```

> `botName` 为本地微信登录账号名称。由于本方案不响应 @mention，此字段对消息路由无影响，留空即可。

在 `agents.list` 中添加：
```json
{
  "id": "{your-agent-id}",
  "workspace": "~/.openclaw/workspace-{your-agent-id}",
  "model": "anthropic/claude-sonnet-4-6",
  "tools": {
    "profile": "minimal",
    "deny": ["exec","read","write","edit","web_search","web_fetch","browser",
             "canvas","nodes","message","sessions_spawn","sessions_send",
             "sessions_list","sessions_history","subagents","session_status",
             "memory_search","memory_get","image","pdf","tts","agents_list"]
  }
}
```

#### Step 4 · 配置 Agent 人设

创建 `~/.openclaw/workspace-{your-agent-id}/` 目录，添加：
- `SOUL.md`：Agent 的人格定义（角色、语气、边界）
- `IDENTITY.md`：Agent 的身份信息

#### Step 5 · 系统权限

系统设置 → 通知 → 微信 → 开启**允许通知**。

#### Step 6 · 启动与绑定群聊

```bash
openclaw gateway restart
```

在目标微信群中发送你的 `{绑定触发词}`，agent 回复确认后即绑定成功。

---

### 日常使用

| 操作 | 方法 |
|------|------|
| 切换目标群 | 在新群发送 `{bindTrigger}` |
| 触发 Agent | 在已绑定群发送 `{botTrigger} 你的问题` |
| 查看当前绑定 | `grep "bound active group" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log \| tail -1` |

---

### 已知限制

- 通知文本最多约 65 字符，长文本被截断
- 微信在前台时不触发通知
- 发送时短暂占用前台和系统剪贴板（约 2 秒）
- Gateway 重启后动态绑定重置，需重新发绑定触发词
- 不支持图片、语音、视频内容
- 发送目标为微信当前激活窗口，需手动保持微信停在目标群聊

---

### 待优化项

- [ ] 自动搜索切换目标群聊
- [ ] chatlog 组件集成（绕过通知截断限制）
- [ ] 多群并发支持
- [ ] 发送时剪贴板内容保护

---

---

## 🇬🇧 English Version

### Reference Project

This setup is based on:

- **Primary reference**: [ahaduoduoduo/openclaw-wechat-plugin](https://github.com/ahaduoduoduo/openclaw-wechat-plugin)
  - Original approach: AppleScript polls macOS Notification Center for WeChat messages and uses UI automation to send replies
  - The original targets older macOS versions — this guide documents all changes required to run on **macOS Tahoe (26.x)**

---

### Prerequisites

| Requirement | Notes |
|-------------|-------|
| macOS Tahoe or later | Notification structure differs from earlier versions |
| WeChat Desktop | Must run in background (**not frontmost**) |
| WeChat notification permission | System Settings → Notifications → WeChat → **Allow notifications** (required) |
| OpenClaw | Installed and running, target agent configured |

> ⚠️ WeChat must not be the frontmost app — notifications only appear when WeChat is in the background.

---

### Required Modifications to the Original Plugin (7 total)

#### Fix 1 · macOS Tahoe Notification Structure

Original AppleScript path:
```
group 1 > scroll area 1 > group 1 > static text
```

Actual path on macOS Tahoe (one extra group layer):
```
group 1 > scroll area 1 > group 1 > group N > {static text 1, static text 2}
```

**Impact**: Must iterate over each `group N` to extract sender and message content.

---

#### Fix 2 · WeChat Window Activation

Original:
```applescript
tell application "WeChat" to activate  -- broken
```

Fix:
```applescript
tell application "System Events" to set frontmost of process "WeChat" to true
```

---

#### Fix 3 · Group Chat Detection

macOS notifications don't distinguish between group and private chats.  
Solution: parse notification body format (`sender: message content` → classify as group chat, `chatType=group`).

---

#### Fix 4 · Trigger Keyword Replaces @mention

@mention notifications are truncated to `sender mentioned you in a group`, losing actual content.  
Solution: use a custom fullwidth character as a trigger keyword (config key `botTrigger`).

> Note: For the same reason, `botName` (your local WeChat account name) has no effect on message routing and can be left empty.

---

#### Fix 5 · Input Field Focus

Original used `Cmd+↓×2`, which jumped to the third chat window.  
Fix: use `Cmd+↓ → Cmd+↑` to focus the first input field.

---

#### Fix 6 · Session-to-Agent Binding

Original sessionKey format `wechat:group:groupName` doesn't include agentId → falls back to the default agent.  
Fix: sessionKey changed to `agent:{agent-id}:wechat:group:groupName` → correctly routes to target agent.

---

#### Fix 7 · Group Allowlist

Added `allowedGroups` config to restrict response to specific groups and avoid false triggers.

---

### Full Setup Steps

#### Step 1 · Install the plugin

```bash
cd ~/.openclaw/workspace/plugins
git clone https://github.com/ahaduoduoduo/openclaw-wechat-plugin
# Apply the 7 fixes above to index.ts
```

---

#### Step 2 · Decide your trigger keywords ⚠️

Before writing any config, decide on your two trigger keywords:

| Keyword | Config key | Purpose | Examples |
|---------|------------|---------|---------|
| **Bind trigger** | `bindTrigger` | Send this in a group to make it the agent's active group | `！bind`, `！assistant` |
| **Message trigger** | `botTrigger` | Include this in a message to invoke the agent | `＆ai`, `＆assistant` |

**Tips:**
- Use fullwidth characters (e.g. `！` `＆`) to reduce accidental triggers
- Make the two keywords clearly distinct
- Keep them short — users will type the message trigger in every request

**Fill in your chosen keywords before continuing.**

---

#### Step 3 · Configure openclaw.json

Backup first:
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak_$(date +%Y%m%d_%H%M%S)
```

Add to `channels` (replace `{...}` placeholders):
```json
"wechat": {
  "enabled": true,
  "blockStreaming": true,
  "groupOnly": true,
  "botName": "",
  "botTrigger": "{your message trigger}",
  "bindTrigger": "{your bind trigger}",
  "agent": "{your-agent-id}",
  "allowedGroups": [],
  "allowedSenders": [],
  "rateLimitPerMinute": 20,
  "dailyTokenBudget": 50000
}
```

> `botName` is your local WeChat account name. Since this setup does not respond to @mentions (mention notifications are truncated and unusable), this field has no effect on routing and can be left empty.

Add to `agents.list`:
```json
{
  "id": "{your-agent-id}",
  "workspace": "~/.openclaw/workspace-{your-agent-id}",
  "model": "anthropic/claude-sonnet-4-6",
  "tools": {
    "profile": "minimal",
    "deny": ["exec","read","write","edit","web_search","web_fetch","browser",
             "canvas","nodes","message","sessions_spawn","sessions_send",
             "sessions_list","sessions_history","subagents","session_status",
             "memory_search","memory_get","image","pdf","tts","agents_list"]
  }
}
```

#### Step 4 · Configure the agent persona

Create `~/.openclaw/workspace-{your-agent-id}/` and add:
- `SOUL.md`: personality definition (role, tone, boundaries)
- `IDENTITY.md`: agent identity info

#### Step 5 · System permissions

System Settings → Notifications → WeChat → enable **Allow notifications**.

#### Step 6 · Start and bind a group

```bash
openclaw gateway restart
```

In your target WeChat group, send your `{bindTrigger}`. When the agent replies to confirm, the group is bound.

---

### Daily Usage

| Action | How |
|--------|-----|
| Switch target group | Send `{bindTrigger}` in the new group |
| Invoke agent | Send `{botTrigger} your question` in the bound group |
| Check current binding | `grep "bound active group" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log \| tail -1` |

---

### Known Limitations

- Notification text truncated at ~65 characters
- WeChat must be in background for notifications to appear
- Sending briefly takes over the foreground and clipboard (~2 seconds)
- Dynamic binding resets after gateway restart — re-send bind trigger
- Images, voice, and video are not supported
- Sends to whichever WeChat chat is currently active — keep WeChat on the target group

---

### Roadmap

- [ ] Auto-navigate to target group before sending
- [ ] chatlog component integration (bypass notification truncation)
- [ ] Multi-group concurrent support
- [ ] Clipboard preservation during send

---

*Last updated: 2026-03-10 · Maintained by [OttoPrua](https://github.com/OttoPrua)*
