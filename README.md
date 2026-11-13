# dsh-doc

DSH 使用与插件机制的实践笔记。所有结论都在本机 `web` profile 上实际验证过。

## 文档目录

| 文档 | 内容 |
| --- | --- |
| [DSH 常用插件选型与实测](dsh-plugin-selection.md) | 按插件市场注册表的热度、能力声明和红线标记选出 10 个插件，逐个记录实测结论、副作用汇总、安装与回退方法 |
| [DSH 插件市场 dshmarket 的加载机制](dshmarket-plugin-loading.md) | profile 的 patch 里没有 dshmarket 配置，它为什么还能被加载：安装时依赖怎么进层、boot 时如何合成进程树、包内自带的 patch 挂载点、热加载验证、卸载回退，以及 `$DSH_HOME` 下的文件改动 |
| [配置 DSH 会话记录目录](session-log-directory.md) | 用 home 级 patch 把会话记录从 `$DSH_HOME/sessions` 移到仓库根的 `dsh-sessions/`，这样重置 `$DSH_HOME` 时不会删掉历史会话；含验证方法和目录结构 |

## 约定

- `$DSH_HOME` 默认是 `~/.dsh`。
- 文中的 profile 均指 `web`。
