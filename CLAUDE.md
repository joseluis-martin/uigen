# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps + generate Prisma client + run migrations
npm run dev          # Start dev server with Turbopack at http://localhost:3000
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all tests with Vitest
npx vitest run src/path/to/__tests__/file.test.tsx  # Run a single test file
npm run db:reset     # Reset database (destructive)
```

Dev server requires `NODE_OPTIONS='--require ./node-compat.cjs'` (already in the script).

Without `ANTHROPIC_API_KEY` in `.env`, a mock provider returns static code instead of calling Claude.

## Architecture

**UIGen** is a Next.js 15 (App Router) AI-powered React component generator. Users describe components in chat; Claude generates code that appears in a live preview.

### Data Flow

1. User sends a message → `/api/chat` route streams a response using Vercel AI SDK
2. Claude calls one of two tools: `str_replace_editor` (create/modify files) or `file_manager` (rename/delete)
3. Tool calls update the **virtual file system** (in-memory only, never written to disk)
4. The virtual FS state serializes to JSON and persists in `Project.data` (SQLite via Prisma)
5. The **PreviewFrame** transforms the virtual FS into an iframe: Babel standalone compiles JSX → HTML with an import map

### Key Subsystems

**Virtual File System** (`src/lib/file-system.ts`, `src/lib/contexts/file-system-context.tsx`)
- In-memory tree of `File`/`Directory` nodes
- Serializable to/from JSON for database persistence
- Context provides global state across editor, preview, and chat

**AI Generation** (`src/app/api/chat/`, `src/lib/tools/`, `src/lib/prompts/`)
- System prompt instructs Claude to always create `/App.jsx` as entrypoint, use Tailwind, use `@/` imports for internal files
- Two tools: `str_replace_editor` handles create/overwrite/str_replace operations; `file_manager` handles rename/delete
- Provider selection in `src/lib/provider.ts`: uses Claude haiku (`claude-haiku-4-5`) when API key present, mock otherwise

**Preview** (`src/components/preview/PreviewFrame.tsx`, `src/lib/transform/jsx-transformer.ts`)
- Renders components inside a sandboxed iframe
- `@babel/standalone` compiles JSX in the browser
- Import map resolves `react`, `react-dom`, and `@/` aliases to virtual FS files

**Authentication** (`src/lib/auth.ts`, `src/actions/index.ts`, `src/middleware.ts`)
- JWT sessions via `jose`, stored in HTTP-only cookies (7-day expiry)
- bcrypt password hashing
- Anonymous users can work on projects; registration links them to an account

**Database** (`prisma/schema.prisma`, SQLite `prisma/dev.db`)
- Reference `prisma/schema.prisma` whenever you need to understand the structure of data stored in the database
- Two models: `User` (email/password) and `Project` (name, messages JSON, virtual FS data JSON)
- Prisma client generated to `src/generated/prisma/`

## Code Style

Use comments sparingly. Only comment complex code.

### Path Aliases

`@/*` maps to `src/*` (configured in `tsconfig.json` and `vitest.config.mts`).

### Testing

Tests live in `__tests__/` directories adjacent to source files. Vitest + jsdom + React Testing Library.
