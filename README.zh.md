[English](README.md) | 简体中文

# DSH 插件

DeepSeek Harness 插件的主仓库。每个插件仍是独立的 Git 仓库，并以 Git 子模块形式包含在此，因此插件的历史记录、分支、发行版和 CI 保持隔离。

## 插件

| 插件 | 仓库 | 用途 |
|---|---|---|
| [dsh-auto-chat-titles](./dsh-auto-chat-titles) | [leonardoxr/dsh-auto-chat-titles](https://github.com/leonardoxr/dsh-auto-chat-titles) | 语义化、可配置的聊天标题 |
| [dsh-claude-usage](./dsh-claude-usage) | [leonardoxr/dsh-claude-usage](https://github.com/leonardoxr/dsh-claude-usage) | Claude 用量 UI |
| [dsh-codex-usage](./dsh-codex-usage) | [leonardoxr/dsh-codex-usage](https://github.com/leonardoxr/dsh-codex-usage) | Codex 用量 UI |
| [dsh-coding-tools](./dsh-coding-tools) | [leonardoxr/dsh-coding-tools](https://github.com/leonardoxr/dsh-coding-tools) | LSP、ast-grep 和版本化编辑工具 |
| [dsh-companion](./dsh-companion) | [leonardoxr/dsh-companion](https://github.com/leonardoxr/dsh-companion) | 只读工作区/会话 API 和通知 |
| [dsh-harness-updater](./dsh-harness-updater) | [leonardoxr/dsh-harness-updater](https://github.com/leonardoxr/dsh-harness-updater) | Claude Code / Codex CLI 更新检测和一键更新 |
| [dsh-image-preview](./dsh-image-preview) | [leonardoxr/dsh-image-preview](https://github.com/leonardoxr/dsh-image-preview) | 内联图像预览和剪贴板复制 |
| [dsh-native](./dsh-native) | [leonardoxr/dsh-native](https://github.com/leonardoxr/dsh-native) | DSH Web 服务器的原生桌面和移动端外壳 |
| [dsh-plugin-manager](./dsh-plugin-manager) | [leonardoxr/dsh-plugin-manager](https://github.com/leonardoxr/dsh-plugin-manager) | 配置文件插件管理 |
| [dsh-routed-subagent](./dsh-routed-subagent) | [leonardoxr/dsh-routed-subagent](https://github.com/leonardoxr/dsh-routed-subagent) | 按复杂度路由的子代理委派 |
| [dsh-status-bar-config](./dsh-status-bar-config) | [leonardoxr/dsh-status-bar-config](https://github.com/leonardoxr/dsh-status-bar-config) | 对话统计信息行 |
| [dsh-workspace-git](./dsh-workspace-git) | [leonardoxr/dsh-workspace-git](https://github.com/leonardoxr/dsh-workspace-git) | 工作区 Git 集成 |

## dsh-all-in-one

本仓库还发布 **dsh-all-in-one**，这是一个聚合 DSH bundle，可一步安装本集合中的所有 Web 插件，并将它们挂载为单个 bundle 层：

```sh
# published channel
dsh plugin --profile <name> add dsh-all-in-one

# or pinned straight from this repository's releases
dsh plugin --profile <name> add https://github.com/leonardoxr/dsh-plugins/releases/download/v<version>/dsh-all-in-one-<version>.tgz
```

特性：

- **每个插件一行，使用规范 ID。** 每个聚合插件都保留其独立 bundle 使用的相同 Cordis 行 ID，因此从独立插件迁移到 all-in-one 时，配置文件补丁中的配置覆盖无需更改即可继续工作。
- **二选一——绝不能同时使用。** 此套件会取代独立 bundle。根据加载器设计，同时挂载独立 bundle 和此套件会导致启动致命错误（“duplicate loader entry id”，即使其中一行已禁用也是如此）。添加此套件前，请移除独立安装项。每个插入项还带有一个 `!!js` 回退防护，以避让之前使用不同 ID 挂载的相同软件包；请*最后*安装 all-in-one（`dsh plugin add` 会自动这样做），使防护能够看到之前挂载的所有内容。
- **通过生成获得，绝不手动编辑。** `aggregate.yml` 是唯一事实来源；`package.json` 依赖项和 `cordis.patch.yml` 由 `scripts/aggregate.mjs` 生成：

  ```sh
  pnpm generate   # regenerate both outputs from aggregate.yml
  pnpm verify     # fail when outputs drifted (runs on prepack too)
  ```

- **聚合后仍可按插件选择退出。** 用户仍可在其配置文件补丁中禁用任意子集（`- id: <row-id>` + `disabled: true`）。

对于已发布的软件包，依赖固定项使用精确的 npm 版本；否则使用 GitHub Release tarball URL。若要将固定项切换到 npm，请编辑 `aggregate.yml` 中的 channel/spec 并重新生成。

## 仓库模型

- 本仓库负责协调插件集合；它不会合并各插件的历史记录。
- 插件目录是独立的 Git 仓库。请使用 `git -C <plugin> ...` 进行插件工作。
- 使用 `git -C <plugin> pull`，然后执行 `git add <plugin>`，即可更新主仓库中固定的插件修订版本。
- 不要在此处或插件内部提交 DSH 源码更改。插件必须通过 DSH 配置文件和 `cordis.patch.yml` 机制挂载。
- 代码更改遵循各插件的贡献规则；仅涉及文档的集合更改可以在此处进行。

## DSH 插件行为

插件是带有可选主机端和客户端部分的 Cordis 模块。软件包清单声明 DSH bundle 补丁和客户端注入元数据。主机端代码必须使用 DSH 公共服务/路由；客户端代码必须使用注入的服务，并且必须能够干净卸载。

## 本地开发

```sh
# Clone this collection with plugin repositories
git clone --recurse-submodules <master-repository-url> dsh-plugins

# Initialize plugins after a normal clone
git submodule update --init --recursive

# Work inside one plugin
cd <plugin-directory>
pnpm install
pnpm build
```

使用官方插件机制将本地插件安装到 DSH Web 配置文件中，然后刷新现有的 DSH UI。不要修改 DSH checkout。

## 文档参考

- [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)
- [DSH 插件注册表](https://github.com/dsh-external/plugin-registry)
