# Hot reload for web-profile plugin development

No more `dsh web` restarts for code changes. The harness ships a Cordis HMR
service (`@deepseek-ai/cordis-plugin-hmr`) in every profile boot; the
web-app bundle layer disables it. Our profile patch re-enables it and points
its watcher at this workspace's build outputs.

## Where it is configured

`C:\Users\leona\.dsh\profiles\web\cordis.patch.yml` — the `- id: hmr`
block at the bottom. It watches every plugin's `lib/` output (and
`dsh-companion/dist`). Delete the block to go back to stock behavior.

That same patch file is itself **hot**: rows you add, remove, disable, or
retune apply to the running harness without a restart.

## The dev loop

```powershell
# in the plugin you are working on:
pnpm exec tsdown --watch     # some plugins already ship this as `pnpm dev`
```

Save a source file → tsdown rebuilds `lib/` → HMR reloads the plugin in the
running web profile (logs: `reload plugin at <path>`). Failed builds roll
back cleanly to the previous module version.

## Caveats

- **Browser assets are not hot.** Host code swaps live; client bundles
  (`client/`) need a page refresh after their rebuild.
- **Reload units are plugin entry files.** A plugin providing a service
  cascades disposal through everything injecting it — expect dependent
  churn on save.
- **Framework files trigger a full restart** by design (they are the CLI
  entry's dependency tree, classified as externals).
- **`--patch` overlays passed on argv are NOT watched.** Only the profile
  patch (`~/.dsh/profiles/web/cordis.patch.yml`) and the home patch
  (`~/.dsh/cordis.patch.yml`) get live config reload. Put long-lived rows in
  the profile patch.
- Newly created files only participate once something imports them; brand-
  new plugins should be added as rows in the watched profile patch.

## One-time activation

The block takes effect on the next `dsh web` boot (the running session
started before it existed). Every boot after that needs no restarts.
