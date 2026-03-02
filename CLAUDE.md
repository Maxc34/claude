# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
npm run dev          # Start dev server with Turbopack at localhost:3000
npm run dev:daemon   # Start dev server in background (logs → logs.txt)
npm run build        # Production build
npm run lint         # ESLint (Next.js config)
npm run test         # Run Vitest tests
npm run test -- --watch  # Watch mode
npm run test -- --run src/components/chat/__tests__/MessageList.test.tsx  # Single test file

# Database
npm run setup        # First-time setup: install, prisma generate, migrate
npm run db:reset     # Reset and reseed database
```

The dev server requires `NODE_OPTIONS='--require ./node-compat.cjs'` (already set in the npm script) due to Node.js compatibility shims.

## Environment Variables

```bash
ANTHROPIC_API_KEY=   # Optional. Without it, falls back to MockLanguageModel
JWT_SECRET=          # Optional. Defaults to "development-secret-key" if unset
```

## Architecture

### Core Concept

UIGen is an AI-powered React component generator. Users describe components in a chat interface; the AI generates code using tool calls that modify a **virtual in-memory file system**; changes are reflected live in an iframe preview.

### Routing

- `src/app/page.tsx` — home page, renders `MainContent` with no project
- `src/app/[projectId]/page.tsx` — loads a saved project by ID, passes it to `MainContent`
- `src/app/main-content.tsx` — root UI container for both routes

### Three-Panel Layout

`src/app/main-content.tsx` is the root container:
- **Left panel (35%):** Chat interface (`src/components/chat/`)
- **Middle/right panel (65%):** Tabs switching between:
  - **Preview tab:** Live iframe rendering via `src/components/preview/PreviewFrame.tsx`
  - **Code tab:** File tree (`src/components/editor/FileTree.tsx`) + Monaco editor (`src/components/editor/CodeEditor.tsx`)

### AI Integration Flow

1. User sends message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. Route calls `streamText()` (Vercel AI SDK) with the model from `src/lib/provider.ts`
3. Model uses two tools (Zod schemas defined in `src/lib/tools/`):
   - `str_replace_editor` — view/create/edit files (`str_replace`, `insert`, `create`, `view`)
   - `file_manager` — rename/delete files
4. Tool calls are streamed back and applied to the **File System Context** (`src/lib/contexts/file-system-context.tsx`)
5. File changes trigger re-render of the preview iframe
6. On completion, project state is saved to the database if the user is authenticated

The route reconstructs `VirtualFileSystem` from serialized data in the request body on every call. `maxSteps` is 40 for real models, 4 for the mock. The system prompt uses `@anthropic cacheControl: ephemeral` for prompt caching.

### Virtual File System

`src/lib/file-system.ts` — in-memory file tree, never writes to disk. Files exist only in React state. Serialized to JSON for database persistence. The context (`file-system-context.tsx`) wraps the app and provides `useFileSystem()`.

`Project.messages` and `Project.data` are stored as raw JSON strings in SQLite (not native JSON columns) and must be manually parsed/serialized in application code.

### Preview Pipeline

`src/lib/transform/jsx-transformer.ts` handles the client-side rendering pipeline:
1. Transpiles JSX/TSX with Babel standalone
2. Builds an ESM import map pointing to esm.sh CDN for React 19, react-dom, and inlined project files
3. Injects the result into a sandboxed iframe (`allow-scripts allow-same-origin allow-forms`)

Entry point detection order: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` (root and `/src/` checked). CSS imports are stripped from transpiled code.

### AI Provider

`src/lib/provider.ts` selects the model:
- **With `ANTHROPIC_API_KEY`:** Uses `claude-haiku-4-5` via `@ai-sdk/anthropic` with prompt caching
- **Without API key:** Falls back to `MockLanguageModel` — deterministic responses that simulate 3–4 tool-call steps, generating preset component types (Counter, ContactForm, Card) based on keywords in the prompt. Useful for local development and tests.

System prompt lives in `src/lib/prompts/generation.ts`. Key constraints it enforces: always create `/App.jsx`, use Tailwind CSS (no inline styles), use `@/` import aliases.

### Authentication

JWT-based (`src/lib/auth.ts`): HS256 tokens in httpOnly cookies (7-day expiry). Middleware at `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes.

Server Actions in `src/actions/`:
- `index.ts` — sign-up, sign-in, logout
- `create-project.ts`, `get-project.ts`, `get-projects.ts` — project persistence

Anonymous usage is supported — projects can exist without a logged-in user. `src/lib/anon-work-tracker.ts` tracks in-progress anonymous work.

### Database

Prisma with SQLite. **Always reference `prisma/schema.prisma` for the full database structure.** Summary:
- `User`: email (unique), hashed password
- `Project`: name, messages (JSON), data (JSON), optional userId

Generated client in `src/generated/prisma/`.

### State Management

- **Chat state:** Vercel AI SDK's `useChat()` hook, wrapped in `src/lib/contexts/chat-context.tsx`
- **File system state:** Custom context in `src/lib/contexts/file-system-context.tsx`
- No Redux or Zustand — state is React context + hooks

### Testing

Tests live in `__tests__/` subdirectories next to the code they test (e.g., `src/components/chat/__tests__/`, `src/lib/__tests__/`, `src/lib/contexts/__tests__/`). Uses Vitest + jsdom + `@testing-library/react`. Path aliases (`@/*`) work via `tsconfigPaths()` plugin in `vitest.config.mts`.

### Code Style

Only add comments for complex or non-obvious logic. Self-explanatory code should speak for itself.

### Key Conventions

- `"use client"` directive required on any component using hooks, browser APIs, or event handlers
- Server Actions (`"use server"`) for auth and project persistence
- Path alias `@/*` maps to `src/*`
- shadcn/ui components (New York style) in `src/components/ui/`
- Tailwind CSS v4 (no `tailwind.config.ts` needed for basic usage; config exists for extensions)
