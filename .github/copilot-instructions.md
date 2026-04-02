# Copilot Instructions

## What This Repository Is

Decompiled TypeScript source of **Claude Code v2.1.88**, extracted from the npm package `@anthropic-ai/claude-code`. The original package ships a single ~12MB bundled `cli.js`; this repo contains the unbundled TypeScript source for research purposes. **108 modules are missing** — they are Anthropic-internal and were dead-code-eliminated at compile time.

> All source code is the intellectual property of Anthropic. Research/educational use only. No commercial use.

---

## Build & Run

```bash
npm install
npm run build      # prepare-src + esbuild bundle → dist/cli.js
npm run check      # type-check only (tsc --noEmit), no output
npm start          # node dist/cli.js
```

There are **no tests** in this codebase.

### Build internals

`scripts/build.mjs` does a multi-phase build because the source uses Bun compile-time intrinsics that esbuild doesn't understand:

1. Copies `src/` → `build-src/`
2. Replaces `feature('FLAG')` → `false` and `MACRO.X` → string literals
3. Removes `import { feature } from 'bun:bundle'`
4. Bundles with esbuild (up to 5 rounds, auto-creating stubs for missing modules)

`stubs/bun-bundle.ts` provides a runtime stub where `feature()` always returns `false`.

---

## Architecture

```
src/entrypoints/cli.tsx   ← CLI entry point (Commander.js)
src/main.tsx              ← Main orchestration (~4,600 lines), startup prefetches, REPL launch
src/QueryEngine.ts        ← Headless SDK entry: submitMessage() → AsyncGenerator<SDKMessage>
src/query.ts              ← Main agent loop (~785KB), streaming, tool dispatch, compaction
src/tools.ts              ← Registry: imports and assembles all Tool instances
src/Tool.ts               ← Core Tool type definitions and ToolUseContext
src/tools/<ToolName>/     ← One directory per tool (40+ tools)
src/services/             ← Analytics, MCP, API client, compact, OAuth, LSP, etc.
src/services/tools/toolExecution.ts  ← StreamingToolExecutor: parallel tool batching
src/state/AppState.ts     ← Global mutable state (React-managed)
src/components/           ← Ink (React terminal) UI components
src/screens/              ← Full-screen Ink views
src/types/                ← Shared TypeScript types (message, permissions, ids, etc.)
src/utils/                ← Utilities (file I/O, shell, permissions, model, path, etc.)
src/hooks/                ← React/custom hooks
src/context.ts            ← System/user context assembly
src/commands.ts           ← Slash-command definitions (~80+ commands)
src/skills/               ← Skill system (loadable tool bundles from directories)
src/memdir/               ← CLAUDE.md / memory file handling
src/coordinator/          ← Multi-agent coordinator
src/tasks/                ← Background shell task management
```

**Root-level directories outside `src/`:**
- `tools/` — Feature-gated testing tool stubs (OverflowTestTool, TungstenTool, etc.) not compiled into builds
- `utils/` — Runtime native integration helpers (`attributionHooks.js`, `systemThemeWatcher.js`, `udsClient.js`)
- `vendor/` — Native module source stubs (audio-capture, image-processor, modifiers-napi, url-handler) — not compiled
- `types/` — Stub connector (`connectorText.js`)

