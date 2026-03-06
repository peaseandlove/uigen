# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup
npm run setup          # installs deps, generates Prisma client, runs migrations

# Development
npm run dev            # starts Next.js dev server with Turbopack on localhost:3000
npm run dev:daemon     # same but runs in background, logs to logs.txt

# Build & lint
npm run build
npm run lint

# Tests
npm test               # run all tests with vitest
npx vitest run src/lib/__tests__/file-system.test.ts  # run a single test file

# Database
npm run db:reset       # reset and re-run all migrations (destructive)
npx prisma migrate dev # run new migrations
npx prisma generate    # regenerate Prisma client after schema changes
```

## Architecture

UIGen is a Next.js 15 App Router app that uses Claude AI to generate React components in a browser-based live preview. The key insight is that all generated files exist only in a **virtual in-memory file system** — nothing is written to disk.

### Request Flow

1. User types a prompt in the chat (`ChatInterface`)
2. `ChatContext` calls `POST /api/chat` with messages + serialized virtual FS state
3. The API route streams responses from Claude (or `MockLanguageModel` if no API key) using Vercel AI SDK
4. Claude uses two tools: `str_replace_editor` (create/edit files) and `file_manager` (rename/delete)
5. Tool calls stream back to the client, where `FileSystemContext.handleToolCall()` applies them to the in-memory FS
6. `PreviewFrame` detects FS changes, transforms all JSX/TSX files with `@babel/standalone`, and renders them in an iframe using an ES module import map

### Key Files

- `src/app/api/chat/route.ts` — streaming chat endpoint; reconstructs VirtualFileSystem from serialized data, runs Claude with tools
- `src/lib/file-system.ts` — `VirtualFileSystem` class; all file operations happen here in memory
- `src/lib/contexts/file-system-context.tsx` — React context wrapping VirtualFileSystem; `handleToolCall()` applies AI tool calls to the FS
- `src/lib/contexts/chat-context.tsx` — manages chat messages, streams AI responses, dispatches tool calls to FileSystemContext
- `src/lib/transform/jsx-transformer.ts` — transforms JSX/TSX with Babel, resolves imports, builds ES module import maps with blob URLs; third-party packages resolved via `esm.sh`
- `src/components/preview/PreviewFrame.tsx` — renders an `<iframe>` with generated HTML; re-renders on FS changes
- `src/lib/provider.ts` — `getLanguageModel()` returns real Anthropic model or `MockLanguageModel` when no API key is set; model is `claude-haiku-4-5`
- `src/lib/prompts/generation.tsx` — system prompt for component generation
- `src/lib/tools/str-replace.ts` / `src/lib/tools/file-manager.ts` — AI SDK tool definitions

### Auth & Persistence

- JWT-based auth via `jose`, stored in an httpOnly cookie (`src/lib/auth.ts`)
- SQLite database via Prisma (`prisma/schema.prisma`); Prisma client output goes to `src/generated/prisma/`
- Authenticated users get projects persisted to DB; anonymous users get an in-memory session only
- Project `data` field stores the serialized VirtualFileSystem as JSON; `messages` stores chat history

### Preview Rendering

The preview iframe uses ES module import maps. `createImportMap()` in `jsx-transformer.ts`:
- Transforms each JS/JSX/TS/TSX file with Babel and creates a blob URL
- Maps all import paths (with/without extensions, with `@/` alias) to their blob URLs
- Resolves unknown third-party imports to `https://esm.sh/<package>`
- Injects Tailwind CSS via CDN
- Entry point defaults to `/App.jsx`

### Layout

`MainContent` (`src/app/main-content.tsx`) is a two-panel layout: chat on the left (35%), preview/code on the right (65%). The right panel switches between `PreviewFrame` and a code editor (`FileTree` + `CodeEditor` using Monaco).
