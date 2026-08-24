# Unified DSH Coding Tools Plugin Plan

## Status

Planning document for a new, independent DSH plugin repository coordinated by this master repository.

Working package and directory name: `dsh-coding-tools`.

The package should present itself to users as **Tools** or **Coding Tools**. The working package name intentionally avoids confusion with the existing core package `@deepseek-ai/dsh-tools`, which owns the DSH tool registry rather than these model-facing coding capabilities.

## Executive Summary

Build one installable Cordis plugin that contributes a cohesive suite of high-value coding tools to DeepSeek Harness without modifying DSH itself:

- Persistent, read-only Language Server Protocol queries
- Fresh diagnostics after controlled file mutations
- Structural search with ast-grep
- Exact-version, observed-range code reads and edits
- Transactional structural edit previews and applies
- Debug Adapter Protocol support after the safer tools are mature
- Compact, typed, paginated outputs designed to reduce model token usage

The plugin is one package, one repository, and one DSH bundle row. Internally it is divided into lifecycle-owned modules so each subsystem can be enabled, disabled, tested, and disposed independently.

The highest-value release is not the final all-features release. The first production milestone should contain read-only LSP, compact results, ast-grep, and exact-version range edits. Structural mutation and debugging follow behind feature flags after their security and transaction requirements are proven.

## Primary Objective

Reduce the number of model turns, file reads, grep calls, build attempts, stale edit retries, and prompt tokens required for repository-scale coding work while improving correctness.

The plugin succeeds when an agent can:

1. Ask semantic questions without reading many files.
2. Find structural code patterns without fragile regexes.
3. Edit only content it actually observed and fail safely on drift.
4. Receive fresh, compact diagnostics immediately after a controlled edit.
5. Preview large structural changes exactly before any bytes are changed.
6. Debug processes through a bounded, session-owned protocol implementation when explicitly enabled.

## Repository and DSH Constraints

The implementation must follow the master repository rules:

- Remain an independent Git repository mounted here as a submodule.
- Never modify DeepSeek Harness source.
- Mount through the plugin package's DSH bundle patch and profile composition.
- Use only public DSH and Cordis services.
- Use package-prefixed Cordis row IDs and service names.
- Own every listener, timer, tool registration, process, pending request, and temporary resource through Cordis lifecycle effects.
- Keep host-only functionality host-only. A client plugin is optional and should not be added until a clear UI requirement exists.
- Preserve unrelated submodule state in this coordination repository.

If a required security or filesystem operation has no suitable public DSH service, that feature must remain disabled and the missing public contract must be documented. The plugin must not bypass the gap with DSH internal imports.

## Scope

### Included

- One unified plugin package
- One Cordis bundle entry
- Multiple model-facing tools registered by that package
- Shared process, path, output, and protocol infrastructure
- Per-feature settings and feature flags
- Code Mode-compatible canonical typed outputs
- Native function-calling compatibility
- Tests, benchmarks, security fixtures, and installation documentation

### Explicitly Excluded

- Patching the built-in DSH filesystem, shell, tool registry, presets, or agent loop
- Automatic execution of project-provided binaries without a trust or approval decision
- Automatic application of server-originated LSP edits
- Unconfined file URIs or process working directories
- Four-character or other short hashes as authoritative file identity
- Structural edits that rerun a query and compare only result counts after writing
- Process-global debugger state shared across DSH sessions
- Unbounded protocol frames, output, diagnostics, memory reads, file counts, or replacement counts
- Making DAP/debug enabled by default in the first production release

## Product Shape

### Package

Working package name:

`dsh-coding-tools`

Suggested metadata:

- Display name: `Tools`
- Description: `Semantic code intelligence, structural search, versioned edits, and debugging tools for DeepSeek Harness.`
- DSH bundle patch: `./cordis.patch.yml`
- Host entry: `lib/index.js`
- No client entry in the initial releases

### Cordis bundle row

`cordis.patch.yml` should insert exactly one profile row:

```yaml
- insert:
    - id: dsh-coding-tools
      name: 'dsh-coding-tools'
```

The user profile may replace the row's complete configuration:

```yaml
- id: dsh-coding-tools
  config:
    lsp:
      enabled: true
      diagnosticsOnWrite: true
    ast:
      grepEnabled: true
      editEnabled: false
    versionedEdit:
      enabled: true
    debug:
      enabled: false
```

### Model-facing tools

The final package may register these tools:

