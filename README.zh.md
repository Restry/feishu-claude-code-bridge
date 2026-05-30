# lark-channel-bridge

把飞书 / Lark 消息和本地 Claude Code CLI 打通的轻量 bot，用一条命令起服务，扫码绑应用，在飞书里和 Claude 对话、让它读图 / 改代码。

[English README](./README.md)

关于能实现的效果，详情可以阅读[飞书文档](https://larkcommunity.feishu.cn/docx/OaRIdFIRFoLM3xxTmKwcetHqn5e)

## 能干什么

- 在飞书（私聊直接发；群里 `@bot`）把消息转给本地的 `claude` CLI，Claude 在你指定的工作目录里工作
- **流式卡片**：Claude 的文本和工具调用实时出现在同一张卡片上，不用傻等
- **会话延续**：每个 chat 独立 session，对话能接着上次说
- **抢占 + 批处理**：中途发新消息会打断旧任务；快速连发几条会合并成一次请求
- **多工作空间**：`/ws` 切换不同项目，session 自己重置
- **图片 / 文件**：直接发给 bot，Claude 会读本地下载的文件路径
- **卡片按钮**：`/help` `/ws list` `/status` 返回交互卡片，点按钮直接操作

## 前置条件

- Node.js **≥ 20**
- `claude` CLI 已安装并完成认证。两条路：
  - **OAuth**：手动跑一次 `claude` 完成交互式登录。
  - **API Key**（无人值守/服务器部署推荐）：在拉起 bridge 的环境里设 `ANTHROPIC_API_KEY`（如走代理/网关再加 `ANTHROPIC_BASE_URL`）。MCP client 拉起 bridge 时怎么把这两个传进去，见下面 [MCP server 模式](#mcp-server-模式)。
- 飞书 / Lark **PersonalAgent** 应用 —— **仅飞书 bridge 模式需要**（首次启动的扫码向导能帮你创建）。**只用 MCP server 不需要。**

## 安装

```bash
npm i -g lark-channel-bridge
# 或
pnpm add -g lark-channel-bridge
```

## 首次启动

```bash
lark-channel-bridge run
```

第一次跑会检测到没配置应用，**自动进入扫码向导**：

1. 终端渲染一个二维码
2. 用飞书 App 扫码
3. 选择 / 创建 PersonalAgent 应用
4. 成功后凭据写入 `~/.lark-channel/config.json`

## 命令速查

### 宿主 CLI

**进程层**（在你自己的 shell 里直接跑 bridge）:

```
lark-channel-bridge run [-c <config>]     前台启动 bot
lark-channel-bridge ps                    列出本机所有正在跑的 bridge 进程
lark-channel-bridge kill <id|#>           kill 指定 bridge 进程（SIGTERM，2s 后 SIGKILL）
lark-channel-bridge --help                列所有命令
```

**服务层**（让 OS 在后台托管 bridge）:

> ⚠️ **服务层命令必须先全局安装,不能直接用 npx**。daemon 的 launchd plist / systemd unit / Windows 任务里会**硬编码** bridge CLI 的路径;通过 `npx lark-channel-bridge start` 调用时,这条路径在 npm 的临时缓存里(`~/.npm/_npx/<hash>/...`),会被 GC 清掉 — 一旦缓存清理,daemon 就起不来了。请先 `npm install -g lark-channel-bridge`,再 `lark-channel-bridge start`。`bridge run` 用 npx 调用没问题(单次进程)。

```
lark-channel-bridge start                 注册（如需）+ 启动后台 daemon
lark-channel-bridge stop                  停止 daemon 并关闭开机自启
lark-channel-bridge restart               重启 daemon
lark-channel-bridge status                查看 daemon 状态（pid、日志路径、上次退出码）
lark-channel-bridge unregister            撤销注册（停止 + 删除服务定义文件）
```

daemon 崩溃会被自动拉起，用户登录时也会自动启动。平台映射:
- **macOS** → `launchd` 用户代理 `~/Library/LaunchAgents/ai.lark-channel-bridge.bot.plist`
- **Linux** → `systemd` 用户单元 `~/.config/systemd/user/lark-channel-bridge.bot.service`。要让 daemon 在退出登录后还能跑，执行一次 `loginctl enable-linger $USER`。
- **Windows** → Task Scheduler 任务 `LarkChannelBridge.Bot`，触发条件为 ONLOGON。启动脚本位于 `~/.lark-channel/daemon-launcher.cmd`。

daemon 的 stdout / stderr 写到 `~/.lark-channel/logs/daemon-stdout.log` 和 `daemon-stderr.log`，跟 bridge 自己的每日结构化日志放在同一个目录。

> 多开同一个 app 时，开放平台会把事件随机推到其中一个长连接。`run` 启动前会检测同 app 已有的进程，TTY 下提示 `[c]ontinue / [k]ill old / [a]bort` 三选；非 TTY 只 warn 并继续。

### 在飞书里用的斜杠命令

| 命令 | 作用 |
|---|---|
| `/new` `/reset` | 清空当前 chat 的会话 |
| `/cd <path>` | 切换工作目录（会重置 session） |
| `/ws list` | 列所有命名工作空间（卡片 + 按钮） |
| `/ws save <name>` | 把当前 cwd 存为命名工作空间 |
| `/ws use <name>` | 切换到命名工作空间 |
| `/ws remove <name>` | 删除命名工作空间 |
| `/status` | 当前 cwd / session / agent（卡片 + 按钮） |
| `/config` | 调整偏好（消息回复方式、工具调用显示等） |
| `/stop` | 终止当前正在跑的 run（也可点卡片底部 ⏹ 终止 按钮） |
| `/timeout [N\|off\|default]` | 当前 session 的 idle 探活（分钟）；`/config` 改全局默认。详见下方"常见问题 — Claude 子进程假死" |
| `/ps` | 列出本机所有 start 进程，标识当前回复的是哪个 |
| `/exit <id\|#>` | 终止指定 start 进程（自己 = graceful 退出；他人 = SIGTERM） |
| `/reconnect` | 强制重连 WebSocket（网络抖动后 bot 没反应时用） |
| `/doctor [描述]` | 把最近运行日志和你的描述喂给 Claude，自助诊断卡住 / 异常的原因 |
| `/help` | 帮助卡片 |
| 其它 `/xxx` | 原样交给 Claude |

**消息策略**：私聊 = 不需要 @，任何消息都回；**群（含话题群）= 默认要 @bot 才回**（0.1.22 起的新默认），不 @ 时 bot 完全沉默；@全员永远不响应；云文档评论必须 @bot。要恢复"群里也不强制 @"的老行为：`/config` → "群里需要 @ bot" → 选"否"。

## MCP server 模式

除了飞书 bridge，同一个包还能把 Claude Code 暴露成一个 **MCP stdio server**，让任何 MCP client（Hermes、Claude Desktop 等）异步调 Claude Code，不阻塞自己的对话主循环。

```bash
lark-channel-bridge mcp \
  --cwd /path/to/default/workspace \
  --cwd-root /path/to/default/workspace \
  --cwd-root /tmp
```

- `--cwd` —— 任务的默认工作目录，调用方可在每次调用时覆盖。
- `--cwd-root` —— 可重复指定的白名单。若设置，任务的 `cwd` 必须落在某个 root 下。**暴露给非信任调用方时强烈建议加上**。

### 暴露的工具

| 工具 | 作用 |
|---|---|
| `claude_run(prompt, cwd?, session_id?, model?)` | 起任务，立即返回 `task_id`。 |
| `claude_status(task_id)` | 快照：状态、已积累文本、当前工具、计数器。 |
| `claude_wait(task_id, from_seq?, timeout_ms?)` | 短阻塞（≤ 120 秒）拉取 `from_seq` 之后的新事件。 |
| `claude_cancel(task_id)` | 先 SIGTERM 再 SIGKILL 中断任务。 |
| `claude_list()` | 列出所有任务（running + 终态）。 |
| `claude_forget(task_id)` | 从内存里释放一个已结束的任务。 |

1–2 小时的长任务**不会**占满一次 MCP 调用：调用方模式是 `claude_run` → 该聊别的聊别的 → 想看进度时调 `claude_wait` / `claude_status` → 下一次 `claude_wait` 带上最高的 `seq` 增量拉取。

### 约束与边界（已实测）

任务状态是**进程内内存**，而 MCP server 是 stdio——它的生命周期绑定单个 MCP client 连接。由此有几条必须知道的边界：

- **不持久 / 不跨进程**：server 进程一退，所有任务记录（`claude_list` / `claude_status` 能查到的一切）清零。另起一个客户端连接 = 另一个 server 进程 = 一份空的任务表。别指望"昨天建的任务今天还能 list 出来"，也别指望两个客户端共享同一份任务视图。
- **定位**：这是一个**会话级任务调度器**，不是持久任务队列。典型用法——`claude_run` 拿 `task_id` → 同一连接内 `claude_wait` / `claude_status` 轮询到终态——完全可靠。
- **resume 是显式的**：续接靠 `claude_run` 显式传 `session_id`（底层 `claude --resume <id>`），不靠 cwd 自动续。同一个 `cwd` 起的多个任务彼此隔离，不会串台。传一个不存在的 `session_id` 会让该任务直接失败（`No conversation found`），不会静默降级为新会话。
- **事件缓冲上限**：每个任务最多保留最近 5000 条事件，更早的会被丢弃（`seq` 仍单调递增）。增量拉取请始终带上次见过的最高 `seq`。

> 想要持久化（跨重启可查、多端共享任务列表）需要自行加一层：把任务表落到磁盘 / SQLite，server 启动时恢复。当前版本不做这件事。

### Hermes 接入示例

编辑 `~/.hermes/config.yaml`，加顶层 `mcp_servers:` 块：

```yaml
mcp_servers:
  cc:
    command: node
    args:
      - /path/to/feishu-claude-code-bridge/bin/lark-channel-bridge.mjs
      - mcp
      - --cwd
      - /home/me/Projects
      - --cwd-root
      - /home/me/Projects
      - --cwd-root
      - /tmp
    env:
      # 显式传 auth —— MCP client 直接 spawn 子进程,不走 shell,
      # 所以 .zshrc / .bashrc 里的环境变量 **不会** 被继承。
      ANTHROPIC_API_KEY: "sk-..."
      # 走代理 / 网关时:
      ANTHROPIC_BASE_URL: "https://api.example.com"
    timeout: 180         # 每次 MCP 调用的最大阻塞时长(claude_wait 内部封顶 120 秒)
    connect_timeout: 30
```

然后在 Hermes 对话里发 `/reload-mcp` 热加载,不用重启。也可以 `hermes mcp test cc` 验证连通。

> ⚠️ **认证坑**: MCP client 是直接 `node ...` 拉起 bridge,**不走交互式 shell**。所以如果你的 `claude` 是 OAuth 登录 + 依赖 shell 环境变量定位配置,这些环境变量**不会被继承**。要么在 `env:` 块里显式写 `ANTHROPIC_API_KEY`(推荐),要么把 `command` 包成 `zsh -ic`。

> ⚠️ **安全**: 任务都是 `--permission-mode bypassPermissions`(无任何确认)。多租户或非信任调用方场景下,**务必**配 `--cwd-root`,并把本机视为整台交给调用方支配。

## 数据目录

| 路径 | 内容 |
|---|---|
| `~/.lark-channel/config.json` | 应用凭据（App ID / Secret），权限 600 |
| `~/.lark-channel/sessions.json` | 每个 chat / 话题 的 Claude session id + cwd（+ 可选的 `/timeout` 覆盖） |
| `~/.lark-channel/workspaces.json` | 工作空间映射 |
| `~/.lark-channel/processes.json` | 当前在跑的 start 进程注册中心（`ps`/`stop` 用），死进程会被自动清理 |
| `~/.lark-channel/media/<chatId>/` | 下载的图片 / 文件，24h 自动清理 |
| `~/.lark-channel/logs/YYYY-MM-DD.log` | 结构化运行日志（JSON line），按天滚动；启动时清理超过 7 天的老文件（`LARK_CHANNEL_LOG_DAYS` 环境变量可改）；`/doctor` 命令读它做诊断 |

> 升级自 0.1.11 之前的版本？跑一次 `lark-channel-bridge migrate` —— 自动把 `~/.config/lark-channel-bridge/` 和 `~/.cache/lark-channel-bridge/` 下的内容搬到新位置，并把 `config.json` 升级到新结构。

## 访问控制

**默认是私有的：开箱即用时，只有"你"能用这个 bot。** 这里的"你" = 创建 / 拥有这个飞书应用的人（也就是扫码把 bot 建起来的那位）。bot 会自动从飞书查出谁是应用 owner，所以**一个人用完全不用配置**——你私聊它、在任意群里 @它都正常工作，其他人发的消息会被静默忽略（bot 不会回"你没权限"，免得暴露自己的存在）。

想让别的同事或某些群也能用，就把他们加进下面三类名单：

| 名单 | 控制谁 | 加入 | 移除 |
|------|--------|------|------|
| **允许私聊的用户** | 谁可以私聊 bot | `/invite user @某人` | `/remove user @某人` |
| **响应的群** | bot 在哪些群里对**群内所有人**响应 | `/invite group`（当前群）/ `/invite all group`（bot 所在的全部群） | `/remove group`（当前群） |
| **管理员** | 谁能改设置、并能在任意群用 bot | `/invite admin @某人` | `/remove admin @某人` |

> `/invite`、`/remove` 这些命令只有**你（创建者）和管理员**能发。命令里 @ 的是**对方**（不是 @ bot），bot 会自动把 @ 解析成对应的人，你不用手动去找 ID。

### 两种"畅通无阻"的身份

- **你（创建者）**：不受任何名单限制——私聊、任意群、所有命令都能用，而且**永远锁不死自己**：哪怕名单配乱了，回到 bot 私聊发 `/config` 总能进来。在飞书后台把应用 owner 转给别人后，bot 也会自动跟着切换。
- **管理员**：能私聊、能用 `/config` 等管理命令，而且**不受"响应的群"名单限制**——无论群在不在名单里，bot 都会回他们。适合给一起维护 bot 的同事。

### 几种常见配置

- **只给自己用** → 什么都不用做，默认就是。
- **让某个同事能私聊 bot** → `/invite user @他`
- **让某个工作群里所有人都能用** → 在那个群里发 `/invite group`
- **第一次配，想把 bot 已经在的群一次性全开放** → 发 `/invite all group` 一键拉取 bot 所在的全部群加入名单，之后再用 `/remove group` 删掉不想要的
- **再拉个人一起当管理员** → `/invite admin @他`

### 还需要知道的

- 改完**下一条消息**就生效，不用重启。
- **群里默认要先 @bot 才会回**（私聊不用 @）。这是另一个独立开关（`/config` →"群里需要 @ bot"），和上面的名单是两回事。
- 陌生人发消息一律静默丢弃，不会有任何回复。唯一的例外：有人在一个还没开放的群里 @bot，bot 会回一句友好提示，告诉他可以让管理员发 `/invite group` 开放这个群。

### 高级：直接改配置文件

不想在飞书里点的话，`/invite`、`/config` 背后写的都是 `~/.lark-channel/config.json` 里的 `preferences.access`：

```json
{
  "preferences": {
    "access": {
      "allowedUsers": ["ou_xxxxxxxxxxxxx"],
      "allowedChats": ["oc_xxxxxxxxxxxxx"],
      "admins":       ["ou_xxxxxxxxxxxxx"]
    }
  }
}
```

`allowedUsers` / `admins` 填用户 `open_id`，`allowedChats` 填群 `chat_id`。手动找 ID 最简单的办法：让对方给 bot 发条消息（群里就 @ 它一下），然后看当天日志：

```bash
grep '"event":"enter"' ~/.lark-channel/logs/$(date +%Y-%m-%d).log | tail -5
```

每行都带 `chatId`（群 / 私聊 ID）和 `senderId`（用户 `open_id`）。手改完后**重启 bridge**，或在任意被允许的会话里发 `/reconnect` 让它生效。日常调整还是 `/invite` / `/config` 更省事，直接改文件主要用于部署脚本预填。

## 常见问题

**Claude 挂住不回复**：通常是 `claude` CLI 本身没登录，或者 session 指向了不存在的 cwd。发 `/status` 看当前状态；`/new` 重开会话往往就好。

**Claude 子进程假死（卡片停在最后一帧不动）**：从 0.1.20 起支持 idle 探活：claude 一段时间没输出就被 SIGTERM kill，卡片末尾会标 "⏱ N 分钟无响应，已自动终止"。默认关闭。开启方式：`/config` 设全局值（分钟），或 `/timeout 10` 只对当前 session 生效；`/timeout off` 关掉某个 session 的探活；`/timeout default` 清掉 session 覆盖回退到全局。

**图片发过去 Claude 说看不到**：升级到最新版，0.1.0 之前的版本有文件名去重 bug。

## 可选：遥测（Telemetry）

默认情况下 bridge **不上报任何数据**：没有指标、没有日志离开你的机器，也不引入任何遥测依赖。下面这个钩子在你主动开启前完全是空操作。

想接自己的监控时，用环境变量指向一个 default export（或导出 `createAdapter`）`AdapterFactory` 的模块：

```bash
LARK_CHANNEL_TELEMETRY_MODULE=your-telemetry-package lark-channel-bridge start
```

该模块会收到每一条 `log.*` 事件，以及错误 / 指标钩子，转发到任何你想要的地方。接口从包根导出：

```ts
import type { AdapterFactory, TelemetryAdapter, TelemetryEvent } from 'lark-channel-bridge';

const createAdapter: AdapterFactory = (meta) => ({
  emit(event) {/* 上报事件 */},
  recordError(err, ctx) {/* 上报异常 */},
  recordMetric(name, value, tags) {/* 上报指标 */},
  flush(timeoutMs) {/* 冲刷缓冲事件 */},
});
export default createAdapter;
```

模块不存在、工厂函数不合法、或者 adapter 抛错，都会降级为空操作——遥测永远不会阻止 bridge 启动，也不会打断日志。

## 许可

[MIT](./LICENSE)

<img src="./assets/feedback-group-qr.png" alt="飞书反馈群二维码" width="360">
