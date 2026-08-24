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
