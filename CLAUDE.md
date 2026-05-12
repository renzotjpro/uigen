# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps + prisma generate + prisma migrate dev
npm run dev          # Dev server with Turbopack (http://localhost:3000)
npm run build        # Production build
npm run test         # Run all tests (Vitest)
npm run test -- --reporter=verbose   # Run tests with verbose output
npx vitest run src/path/to/file.test.ts   # Run a single test file
npm run lint         # ESLint
npm run db:reset     # Wipe and re-migrate the SQLite database
```

**Do not run `npm audit fix`** — dependencies are pinned to versions that work together; bumping them breaks the app.

## Architecture

### AI generation pipeline

The core flow: user types a prompt → `/api/chat` → Claude streams tool calls → VirtualFileSystem is updated in memory → PreviewFrame re-renders the iframe.

- `src/app/api/chat/route.ts` — Streaming POST handler. Uses Vercel AI SDK `streamText` with two tools (`str_replace_editor`, `file_manager`) and up to 40 agentic steps. Saves completed conversation + file state to the database if `projectId` is provided and user is authenticated.
- `src/lib/provider.ts` — Returns `anthropic("claude-haiku-4-5")` when `ANTHROPIC_API_KEY` is set; otherwise returns `MockLanguageModel` with canned responses. The mock simulates the same tool-call protocol.
- `src/lib/prompts/generation.tsx` — System prompt injected with `cacheControl: { type: "ephemeral" }` for Anthropic prompt caching.
- `src/lib/tools/str-replace.ts` — `str_replace_editor` tool: view/create/str_replace/insert commands operating on the VirtualFileSystem.
- `src/lib/tools/file-manager.ts` — `file_manager` tool for additional file operations.

### Virtual file system

`src/lib/file-system.ts` — `VirtualFileSystem` class. All generated files live in memory (never written to disk). Files are stored in a `Map<string, FileNode>` with a tree structure. Key methods: `createFile`, `readFile`, `updateFile`, `deleteFile`, `rename`, `serialize`/`deserialize`/`deserializeFromNodes`. The serialized form (`Record<string, FileNode>`) is what's stored in the database `data` column and sent back and forth with the chat API.

### Preview rendering

`src/lib/transform/jsx-transformer.ts` — Client-side JSX transformation pipeline:
1. `transformJSX` — Babel standalone transforms JSX/TSX → JS. Strips CSS imports.
2. `createImportMap` — Walks all files, transforms each, creates blob URLs, builds an ES module import map. Third-party packages resolve to `esm.sh`. Missing local imports get placeholder stub modules.
3. `createPreviewHTML` — Generates full HTML with `<script type="importmap">`, Tailwind CDN, and a `<script type="module">` that dynamically imports the entry point and renders it into a React root.

`src/components/preview/PreviewFrame.tsx` — Renders a sandboxed `<iframe srcdoc={...}>`. Watches `refreshTrigger` from `FileSystemContext` to regenerate preview on every file change. Auto-discovers the entry point (`/App.jsx`, `/App.tsx`, `/index.jsx`, etc.).

### State management

Two React contexts wrap the main UI:

- `src/lib/contexts/file-system-context.tsx` — Holds the `VirtualFileSystem` instance and a `refreshTrigger` counter. Exposes `handleToolCall` so the chat context can apply tool results (file creates/edits) directly to the in-memory VFS and trigger a preview re-render.
- `src/lib/contexts/chat-context.tsx` — Wraps Vercel AI SDK `useChat`. Sends the serialized VFS and `projectId` in every request body. On `onToolCall`, delegates to the file system context's `handleToolCall`.

### Auth

`src/lib/auth.ts` (marked `server-only`) — JWT sessions via `jose`. Tokens stored in `auth-token` httpOnly cookie (7-day expiry). `JWT_SECRET` env var defaults to a dev placeholder. Uses `bcrypt` for password hashing.

`src/middleware.ts` — Protects `/api/projects` and `/api/filesystem` routes. The `/api/chat` route is intentionally unprotected to allow anonymous use; project saving is gated inside the route handler instead.

### Database

Prisma + SQLite (`prisma/dev.db`). Client generated to `src/generated/prisma` (non-default path). Two models:
- `User` — email + bcrypt password hash
- `Project` — `messages` (JSON array of AI SDK messages) + `data` (serialized VFS nodes), optional `userId` (null for anonymous projects that aren't persisted)

Anonymous users can generate components freely; project persistence requires authentication.

## Environment variables

| Variable | Required | Notes |
|---|---|---|
| `ANTHROPIC_API_KEY` | No | Falls back to MockLanguageModel if absent or set to `your-api-key-here` |
| `JWT_SECRET` | No | Defaults to `development-secret-key`; set a real value in production |

## Testing

Tests use Vitest with jsdom. Test files are colocated under `__tests__/` directories next to their source. Coverage includes `VirtualFileSystem`, JSX transformer, chat context, and file system context.
