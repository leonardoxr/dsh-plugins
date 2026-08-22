# DSH Plugins Master Repository

This is a coordination repository. Plugin directories retain independent Git histories and remotes. Do not flatten or remove nested `.git` repositories.

## Rules

- Never modify DeepSeek Harness source.
- Mount plugins through DSH profiles and `cordis.patch.yml`; do not patch DSH.
- Keep changes scoped to the plugin being changed.
- Preserve plugin-specific AGENTS.md instructions.
- Use package-prefixed IDs and public DSH service APIs.
- For better-sidebar consumers, use `ctx.effect(() => register...)` and optional peer dependencies.
