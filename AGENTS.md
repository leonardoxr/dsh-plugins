# DSH Plugins Master Repository

This is a coordination repository. Plugin directories retain independent Git histories and remotes. Do not flatten or remove nested `.git` repositories.

## Rules

- Never modify DeepSeek Harness source.
- Never restart DSH Web without asking the user first and receiving explicit approval.
- Mount plugins through DSH profiles and `cordis.patch.yml`; do not patch DSH.
- Keep changes scoped to the plugin being changed.
- Preserve plugin-specific AGENTS.md instructions.
- Use package-prefixed IDs and public DSH service APIs.
- For better-sidebar consumers, use `ctx.effect(() => register...)` and optional peer dependencies.

## Plugin authoring standards

Follow the official guides at https://deepseek-harness.github.io/deepseek-harness/en/develop/basic/ ("Your first plugin", "Build a tool", "Plugin configuration", "Package and install a plugin"):

- A plugin is a TypeScript module exporting an `apply(ctx)` function and a unique exported `name`. The framework calls `apply` on load and passes the `ctx` used to register capabilities.
- Prefer function form. Use object form when convenient; use class form (`extends Service`) only when the plugin provides a service consumed by other plugins.
- Declare every consumed service in the exported `inject` array (e.g. `export const inject = ['tools']`). The framework waits for required services before loading.
- Anything registered through `ctx` cleans up automatically on unload — never call `removeListener`/`clearInterval` manually. Use `ctx.effect(() => { ...; return () => dispose() })` only for resources Cordis does not own (network connections, external handles).
- Tools: define them with `defineTool` from `@deepseek-ai/dsh-tools`, inject `'tools'`, and register via `ctx.tools.register(...)`. `execute` returns the canonical value declared by `output.schema`; `output.render` converts it to model-facing content.
- Configuration: export a `Config` type plus a same-named Schemastery schema with defaults on the schema fields; never export a plain object as `Config`. Never hardcode tunable values — anything two deployments may set differently must be a config field, and constraints must fail loudly at load time through the schema.
- Local mounting: insert plugin rows in a `cordis.patch.yml` overlay with absolute source paths, then run `pnpm dsh web --patch ./<plugin>/cordis.patch.yml`. A patch contributes config but does not change the profile directory modules resolve from.
- Layer order matters: bundle patches apply in the profile's `dsh.profile.bundles` order, then the profile patch, then the home patch, then `--patch` overlays in argv order. Later layers win per row and replace a row's entire config value — overrides must restate every key, no deep merging.
- Packaging: an installable plugin is a bundle whose `package.json` declares `dsh.bundle` pointing at its `cordis.patch.yml`; patch rows reference modules by package name. For git-host installs ship a self-contained `prepare` script that builds the published entry points from source without assuming sibling monorepo checkouts.

### Always expose settings in the Web UI

Any option a user can toggle at runtime ships in the DSH settings UI, like every existing plugin does — never bury preferences behind hand-edited YAML, env vars, or code edits. Follow the established house pattern:

- Give the plugin a client half with `inject = ['slots', 'settingsScope']` and declare `@deepseek-ai/dsh-client-ui-settings` under `dsh.client.inject` in `package.json`; add `import type {} from '@deepseek-ai/dsh-client-ui-settings/client'` for the slot type augmentation.
- Persist preferences through a namespaced settings scope: `ctx.settingsScope.bind<Settings>({ namespace: '<plugin-name>', decode })`.
- Register the settings card through the slot API inside a lifecycle-managed registration: `ctx.slots.inject('settings.plugin.item', () => ctx.slots.register({ name: 'settings.plugin.item', key: '<plugin-name>' }, SettingsCard))`.
- Build the card like the existing ones (`SettingsCard.tsx`): typed draft state, explicit save/reset actions, styles scoped with a plugin prefix and themed only via DSH design tokens (`--dsw-alias-*` / `--ds-*`) so skins work.
- Keep it HMR-safe (all registrations via `ctx.effect`/`ctx.slots.inject` disposers) and cover the card with a client test.
- Reference implementations: `dsh-claude-usage`, `dsh-codex-usage`, `dsh-conversation-stats`, `dsh-image-preview`, `dsh-workspace-git`.

