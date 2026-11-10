# 我的 pi 配置

基于 **Pi 1.4.0+** 的个人主力 harness。包含 10 个精选扩展、一套子代理与记忆配置、一份全局工作约定，以及一层 herdr 多会话运行时。

这套配置的核心理念只有三条：
1. **极限降噪**：过滤树形工具调用展开，上下文与终端滚动区干净一个数量级。
2. **三层预算分流**：主会话用旗舰模型，观察/反思/子代理/网页摘要全部钉死廉价 Flash 模型，杜绝后台隐形吞 token。
3. **长会话保活**：故意将声明上下文窗口砍半，配合比率压缩，规避长文本“中间遗忘”，确保 50 万 token 以上长会话依然敏锐。

三条之外还有一层可选运行时：**herdr** 把多个 pi 会话放进同一终端的不同 pane，进程常驻、状态可见（见 §2.2）。

不含任何私有服务地址与敏感密钥。**照抄能跑，每一处都写清了取舍逻辑。**

> 💡 **让 Agent 代劳？** 你可以直接把下面这段话复制并发送给当前正在用的 agent（如 pi、OpenCode、Claude Code 等）：
> 
> ```text
> 请阅读https://github.com/Echokovo/pi-share中的README.md，按照「1. 快速开始」的步骤 1、步骤 2 帮我安装扩展并落位配置。
> 要求：
> 1. 落位前如果本地 ~/.pi/agent 下已有旧配置，先提醒我备份并展示 diff，不要直接覆盖。
> 2. 遇到任何需要填写密钥、选择模型、修改 recipient 飞书 ID 或存在决策分歧的地方，务必停下来询问我，不要替我擅自决定。
> ```

```text
README.md             # 本说明文档，照着做即可
agent/                # 对应 ~/.pi/agent/ 目录（只入库配置类文件，排除清单见步骤 2）
├── settings.json      # 主配置：扩展清单 + 工具裁剪 + TUI + 插件参数
├── subagents.json     # 子代理隔离与并发策略（worktree 隔离、禁止套娃）
├── web-search.json    # web_access 路由与摘要模型配置
├── AGENTS.md          # 全局工作约定（每次会话默认注入上下文）
└── agents/            # 自定义子代理（Explore / general-purpose / Plan）
```

---

## 1. 快速开始（3 步落位）

### 步骤 1：安装扩展

确保环境中 `git` 与 `npm` 可用，运行以下命令（已与 `agent/settings.json` 完全对齐）：

```bash
# 核心记忆与文档裁剪
pi install npm:@ssk_dev/pi-docs-slim
pi install npm:pi-observational-memory

# 交互、通知与运维
pi install npm:@amaster.ai/pi-channels
pi install git:github.com/Echokovo/pi-ask-channel-reminder
pi install npm:@ssk_dev/rpiv-ask-user-question-lean
pi install git:github.com/Echokovo/pi-ssh-tools
pi install npm:pi-token-speed
pi install git:github.com/Echokovo/pi-update-changelog

# 工具与子代理增强
pi install npm:@ssk_dev/pi-subagents-lean
pi install npm:@ssk_dev/pi-web-access-lean
```

### 步骤 2：落位配置文件

仓库中的 `agent/` 目录对应 `~/.pi/agent/`（排除凭据与运行时缓存）。直接同步即可：

```bash
mkdir -p ~/.pi/agent
cp -r agent/* ~/.pi/agent/
```

> ⚠️ **落地检查**：
> - 若已有配置，建议先 `diff` 备份后再覆盖。
> - 检查 `~/.pi/agent/settings.json`：若使用飞书通知，将 `pi-channels.routes.ops.recipient` 改为你的飞书 chat ID；若暂不需要通知，将 `pi-ask-reminder.enabled` 设为 `false` 即可。

### 步骤 3：配置环境变量

后台压缩与摘要统一走 DeepSeek Flash，如启用 Exa 搜索需配其 Key（建议写入 `~/.bashrc` 或 `~/.zshrc`）：

```bash
export DEEPSEEK_API_KEY="your-deepseek-key"
export EXA_API_KEY="your-exa-key"  # 若不用 Exa，需从 web-search.json 移除对应提供商
```

配置完毕后，启动 `pi` 或在已有会话中输入 `/reload` 即可生效。

