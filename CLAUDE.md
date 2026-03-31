# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server (Turbopack)
npm run dev:daemon   # Start dev server in background (logs to logs.txt)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest (all tests)
npm run db:reset     # Reset database to initial state
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

Environment: copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY` (optional — falls back to a mock provider if absent).

## Architecture

UIGen is a **Next.js 15 (App Router)** app where users describe a React component in a chat and see it rendered live. The core loop: user sends a message → Claude streams back tool calls that read/write a virtual file system → the client transforms JSX with Babel Standalone and renders it in a sandboxed iframe.

### Key Data Flows

**AI Generation (`/src/app/api/chat/route.ts`)**
- Uses Vercel AI SDK `streamText` with Claude (model defined in `/src/lib/provider.ts`)
- Claude has two tools: `str_replace_editor` (read/write file content) and `file_manager` (create/delete/list paths) — defined in `/src/lib/tools/`
- System prompt lives in `/src/lib/prompts/generation.tsx`
- Tool results are streamed back to the client as a data stream

**Virtual File System (`/src/lib/file-system.ts`)**
- `VirtualFileSystem` class — in-memory only, no disk writes
- Exposed via `FileSystemContext` (`/src/lib/contexts/FileSystemContext.tsx`)
- Serialized as JSON and persisted per-project in the `Project.data` database column

**Live Preview (`/src/components/preview/PreviewFrame.tsx`)**
- Babel Standalone transforms JSX on the client
- An import map in the iframe resolves `react`/`react-dom` from esm.sh and local virtual files as blob URLs
- JSX transformation logic is in `/src/lib/transform/jsx-transformer.ts`

**Authentication**
- JWT in HTTP-only cookies; session helpers in `/src/lib/auth.ts`
- Server Actions in `/src/actions/` handle sign-up, sign-in, sign-out, and project CRUD
- `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes
- Anonymous users get localStorage persistence; `use-auth.ts` migrates their work on sign-in

**State Management**
- `FileSystemContext` — virtual FS state and file operations
- `ChatContext` — chat messages and streaming state
- No external state library

### Directory Map

| Path | Purpose |
|---|---|
| `src/app/` | Next.js routes and API handlers |
| `src/components/chat/` | Chat UI (interface, message list, input, markdown renderer) |
| `src/components/editor/` | Monaco code editor + file tree |
| `src/components/preview/` | Iframe-based live preview |
| `src/components/auth/` | Auth forms |
| `src/components/ui/` | Radix UI primitives (shadcn/ui, New York style, Tailwind v4) |
| `src/lib/contexts/` | React contexts for FS and chat state |
| `src/lib/tools/` | AI tool schemas and handlers |
| `src/lib/transform/` | Babel JSX transformation utilities |
| `src/lib/prompts/` | Claude system prompt |
| `src/actions/` | Next.js Server Actions (auth + project operations) |
| `prisma/` | Schema and migrations (SQLite, models: User, Project) |

### Database Schema

`User`: `id`, `email`, `password` (bcrypt), timestamps
`Project`: `id`, `name`, `userId` (nullable), `messages` (JSON string), `data` (JSON string), timestamps

Projects store the full chat history and virtual FS snapshot as JSON strings.
