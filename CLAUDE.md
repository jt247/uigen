# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Run tests in watch mode
npx vitest

# Lint
npm run lint

# Reset the database
npm run db:reset

# Regenerate Prisma client after schema changes
npx prisma generate && npx prisma migrate dev
```

## Environment

Copy `.env` and add `ANTHROPIC_API_KEY` to use real Claude. Without a key, the app uses `MockLanguageModel` (defined in `src/lib/provider.ts`), which returns static hardcoded responses — useful for UI development without burning API credits.

The model used when a real key is present is set by the `MODEL` constant in `src/lib/provider.ts` (currently `claude-haiku-4-5`).

## Architecture

### Request / AI flow

1. User sends a message → `ChatContext` (`src/lib/contexts/chat-context.tsx`) calls `POST /api/chat` via the Vercel AI SDK `useChat` hook.
2. The API route (`src/app/api/chat/route.ts`) reconstructs a `VirtualFileSystem` from the serialized file state sent in the request body, then streams Claude responses using `streamText` with two tools:
   - **`str_replace_editor`** — view, create, str_replace, insert operations on the VFS (built by `src/lib/tools/str-replace.ts`)
   - **`file_manager`** — rename and delete operations on the VFS (built by `src/lib/tools/file-manager.ts`)
3. As tool calls stream back, `ChatContext` passes them to `FileSystemContext.handleToolCall`, which mirrors every mutation against the *client-side* VFS instance so the editor and preview update in real time — without waiting for the response to finish.
4. On stream completion, the server saves the full message history and serialized VFS to the database (only for authenticated users with a `projectId`).

### Virtual File System

`VirtualFileSystem` (`src/lib/file-system.ts`) is an in-memory tree — no files are written to disk. It exists on both the server (per-request) and client (in React context). The client copy is kept in sync via `handleToolCall`; the server copy is reconstructed from the serialized JSON sent with each request.

Serialization: `fileSystem.serialize()` → `Record<string, FileNode>` (strips `Map` children), stored as a JSON string in the `Project.data` column.

### Preview rendering

`PreviewFrame` renders an `<iframe>` whose `srcdoc` is regenerated whenever the VFS changes. The pipeline (in `src/lib/transform/jsx-transformer.ts`):

1. Transform every `.js/.jsx/.ts/.tsx` file with **Babel standalone** (in-browser).
2. Convert each transformed file into a `blob:` URL.
3. Build a native ES module **import map** that maps local `@/` alias paths and third-party package names (resolved via `esm.sh`) to those blob URLs.
4. Inject the import map + Tailwind CDN + an error boundary into the preview HTML.

The app entry point is always `/App.jsx`. The system prompt enforces this convention.

### Auth

JWT-based, server-only (`src/lib/auth.ts`). Sessions are stored in an `httpOnly` cookie (`auth-token`). `getSession()` is called from server actions and the API route; the middleware at `src/middleware.ts` handles route protection. Passwords are hashed with bcrypt. There is no OAuth — email/password only.

Anonymous users can use the app without signing in; their in-progress work is tracked in `src/lib/anon-work-tracker.ts` (localStorage) so it can be offered for save after sign-up.

### Data model (SQLite via Prisma)

- **`User`**: id, email (unique), password (bcrypt hash)
- **`Project`**: id, name, userId (optional — anonymous projects have no owner), `messages` (JSON array of AI SDK `Message[]`), `data` (JSON of serialized VFS)

### React context tree

```
FileSystemProvider        ← owns VirtualFileSystem instance + selectedFile
  └── ChatProvider        ← owns useChat (Vercel AI SDK), calls handleToolCall on tool events
        └── UI components (ChatInterface, CodeEditor, PreviewFrame, FileTree)
```

`FileSystemProvider` can accept `initialData` (a serialized VFS from the DB) for hydrating saved projects.

### UI components

Shadcn/ui pattern: primitives in `src/components/ui/` are thin wrappers over Radix UI. Feature components live in `src/components/chat/`, `src/components/editor/`, and `src/components/preview/`.

### Testing

Vitest + Testing Library. Test files sit next to source in `__tests__/` subdirectories. The vitest config is at `vitest.config.mts`; it uses jsdom and `@vitejs/plugin-react`.
