# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (uses Turbopack + node-compat.cjs shim)
npm run dev

# Build for production
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Reset database
npm run db:reset
```

## Environment

Copy `.env` and set `ANTHROPIC_API_KEY`. Without it, the app runs with a `MockLanguageModel` that returns static component code instead of calling Claude.

## Architecture

### Request Flow

1. User types a prompt in `ChatInterface` → `ChatContext` calls `/api/chat` via Vercel AI SDK's `useChat`
2. The API route (`src/app/api/chat/route.ts`) reconstructs a `VirtualFileSystem` from serialized file data sent in the request body, then streams a response from Claude using `streamText`
3. Claude uses two tools — `str_replace_editor` and `file_manager` — to create/modify files in the virtual FS
4. Tool calls flow back to the client via the AI SDK stream; `FileSystemContext.handleToolCall` applies them to the client-side VFS
5. The `PreviewFrame` component re-renders on each VFS change: it uses Babel standalone to transpile JSX/TSX to JS, creates blob URLs for each file, builds a native ES module import map, and injects the result into a sandboxed `<iframe>` as `srcdoc`

### Key Abstractions

- **`VirtualFileSystem`** (`src/lib/file-system.ts`): In-memory tree of `FileNode` objects. All generated code lives here — nothing is written to disk. Has `serialize()`/`deserializeFromNodes()` for JSON round-tripping through the chat API.
- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`): React context wrapping the client-side VFS instance; exposes `handleToolCall` for applying AI tool calls and `refreshTrigger` to signal the preview to update.
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`): Wraps Vercel AI SDK's `useChat`, serializes the current VFS state into every request body so the server can reconstruct it.
- **JSX transformer** (`src/lib/transform/jsx-transformer.ts`): `transformJSX` uses `@babel/standalone` to compile TSX/JSX. `createImportMap` builds an ES import map, mapping local paths to blob URLs and third-party packages to `esm.sh`. Missing local imports get placeholder stub modules so the preview doesn't crash.
- **`getLanguageModel`** (`src/lib/provider.ts`): Returns the real `claude-haiku-4-5` model if `ANTHROPIC_API_KEY` is set, otherwise returns `MockLanguageModel`.

### Auth

JWT-based sessions stored in an `httpOnly` cookie (`auth-token`). `src/lib/auth.ts` is `server-only`. Users may also interact anonymously; anonymous work is tracked in `src/lib/anon-work-tracker.ts` and can be claimed on sign-up.

### Database

Prisma + SQLite (`prisma/dev.db`). Two models: `User` and `Project`. A `Project` stores serialized chat messages and VFS state as JSON strings in `messages` and `data` columns. Projects with `userId = null` are anonymous. Prisma client is generated into `src/generated/prisma`.

### Preview Entry Point

`PreviewFrame` looks for entry files in this order: `/App.jsx`, `/App.tsx`, `/index.jsx`, `/index.tsx`, `/src/App.jsx`, `/src/App.tsx`. If none match, it falls back to the first `.jsx`/`.tsx` file. The preview iframe has `sandbox="allow-scripts allow-same-origin allow-forms"` — required for blob URL import maps to work.

### Tests

Vitest with jsdom + React Testing Library. Test files live alongside source in `__tests__/` subdirectories. `vite-tsconfig-paths` resolves `@/` aliases in tests.
