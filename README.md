# DSH Plugins

A master repository for DeepSeek Harness plugins. Each plugin remains an independent Git repository and is included here as a Git submodule, so plugin history, branches, releases, and CI stay isolated.

## Plugins

| Plugin | Repository | Purpose |
|---|---|---|
| [dsh-auto-chat-titles](./dsh-auto-chat-titles) | [leonardoxr/dsh-auto-chat-titles](https://github.com/leonardoxr/dsh-auto-chat-titles) | Semantic, configurable chat titles |
| [dsh-better-sidebar](./dsh-better-sidebar) | [leonardoxr/DSH-better-sidebar](https://github.com/leonardoxr/DSH-better-sidebar) | Sidebar workbench and extension API |
| [dsh-codex-usage](./dsh-codex-usage) | [leonardoxr/dsh-codex-usage](https://github.com/leonardoxr/dsh-codex-usage) | Codex usage UI |
| [dsh-companion](./dsh-companion) | [leonardoxr/dsh-companion](https://github.com/leonardoxr/dsh-companion) | Read-only workspace/session API and notifications |
| [dsh-native](./dsh-native) | [leonardoxr/dsh-native](https://github.com/leonardoxr/dsh-native) | Native desktop and mobile shell for DSH web servers |
| [dsh-routed-subagent](./dsh-routed-subagent) | [leonardoxr/dsh-routed-subagent](https://github.com/leonardoxr/dsh-routed-subagent) | Complexity-routed subagent delegation |
| [dsh-workspace-git](./dsh-workspace-git) | [leonardoxr/dsh-workspace-git](https://github.com/leonardoxr/dsh-workspace-git) | Workspace Git integration |

## Repository model

- This repository coordinates the plugin collection; it does not merge their histories.
- Plugin directories are independent Git repositories. Use `git -C <plugin> ...` for plugin work.
- Update the pinned plugin revision in the master repo with `git -C <plugin> pull` followed by `git add <plugin>`.
- Do not commit DSH source changes here or inside a plugin. Plugins must be mounted through the DSH profile and `cordis.patch.yml` mechanism.
- Code changes follow each plugin’s contribution rules; documentation-only collection changes may be made here.

## DSH plugin behavior

Plugins are Cordis modules with optional host and client halves. The package manifest declares the DSH bundle patch and client injection metadata. Host code must use public DSH services/routes; client code must use injected services and must unload cleanly.

For better-sidebar integrations, see [`dsh-better-sidebar/AGENTS.md`](./dsh-better-sidebar/AGENTS.md). Key rules: use `inject = ['betterSidebar']`, register tabs/viewers through `ctx.betterSidebar`, wrap registration in `ctx.effect()` so the disposer is HMR-safe, use package-prefixed IDs, and keep `dsh-better-sidebar` as an optional peer dependency. The service exists only on the client side.

## Local development

```sh
# Clone this collection with plugin repositories
git clone --recurse-submodules <master-repository-url> dsh-plugins

# Initialize plugins after a normal clone
git submodule update --init --recursive

# Work inside one plugin
cd dsh-better-sidebar
pnpm install
pnpm build
```

Install a local plugin into a DSH web profile using the official plugin mechanism, then refresh the existing DSH UI. Do not modify the DSH checkout.

## Documentation references

- [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)
- [DSH plugin registry](https://github.com/dsh-external/plugin-registry)
- [better-sidebar consumer guide](./dsh-better-sidebar/AGENTS.md)
