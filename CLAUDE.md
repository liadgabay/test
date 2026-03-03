# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `npm run setup` — install deps, generate Prisma client, run migrations
- `npm run dev` — start dev server (Next.js + Turbopack) on localhost:3000
- `npm run build` — production build
- `npm run lint` — ESLint
- `npm test` — run all tests (vitest, jsdom environment)
- `npx vitest run src/path/to/file.test.ts` — run a single test file
- `npx prisma generate` — regenerate Prisma client after schema changes
- `npx prisma migrate dev` — apply schema migrations
- `npm run db:reset` — reset the database

## Code Style

- Do not use semicolons at the end of code lines
- Path alias: `@/*` maps to `./src/*`
- UI components use shadcn/ui (new-york style) with Radix primitives in `src/components/ui/`
- Tailwind CSS v4 for styling
- Always read `prisma/schema.prisma` for database schema information — do not assume or guess model fields

## Architecture

**UIGen** is an AI-powered React component generator. Users describe components in a chat, an LLM generates code via tool calls, and a live preview renders the result — all without writing files to disk.

### Core data flow

1. **ChatContext** (`src/lib/contexts/chat-context.tsx`) wraps the Vercel AI SDK `useChat` hook, sending messages to `/api/chat`
2. **API route** (`src/app/api/chat/route.ts`) reconstructs a `VirtualFileSystem` from the request, attaches `str_replace_editor` and `file_manager` tools, and streams an LLM response
3. Tool calls from the LLM (create/edit/delete files) execute against the server-side VFS *and* are replayed client-side by **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`) via `onToolCall`
4. **PreviewFrame** (`src/components/preview/PreviewFrame.tsx`) watches the VFS for changes, transforms JSX/TSX with Babel standalone, builds an import map with blob URLs, and renders inside a sandboxed iframe

### Virtual File System

`VirtualFileSystem` (`src/lib/file-system.ts`) is an in-memory file tree (no disk I/O). It's serialized as JSON for transport between client and server. The LLM interacts with it through two tools:
- `str_replace_editor` — view/create/str_replace/insert operations (modeled after Claude's text editor tool)
- `file_manager` — rename/delete operations

### Preview pipeline

`src/lib/transform/jsx-transformer.ts` handles the entire preview build:
- Transforms JSX/TSX via `@babel/standalone`
- Creates blob URLs for each transformed file
- Builds an import map with `@/` alias support and extension resolution
- Third-party imports resolve to `esm.sh`
- CSS files are collected into a `<style>` block
- Missing local imports get placeholder modules

### LLM provider

`src/lib/provider.ts` — uses `claude-haiku-4-5` via `@ai-sdk/anthropic` when `ANTHROPIC_API_KEY` is set; otherwise falls back to a `MockLanguageModel` that returns static component code (counter/form/card).

### Auth & persistence

- JWT-based auth with `jose` (cookie `auth-token`, 7-day expiry)
- Server actions in `src/actions/` handle sign-up/sign-in/sign-out (bcrypt password hashing)
- Prisma + SQLite (`prisma/dev.db`) stores `User` and `Project` models
- Anonymous users can use the app; authenticated users get persistent projects
- Middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem` routes

### Generation prompt

`src/lib/prompts/generation.tsx` — the system prompt instructs the LLM to produce React + Tailwind components with `/App.jsx` as the entry point, using `@/` import aliases on a virtual root filesystem.

### App layout

`src/app/main-content.tsx` — resizable two-panel layout: chat on the left, preview/code on the right. The code view has a file tree + Monaco editor. Wrapped in `FileSystemProvider` > `ChatProvider`.
