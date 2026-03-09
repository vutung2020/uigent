# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Code Style

Use comments sparingly. Only comment complex code.

## Commands

```bash
# First-time setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (uses Turbopack)
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

## Environment

- `ANTHROPIC_API_KEY` in `.env` — optional. Without it, a `MockLanguageModel` is used that returns static component examples. The app is fully functional without an API key.
- `JWT_SECRET` — defaults to `"development-secret-key"` if not set.
- The `NODE_OPTIONS='--require ./node-compat.cjs'` prefix is already embedded in all npm scripts.

## Architecture

### Data Flow

1. **User sends a chat message** → `ChatProvider` (chat-context.tsx) calls `/api/chat` via Vercel AI SDK's `useChat`.
2. **API route** (`src/app/api/chat/route.ts`) reconstructs a `VirtualFileSystem` from serialized file data sent in the request body, then streams a response using `streamText` with two AI tools: `str_replace_editor` and `file_manager`.
3. **AI tool calls** stream back to the client; `ChatProvider.onToolCall` dispatches them to `FileSystemContext.handleToolCall`, which mutates the in-memory `VirtualFileSystem` and triggers a React re-render.
4. **Preview** (`PreviewFrame`) reads all files from `FileSystemContext`, runs them through `createImportMap` (which Babel-transforms JSX/TSX to JS via `@babel/standalone` and creates blob URLs), then renders the result in an `<iframe>` using native ES module import maps. External npm packages are resolved via `esm.sh`.
5. **Persistence**: on `onFinish`, the API route saves messages + serialized file system to the `Project` row in SQLite (only for authenticated users with a `projectId`).

### Key Abstractions

- **`VirtualFileSystem`** (`src/lib/file-system.ts`) — in-memory tree of `FileNode` objects. No disk I/O. Serializes to/from plain JSON for API transport and database storage. Files stored in `Project.data` as JSON.
- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — React context wrapping a `VirtualFileSystem` instance. Exposes CRUD operations and `handleToolCall` which maps AI tool calls (`str_replace_editor`, `file_manager`) to file system mutations.
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — thin wrapper around `useAIChat` that wires `onToolCall` to `FileSystemContext.handleToolCall`.
- **`jsx-transformer.ts`** (`src/lib/transform/`) — Babel-transforms JSX/TSX files to browser-runnable JS, builds an import map with blob URLs for local files, resolves `@/` aliases to root-level paths, and routes third-party imports through `esm.sh`.
- **`getLanguageModel()`** (`src/lib/provider.ts`) — returns `anthropic("claude-haiku-4-5")` when `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that generates static counter/form/card components.

### Authentication

Custom JWT-based auth (`src/lib/auth.ts`) using `jose`. Sessions stored in an httpOnly cookie (`auth-token`). Middleware (`src/middleware.ts`) validates routes. Users can use the app anonymously (without a project ID); anonymous work is tracked in `src/lib/anon-work-tracker.ts` and can be claimed on sign-up.

### AI Tools Available to the Model

- `str_replace_editor` — commands: `view`, `create`, `str_replace`, `insert`
- `file_manager` — commands: `rename`, `delete`

### Generated Component Conventions

The system prompt (`src/lib/prompts/generation.tsx`) instructs the AI to:
- Always create `/App.jsx` as the entry point with a default export
- Use `@/` import alias for all local files (e.g., `import X from '@/components/X'`)
- Style with Tailwind CSS only (no hardcoded styles)
- No HTML files — the virtual FS entry point is always `/App.jsx`

### Database

The schema is defined in `prisma/schema.prisma` — reference it any time you need to understand the structure of data stored in the database. Prisma with SQLite (`prisma/dev.db`). Two models: `User` (email/password) and `Project` (stores `messages` and `data` as JSON strings). Prisma client generated to `src/generated/prisma`.

### Testing

Vitest with jsdom and React Testing Library. Tests colocated in `__tests__` directories next to source files.
