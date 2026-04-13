# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe UI components in a chat interface; Claude generates code via tool calls; an in-browser Babel transform renders the output in an iframe — no build step required for generated code.

## Commands

```bash
npm run setup        # Install deps + generate Prisma client + run migrations (first-time setup)
npm run dev          # Development server (Next.js 15 with Turbopack)
npm run dev:daemon   # Dev server in background, logging to logs.txt
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest (jsdom)
npm run db:reset     # Force-reset Prisma SQLite database
```

**Run a single test file:**
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

**Environment:** Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. Without it, a mock provider returns static component code (useful for UI development without an API key).

## Architecture

### Core Data Flow

```
Chat message → POST /api/chat → Claude (streamText) → tool calls → VirtualFileSystem
                                                                         ↓
                                              iframe ← Babel JSX transform ← FileSystemContext
```

1. User sends a prompt via `ChatContext`
2. `/api/chat/route.ts` streams a response using Vercel AI SDK with two tools: `str_replace_editor` and `file_manager`
3. Claude manipulates the **VirtualFileSystem** (`src/lib/file-system.ts`) — an in-memory tree; nothing is written to disk
4. `FileSystemContext` propagates changes to the preview iframe
5. The iframe fetches an import map pointing to esm.sh CDN and runs Babel-compiled JSX at runtime

### Virtual File System (`src/lib/file-system.ts`)

Central abstraction. All generated code lives here. Key operations:
- `viewFile` / `createFile` / `deleteFile` / `renameFile`
- `replaceInFile` (str_replace semantics — must match exactly once)
- `insertInFile` (insert after line N)
- `serialize` / `deserialize` for Prisma persistence (stored as JSON)

The AI tools (`str_replace_editor`, `file_manager`) are thin adapters over these VFS methods.

### Preview Rendering (`src/components/preview/`)

- Finds entry point: `App.jsx`, `App.tsx`, `index.jsx`, or `index.tsx`
- Builds a browser import map with esm.sh CDN URLs for React/ReactDOM and any detected deps
- Compiles all VFS files with `@babel/standalone` at runtime
- Loads the result into an isolated iframe via a Blob URL
- Errors from compilation or runtime are caught and displayed inline

### AI Interaction (`src/app/api/chat/route.ts`)

- System prompt uses Anthropic cache control (ephemeral) for prompt caching
- `maxSteps: 40` (4 for mock) — Claude iterates tool calls until complete
- Auto-saves project to Prisma on completion if user is authenticated
- Mock provider (`src/lib/provider.ts`) activates when `ANTHROPIC_API_KEY` is absent

### Auth & Persistence

- JWT sessions (jose, 7-day expiry) validated in `src/middleware.ts`
- Middleware protects `/api/projects` and `/api/filesystem` routes
- Anonymous users: work tracked via `src/lib/anon-work-tracker.ts` using localStorage
- Prisma + SQLite: `User` and `Project` models; Project stores `messages` and VFS `data` as JSON

### State Management

- `FileSystemContext` — VFS state + active file, shared across Editor, FileTree, Preview
- `ChatContext` — conversation history + streaming state
- No external state library; pure React context + hooks

## Key Conventions

- Use comments sparingly — only for complex or non-obvious logic
- Path alias `@/` maps to `src/`
- All internal imports use `@/` — no relative paths crossing directory boundaries
- Tests colocated in `__tests__/` subdirectories, using Vitest + React Testing Library
- Components follow shadcn/ui patterns (New York style, Tailwind CSS v4)
- The VFS is the source of truth for generated code — never bypass it to write files directly