Deployment-level tunables still belong in the host-side Schemastery `Config`; the settings UI carries the user's runtime preferences.

### Hot-reload development loop

Hot module reload is enabled in this deployment: the web profile patch (`C:\Users\leona\.dsh\profiles\web\cordis.patch.yml`) re-enables the harness-shipped Cordis HMR service with watchers on every plugin's build output (`lib/`, `dist/` for `dsh-companion`). Full details in [HOT-RELOAD.md](./HOT-RELOAD.md).

- Never restart DSH Web to apply plugin code changes. Run a build watcher in the plugin you are editing (`pnpm exec tsdown --watch`, or `pnpm dev` where defined) — each save rebuilds `lib/` and the running harness hot-reloads that plugin in place. Failed builds roll back to the previous version automatically.
- The profile patch and home patch files are themselves hot: adding, removing, disabling, or retuning rows applies live without a restart. Long-lived rows belong there — `--patch` overlays passed on argv are NOT watched and stay frozen until the next boot.
- When a new plugin is mounted or an existing one needs a permanent config override, edit the watched profile patch rather than passing another argv overlay.
- Reload units are whole plugin entry files: a plugin that provides a service cascades disposal through everything injecting it, so expect dependent churn on save. Registering everything through `ctx` keeps that safe.
- Browser-only client bundles (`client/`) are not hot: refresh the page after their rebuild. Host code never needs a refresh or restart.
- Touching framework files (the CLI entry's dependency tree) intentionally triggers a full process restart — do not work around it; keep plugin code self-contained so it stays in the fast path.
- After changing what HMR watches (the `- id: hmr` block's `root` list when a plugin gains a different output directory), one `dsh web` boot is needed — ask the user first per the rules above.

## New plugin lifecycle

Whenever we create a new plugin:

1. Create its own directory in this workspace with its own independent Git history (`git init`, default branch `main`). Never nest one plugin repo inside another.
2. Follow the proper setup: create the GitHub repository under the owner account, add it as `origin`, push the initial commit of `main`, then register the plugin as a submodule of this coordination repo (updates `.gitmodules`) and commit that here too.
3. Add repository topics with `gh repo edit <owner>/<repo> --add-topic ...`. The topic `dsh-plugin` is mandatory on every plugin repository — it is the main topic — alongside accurate extras such as `deepseek-harness`, `cordis`, and what the plugin does.
4. Package names start with `dsh-`; plugin IDs are package-prefixed.

## Commits and releases

- We always commit to `main` and push whenever we finish a new version of stuff. Finishing work means: changes committed on `main`, pushed to `origin/main` — including any version bump in `package.json`.
- Commit each coherent unit of work as it completes rather than in bulk at the end of a session.
- Feature branches may be used for experiments, but a finished version always lands on `main` and is pushed before the work is reported done.

### Correct publish on every finished change

Never finish new features or fixes without shipping them properly. Every completed unit of work gets the full release treatment:

1. **Bump the version** — update `version` in the plugin's `package.json` following semver before committing the work: patch (`0.1.6` → `0.1.7`) for bug fixes, minor (`0.1.x` → `0.2.0`) for new features, major (`0.x.y` → `1.0.0`) for breaking changes. Never commit shipped changes while leaving the version untouched.
2. **Verify green** — run the plugin's full gate (`typecheck`, `test`, `build`) before publishing. The `prepack`/`prepublishOnly` scripts enforce this; let them run, never bypass or skip them.
3. **Commit and push** — land the version bump together with the finished work on `main` and push to `origin/main`.
4. **Tag the release** — create git tag `vX.Y.Z` exactly matching `package.json` and push it (`git push origin vX.Y.Z`). A mismatched tag must never ship; fix the version instead of retagging.
5. **Publish the package** — if the repo has a release workflow (e.g. `release.yml`), creating the GitHub Release for the tag triggers the automatic npm publish with provenance; otherwise publish manually with `pnpm publish --access public` from a clean, checked-out `main` after the checks pass. For git-host-only distribution, confirm the `prepare` script builds the published entry points standalone before tagging.

Keep `main` release-ready at all times: any commit on `main` must be safe to tag and publish as-is.
