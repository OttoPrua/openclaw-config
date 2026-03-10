---
name: wechat-rossi
description: 微信群聊洛茜管理。涉及微信插件运维、群绑定切换、洛茜人设调整、故障排查时使用。
---

# 微信群聊洛茜管理 Skill

管理微信插件（macOS 原生通知方案）+ 洛茜 agent 的日常运维。

---

## 架构概览

```
微信群消息 → macOS 通知弹窗
  → AppleScript 轮询（0.5s）→ 解析群名/发送者/内容
  → 绑定指令（！洛茜）/ 触发符（＆洛茜）检测
  → dispatch 到 rossi agent（Sonnet 4.6）
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
| `~/.openclaw/workspace-rossi/SOUL.md` | 洛茜人设 |
| `~/.openclaw/workspace-rossi/IDENTITY.md` | 洛茜身份 |
| `~/.openclaw/openclaw.json` → `channels.wechat` | 微信频道配置 |
| `~/.openclaw/openclaw.json` → `agents.list[rossi]` | rossi agent 配置 |

## 配置参数

```json
{
  "channels": {
    "wechat": {
      "enabled": true,
      "blockStreaming": true,
      "groupOnly": true,
      "botName": "扫拖一体🤖",
      "botTrigger": "＆洛茜",
      "bindTrigger": "！洛茜",
      "agent": "rossi",
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
| `botTrigger` | 群聊触发符（全角），消息包含此触发符才会被洛茜处理 |
| `bindTrigger` | 群绑定指令（全角），在群里发此指令将该群设为激活群 |
| `agent` | 绑定的 agent ID |
| `allowedGroups` | 群名白名单（空=仅依赖动态绑定） |
| `rateLimitPerMinute` | 每分钟最大回复数 |
| `dailyTokenBudget` | 每日 token 预算 |

## 日常操作

### 切换目标群聊
在目标群里发送 `！洛茜`（全角感叹号），洛茜会回复确认绑定。

### 触发洛茜回复
在已绑定的群里发送 `＆洛茜 你的问题`（全角 &），洛茜会回复。

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

### 修改洛茜人设
编辑 `~/.openclaw/workspace-rossi/SOUL.md`，然后重启 gateway：
```bash
openclaw gateway restart
```

### 修改限额
```bash
python3 -c "
import json
with open('/Users/ottoprua/.openclaw/openclaw.json') as f:
    cfg = json.load(f)
cfg['channels']['wechat']['rateLimitPerMinute'] = 新数值
cfg['channels']['wechat']['dailyTokenBudget'] = 新数值
with open('/Users/ottoprua/.openclaw/openclaw.json', 'w') as f:
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
- `not active group` → 群未绑定，需发 `！洛茜`
- `without trigger` → 消息不含 `＆洛茜`
- `rate limited` → 触发频率限制

### 回复发到错误窗口
洛茜回复到微信当前激活的聊天窗口。确保微信停留在目标群聊。

### 回复身份错误（不是洛茜）
1. 检查 `channels.wechat.agent` 是否为 `rossi`
2. sessionKey 是否包含 agentId（格式 `agent:rossi:wechat:group:群名`）
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
- 重启 gateway 后动态绑定重置（需重新发 `！洛茜`）
- 不支持接收图片/语音/视频内容