| Tool | Default | Purpose |
|---|---:|---|
| `code_read` | On | Read bounded code ranges while recording exact file identity and observed ranges. |
| `edit_ranges` | On | Apply exact-version, non-overlapping line/range edits to observed content. |
| `lsp` | On, read-only actions | Diagnostics and semantic navigation through a persistent language server. |
| `ast_grep` | On | Read-only structural search with bounded, paginated results. |
| `ast_edit` | Off until hardened | Exact preview and transactional apply of structural rewrites. |
| `debug` | Off | Session-owned DAP debugging with strict process and protocol policy. |

One package does not imply one giant tool schema. Each capability remains a distinct model tool so permissions, result budgets, descriptions, and failure semantics stay clear. DSH Code Mode will make them available through the generated typed SDK.

## High-Level Architecture

```text
DSH profile
  └─ dsh-coding-tools Cordis entry
       ├─ Configuration and feature manager
       ├─ Workspace/path authority service
       ├─ Compact result and pagination service
       ├─ Snapshot/observation service
       ├─ LSP workspace pool
       │    └─ lsp tool
       ├─ AST runner
       │    ├─ ast_grep tool
       │    └─ ast_edit tool
       ├─ Versioned filesystem tools
       │    ├─ code_read tool
       │    └─ edit_ranges tool
       └─ DAP session manager
            └─ debug tool
```

### One package, isolated internal modules

The top-level plugin should mount internal Cordis plugins or effect-owned modules rather than placing every subsystem in one function. This provides independent cleanup and makes feature toggles predictable.

Suggested internal modules:

- `ConfigurationModule`
- `WorkspaceAuthorityModule`
- `ResultBudgetModule`
- `SnapshotModule`
- `LspModule`
- `AstModule`
- `VersionedEditModule`
- `DebugModule`

Each module must unregister tools and stop owned resources when disabled, reconfigured, or disposed.

### State ownership

- Tool registration: owned by the plugin/module fiber.
- File observations: keyed by DSH agent or session plus canonical workspace and file path.
- LSP clients: shared only by canonical workspace root plus complete server configuration fingerprint.
- DAP sessions: keyed by DSH ToolSession/Agent; never module-global across sessions.
- Pagination cursors: scoped to owner session and bounded by count, bytes, and TTL.
- Pending AST previews: scoped to owner session, single-use, expiring, and bound to exact source identities.

## Phase 0: Public Contract and Threat-Model Spike

Before implementation, prove the public seams required by the plugin.

### DSH contracts to verify

- `ctx.tools.register(defineTool(...))`
- Canonical structured output schemas and rendering
- Tool cancellation through `exec.signal`
- Agent/session identity through tool execution context
- Public filesystem provider methods
- Public filesystem observation and mutation-intent events
- Public sandbox-policy or workspace-root resolution
- Public user-approval integration for process launch and mutations
- Public output retention/spill APIs
- Public settings-section registration
- Public jobs APIs only if background work becomes model-visible
- Public events suitable for observing built-in `read`, `grep`, `write`, and `edit` without replacing them

### Mandatory decision gates

1. If project-configured process execution cannot be routed through a public approval/policy contract, project LSP/DAP config remains disabled.
2. If filesystem mutations cannot be routed through public DSH policy, `edit_ranges`, LSP mutation actions, and `ast_edit` remain disabled.
3. If built-in read observations cannot be consumed safely, the package uses its own `code_read` tool rather than importing DSH internals.
4. If DSH output spilling is unavailable publicly, results use bounded pagination and never return an unbounded inline payload.

### Threat model

Treat these inputs as untrusted:

- Repository contents
- Project-local LSP and DAP configuration
- LSP and DAP protocol messages
- File URIs returned by servers
- Server-originated commands and edit requests
- AST patterns and replacements
- Paths, globs, cursors, and snapshot tokens supplied by the model
- Debuggee output, variables, memory, and adapter events

Assume a repository can be intentionally malicious.

## Shared Security Foundation

### Canonical workspace authority

Every filesystem path must pass through one authority function that:

1. Decodes and normalizes the input or file URI.
2. Resolves it against the declared workspace when relative.
3. Resolves symlinks for existing path components.
4. rejects device paths, alternate data streams where applicable, and malformed URIs.
5. Proves the resulting path is within one of the DSH-authorized roots.
6. Repeats the authority check immediately before mutation.

A path being display-relative to `cwd` is not evidence that it is confined to `cwd`.

### Process authority

Every external executable must have a resolved execution plan:

- Executable path
- Arguments
- Working directory
- Environment allowlist
- Workspace/session owner
- Reason for execution
- Timeout and output budgets
- Cancellation behavior

Project configuration must never silently select an executable for a read-tier tool. The resolved plan must pass DSH policy and, where required, user approval.

Ambient credentials must not be inherited by default. Environment variables should use an explicit allowlist.

### Protocol limits

All JSON-RPC, LSP, and DAP transports must enforce:

- Maximum header bytes
- Maximum declared content length
- Maximum decoded JSON depth if practical
- Maximum pending requests
- Maximum event queue length
- Maximum output bytes
- Maximum log bytes
- Per-request timeout
- Initialization timeout
- Idle timeout
- Graceful shutdown timeout followed by forced cleanup

Transport writes and flushes must be awaited. Backpressure must not be discarded.

### Mutation policy

No protocol peer may directly write files.

- LSP `workspace/applyEdit`: reject by default or convert to a plugin-owned pending proposal.
- LSP rename/code actions: return an exact edit manifest and require explicit apply.
- AST edit: preview exact bytes, then apply that exact preview.
- DAP `runInTerminal`: route through normal process authority and a fresh policy decision.
- DAP `startDebugging`: route each child session through a fresh policy decision.

### Approval quality

Approval details must show all consequential values, not only the high-level action:

- Resolved executable and arguments
- Working directory
- Target file paths
- Rename destinations
- LSP method and command identifier
- DAP attach endpoint or process ID
- Whether termination may kill an attached process
- Estimated number of files, replacements, and changed bytes

## Tool Contract: `code_read`

### Purpose

Provide a bounded code read that records exact file identity and the precise ranges returned to the model. It exists because a standalone plugin must not depend on private built-in read bookkeeping.

### Input

```ts
{
  file: string
  offset?: number
  limit?: number
}
```

### Output

```ts
{
  file: string
  version: string
  offset: number
  totalLines: number
  lines: Array<{ number: number; text: string }>
  truncated: boolean
}
```

### Requirements

- UTF-8 text only in the first release.
- Exact-byte content identity using SHA-256 truncated to at least 128 bits or an equivalent 128-bit-or-stronger digest.
- Do not normalize line endings or trailing whitespace before computing identity.
- Record observed line intervals per agent/session, path, and version.
- Bound returned lines and bytes.
- Reject unsupported binary content cleanly.
- Preserve a small bounded snapshot only when needed for later conflict diagnostics.
- Prefer a compact base64url or similarly token-efficient version encoding.

### Token behavior

- Default to a small line window.
- Do not repeat unchanged file metadata in verbose prose.
- Return line objects canonically in Code Mode and a compact rendered form for native mode.

## Tool Contract: `edit_ranges`

### Purpose

Apply multiple exact-version edits without requiring the model to resend large `old_string` blocks.

### Input

```ts
{
  file: string
  version: string
  edits: Array<{
    startLine: number
    endLine: number
    replacement: string
  }>
}
```

Line-number semantics must be documented precisely. The preferred initial contract is one-indexed inclusive source lines, with insertion represented by a separately discriminated operation rather than special line-number sentinels.

A stronger final schema should use discriminated operations:

```ts
type RangeEdit =
  | { kind: 'replace'; startLine: number; endLine: number; replacement: string }
  | { kind: 'insert_before'; line: number; text: string }
  | { kind: 'insert_after'; line: number; text: string }
  | { kind: 'append'; text: string }
  | { kind: 'delete'; startLine: number; endLine: number }
```

### Validation

- Current exact file version must match.
- Every touched source range must have been observed at that same version unless the operation is an explicitly allowed append.
- Ranges must be ordered or normalized and must not overlap.
- All operations are evaluated against the original version, not earlier edits in the same call.
- File and changed-byte limits apply before mutation.
- DSH filesystem policy is rechecked immediately before write.
- If anything fails, write zero bytes.

### Output

```ts
{
  file: string
  oldVersion: string
  newVersion: string
  changedLines: number
  diffPreview: string
  diagnostics?: DiagnosticSummary
}
```

Large diffs must spill or paginate. The inline preview should focus on changed regions.

### Conflict response

On version mismatch, return:

- Current version
- Whether the original snapshot is still retained
- Small bounded context around attempted ranges when safe
- A clear instruction to reread

Do not automatically remap stale edits in the first release. Conservative remapping may be considered later only if it fails closed on ambiguity and has dedicated tests.

## Tool Contract: `lsp`

### Initial read-only actions

- `status`
- `diagnostics`
- `definition`
- `references`
- `hover`
- `symbols`
- `implementation`
- `type_definition`
- `capabilities`
- `reload`

`reload` changes server state and must use the appropriate policy tier even though it does not directly edit files.

### Later mutation actions

- `rename`
- `rename_file`
- `code_actions`
- Explicit raw `request` only if a safe policy can be expressed

Mutation actions should produce proposals by default. Applying a proposal must be a separate action with an opaque, exact manifest token.

### Schema design

Prefer a discriminated union keyed by `action`, so irrelevant fields cannot be supplied and approvals can inspect typed consequences.

