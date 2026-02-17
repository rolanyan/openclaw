# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is OpenClaw?

OpenClaw is a personal AI assistant gateway that connects to messaging channels (WhatsApp, Telegram, Slack, Discord, Google Chat, Signal, iMessage, Microsoft Teams, Matrix, Zalo, WebChat) and provides a unified AI assistant experience. It runs locally and uses a Pi agent runtime for AI responses. It also has macOS/iOS/Android companion apps.

## Build & Development Commands

```bash
pnpm install              # Install dependencies (Node >=22 required, pnpm 10.23.0)
pnpm build                # Full production build (tsdown → dist/)
pnpm dev                  # Run CLI in dev mode (tsx, auto-reload)
pnpm gateway:watch        # Dev gateway with file watching
pnpm check                # format:check + tsgo + lint (run before commits)
pnpm format               # Format with oxfmt (--write)
pnpm lint                 # Lint with oxlint (--type-aware)
pnpm lint:fix             # Auto-fix lint + format
```

## Testing

```bash
pnpm test                 # Unit tests via vitest (parallel, forks pool)
pnpm test:fast            # Unit tests only (excludes gateway/extensions)
pnpm test:e2e             # E2E tests (vmForks pool)
pnpm test:coverage        # Unit tests with V8 coverage (70% thresholds)
pnpm test:live            # Live tests requiring real API keys (CLAWDBOT_LIVE_TEST=1)
pnpm test:watch           # Watch mode
pnpm test:ui              # UI package tests
```

Run a single test file:
```bash
npx vitest run src/path/to/file.test.ts
```

Test naming: colocated `*.test.ts` for unit, `*.e2e.test.ts` for E2E, `*.live.test.ts` for live API tests.

## Pre-commit Hooks

Git hooks in `git-hooks/` run oxlint + oxfmt on staged files automatically. The hook auto-fixes and re-stages.

## Commit Convention

Use `scripts/committer "<msg>" <file...>` instead of manual `git add`/`git commit` to keep staging scoped. Follow concise action-oriented messages (e.g., `CLI: add verbose flag to send`).

## Architecture Overview

### Core Layers

- **CLI** (`src/cli/`, `src/commands/`): Commander-based CLI surface. Entry point is `src/index.ts` → `src/cli/program.ts`. Dependencies injected via `createDefaultDeps`.
- **Gateway** (`src/gateway/`): WebSocket server that acts as the control plane. Manages sessions, channels, tools, events, cron, config, and the Control UI. The main server implementation is in `src/gateway/server.impl.ts`.
- **Agent Runtime** (`src/agents/`): Pi agent runner (`pi-embedded-runner.ts`) that handles AI model interactions, tool execution, compaction, system prompts, and streaming. Subagent support via `subagent-registry.ts` and `subagent-spawn.ts`.
- **Channels** (`src/channels/`, plus `src/telegram/`, `src/discord/`, `src/slack/`, `src/signal/`, `src/imessage/`, `src/web/`, `src/whatsapp/`): Messaging channel integrations. Each channel has its own directory. Routing logic in `src/routing/`.
- **Config** (`src/config/`): Configuration loading, session store, credentials.
- **Infra** (`src/infra/`): Runtime utilities — ports, dotenv, errors, fetch, updates, state migrations, heartbeat, provider usage tracking, Tailscale integration.

### Extensions & Plugins

- **Extensions** (`extensions/`): Workspace packages (each with own `package.json`). Includes channel plugins (msteams, matrix, zalo, etc.), memory backends, voice, and more. Keep plugin-only deps in the extension `package.json`, not root.
- **Plugin SDK** (`src/plugin-sdk/`): Published as `openclaw/plugin-sdk`. Runtime resolves via jiti alias. Avoid `workspace:*` in plugin `dependencies`.

### UI

- **Control UI** (`ui/`): Lit-based web UI served from the gateway. Uses **legacy decorators** (`@state()`, `@property()`). Built with Vite. Separate pnpm workspace package.
- **Canvas** (`src/canvas-host/`): A2UI visual workspace. Bundle hash in `src/canvas-host/a2ui/.bundle.hash` is auto-generated.

### Apps

- **macOS** (`apps/macos/`): SwiftUI menu bar app. Prefer `Observation` framework (`@Observable`) over `ObservableObject`.
- **iOS** (`apps/ios/`): SwiftUI, uses xcodegen for project generation.
- **Android** (`apps/android/`): Kotlin/Gradle.
- **Shared** (`apps/shared/`): OpenClawKit shared between iOS/macOS.

### Key Patterns

- **Session model**: `main` session for direct chats, group isolation, per-channel routing.
- **Model failover**: Auth profile rotation with cooldown/expiry (`src/agents/auth-profiles/`).
- **Media pipeline** (`src/media/`, `src/media-understanding/`): Image/audio/video processing, transcription.
- **Tool system** (`src/agents/tools/`): Browser control, canvas, nodes, cron, Discord/Slack actions. Tool schemas must avoid `Type.Union` / `anyOf`/`oneOf` (Google compatibility).
- **Skills** (`src/agents/skills/`): Bundled, managed, and workspace skills with install gating.

## Coding Standards

- TypeScript ESM. Strict typing, avoid `any`. Never add `@ts-nocheck`.
- Format/lint: oxfmt + oxlint. Run `pnpm check` before commits.
- Keep files under ~500 LOC; split/refactor for clarity.
- No prototype mutation for sharing class behavior. Use explicit inheritance/composition.
- CLI progress: use `src/cli/progress.ts`. Status tables: `src/terminal/table.ts`. Colors: `src/terminal/palette.ts`.
- Tool schema guardrails: no `Type.Union` in tool input schemas, use `stringEnum`/`optionalStringEnum` for string enums, avoid raw `format` property names.

## Workspace Structure

```
src/           → Core source (CLI, gateway, agents, channels, infra)
extensions/    → Channel plugins and feature extensions (workspace packages)
ui/            → Lit-based Control UI (separate pnpm workspace)
apps/          → Native apps (macos, ios, android, shared)
packages/      → Additional workspace packages
docs/          → Mintlify documentation
scripts/       → Build, test, release scripts
test/          → Test setup and shared E2E tests
dist/          → Build output
```

## Also See

- `AGENTS.md` — Detailed operational guidelines (release workflow, multi-agent safety, VM ops, deployment).
- `CONTRIBUTING.md` — Contributor guidelines and PR checklist.
- `SOUL.md` — Assistant personality and behavioral guidelines.
