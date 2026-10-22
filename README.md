# wx-bridge

用**手机微信**跟一个自定义人格的 AI 聊天。

走的是微信官方的 **iLink Bot API**——扫码登录后你在微信里多了个 Bot 联系人，不 hook 个人账号，不封号。

```
手机微信
   │  你给 Bot 发消息
   ▼
微信 iLink 服务器（官方，ilinkai.weixin.qq.com）
   │  wx-bridge 长轮询取消息
   ▼
本机 wx-bridge
   ├─ 维护对话记忆、场景、情绪状态
   ├─ 组装「人设 + 历史 + 本轮」调用模型
   ▼
回复发回你的微信
```

---

## 快速开始

```bash
# 1. 装 Node 22+
node -v

# 2. 配 Key（二选一）
#    a. 建 data/apikey.txt，写一行 sk-xxx
#    b. 或设环境变量 DEEPSEEK_API_KEY

# 3. 启动
node src/index.js --mode=chat
```

首次运行会打印一个登录链接，**用手机微信打开并确认**。凭证存进 `data/credentials.json`，之后不用再扫。

看到这行就是好了：

```
开始长轮询，等待微信消息…
```

Windows 用户可以直接双击 `START.cmd`。

---

## 配置

复制 `data/config.example.json` 成 `data/config.json` 再改。

**必改的**：

| 字段 | 说明 |
|---|---|
| `allowedUsers` | 只响应这些微信 id。**留空 = 谁都能用你的 Key，别留空** |
| `doUsers` | 允许执行 `/do` 的微信 id，建议和上面一致 |

**怎么拿到自己的 id**：启动后在微信里给 Bot 发一条消息，日志里会打印 `收到  来自=xxxxx@im.wechat`。

**主要参数**：

| 字段 | 默认 | 说明 |
|---|---|---|
| `chatModel` | `deepseek-v4-flash` | 模型 |
| `chatTemperature` | 1.3 | 越高越随机 |
| `chatPresencePenalty` | 0.6 | 抑制重复用词 |
| `chatFrequencyPenalty` | 0.45 | 抑制口头禅 |
| `maxBubbles` | 4 | 一条回复最多拆几条 |
| `bubbleGapMs` | 1200 | 气泡间隔 |
| `proactive` | false | 是否主动发消息 |
| `proactiveMinMinutes` / `Max` | 5 / 12 | 主动消息间隔（注意：实际 = 静默 + 间隔） |
| `proactiveContextTtlMinutes` | 30 | 发送窗口，超过就发不出去（服务端限制） |

---

## 人设

人设在 `personas/*.md`，是**纯文本 + front matter**：

```markdown
---
title: 沈迟
description: 一句话简介
---

（正文就是提示词）
```

改完**存盘即生效**，不用重启。

- 默认人设由 `config.json` 的 `defaultPersona` 指定
- 微信里用 `/role` 切换，`/role off` 关掉
- `/do` 干活时用 `doPersona`（默认 `gongju`），因为技术汇报和闲聊语感不一样

### 写人设的两条经验

**1. 少写规则，多写人。**

我们试过写 1.8 万字符的人设——17 条情境对照表、频率配额、禁忌清单。
结果是模型忙着对号入座，输出反而机械。砍到 3 千字符、只留"她是谁"之后，话少了但像人了。

**2. 「可以」压不住，要写「必须」。**

反过来也成立：想让她撒娇，写"你可以撒娇"没用，写"看到这些必须做出反应"才生效。
重要的行为要配**具体触发条件 + 例句**。

---

## 指令

微信里发：

| 指令 | 作用 |
|---|---|
| `/new` 或 `/clear` | 清空上下文 |
| `/role` | 查看/切换人设 |
| `/do 动作名` | 执行预置动作（见 `data/actions.json`） |
| `/do 任意任务` | 交给 DSH agent 现场处理（需要装 `@deepseek-ai/dsh`） |
| `/help` | 全部指令 |

---

## 两种模式

| 模式 | 命令 | 特点 |
|---|---|---|
| `chat` | `--mode=chat` | 直连 API，约 1500-6000 tokens/轮，便宜快 |
| `dsh` | `--mode=dsh` | 走 DSH headless，带全套工具，贵但能干活 |
| `echo` | `--mode=echo` | 回声，排查链路用 |

---

## /do 干活（可选）

装了 DSH CLI 之后，`/do 帮我写个脚本` 会拉起一个 agent 真的动手。

```bash
mkdir ../dsh-cli && cd ../dsh-cli
pnpm add "@deepseek-ai/dsh@0.2.1-rc.2"
```

目录要和本项目**同级**（代码按 `项目根/../dsh-cli/` 找），或者设 `DSH_BIN` 环境变量。

agent 被沙箱限制在 `../workspace` 目录里，碰不到项目源码和凭证。

> Windows 第一次跑可能报 `SetNamedSecurityInfoW failed (Win32 5)`，
> 是沙箱授权失败。给项目父目录和工作区目录补上当前用户的完全控制权即可：
> `icacls "D:\path\to\parent" /grant "%USERNAME%:(OI)(CI)F"`

---

## 目录结构

```
wx-bridge/
├── src/
│   ├── index.js          主程序：长轮询、消息分发、场景编排
│   ├── ilink.js          微信 iLink 协议（扫码/长轮询/发消息）
│   ├── chat.js           直连 Chat API
│   ├── dsh.js            DSH headless 通道
│   ├── persona.js        人设解析
│   ├── memory.js         对话记忆 + 归档
│   ├── mood.js           情绪状态机（生气/软化/消气）
│   ├── scene.js          场景延续
│   ├── output.js         气泡拆分、去动作、表情包标记
│   ├── proactive.js      主动消息调度
│   └── ...
├── personas/             人设（md + front matter）
├── stickers/             表情包（6 类）
├── data/
│   ├── config.example.json
│   └── actions.json      /do 预置动作
├── tools/                测试脚本
└── docs/                 技术笔记
```

---

## 测试

`tools/` 下有几十个场景测试脚本，走的是和线上完全同一条链路
（`chatComplete` → `splitIntoBubbles` → `extractStickers`），只是不经过微信发送。

```bash
node tools/scenario-test.mjs     # 长短对照
node tools/scenario-test2.mjs    # 倒霉/作死/问事/示好
node tools/contrast-test.mjs     # 情绪反差切换
node tools/anger-flow2.mjs       # 生气流程
node tools/profanity-rate.mjs    # 统计脏话比例
```

改完人设跑一遍，能立刻看出哪里崩。

---

## 已知限制

这些是**没做**，不是坏了：

- **Bot 不能主动发起会话**（服务端硬限制：`context_token` 只能来自用户消息）。
  所有主动消息都必须在**你最后一条消息之后约 30 分钟内**发出，超时会被拒。
- **只支持 1:1 私聊**，没有群聊。
- **每条消息 spawn 一次 API 调用**，约 1-3 秒。
- **记忆是最近 N 轮 + 摘要**，长对话会丢早期内容。

---

## 免责

- 用的是微信官方 iLink Bot API，不是逆向协议。
- 请自己配置 `allowedUsers`，不要把带 Key 的 Bot 暴露给陌生人。
- 别把 `data/` 提交进版本库——里面有登录凭证和 API Key，`.gitignore` 已经排除了。

## License

MIT