If the public DSH tool schema cannot render a discriminated union, use strict action-specific runtime validation and consider registering separate read and mutation tools rather than falling back to a broad all-optional schema.

### LSP lifecycle

- Server pool key: canonical workspace root plus complete normalized server configuration fingerprint.
- Single-flight initialization.
- Initialization-failure backoff.
- Lazy startup.
- Bounded warm query timeout.
- Versioned `didOpen` and `didChange` overlays.
- `didSave` only after the filesystem reports the landed content.
- Bounded graceful shutdown followed by forced process cleanup.
- No publication of a client until initialization completes.

### Server discovery

Initial production support should be explicit configuration plus a safe built-in `clangd` definition. Auto-discovery of repository-local binaries is disabled until trusted-workspace and process-approval behavior is complete.

For LostTale and other C++ workspaces, support explicit selection of the compilation database directory. Do not guess between incompatible platform build trees silently.

### Diagnostics after write

For plugin-owned `edit_ranges` and `ast_edit` applies:

1. Synchronize exactly what landed.
2. Capture the expected document/server version.
3. Wait for a short inline diagnostics window.
4. Return only fresh diagnostics relevant to touched files, prioritizing errors and touched ranges.
5. Continue a bounded deferred wait only through a DSH-supported delivery mechanism.
6. Deduplicate using location, message, severity, source, and diagnostic code.

Observing built-in DSH `write` and `edit` is optional and may be added only through public events without replacing those tools.

### Result budget

- Definitions: one primary result plus a small alternative set.
- References: grouped by file, default page of 25-50.
- Symbols: paginated, compact names and locations.
- Hover: bounded plaintext/markdown extraction.
- Diagnostics: errors first, touched files first, hard cap plus cursor or spill locator.
- Never return entire server capability documents inline; summarize and spill the raw form.

## Tool Contract: `ast_grep`

### Input

```ts
{
  pattern: string
  paths?: string[]
  language?: string
  limit?: number
  cursor?: string
  strictness?: 'smart' | 'ast' | 'relaxed'
}
```

### Output

```ts
{
  matches: Array<{
    file: string
    startLine: number
    endLine: number
    preview: string
    captures: Record<string, string>
  }>
  filesSearched: number
  totalMatches?: number
  parseErrors: Array<{ file: string; summary: string }>
  nextCursor?: string
}
```

### Implementation approach

Start by invoking a pinned or user-configured `ast-grep` executable through process authority. Do not port OMP's Rust/native layer for the first release.

### Requirements

- Read-only policy tier.
- Explicit file, byte, time, match, and parse-error caps.
- Gitignore-aware scanning where supported.
- Do not follow symlinks by default.
- Deterministic result ordering.
- Honest incomplete-coverage reporting when parse errors occur.
- Cursor must be owner-scoped and tamper-resistant or server-stored.
- Avoid a `skip` design that requires retaining an unbounded prefix.

## Tool Contract: `ast_edit`

### Availability

Disabled by default until all transactional requirements and security tests pass.

### Preview input

```ts
{
  action: 'preview'
  operations: Array<{ pattern: string; replacement: string }>
  paths: string[]
  language?: string
  maxFiles?: number
  maxReplacements?: number
}
```

### Preview output

```ts
{
  action: 'preview'
  previewId: string
  expiresAt: string
  files: number
  replacements: number
  changedBytes: number
  diffPreview: string
  fullDiffLocator?: string
  parseErrors: Array<{ file: string; summary: string }>
}
```

### Immutable preview manifest

Internally, the preview must store exact entries:

```ts
{
  canonicalPath: string
  oldFileVersion: string
  byteRange: { start: number; end: number }
  oldTextDigest: string
  replacementBytes: Uint8Array
}
```

### Apply input

```ts
{
  action: 'apply'
  previewId: string
}
```

### Apply rules

- Apply the exact stored manifest; do not rerun the AST query.
- Revalidate every file identity and policy before the first write.
- If any source differs, write zero files.
- Enforce file, replacement, and changed-byte limits.
- Route all writes through the same public DSH filesystem path as `edit_ranges`.
- Preserve BOM and line endings.
- Hash and report what actually landed.
- Synchronize LSP and collect fresh diagnostics afterward.
- A multi-file OS error must produce explicit partial-write evidence and attempt bounded rollback from prepared original bytes when policy permits.

### Parse errors

The default for refactoring should be fail-on-parse-error for files in scope. Partial application requires an explicit option and a prominent result warning.

## Tool Contract: `debug`

### Availability

Disabled by default. Implement only after LSP, AST search, and versioned editing are production-stable.

### Initial action set

