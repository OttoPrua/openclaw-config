---
name: openclaw-wechat-bot
description: Connect an OpenClaw Agent to WeChat group chats via macOS notification monitoring — receive group messages and send AI-powered replies automatically.
---

# openclaw-wechat-bot

管理微信插件（macOS 原生通知方案）+ 绑定 Agent 的日常运维。  
Manage the WeChat plugin (macOS native notification approach) + daily ops for the bound Agent.

> **Credits**: Based on [ahaduoduoduo/openclaw-wechat-plugin](https://github.com/ahaduoduoduo/openclaw-wechat-plugin). Thanks to the original author for the macOS notification monitoring approach.

---

## 架构概览 / Architecture

```
微信群消息 / WeChat group message
  → macOS 通知弹窗 / macOS notification
  → AppleScript 轮询（0.5s）/ AppleScript poll (0.5s)
  → 解析群名/发送者/内容 / parse group name, sender, content
  → 绑定指令（{bindTrigger}）/ 触发符（{botTrigger}）检测
    bind command ({bindTrigger}) / trigger keyword ({botTrigger}) detection
  → dispatch 到绑定 agent / dispatch to bound agent
  → 回复 → 激活微信 → 粘贴+发送 → 切后台
    reply → activate WeChat → paste+send → switch to background
```

## 前置条件 / Prerequisites

- macOS Tahoe，微信桌面版在后台运行 / macOS Tahoe, WeChat Desktop running in background
- 系统设置 → 通知 → 微信 → **允许通知**（必须）/ System Settings → Notifications → WeChat → **Allow notifications** (required)
- 微信**不能在最前面**（前台不弹通知）/ WeChat must **not** be frontmost (no notifications when in foreground)
- 微信窗口需停留在目标群聊（发送到当前激活窗口）/ WeChat window must stay on the target group (sends to active window)
- **推荐**：通知提醒样式设为「持续」（系统设置 → 通知 → 微信 → 提醒样式 → 持续），避免通知自动消失导致漏消息 / **Recommended**: Set alert style to "Persistent" (System Settings → Notifications → WeChat → Alert style → Persistent) to prevent notifications from auto-dismissing

## 关键路径 / Key Paths

| 文件 / File | 用途 / Purpose |
|------|------|
| `~/.openclaw/workspace/plugins/openclaw-wechat-plugin/index.ts` | 插件主代码 / Plugin main code |
| `~/.openclaw/workspace-{agent}/SOUL.md` | Agent 人设 / Agent persona |
| `~/.openclaw/workspace-{agent}/IDENTITY.md` | Agent 身份 / Agent identity |
| `~/.openclaw/openclaw.json` → `channels.wechat` | 微信频道配置 / WeChat channel config |
| `~/.openclaw/openclaw.json` → `agents.list[{agent-id}]` | Agent 配置 / Agent config |

## 配置参数 / Configuration

```json
{
  "channels": {
    "wechat": {
      "enabled": true,
      "blockStreaming": true,
      "groupOnly": true,
      "botName": "",
      "botTrigger": "{botTrigger}",
      "bindTrigger": "{bindTrigger}",
      "agent": "{your-agent-id}",
      "allowedGroups": [],
      "allowedSenders": [],
      "rateLimitPerMinute": 20,
      "dailyTokenBudget": 50000
    }
  }
}
```

### 参数说明 / Parameter Reference

| 参数 / Param | 说明 / Description |
|------|------|
| `botName` | 当前微信登录账号的名称。由于本方案不响应 @mention（通知内容被截断），此字段对消息路由无影响，可留空。/ Your local WeChat account name. Since this setup does not respond to @mentions (notification content is truncated), this field has no effect on routing and can be left empty. |
| `botTrigger` | 群聊触发符（建议全角字符），消息包含此触发符才会被 agent 处理 / Message trigger keyword (fullwidth recommended). Agent only processes messages containing this keyword. |
| `bindTrigger` | 群绑定指令（建议全角字符），在群里发此指令将该群设为激活群 / Bind command (fullwidth recommended). Send this in a group to make it the active group. |
| `agent` | 绑定的 agent ID / Bound agent ID |
| `allowedGroups` | 群名白名单（空=仅依赖动态绑定）/ Group allowlist (empty = dynamic binding only) |
| `rateLimitPerMinute` | 每分钟最大回复数 / Max replies per minute |
| `dailyTokenBudget` | 每日 token 预算 / Daily token budget |

## 日常操作 / Daily Operations

### 切换目标群聊 / Switch Target Group
在目标群里发送 `{bindTrigger}`，agent 回复确认后生效。  
Send `{bindTrigger}` in the target group. Effective after agent confirms.

### 触发 Agent 回复 / Invoke Agent
在已绑定的群里发送 `{botTrigger} 你的问题`，agent 会回复。  
Send `{botTrigger} your question` in the bound group to get a reply.

### 查看当前绑定 / Check Current Binding
```bash
grep "bound active group" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | tail -1
```

### 查看最近消息流 / View Recent Messages
```bash
grep "wechat" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | tail -20 | python3 -c "
import json, sys
for line in sys.stdin:
    try:
        d = json.loads(line.strip())
        print(f'{d.get(\"time\",\"\")[-12:]} | {d.get(\"1\",\"\")[:150]}')
    except: pass
"
```

### 修改 Agent 人设 / Update Agent Persona
编辑 `~/.openclaw/workspace-{agent}/SOUL.md`，然后重启：  
Edit `~/.openclaw/workspace-{agent}/SOUL.md`, then restart:
```bash
openclaw gateway restart
```

### 修改限额 / Update Rate Limits
```bash
python3 -c "
import json
with open('/Users/YOUR_USER/.openclaw/openclaw.json') as f:
    cfg = json.load(f)
cfg['channels']['wechat']['rateLimitPerMinute'] = NEW_VALUE
cfg['channels']['wechat']['dailyTokenBudget'] = NEW_VALUE
with open('/Users/YOUR_USER/.openclaw/openclaw.json', 'w') as f:
    json.dump(cfg, f, ensure_ascii=False, indent=2)
"
```
热重载自动生效，无需重启。/ Hot-reload applies automatically, no restart needed.

## 故障排查 / Troubleshooting

### 通知不触发 / Notifications Not Firing
1. 检查微信通知权限：系统设置 → 通知 → 微信 / Check WeChat notification permission: System Settings → Notifications → WeChat
2. 确认微信在后台（不是最前面的窗口）/ Confirm WeChat is in background (not frontmost)
3. 确认群聊未开免打扰 / Confirm group chat is not muted
4. 确认通知提醒样式为「持续」而非「临时」/ Confirm alert style is "Persistent" not "Temporary"
5. 检查通知监控是否运行 / Check if notification monitor is running:
```bash
grep "wechat-notify.*Starting" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | tail -1
```

### 消息未被处理 / Messages Not Processed
检查日志中的 skipping 原因 / Check skip reasons in logs:
```bash
grep "skipping\|ignoring" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep wechat | tail -10
```
常见原因 / Common reasons:
- `ignoring DM` → 私聊被过滤（groupOnly=true）/ Private chat filtered
- `not active group` → 群未绑定，需发 `{bindTrigger}` / Group not bound
- `without trigger` → 消息不含 `{botTrigger}` / Message missing trigger keyword
- `rate limited` → 触发频率限制 / Rate limit hit

### 回复发到错误窗口 / Reply Sent to Wrong Window
Agent 回复到微信当前激活的聊天窗口。确保微信停留在目标群聊。  
Agent sends to whichever WeChat chat is active. Keep WeChat on the target group.

### 回复身份错误 / Wrong Agent Identity
1. 检查 `channels.wechat.agent` 是否为正确的 agent ID / Verify correct agent ID
2. sessionKey 是否包含 agentId（格式 `agent:{agent-id}:wechat:group:群名`）/ Check sessionKey format
3. 重启 gateway 清除旧 session / Restart gateway to clear old sessions

## macOS Tahoe 通知 UI 结构 / Notification UI Structure

```
window "Notification Center"
  > group 1
    > group 1
      > scroll area 1
        > group 1
          > group N (每个通知一个 group / one group per notification)
            > static text 1 (sender/群名 / sender/group name)
            > static text 2 (body/消息内容 / body/message content)
```

通知 body 格式 / Notification body format:
- 群消息 / Group message: `发送者: 消息内容` / `sender: message content`
- @mention: `发送者在群聊中@了你` / `sender mentioned you in a group`（无法获取实际内容，已用触发符替代 / actual content lost, replaced by trigger keyword）

## 已知限制 / Known Limitations

- 通知文本截断约 65 字符 / Notification text truncated at ~65 characters
- 微信在前台时不弹通知 / No notifications when WeChat is frontmost
- 发送占用前台和剪贴板（约 2 秒）/ Sending takes over foreground and clipboard (~2s)
- 重启 gateway 后动态绑定重置（需重新发 bindTrigger）/ Dynamic binding resets after restart
- 不支持接收图片/语音/视频内容 / Images, voice, and video not supported
