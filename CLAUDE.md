# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator. Users describe components in chat; Claude generates working React code displayed in a live preview alongside a Monaco code editor. It uses a **virtual file system** (in-memory, no disk I/O) to manage generated files.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development (uses Turbopack + node-compat shim)
npm run dev

# Build
npm run build

# Lint
npm run lint

# Tests (all)
npm test

# Tests (single file)
npx vitest src/lib/__tests__/file-system.test.ts

# Database reset
npm run db:reset
```

## Environment

- Add `ANTHROPIC_API_KEY` to `.env` to use real Claude AI. Without it, the app falls back to a `MockLanguageModel` in `src/lib/provider.ts` that returns demo components.
- Database: SQLite via Prisma (file-based, local).

## Architecture

### Data Flow

1. User types prompt → `ChatContext` captures it → POST to `/api/chat`
2. Chat API reconstructs `VirtualFileSystem` from serialized client state
3. Claude receives prompt + system prompt + current file state
4. Claude calls AI tools (`str_replace_editor`, `file_manager`) to create/modify files
5. Tool results stream back → `FileSystemContext` updates in-memory state
6. `PreviewFrame` re-renders the component using **Babel standalone** (client-side JSX evaluation)
7. For authenticated users, final state is persisted to SQLite via Prisma

### Key Abstractions

**Virtual File System** (`src/lib/file-system.ts`): In-memory tree structure. The same `VirtualFileSystem` class is used server-side (in the API route) and its serialized form is sent to/from the client in every chat request.

**AI Tools** (`src/lib/tools/`):
- `str_replace_editor.ts` — view/create/str_replace/insert operations on files
- `file_manager.ts` — rename/delete operations

**Contexts** (`src/lib/contexts/`):
- `ChatContext` — conversation messages, input state, AI loading status
- `FileSystemContext` — virtual file tree state, selected file, refresh triggers

**Language Model Provider** (`src/lib/provider.ts`): Wraps `@ai-sdk/anthropic`. Uses `claude-haiku-4-5-20251001`. Falls back to `MockLanguageModel` when `ANTHROPIC_API_KEY` is absent.

### Auth

JWT sessions via `jose` (7-day expiry, stored in cookies). Passwords hashed with bcrypt. Middleware at `src/middleware.ts` protects the `/api/chat` route. Anonymous users can use the app without an account but projects aren't persisted.

### Routing

- `/` — home page; authenticated users are redirected to their most recent project
- `/[projectId]` — project detail page (loads messages + file state from DB)
- `/api/chat` — streaming chat endpoint (Vercel AI SDK `streamText`)

### UI Layout

Three-panel resizable layout (via `react-resizable-panels`) in `src/app/main-content.tsx`:
- Left: Chat interface
- Right top: Preview (iframe with Babel-evaluated JSX) or Code editor (Monaco)
- Components use shadcn/ui (new-york style, neutral base) built on Radix UI primitives

### Database Schema (Prisma + SQLite)

- `User`: id, email, hashedPassword, timestamps
- `Project`: id, name, userId (FK), messages (JSON string), data (JSON string), timestamps
- Projects cascade-delete with user

## Path Aliases

`@/*` maps to `src/*` (configured in `tsconfig.json`).

## Testing

Uses Vitest + jsdom + Testing Library. Tests live in `__tests__/` directories next to the code they test. The `vitest.config.mts` uses `vite-tsconfig-paths` so `@/` imports work in tests.