- `launch`
- `attach`
- `sessions`
- `set_breakpoint`
- `remove_breakpoint`
- `continue`
- `pause`
- `step_over`
- `step_in`
- `step_out`
- `threads`
- `stack_trace`
- `scopes`
- `variables`
- `evaluate`
- `output`
- `terminate`

Defer instruction/data breakpoints, memory writes, raw requests, and other advanced operations until the base model is secure.

### Ownership

- One DAP manager per DSH owner session, not one module-global singleton.
- Child sessions remain inside the same approved debug tree.
- Every adapter-created child through `startDebugging` receives a fresh policy decision.
- Active focus is local to the owner session.

### Launch versus attach

Ownership must be explicit:

- A launched process may be terminated if the approved plan allows it.
- An attached process defaults to detach-only.
- Killing an attached process requires a separate explicit option and approval.

### Adapter trust

Project DAP configuration is executable code. Approval must show the resolved adapter executable, argv, cwd, environment policy, connection mode, and target.

### Reverse requests

- `runInTerminal`: disabled by default; when enabled, send the exact command through process authority and a new approval.
- `startDebugging`: disabled by default; when enabled, validate and approve each child session.

### Bounds

- Cap DAP frame size before allocation.
- Cap pending requests and event queues.
- Cap stack frames, variables, variable depth, output, source content, disassembly, and memory reads.
- Classify memory reads conservatively because they may expose secrets.
- Idle cleanup must observe adapter/debuggee events, not only model tool calls.
- Breakpoint state changes only after adapter acknowledgement.
- Await all transport writes and flushes.

### Windows C++ note

Do not claim complete Windows/MSVC debugging until an adapter with verified PDB behavior is installed, licensed appropriately, and covered by integration tests. GDB/LLDB support alone is not sufficient for the primary LostTale Windows build.

## Configuration Model

Suggested initial schema:

```yaml
inlineMaxBytes: 8192
cursorTtlMs: 300000
workspaceRoots: []

codeRead:
  enabled: true
  defaultLines: 200
  maxLines: 2000
  maxBytes: 262144
  snapshotMaxBytes: 67108864

versionedEdit:
  enabled: true
  requireObservedRanges: true
  maxOperations: 100
  maxChangedBytes: 1048576
  diagnosticsOnWrite: true

lsp:
  enabled: true
  allowProjectConfig: false
  allowProjectExecutables: false
  idleTimeoutMs: 300000
  initializeTimeoutMs: 20000
  requestTimeoutMs: 20000
  inlineDiagnosticsWaitMs: 1200
  maxReferencesPerPage: 50
  allowServerApplyEdit: false
  mutationActionsEnabled: false
  servers: {}

ast:
  grepEnabled: true
  editEnabled: false
  executable: ast-grep
  maxFiles: 1000
  maxMatchesPerPage: 50
  maxReplacements: 1000
  maxChangedBytes: 1048576
  timeoutMs: 30000

debug:
  enabled: false
  allowProjectConfig: false
  allowRunInTerminal: false
  allowStartDebugging: false
  maxFrameBytes: 4194304
  maxOutputBytes: 1048576
  maxMemoryReadBytes: 65536
  idleTimeoutMs: 600000
  adapters: {}
```

The actual schema must reject unknown keys where DSH settings infrastructure supports strict schemas. Environment-specific executable paths belong in local configuration, never maintained documentation.

## Compact Output Standard

All tools should share one output policy.

### Defaults

- Inline rendered content target: no more than 8 KiB.
- Return canonical structured data to Code Mode.
- Render only summary, first page, warnings, and continuation locator.
- Group repetitive results by file.
- Use short relative paths where unambiguous but retain canonical paths internally.
- Do not duplicate the same data in both verbose prose and JSON.
- Prioritize errors, touched ranges, and primary definitions.

### Pagination

Cursors must encode or reference:

- Owner session
- Tool and query fingerprint
- Workspace/config generation
- Offset
- Expiration

Reject cross-session, expired, modified-query, or modified-generation cursors.

### Spilling

If DSH exposes public output retention, spill full diagnostics, diffs, capability documents, and debugger payloads through it. Otherwise retain bounded server-side pages and return cursors. Never write undocumented temporary files solely to evade output limits.

## Error Model

Every tool should return or throw errors that distinguish:

- Invalid input
- Policy denial
- Approval denial
- Unsupported capability
- Missing executable
- Untrusted project configuration
- Workspace confinement violation
- Stale file version
- Stale preview
- Parse-incomplete result
- Timeout
- Cancellation
- Protocol failure
- Process exited
- Partial filesystem commit
- Internal invariant failure

A rendered error must never coexist with canonical `success: true` metadata.

## Proposed Source Layout

