# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup (install deps + Prisma generate + migrate)
npm run setup

# Development server (Turbopack)
npm run dev

# Build for production
npm run build

# Lint
npm lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Reset database
npm run db:reset
```

The dev server requires `NODE_OPTIONS='--require ./node-compat.cjs'` — this is already included in all npm scripts. Don't run `next dev` directly.

## Architecture

UIGen is an AI-powered React component generator. Users describe a UI in chat; Claude generates React files into a virtual file system; the result is rendered live in a sandboxed iframe.

### Request flow

1. User sends a chat message in `ChatInterface` → POST to `/api/chat`
2. `/api/chat/route.ts` calls `streamText` (Vercel AI SDK) with Claude (`claude-haiku-4-5`) plus two tools: `str_replace_editor` and `file_manager`
3. Claude uses those tools to create/edit files in a server-side `VirtualFileSystem` instance
4. Tool calls are streamed back to the client; `FileSystemContext.handleToolCall` applies them to the client-side `VirtualFileSystem`
5. `PreviewFrame` detects the change (via `refreshTrigger`), calls `createImportMap` to Babel-transform all files to blob URLs, injects an `<importmap>` into an `srcdoc` iframe, and renders the component

### Key abstractions

- **`VirtualFileSystem`** (`src/lib/file-system.ts`) — in-memory tree of `FileNode`s. All file I/O happens here; nothing is written to disk. The server serialises it to JSON and persists it in the `Project.data` column.
- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — React context wrapping a `VirtualFileSystem` instance; exposes CRUD helpers and `handleToolCall` which applies streamed AI tool calls.
- **`jsx-transformer.ts`** (`src/lib/transform/`) — Babel-transforms JSX/TSX files in-browser, builds an ES module import map (local files → blob URLs, npm packages → `esm.sh`), and generates the full `srcdoc` HTML for the preview iframe.
- **AI tools** (`src/lib/tools/`) — `str_replace_editor` (view/create/str_replace/insert) and `file_manager` (rename/delete) are the two tools the model uses. Both operate on a `VirtualFileSystem` passed in at construction.
- **`getLanguageModel()`** (`src/lib/provider.ts`) — returns real `anthropic("claude-haiku-4-5")` when `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that streams static component code.

### Auth & persistence

- JWT sessions stored in an `httpOnly` cookie (`jose`); helpers in `src/lib/auth.ts`
- Prisma + SQLite (`prisma/dev.db`). Two models: `User` and `Project`. `Project.messages` and `Project.data` store the full chat history and serialised file system as JSON strings.
- Anonymous users get a session-storage-backed tracker (`anon-work-tracker.ts`); their work is not persisted to the DB.
- Middleware (`src/middleware.ts`) redirects unauthenticated users away from project routes.

### Routing

- `/` — landing / anonymous workspace (`src/app/page.tsx`). Authenticated users are immediately redirected to their latest project.
- `/[projectId]` — per-project workspace that hydrates messages and file system from the DB.

### UI layout

The workspace is a three-panel resizable layout (chat | preview | code editor) built with `react-resizable-panels`. The chat context (`src/lib/contexts/chat-context.tsx`) manages message streaming state separately from the file system context.

### Environment variables

| Variable | Required | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` | No | Use real Claude; falls back to mock if absent |
| `JWT_SECRET` | No | Falls back to `"development-secret-key"` |