The agent loop (`query.ts` / `QueryEngine.ts`) calls the Anthropic API, receives tool_use blocks, dispatches each to its `Tool.call()` (concurrent tools via `StreamingToolExecutor`), and feeds results back. The UI layer is React + [Ink](https://github.com/vadimdemedes/ink) for terminal rendering.

---

## Tool System

Every tool lives in `src/tools/<ToolName>/` and is registered in `src/tools.ts`.

### Tool interface (from `src/Tool.ts`)

```ts
type Tool = {
  name: string
  aliases?: string[]
  searchHint?: string          // 3–10 word hint for ToolSearch deferred loading
  readonly inputSchema: z.ZodType   // Zod v4 schema
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult>
  description(input, options): Promise<string>
  checkPermissions(input, context): Promise<PermissionResult>
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isConcurrencySafe(input): boolean
  isEnabled(): boolean
  validateInput?(input, context): Promise<ValidationResult>
  maxResultSizeChars: number
  shouldDefer?: boolean        // requires ToolSearch round-trip before use
  alwaysLoad?: boolean         // never deferred; always in initial prompt
}
```

Use the `buildTool()` factory from `src/Tool.ts` to construct tools. Each tool directory typically contains:
- `<ToolName>.ts` or `<ToolName>.tsx` — main tool implementation
- `UI.tsx` — Ink component for the tool's terminal display
- `prompt.ts` — prompt text / description strings
- `toolName.ts` — exported `TOOL_NAME` constant
- Additional helpers (permissions, validation, utils)

### Feature-gated tools

Many tools are conditionally loaded in `src/tools.ts`:

```ts
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool
  : null

const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

`feature()` always returns `false` in the built output (replaced at build time). `USER_TYPE === 'ant'` gates Anthropic-internal tools.

---

## Key Conventions

### Imports

- ESM throughout (`"type": "module"` in package.json)
- Path alias `src/*` → `src/*` (tsconfig). Files use both `src/utils/file.js` and `../../utils/file.js` style — both work.
- All imports use `.js` extensions even for `.ts` source files (ESM resolution).
- Zod is imported as `from 'zod/v4'` (not `from 'zod'`).
- `@anthropic-ai/sdk` types come from `@anthropic-ai/sdk/resources/index.mjs`.

### TypeScript

- `strict: false` — no strict null checks enforced
- Target: ES2022, module: ESNext, jsx: react-jsx
- `resolveJsonModule: true` — JSON files importable
- `bun:bundle` imports must be handled via the `stubs/bun-bundle.ts` stub when working outside Bun

### Analytics type safety pattern

Sensitive strings (code content, file paths) use a nominal type to prevent accidental logging:

```ts
type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = ...
```

This naming convention enforces review at the call site.

### Commands

Slash commands (`/plan`, `/config`, `/login`, etc.) are defined in `src/commands/` using a `satisfies Command` pattern:

```ts
export default {
  type: 'local-jsx',            // 'local' | 'local-jsx' | 'prompt'
  name: 'plan',
  description: 'Enter plan mode',
  supportsNonInteractive: false,
  isEnabled: () => true,
  load: () => import('./plan.js'),
} satisfies Command
```

### Errors

Custom error classes in `src/utils/errors.ts`: `ClaudeError`, `AbortError`, `ShellError` (stdout/stderr/exit code), `ConfigParseError`. Use `isAbortError()` to check for any abort-shaped error (handles `AbortError`, `DOMException`, and SDK `APIUserAbortError`). Tool input validation returns `ValidationResult = { result: true } | { result: false, message: string, errorCode: number }`.

### Branded ID types

```ts
// src/types/ids.ts
type SessionId = string & { readonly __brand: 'SessionId' }
type AgentId   = string & { readonly __brand: 'AgentId' }
```

### Permissions

Permission logic is centralized in `src/utils/permissions/`. The `ToolUseContext.options` carries a `ToolPermissionContext` with `mode`, `alwaysAllowRules`, `alwaysDenyRules`, and `alwaysAskRules`. Tools call `checkPermissions()` which feeds into `canUseTool`.

### `ToolUseContext`

The large `ToolUseContext` object (defined in `Tool.ts`) is threaded through every tool call. It carries: app state accessors, abort controller, permission context, MCP clients, message history, progress callbacks, and more. Subagent contexts are created via `createSubagentContext()` in `src/tools/AgentTool/`.

### React/Ink UI

Terminal UI uses Ink (React for CLIs). Tool display components render in `UI.tsx` files alongside each tool. `setToolJSX` in `ToolUseContext` sets the current tool's rendered output.

### Missing modules

If you add imports that reference missing Anthropic-internal modules (anything from the 108 missing list), the build script will auto-create empty stubs. Do not add real implementation for these — they are intentionally absent.

---

## Key Files for Understanding the System

| File | Purpose |
|------|---------|
| `src/Tool.ts` | Core types: `Tool`, `ToolUseContext`, `ToolResult`, `PermissionResult` |
| `src/tools.ts` | Full tool registry |
| `src/query.ts` | Agent loop implementation |
| `src/entrypoints/cli.tsx` | CLI entry, Commander.js setup |
| `src/main.tsx` | Startup orchestration |
| `src/services/mcp/` | MCP server connection handling |
| `src/utils/permissions/` | Permission rule matching and evaluation |
| `src/types/message.ts` | `Message`, `UserMessage`, `AssistantMessage` types |
| `src/types/permissions.ts` | `PermissionMode`, `PermissionResult`, `AdditionalWorkingDirectory` |
