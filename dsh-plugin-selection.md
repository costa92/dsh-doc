# DSH 常用插件选型与实测

本文记录本工作区实际安装并验证过的 DSH 插件。选型依据是插件市场官方注册表的热度数据（star 数与 30 天下载量）、能力声明和红线标记；每个插件都在当前 `web` profile 里装过，并做了功能验证。

## 选型依据

插件目录来自 [`awesome-dsh-plugin.com/plugins.json`](https://awesome-dsh-plugin.com/plugins.json)，当前收录 10581 条。每条记录带三类可用信号：

- `stars` / `downloads` —— 热度
- `capabilities` —— 声明的能力（`fs-read`、`fs-write`、`shell`、`network`、`env`、`credentials`、`llm`、`subagent`、`dynamic-code`、`host-runtime`）
- `capabilityRedLines` —— 自动化审计出的风险组合，例如「读取凭据 **且** 有网络访问」

筛选口径：npm 可装、Linux 可跑、能力可验证。排除了 Windows 专用插件（`dsh-windows-mcp`、`dsh-ops`）和需要桌面壳的皮肤类插件（`dsh-wallpaper-engine` 依赖 Wallpaper Engine 本机程序）。

## 环境与验证方法

| 项 | 值 |
| --- | --- |
| dsh | 0.2.2-rc.1 |
| profile | `web` |
| 安装方式 | `dsh plugin --profile web add <包名>` |

验证分三层，逐层加严：

1. **清单层** —— 包是否进入 `package.json` 的 `dependencies` 与 `dsh.profile.bundles`；
2. **合成层** —— 插件自带的 patch 行是否出现在 `dsh --profile web --dump-config` 里；
3. **运行层** —— 宿主里是否真的挂载，以及插件功能是否可用。

第 3 层最容易被跳过，也最容易出问题。两个可用的观察点：

- `plugin_manager` 工具的 `list_bundles` 会给出每行的 `entryId`，只有真正挂进 Loader 的行才有；
- 插件注册的工具会出现在 agent 的工具列表里，直接调用就知道通不通。

## 十个插件的实测结论

### dshmarket —— 插件市场本体

star 10295，30 天下载 117 万，注册表里热度最高的市场类插件。装完后 Web 端 Plugins 页多出「插件市场」标签页，可以浏览、搜索、一键安装社区插件与主题。

实测：`GET /dsh-market/status` 返回 401（鉴权拦截，说明路由已挂载；未知路径返回 404）。没有额外能力声明，也没有红线。

副作用：它自己不读凭据、不联网装东西，但**它装的插件会**。市场是入口，不是安全边界。

### dsh-context —— 上下文洞察

star 4963，30 天下载 46 万。给 Web 端加一个 Context 仪表盘、一个 `/context` 命令和一个 Context 浏览器，用来看上下文的分类组成、演进趋势、压缩与注入事件。

能力：`fs-write`、`fs-read`、`network`、`env`。没有红线。

实测：行 `dsh-context` 在合成树中，宿主里 `fiberPhase: active`。

副作用：会读写 profile 目录下的数据来存上下文统计，并且有网络访问（`network`）。排查长会话的 token 花在哪时比较有用。

### dsh-better-sidebar —— 侧边栏工作台

star 5092，30 天下载 32 万。把文件树、编辑器、终端、Git、子代理塞进右侧栏一列，每个会话独立。它还对外提供 service，别的插件可以往里注册新 Tab。

能力：`shell`、`fs-write`、`fs-read`、`network`、`env`、`host-runtime`。没有红线。

实测：行 `better-sidebar` 是 active；隔离宿主启动后，它的客户端半边出现在下发的 client manifest 里。

副作用：带 `shell` 和 `host-runtime`，意味着它能在宿主进程里跑命令。这是本组里权限最宽的一个，装之前要想清楚。

注意它自带一条去重守卫：如果 Loader 里出现第二个同名但不同 id 的 `dsh-better-sidebar` 条目，这一行会自己停用。手工往 `cordis.patch.yml` 里插同一个插件，会让它自己关掉自己。

### @liustack/modlens —— 给纯文本模型加视觉

star 4725，30 天下载 29 万。把图片变成结构化 JSON 证据：OCR 全文、版面区域、语义、实体关系、不确定项。

能力：`shell`、`fs-write`、`fs-read`、`network`、`env`、`llm`。没有红线。

实测：这一条验证最实在。用仓库里的一张产品截图调用 `modlens_read_image`，返回了完整的中文 OCR 转录（「DSH 本地构建 0.1.7-rc.1」「新会话」「插件」「工作区」等界面文字）加上场景与意图描述。**视觉能力真的可用**。

副作用：它靠外部引擎跑推理（默认走免费的 Antigravity CLI），需要本机能启动那个引擎；用 `npx @liustack/modlens doctor` 可以自检。图片内容会离开本机，这一点在处理敏感截图时要留意。

### @michengai/dsh-skills-manager —— 技能管理

star 444，30 天下载 15.7 万。在设置里管理本地 DSH 技能，并只读浏览公共 Agent 技能。

能力：`shell`、`fs-write`、`fs-read`、`network`、`env`、`host-runtime`。没有红线。

实测：行 `skills-manager` 在合成树中且 active。

副作用：带 `host-runtime`，能碰宿主。技能本质是提示词文件，这个插件会改写 `DSH_HOME` 下的技能目录。

### @linxin666/dsh-client-ui-task-board —— 任务看板

star 10357，是这批里 star 最高的。多列看板，卡片交给真实的 DSH 智能体会话执行，支持 cron 定时，且定时由 Host 侧到点执行，关掉浏览器也生效。

能力：注册表里未声明能力（纯客户端 UI + Host 服务）。

实测：调用 `task_board_list` 返回完整看板摘要 —— `revision: 0`、`timeZone: Asia/Singapore`、`maxSubtaskDepth: 1`、`sessionDefaultPermission: workspace-write`、各列计数全 0。**看板服务真的在跑**。

副作用：卡片执行会开真实会话、消耗真实 API 额度。`sessionDefaultPermission` 是 `workspace-write`，意味着卡片默认就能写工作区；定时任务在你不在的时候也会跑。建议先只放只读任务试。

### @vectorize-io/hindsight-coding-agents —— 长期记忆

star 60140，是整份注册表里 star 最高的插件。给 agent 加跨会话的长期记忆：自动召回与保存、知识页、深度反思、按仓库隔离的记忆银行。

能力：注册表里未声明能力。

实测：插件挂载后注入了 `hindsight_*` 系列工具和一个 `hindsight-coding-agent` 技能。调用 `hindsight_list_pages` 返回的是后端的真实错误 —— `401 Authentication failed: API key required`。也就是说**链路是通的，缺的只是一个 API token**。

副作用：这是本组里唯一一个把会话内容外发的插件 —— 记忆存在 Vectorize 的云服务（`api.hindsight.vectorize.io`）。没配 token 时它对每次工具调用都会失败一次。要本机优先的话得另外配置。

### dsh-remote —— 多机远程工作区

star 264，30 天下载 11.7 万。管理多台 SSH 主机，把远程目录镜像成本地文件夹，用工具操作。

能力：`shell`、`fs-write`、`fs-read`、`network`、`credentials`、`env`、`host-runtime`。有 `credentials` 但注册表没判红线。

实测：挂载后注入了 **21 个 `rw_*` 工具**（`rw_connect`、`rw_exec`、`rw_read_file`、`rw_upload`、`rw_sync`、`rw_forward` 等）。调用 `rw_info` 返回「本会话没有绑定远程工作区」，符合预期 —— 插件就位，只是还没连主机。

副作用：会存 SSH 凭据（密码走系统钥匙串），并且能在远程主机上执行任意命令。`rw_remove` 是真删。用之前确认目标主机和路径。

### cc-safety-net —— 执行前守卫

star 1900，30 天下载 15 万。在工具调用运行前拦截破坏性 Git 与文件系统命令，以及访问 SSH 密钥、`.env` 等机密文件。

能力：`shell`、`fs-write`、`fs-read`、`network`、`credentials`、`env`。**红线：读取凭据且能联网**。

实测：拦得住的。我试着读 `~/.ssh/id_rsa`，调用被直接拒绝：

```
Error: BLOCKED by CC Safety Net
Reason: Access to a sensitive path is not allowed.
Rule: secret.home.ssh
Segment: ~/.ssh/id_rsa
```

**这是本组里最值得装的一个**，尤其在有 agent 自动执行命令的场景下。

副作用：红线是从它的能力推出来的，不是它做错了什么 —— 它要认出机密路径就得读路径与配置。但一条红线也意味着它自己是个高权限组件，要跟其他高权限插件一起评估。

### dsh-cost-meter —— 费用统计

star 675，30 天下载 15 万。会话与当日 API 费用、预算进度、官方余额、历史看板，支持峰谷计价。

能力：`shell`、`fs-write`、`fs-read`、`network`、`credentials`、`env`。**红线：读取凭据且能联网**。

实测：装上了，行进入合成树。它的界面是对话标题栏的费用芯片，需要浏览器看。

副作用：要读 DeepSeek 的 API key 才能查余额，所以带凭据能力。它的 key 读取在本地完成，但红线标记提醒了组合风险。

## 副作用汇总

按能力从宽到窄排，判断装不装时看这张表比看功能描述更有效。

| 插件 | shell | fs 写 | 网络 | 凭据 | host-runtime | 红线 |
| --- | :-: | :-: | :-: | :-: | :-: | --- |
| dsh-better-sidebar | ● | ● | ● | | ● | 无 |
| cc-safety-net | ● | ● | ● | ● | | 凭据+网络 |
| dsh-remote | ● | ● | ● | ● | ● | 无 |
| dsh-cost-meter | ● | ● | ● | ● | | 凭据+网络 |
| @liustack/modlens | ● | ● | ● | | | 无 |
| @michengai/dsh-skills-manager | ● | ● | ● | | ● | 无 |
| dsh-context | | ● | ● | | | 无 |
| dshmarket | | | | | | 无 |
| @linxin666/dsh-client-ui-task-board | | | | | | 无 |
| @vectorize-io/hindsight-coding-agents | | | | | | 无 |

两个带红线的插件（`cc-safety-net`、`dsh-cost-meter`）都是「读凭据 + 能联网」的组合。这类组合的风险不在单个插件，而在链式后果：一旦有提示注入或工具滥用，凭据可能顺着网络出去。装之前想清楚是否必要。

注意最后两行：`dsh-client-ui-task-board` 和 `hindsight-coding-agents` 在注册表里没有能力声明，**不等于它们没有能力**，只说明自动化审计没测出来。看板能开真实会话，hindsight 会把内容发到云端 —— 这些都不在能力矩阵里。

## 安装与回退

```bash
# 安装（可一次装多个）
dsh plugin --profile web add dsh-context dsh-better-sidebar

# 卸载
dsh plugin --profile web remove cc-safety-net
```

装完是否需要重启，取决于宿主是否在做重组。批量连续安装时 HMR 的 debounce 会合并事件，可能出现「清单里有、宿主里没挂上」的情况。判断方法：

```bash
dsh --profile web --dump-config | grep -c '^- id: '
```

再对比 `plugin_manager` 的 `list_bundles` 里每行有没有 `entryId`。对不上就重启一次宿主，重启后所有在 `bundles` 里的插件都会挂载。

## 注意事项

- **先看红线，再看功能**。功能描述是作者写的，能力和红线是审计出来的。
- **能力矩阵不完整**。没声明能力的插件不代表安全，只代表没测出能力。看板、记忆这类插件的实际影响要看它的行为。
- **插件默认会拿到你 profile 的全部权限**。DSH 没有按插件隔离权限的机制，装一个带 `shell` 的插件等于给它宿主执行权。
- **批量安装后确认挂载状态**。清单里有不等于跑起来了，用 `entryId` 或工具列表确认。
- **带凭据能力的插件要单独评估**。`cc-safety-net` 虽然是自己人，但它读的就是机密路径；它挡住别人的同时也站在最前面。
- `dshmarket` 里的一键安装等同于执行 `dsh plugin add`，装的都是第三方代码，速度不代表可信度。