> 🚀 **核心能力现已就绪！** 若需要**多个会话并行、终端后台常驻**，继续参考选装中的 [§2.2 多会话运行时 (herdr)](#22-多会话用-herdr-承载多个-pi)。

---

## 2. 选装配置（按需加装）

### 2.1 文档与思维 Skills

Skills 独立于配置目录，由 `skills` CLI 统一管理并落位至 `~/.agents/skills/`，pi 会自动发现：

```bash
# 方案拷问组：强迫 LLM 批判性反思取舍与技术边界，杜绝盲目推进
npx skills add mattpocock/skills -g -s grilling
npx skills add mattpocock/skills -g -s grill-me
npx skills add mattpocock/skills -g -s grill-with-docs

# 文档工程规范组：对齐工业级大厂文风（编写 README / API / 交互报错时启用）
npx skills add Neeeophytee/agent-stylebooks -g -s github-docs
npx skills add Neeeophytee/agent-stylebooks -g -s google-developer-docs
npx skills add Neeeophytee/agent-stylebooks -g -s microsoft-writing-style
```

- **常用命令**：`npx skills ls -g` 查看列表，`npx skills update -g` 同步上游更新。
- **使用方式**：按需自动触发，也可通过 `/skill:<name>` 强制载入会话。

### 2.2 多会话：用 herdr 承载多个 pi

pi 的 TUI 一个进程一次只呈现一个会话。A 项目等测试、想同时处理 B 项目报错时，多开终端窗口切起来费劲，断网或手滑关窗进程就丢，也看不清后台会话是否正在等待交互。

我们使用 **herdr** 来承载运行时。它类似 tmux，但原生为 Agent 场景设计：pane 里的 pi 会被识别并把 `working/blocked/idle/done` 状态汇聚在侧边栏，同时支持 CLI/Socket 供外部脚本调度。

```text
herdr server（后台常驻，持有所有 PTY）
└── workspace：pi-share
    └── tab：1 · <auto-title 生成的任务标题>
        ├── pane w1:p1  pi 会话 A（working）
        └── pane w1:p2  pi 会话 B（idle - 等待输入）
```

#### 为什么不用“多开几个终端”？

| 痛点 | 普通多终端 | herdr |
|---|---|---|
| **断线 / 关窗** | 进程直接退出 | pane 归 server 所有，detach、关窗、断 SSH 都不影响 |
| **状态透视** | 只能逐个切窗口看 | sidebar 与 `herdr agent list` 实时聚合各 pane 状态 |
| **跨会话调度** | 人手手动复制粘贴 | CLI 一行命令调度：`herdr agent prompt <name> "..." --wait` |
| **重启恢复** | 窗口全丢 | 根据 `session.json` 复原 workspace/tab/pane 与工作目录 |
| **并行改代码** | 互相踩 Git 文件 | `herdr worktree create` 一条命令生成并绑定独立隔离分支 |

代价是多一层键位与一个后台 server；不需要并行时完全可以不装，pi 单会话照常工作。

#### 安装与集成

```bash
# 1. 安装 herdr（Linux/macOS；后续 herdr update 自更新）
curl -fsSL https://herdr.dev/install.sh | sh

# 2. 安装 pi 集成（生成 ~/.pi/agent/extensions/herdr-agent-state.ts）
herdr integration install pi

# 3. 启动并附着默认 session
herdr
```

herdr 不强制要求配置文件，默认值即可用；需要改键位或实验项时再建 `~/.config/herdr/config.toml`（用 `herdr --default-config` 取全量字段）。

#### 日常操作速记

- **快捷键**：默认前缀键为 `ctrl+b`（按后松开，再按动作键）：
  - `ctrl+b c`：新建 tab → `cd ~/ws/project-b && pi`
  - `ctrl+b v`：垂直右分屏；`ctrl+b -`：水平下分屏
  - `ctrl+b h/j/k/l`：在各 pane 间导航
  - `ctrl+b z`：全屏缩放当前 pane
  - `ctrl+b w`：切换 workspace
  - `ctrl+b ?`：查看完整键位表
  - `ctrl+b q`：detach 断开（后台保持运行，下次直接敲 `herdr` 原样接回）
- **鼠标支持**：pane、tab、workspace、分割线与右键菜单均支持鼠标点击与拖拽，不背键位也能平滑上手。
- **独立环境**：一个 pane 一个 pi 进程 = 一份独立 session 文件。`cd` 决定 pi 的会话目录，不同项目天然隔离；同一目录想接着上次跑，用 `pi --continue` 或 `pi --session <path>`。

#### 存活与恢复

| 场景 | 结果 |
|---|---|
| `ctrl+b q` detach、关终端、SSH 中断 | server 继续跑，pi 进程不中断；下次 `herdr` 原样接回 |
| server 重启 / 机器重启 | 进程结束；layout 与 cwd 从 `session.json` 复原 |
| server 重启后的 pi 会话 | 集成模块记录会话路径，重启后自动以 `pi --session <path>` 恢复现场 |
| 终端屏幕历史 | 默认不回放。设 `[experimental] pane_history = true` 后仅在布局完全一致时回放 |

> ⚠️ `pane_history` 上游默认关闭是有理由的：它把终端输出落盘到 `~/.config/herdr/session-history.json`，可能包含 token、密钥或私有代码，请按 shell history 同等敏感度对待。

#### 推荐配套插件

推荐两个 herdr 的常用插件：

| 插件 | 版本 | 作用 |
|---|---|---|
| `herdr-sidebar`（alexarthurs） | 0.15.1 | 在 tab 左侧开一个约 15% 宽的文件树 + 源代码管理 pane，含 AI commit message、Quick Open、项目文本搜索 |
| `herdr.auto-title`（kryptamine） | 0.21.0 | 按 pane 里的任务内容自动给 tab 起名，省掉手动重命名 |

```bash
herdr plugin install alexarthurs/herdr-sidebar/plugins/herdr-sidebar
herdr plugin install kryptamine/herdr-auto-title --ref stable
herdr plugin action invoke herdr.auto-title.restart
```

---

## 3. 核心机制：为什么这样调？

### 3.1 便宜模型分流：后台调用杜绝烧钱

主会话通常选用昂贵的旗舰推理模型，但以下三类后台高频任务**只做归纳、检索与执行**，统一绑定为 `deepseek-flash` 并强制关闭 thinking：

| 触发场景 | 配置文件与字段 | 绑定值 |
|---|---|---|
| **子代理任务**（Explore / general-purpose） | `~/.pi/agent/agents/*.md` 的 frontmatter | `model: deepseek/deepseek-flash`<br>`thinking: off` |
| **网页检索摘要**（web_access） | `~/.pi/agent/web-search.json` 的 `summaryModel` | `"deepseek/deepseek-flash:off"` |
| **长期记忆提炼**（观察 / 反思） | `settings.json` 的 `observational-memory.model` | `provider: deepseek`<br>`id: deepseek-flash`<br>`thinking: off` |

> 📌 **关键前提**：
> - 模型 ID 需通过 pi 内置 `/model` 确认存在。
> - 若配置了 `enabledModels` 白名单，务必将该便宜模型加入其中。

---

### 3.2 上下文预算：故意将声明窗口砍半

`pi` 计算记忆压缩阈值依据的是客户端声明的 `contextWindow`，而非服务端的实际承载上限。
在这套配置中，**所有云端长上下文模型全部将声明值人为截断为 `500,000`**，并配合 `compactAfterTokensRatio: 0.5`：

```text
模型实际承载: 1,000,000 token
声明窗口截断:   500,000 token  ──(ratio: 0.5)──>  约 250,000 token 即触发压缩
实际工作区间: 保持在 0 ~ 250k token 最敏锐区间
```

**为什么值得做？**
1. **规避“中间遗忘（Lost in the Middle）”**：长上下文的注意力随长度显著摊薄。200k 内的信息检索与指令遵从准确率远高于 900k。
2. **把握最高质量的摘要节点**：若等到 80%~90% 才压缩，模型已处于低敏状态，产出的摘要往往丢失关键细节；在 50% 截断处压缩，被压缩的内容恰好是处于最佳注意力期产出的产物。
3. **精准规避阶梯计费溢价**：阻止上下文触碰部分供应商对超长请求设置的高倍率溢价区间（`cost.tiers`）。

> 该策略配置在 `~/.pi/agent/models.json`，因含私有 `baseUrl` 本仓库**不入库**，此处给出标准写法。

**配置写法示例**（自建 provider 直接声明窗口；内置 provider 用 `modelOverrides` 只改窗口，其余沿用上游定义）：
```json
{
  "providers": {
    "acme": {
      "baseUrl": "https://api.acme.example/v1",
      "apiKey": "$ACME_API_KEY",
      "models": [
        { "id": "acme-big", "contextWindow": 500000, "maxTokens": 64000 }
      ]
    },
    "deepseek": {
      "modelOverrides": {
        "deepseek-flash": { "contextWindow": 500000 }
      }
    }
  }
}
```

> ⚠️ **边界原则**：
> - 这只是 pi 内部的预算策略，无法替代服务端的硬性配额。
> - **本地 / 小参数模型必须按真实上限填写**（如 32k/128k 模型不可再盲目砍半，否则会严重挤压可用工作空间）。

---

## 4. `settings.json` 关键配置拆解

### 工具裁剪与安全边界
```json
"defaultTools": ["+codemode"],
"extensions": ["-builtin:llama.cpp", "-builtin:mcp", "-builtin:tool-search"]
```
- **保留 `codemode`**：允许直接运行 JS 批量调用工具链，大幅减少高频多步调用的上下文往返开销。
- **剔除冗余扩展**：每个内置工具都会常驻占据 Prompt 描述空间；禁用未使用的 `mcp` 与本地运行时，避免无谓的模型误导与上下文污染。

### 视图降噪与 TUI
```json
"treeFilterMode": "no-tools",
"collapseChangelog": true,
"quietStartup": "header",
"showCacheMissNotices": true,
"tuiMode": "fullscreen",
"fullscreenExitOutput": "resume-hint"
```
- `"treeFilterMode": "no-tools"`：**体感最明显的降噪项**。工具执行详情不折叠渲染进交互树，会话界面极度清爽（临时看详情可用 `/tree-filter all`）。
- `"showCacheMissNotices": true`：刻意开启。提示词缓存未命中时即时感知，便于定位 Prompt 破坏性变更。
- `"fullscreenExitOutput": "resume-hint"`：全屏退出时打印一键恢复会话命令，保护上下文不丢失。

### 记忆调度（pi-observational-memory）
```json
"observeAfterTokens": 25000,
"reflectAfterTokens": 50000,
"compactAfterTokensMode": "ratio",
"compactAfterTokensRatio": 0.5,
"observationsPoolMaxTokens": 20000,
"observationsPoolTargetTokens": 10000,
"observerChunkMaxTokens": 120000,
"agentMaxTurns": 8,
"model": { "provider": "deepseek", "id": "deepseek-flash", "thinking": "off" },
"debugLog": true
```
- 25k token 触发一次即时观察，50k token 触发阶段反思。
- 达到上下文截断阈值（约 250k）触发压缩，压缩后依然保留约 10k token 的高价值观察池。
- `model` 显式绑定低成本 Flash 模型（参见 §3.1），杜绝后台提炼蚕食主预算。
- `debugLog: true` 会在 `~/.pi/agent/observational-memory/debug/` 保留跟踪日志，便于诊断压缩质量；介意磁盘占用可设为 `false`。

### 运维告警与异步通知
```json
"pi-channels": {
  "adapters": { "feishu": { "type": "feishu", "eventMode": "off" } },
  "routes": { "ops": { "adapter": "feishu", "recipient": "YOUR_CHAT_ID" } },
  "bridge": { "enabled": false }
},
"pi-ask-reminder": {
  "enabled": true,
  "route": "ops",
  "afterMs": 180000,
  "maxReminders": 1
}
```
- **纯单向输出（`eventMode: "off"`）**：仅向外推送长任务完成、挂起提问通知；不接收入站消息，杜绝终端被外部消息劫持。
- **超时防死锁**：若交互提问（`ask_user_question`）挂起超过 3 分钟未应答，自动向移动端推送催促消息，离开工位也不会阻塞流水线。

---

## 5. 子代理治理（`agent/agents/`）

由 `@ssk_dev/pi-subagents-lean` 调度，置于 `~/.pi/agent/agents/`：

1. **`Explore.md`（只读探索）**：
   - 限制仅可用 `read`, `bash`, `grep`, `find`, `ls`。
   - `exclude_extensions` 剔除 ssh、ask、记忆系统等无关扩展，极致压缩单次启动体积。
   - `prompt_mode: replace` 彻底替换为人格独立的分析器。
2. **`general-purpose.md`（通用子代理）**：
   - `prompt_mode: append` 保留主逻辑前提下增加执行约束，严禁执行 `copy/move` 等破坏性整理。
3. **`Plan.md`**：
   - `enabled: false`。核心规划强保留在主会话中由主模型完成，杜绝多代理自由发散。

**系统级并发隔离（`subagents.json`）**：
- `"worktreeIsolation": true`：每个子代理独立分配 Git worktree，多任务并行代码修改互不干扰。
- `"maxSubagentDepth": 1` & `"fallbackSubagent": "none"`：禁止代理嵌套调用，禁止隐式回退。

---

## 6. 全局工作约定（`agent/AGENTS.md`）

置于 `~/.pi/agent/AGENTS.md`，对当前设备上运行的所有 pi 进程强制生效。核心包含：
- **硬性安全防线**：密钥严禁明文进任何文件或命令；破坏性命令（`git commit/push`、`reset`）执行前必须向用户显式确认并附带 `git status`。
- **任务交付自省交代**：阶段任务汇报必须包含 **「验证 / 盲区 / 假设 / 副作用」** 四段式自检，杜绝“全绿即通过”的盲区。
- **分层优先级**：项目级 `AGENTS.md` 永远高于全局配置，全局约定仅承载最底层的操作安全基准。
