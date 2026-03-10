---
name: openclaw-wechat-bot
description: Connect an OpenClaw Agent to WeChat group chats via macOS notification monitoring — receive group messages and send AI-powered replies automatically.
---

# openclaw-wechat-bot

管理微信插件（macOS 原生通知方案）+ 绑定 Agent 的日常运维。

---

## 架构概览

```
微信群消息 → macOS 通知弹窗
  → AppleScript 轮询（0.5s）→ 解析群名/发送者/内容
  → 绑定指令（{bindTrigger}）/ 触发符（{botTrigger}）检测
  → dispatch 到绑定 agent
  → 回复 → 激活微信 → 粘贴+发送 → 切后台
```

## 前置条件

- macOS Tahoe，微信桌面版在后台运行
- 系统设置 → 通知 → 微信 → **允许通知**（必须）
- 微信**不能在最前面**（前台不弹通知）
- 微信窗口需停留在目标群聊（发送到当前激活窗口）

## 关键路径

| 文件 | 用途 |
|------|------|
| `~/.openclaw/workspace/plugins/openclaw-wechat-plugin/index.ts` | 插件主代码 |
| `~/.openclaw/workspace-{agent}/SOUL.md` | Agent 人设 |
| `~/.openclaw/workspace-{agent}/IDENTITY.md` | Agent 身份 |
| `~/.openclaw/openclaw.json` → `channels.wechat` | 微信频道配置 |
| `~/.openclaw/openclaw.json` → `agents.list[{agent-id}]` | Agent 配置 |

## 配置参数

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

### 参数说明

| 参数 | 说明 |
|------|------|
| `botName` | 当前微信登录账号的名称。由于本方案不响应 @mention（@mention 通知内容被截断，无法提取实际消息），此字段对消息路由无实质影响，可留空。 |
| `botTrigger` | 群聊触发符（建议使用全角字符），消息包含此触发符才会被 agent 处理 |
| `bindTrigger` | 群绑定指令（建议使用全角字符），在群里发此指令将该群设为激活群 |
| `agent` | 绑定的 agent ID |
| `allowedGroups` | 群名白名单（空=仅依赖动态绑定） |
| `rateLimitPerMinute` | 每分钟最大回复数 |
| `dailyTokenBudget` | 每日 token 预算 |

## 日常操作

### 切换目标群聊
在目标群里发送 `{bindTrigger}`（你配置的绑定触发词），agent 回复确认后生效。

### 触发 Agent 回复
在已绑定的群里发送 `{botTrigger} 你的问题`，agent 会回复。

### 查看当前绑定
```bash
grep "bound active group" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | tail -1
```

### 查看最近消息流
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

### 修改 Agent 人设
编辑 `~/.openclaw/workspace-{agent}/SOUL.md`，然后重启 gateway：
```bash
openclaw gateway restart
```

### 修改限额
```bash
python3 -c "
import json
with open('/Users/YOUR_USER/.openclaw/openclaw.json') as f:
    cfg = json.load(f)
cfg['channels']['wechat']['rateLimitPerMinute'] = 新数值
cfg['channels']['wechat']['dailyTokenBudget'] = 新数值
with open('/Users/YOUR_USER/.openclaw/openclaw.json', 'w') as f:
    json.dump(cfg, f, ensure_ascii=False, indent=2)
"
```
热重载自动生效，无需重启。

## 故障排查

### 通知不触发
1. 检查微信通知权限：系统设置 → 通知 → 微信
2. 确认微信在后台（不是最前面的窗口）
3. 确认群聊未开免打扰
4. 检查通知监控是否运行：
```bash
grep "wechat-notify.*Starting" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | tail -1
```

### 消息未被处理
检查日志中的 skipping 原因：
```bash
grep "skipping\|ignoring" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep wechat | tail -10
```
常见原因：
- `ignoring DM` → 私聊被过滤（groupOnly=true）
- `not active group` → 群未绑定，需发 `{bindTrigger}`
- `without trigger` → 消息不含 `{botTrigger}`
- `rate limited` → 触发频率限制

### 回复发到错误窗口
Agent 回复到微信当前激活的聊天窗口。确保微信停留在目标群聊。

### 回复身份错误（不是预期 Agent）
1. 检查 `channels.wechat.agent` 是否为正确的 agent ID
2. sessionKey 是否包含 agentId（格式 `agent:{agent-id}:wechat:group:群名`）
3. 重启 gateway 清除旧 session

## macOS Tahoe 通知 UI 结构

```
window "Notification Center"
  > group 1
    > group 1
      > scroll area 1
        > group 1
          > group N (每个通知一个 group)
            > static text 1 (sender/群名)
            > static text 2 (body/消息内容)
```

通知 body 格式：
- 群消息：`发送者: 消息内容`
- @mention：`发送者在群聊中@了你`（无法获取实际内容，已用触发符替代）

## 已知限制

- 通知文本截断约 65 字符
- 微信在前台时不弹通知
- 发送占用前台和剪贴板（约 2 秒）
- 重启 gateway 后动态绑定重置（需重新发 bindTrigger）
- 不支持接收图片/语音/视频内容