```text
dsh-coding-tools/
├─ AGENTS.md
├─ README.md
├─ LICENSE
├─ package.json
├─ cordis.patch.yml
├─ tsconfig.json
├─ vitest.config.ts
├─ src/
│  ├─ index.ts
│  ├─ config.ts
│  ├─ invariant.ts
│  ├─ services/
│  │  ├─ workspace-authority.ts
│  │  ├─ process-authority.ts
│  │  ├─ result-budget.ts
│  │  ├─ cursors.ts
│  │  └─ snapshots.ts
│  ├─ protocol/
│  │  ├─ jsonrpc-framer.ts
│  │  ├─ request-registry.ts
│  │  └─ process-transport.ts
│  ├─ lsp/
│  │  ├─ module.ts
│  │  ├─ config.ts
│  │  ├─ client.ts
│  │  ├─ pool.ts
│  │  ├─ diagnostics.ts
│  │  ├─ workspace-edits.ts
│  │  └─ tool.ts
│  ├─ ast/
│  │  ├─ module.ts
│  │  ├─ runner.ts
│  │  ├─ grep-tool.ts
│  │  ├─ edit-tool.ts
│  │  └─ previews.ts
│  ├─ edit/
│  │  ├─ module.ts
│  │  ├─ code-read-tool.ts
│  │  ├─ edit-ranges-tool.ts
│  │  ├─ versions.ts
│  │  └─ apply.ts
│  └─ dap/
│     ├─ module.ts
│     ├─ config.ts
│     ├─ client.ts
│     ├─ session-manager.ts
│     ├─ reverse-requests.ts
│     └─ tool.ts
├─ test/
│  ├─ fixtures/
│  ├─ security/
│  ├─ protocol/
│  ├─ lsp/
│  ├─ ast/
│  ├─ edit/
│  ├─ dap/
│  └─ integration/
└─ scripts/
   └─ copy-assets.mjs
```

Avoid creating a client bundle unless settings integration or a dedicated diagnostics/debug viewer proves necessary.

## Implementation Phases

### Phase 0: Contracts and skeleton

Deliverables:

- Independent plugin repository
- Package manifest and one-row bundle patch
- Strict configuration schema
- Cordis top-level module and lifecycle tests
- Public DSH contract matrix
- Threat model and security invariants
- No external processes or filesystem mutations yet

Exit criteria:

- Installable through the official DSH plugin mechanism
- Registers and unregisters a harmless status tool cleanly
- HMR/config reload leaves no duplicate tools or resources

### Phase 1: Shared infrastructure

Deliverables:

- Workspace authority
- Process authority integration
- JSON-RPC framing with hard bounds
- Cancellation and timeout helpers
- Result budgets, cursors, and spill adapter
- Agent/session ownership helpers

Exit criteria:

- Security tests reject traversal, URI escapes, symlink escapes, device paths, oversized frames, expired cursors, and cross-session cursors
- Process plans cannot execute without the expected policy path

### Phase 2: Read-only LSP

Deliverables:

- Explicit `clangd` configuration
- Lazy per-workspace pool
- Read-only actions
- Compact typed outputs and pagination
- Fake-server test suite

Exit criteria:

- No LSP action can mutate files
- Server-originated `workspace/applyEdit` is rejected
- Warm query latency and token benchmarks meet targets
- Shutdown leaves no processes

### Phase 3: Diagnostics integration

Deliverables:

- Version-aware diagnostics
- Inline diagnostics after plugin-owned edits
- Bounded deferred handling
- Deduplication and touched-range prioritization

Exit criteria:

- Stale diagnostics never appear as fresh
- Slow servers do not block beyond configured limits
- Large diagnostic sets paginate or spill

### Phase 4: Structural search

Deliverables:

- `ast_grep` tool
- Explicit executable policy
- Deterministic, paginated results
- Parse-error disclosure

Exit criteria:

- Broad scans respect file, byte, result, and time limits
- Search is read-only and confined to approved roots

### Phase 5: Versioned reads and edits

Deliverables:

- `code_read`
- Per-agent observed-range store
- `edit_ranges`
- Exact-byte version tokens
- Compact diff and conflict output

Exit criteria:

- Stale, unobserved, overlapping, out-of-root, or oversized edits write zero bytes
- Successful edits preserve BOM and line endings
- Fresh diagnostics attach when enabled

### Phase 6: Transactional AST edits

Deliverables:

- Preview manifests
- Full bounded diff artifacts
- Single-use apply tokens
- Exact pre-write validation
- DSH filesystem routing

Exit criteria:

- Same-count-but-different-content drift is rejected
- No query is rerun during apply
- A stale preview writes zero files
- Direct native filesystem writes are absent

### Phase 7: DAP debugging

Deliverables:

