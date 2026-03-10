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
| OpenClaw | 已安装并运行，`rossi` agent 已配置 |

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
解决方案：改用全角字符 `＆洛茜` 作为触发符（配置项 `botTrigger`）。

---

#### 修改 5 · 输入框聚焦

原插件用 `Cmd+↓×2` 跳转，会跳到第三个聊天窗口。  
修复：改为 `Cmd+↓ → Cmd+↑` 聚焦第一个输入框。

---

#### 修改 6 · Session 绑定 Agent

原始 sessionKey 格式 `wechat:group:群名` 不含 agentId，会 fallback 到默认 agent（perlica）。  
修复：sessionKey 改为 `agent:rossi:wechat:group:群名`，正确路由到洛茜。

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

#### Step 2 · 配置 openclaw.json

备份：
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak_$(date +%Y%m%d_%H%M%S)
```

在 `channels` 中添加：
```json
"wechat": {
  "enabled": true,
  "blockStreaming": true,
  "groupOnly": true,
  "botName": "扫拖一体🤖",
  "botTrigger": "＆洛茜",
  "bindTrigger": "！洛茜",
  "agent": "rossi",
  "allowedGroups": ["你的群名"],
  "allowedSenders": [],
  "rateLimitPerMinute": 20,
  "dailyTokenBudget": 50000
}
```

在 `agents.list` 中添加：
```json
{
  "id": "rossi",
  "workspace": "~/.openclaw/workspace-rossi",
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

#### Step 3 · 配置洛茜人设

创建 `~/.openclaw/workspace-rossi/` 目录，添加：
- `SOUL.md`：洛茜的人格定义（角色、语气、边界）
- `IDENTITY.md`：洛茜的身份信息

#### Step 4 · 系统权限

系统设置 → 通知 → 微信 → 开启**允许通知**。

#### Step 5 · 启动与绑定群聊

```bash
openclaw gateway restart
```

在目标微信群中发送 `！洛茜`（全角感叹号），洛茜回复确认后即绑定成功。

---

### 日常使用

| 操作 | 方法 |
|------|------|
| 切换目标群 | 在新群发送 `！洛茜` |
| 触发洛茜 | 在已绑定群发送 `＆洛茜 你的问题` |
| 查看当前绑定 | `grep "bound active group" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log \| tail -1` |

---

### 已知限制

- 通知文本最多约 65 字符，长文本被截断
- 微信在前台时不触发通知
- 发送时短暂占用前台和系统剪贴板（约 2 秒）
- Gateway 重启后动态绑定重置，需重新发 `！洛茜`
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
| OpenClaw | Installed and running, `rossi` agent configured |

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
Solution: use fullwidth `＆洛茜` as a trigger keyword (config key `botTrigger`).

---

#### Fix 5 · Input Field Focus

Original used `Cmd+↓×2`, which jumped to the third chat window.  
Fix: use `Cmd+↓ → Cmd+↑` to focus the first input field.

---

#### Fix 6 · Session-to-Agent Binding

Original sessionKey format `wechat:group:groupName` doesn't include agentId → falls back to the default agent (perlica).  
Fix: sessionKey changed to `agent:rossi:wechat:group:groupName` → correctly routes to rossi.

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

#### Step 2 · Configure openclaw.json

Backup first:
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak_$(date +%Y%m%d_%H%M%S)
```

Add to `channels`:
```json
"wechat": {
  "enabled": true,
  "blockStreaming": true,
  "groupOnly": true,
  "botName": "SweepBot🤖",
  "botTrigger": "＆rossi",
  "bindTrigger": "！rossi",
  "agent": "rossi",
  "allowedGroups": ["your-group-name"],
  "allowedSenders": [],
  "rateLimitPerMinute": 20,
  "dailyTokenBudget": 50000
}
```

Add to `agents.list`:
```json
{
  "id": "rossi",
  "workspace": "~/.openclaw/workspace-rossi",
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

#### Step 3 · Configure the agent persona

Create `~/.openclaw/workspace-rossi/` and add:
- `SOUL.md`: personality definition (role, tone, boundaries)
- `IDENTITY.md`: agent identity info

#### Step 4 · System permissions

System Settings → Notifications → WeChat → enable **Allow notifications**.

#### Step 5 · Start and bind a group

```bash
openclaw gateway restart
```

In your target WeChat group, send `！rossi` (fullwidth exclamation). When rossi replies to confirm, the group is bound.

---

### Daily Usage

| Action | How |
|--------|-----|
| Switch target group | Send `！rossi` in the new group |
| Trigger rossi | Send `＆rossi your question` in the bound group |
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
