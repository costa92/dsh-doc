# 配置 DSH 会话记录目录

DSH 的会话记录由 `@deepseek-ai/dsh-session-persistence-jsonl` 插件写入，目录由它的 `root` 配置决定。base bundle 给出的值是 `root: !!js dshHomePath('sessions')`，即 `$DSH_HOME/sessions`（默认 `$DSH_HOME` 是 `~/.dsh`）。

本工作区把会话记录放在 `$DSH_HOME` 之外，落到仓库根的 `dsh-sessions/` 目录，这样重置 `$DSH_HOME` 时不会连带清掉历史会话。

## 配置方式

在 home 级 patch 层 `$DSH_HOME/cordis.patch.yml` 中按插件 id 覆盖 `root`：

```yaml
# Home-level patch layer, applied to every profile after its own cordis.patch.yml.
# Keep session logs outside $DSH_HOME so resetting ~/.dsh does not wipe them.
- id: session-persistence-jsonl
  name: '@deepseek-ai/dsh-session-persistence-jsonl'
  config:
    root: /absolute/path/to/dsh-sessions
```

`root` 必须是绝对路径，目录不存在时会在首次写入会话时创建。填本工作区的值时，在仓库根执行 `echo "$(pwd)/dsh-sessions"` 拿到绝对路径。

改完需重启 dsh 才生效，除非 profile 启用了 `dsh-hmr`。

### 为什么写在 home 级 patch

profile 配置按以下顺序叠加，后者覆盖前者：

1. `dsh.profile.bundles` 中各组合包的 patch
2. profile 自身的 `$DSH_HOME/profiles/<name>/cordis.patch.yml`
3. home 级的 `$DSH_HOME/cordis.patch.yml`
4. 命令行 `--patch` 指定的覆盖层

写在第 3 层，web、sdk 等所有 profile 共用同一个会话目录。只想对单个 profile 生效时，改写到该 profile 的 `cordis.patch.yml`。

## 验证

不启动 dsh，直接查看合成后的配置：

```bash
dsh --profile web --dump-config | grep -A3 'id: session-persistence-jsonl'
dsh --profile sdk --dump-config | grep -A3 'id: session-persistence-jsonl'
```

两条命令的 `root` 应指向同一个目录。

## 目录结构与读取

```text
dsh-sessions/
  --<normalized-cwd>--/        # 按会话工作目录分组（无 cwd 时为 _no-cwd/）
    <encoded-session-id>/
      session.v4.jsonl.zstd     # 当前格式，默认 zstd 压缩
      session.lock
```

压缩日志可以这样逐行查看：

```bash
zstd -dc dsh-sessions/<project>/<session-id>/session.v4.jsonl.zstd | head
```

需要外部工具直接按行读取时，可加 `compression: 'none'`。但一个 `root` 只能使用一种编码，切换编码需换一个新目录。

## 注意事项

- 不会自动清理：插件没有删除接口，会话文件会在 `root` 下一直累积，需手动清理。
- 迁移旧会话：目录结构与 `$DSH_HOME/sessions` 相同，把旧的项目目录整体拷到新 `root` 下即可继续使用。
- 单写者：同一会话同时只允许一个进程写入（`session.lock` 上的 `flock`），不要把 `root` 放在 NFSv3 等 flock 不可靠的网络文件系统上，也不要放在 OneDrive 同步目录中。
- 格式代次：文件名里的 `v4` 是当前 Session 格式版本（`SESSION_FORMAT_VERSION = 4`）。旧代文件保持原名不改写，新写入走 v4 路径。