- Per-session DAP manager
- Initial bounded action set
- Explicit adapter approval
- Reverse-request policy gates
- Launch/attach ownership distinction
- Fake adapter test suite

Exit criteria:

- Sessions cannot leak across DSH owners
- Attach termination defaults to detach-only
- `runInTerminal` and `startDebugging` cannot execute without fresh policy checks
- Oversized frames and memory/output requests are rejected
- Transport backpressure is tested

### Phase 8: Integration, benchmarking, and stable release

Deliverables:

- Complete README and settings reference
- Profile installation and configuration examples
- Compatibility matrix by OS/language/server/adapter
- Performance and token benchmark report
- Security test report
- Migration notes for configuration changes

Exit criteria:

- All default-enabled features pass cross-platform tests
- Debug remains opt-in unless its complete acceptance suite passes
- No DSH source changes are required

## Test Strategy

### Unit tests

- Strict schemas and cross-field validation
- Canonical path confinement
- URI decoding and symlink escapes
- Digest and observed-range behavior
- Range normalization and overlap rejection
- Cursor ownership and expiration
- Frame parsing, content-length limits, and malformed JSON
- Result truncation and pagination

### LSP fake-server tests

- Initialization ordering and cancellation
- Single-flight startup
- Failure backoff
- Diagnostics versions and deduplication
- Definition/reference/symbol pagination
- Server `workspace/applyEdit` rejection
- Out-of-workspace URI rejection
- Raw request policy
- Shutdown during initialization
- Project executable trust gate

### AST tests

- Pattern validation
- Deterministic ordering
- Parse-error disclosure
- File and replacement caps
- Preview exactness
- Same-count stale changes
- Multi-file stale change
- Full manifest validation before writes
- Policy and diagnostics routing

### Versioned edit tests

- Exact bytes, CRLF, LF, BOM, and trailing whitespace
- Stale digest collision resistance
- Observed and unobserved ranges
- Concurrent external modification
- Overlap and no-op behavior
- Failed write and rollback evidence
- Formatter or external write-through changes

### DAP fake-adapter tests

- Launch and attach handshakes
- Configuration-done ordering
- Per-owner isolation
- Reverse request approvals
- Child-session policy
- Attach detach versus kill
- Breakpoint acknowledgement and failure
- Output and frame caps
- Memory read caps
- Partial/slow transport writes
- Adapter and debuggee cleanup

### Integration tests

- Native tool presentation
- Code Mode generated SDK
- Multiple concurrent DSH sessions
- Plugin disable/re-enable
- Configuration reload
- Cordis disposal and HMR
- DSH profile installation
- Windows path and process behavior
- Linux/macOS behavior where builders are available

## Benchmark Plan

Use representative repository tasks, including a large C++ workspace.

### Baseline capture

For each task, record:

- Total model input tokens
- Total model output tokens
- Tool call count by tool
- Bytes returned by tools
- Number of `read` and `grep` calls
- Number of stale edit failures
- Number of build attempts before a clean build
- Wall-clock time
- LSP cold and warm latency

### Example tasks

- Find the definition and all call sites of a C++ method.
- Trace an interface implementation across modules.
- Find every structural use of a deprecated API.
- Make a three-location exact edit after reading one bounded range.
- Diagnose a type error introduced by an edit.
- Preview and apply a multi-file AST migration.

### Initial success targets

- At least 30% lower model input tokens on navigation-heavy tasks.
- At least 50% fewer read/grep calls on semantic-navigation tasks.
- Warm LSP query p95 below 500 ms on the benchmark workspace where the server supports it.
- Fresh touched-file diagnostics returned within the configured inline window or explicitly deferred.
- Zero stale edits that silently land.
- Zero out-of-workspace mutations in adversarial tests.
- No orphan LSP, AST, or DAP processes after cancellation, HMR, or shutdown.

Targets are release gates only after the benchmark harness and baseline are checked in.

## Observability

Log structured events without leaking source content or secrets:

- Feature enable/disable
- Server/adapter identity and lifecycle
- Workspace key hash, not sensitive absolute paths in durable telemetry
- Cold/warm startup duration
- Query duration and result counts
- Truncation/pagination/spill events
- Stale version and stale preview counts
- Policy denials by category
- Process exits and forced cleanup
- Protocol limit violations

Never log file contents, environment secrets, debugger memory, or full protocol payloads by default.

## Release and Compatibility Policy

- The package should declare peer dependencies on the public DSH/Cordis packages it imports.
- Dev dependencies should pin the DSH version used for validation.
- Feature detection must fail loudly when a required public service is absent.
- New mutating capabilities release disabled by default first.
- Configuration defaults must remain conservative across upgrades.
- Breaking tool schemas require a major package version or an explicit migration strategy.
- The README must distinguish supported, experimental, and disabled features.

## Risks and Mitigations

| Risk | Mitigation |
|---|---|
| One package becomes a monolith | Internal Cordis modules, separate test suites, strict dependency direction, feature flags. |
| LSP server consumes excessive resources | Lazy pool, per-workspace keying, idle shutdown, process and protocol limits. |
| Project configuration executes malware | Disabled by default, explicit trust/approval, sandboxed process plan, scrubbed environment. |
| Server writes outside workspace | Never direct-apply; canonical authority check for every URI and mutation. |
| Version token adds prompt cost | Compact 128-bit-or-stronger base64url token and no duplicated prose. |
| Snapshot memory grows | Per-owner byte/path/version caps and deterministic eviction. |
| AST edit applies stale changes | Immutable preview manifest and complete validation before writes. |
| Multi-file writes partially fail | Preflight, prepared originals, explicit partial evidence, bounded rollback. |
| Debugger crosses sessions | Owner-keyed manager and no module-global active session. |
| Adapter launches hidden commands | Exact execution plan in approval; reverse requests re-enter policy. |
| Tool results consume context | Shared inline budgets, canonical structures, pagination, and spill support. |
| DSH public API lacks a needed seam | Keep feature disabled; document the missing public contract; never import internals. |

## Definition of Done

The unified plugin is complete when:

- One package installation and one Cordis row make all implemented tools available.
- Read-only LSP, ast-grep, code-read, and exact-version edits are stable and enabled by default.
- All model-facing outputs are typed, compact, bounded, and paginated or spillable.
- Every filesystem and process effect routes through public DSH policy.
- Every workspace path is canonically confined.
- Server-originated mutations cannot bypass approval.
- Structural apply uses immutable previews and validates before writing.
- DAP state is session-owned, protocol-bounded, and opt-in.
- Cordis disposal leaves no tools, listeners, timers, requests, cursors, snapshots, or child processes behind.
- Native and Code Mode presentations both work.
- Benchmarks demonstrate material token and tool-call reductions.
- Security fixtures cover malicious repositories, servers, adapters, paths, payloads, and concurrent sessions.
- Installation, configuration, feature maturity, and limitations are documented.
- No DeepSeek Harness source files are modified.

## Recommended First Release

The first useful release should contain:

- `code_read`
- `edit_ranges`
- Read-only `lsp`
- Fresh diagnostics for plugin-owned edits
- `ast_grep`
- Shared path, process, protocol, pagination, and result-budget infrastructure

It should not yet contain enabled LSP mutation actions, `ast_edit`, or `debug`.

This subset provides the majority of the expected performance and token savings while establishing the security and lifecycle foundations needed by every later capability.

## Reference Design Sources

- DeepSeek Harness tool registry and Code Mode: https://github.com/deepseek-ai/deepseek-harness/tree/main/packages/core/tools
- DeepSeek Harness Cordis runtime: https://github.com/deepseek-ai/deepseek-harness/tree/main/vendor/cordis
- Oh My Pi reviewed revision: https://github.com/can1357/oh-my-pi/commit/b214f5a1b6ddda185c618f1339d5fd4ccd45f7c0
- OMP LSP schema: https://github.com/can1357/oh-my-pi/blob/b214f5a1b6ddda185c618f1339d5fd4ccd45f7c0/packages/coding-agent/src/lsp/types.ts#L8-L24
- OMP server-originated apply-edit path: https://github.com/can1357/oh-my-pi/blob/b214f5a1b6ddda185c618f1339d5fd4ccd45f7c0/packages/coding-agent/src/lsp/client.ts#L488-L507
- OMP workspace edit implementation: https://github.com/can1357/oh-my-pi/blob/b214f5a1b6ddda185c618f1339d5fd4ccd45f7c0/packages/coding-agent/src/lsp/edits.ts#L316-L451
- OMP hashline digest: https://github.com/can1357/oh-my-pi/blob/b214f5a1b6ddda185c618f1339d5fd4ccd45f7c0/packages/hashline/src/format.ts#L90-L120
- OMP AST edit preview/apply path: https://github.com/can1357/oh-my-pi/blob/b214f5a1b6ddda185c618f1339d5fd4ccd45f7c0/packages/coding-agent/src/tools/ast-edit.ts#L439-L532
- OMP DAP reverse requests: https://github.com/can1357/oh-my-pi/blob/b214f5a1b6ddda185c618f1339d5fd4ccd45f7c0/packages/coding-agent/src/dap/session.ts#L1369-L1405
- OMP DAP global manager: https://github.com/can1357/oh-my-pi/blob/b214f5a1b6ddda185c618f1339d5fd4ccd45f7c0/packages/coding-agent/src/dap/session.ts#L1877
