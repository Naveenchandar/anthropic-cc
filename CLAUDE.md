# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude AI generates them in real-time using tool calls to manipulate a virtual file system. The preview renders in a sandboxed iframe via Babel-transformed JSX.

## Commands

```bash
# Install dependencies + initialize database
npm run setup

# Development server (Turbopack)
npm run dev

# Production build
npm run build

# Lint
npm run lint

# Run all tests
npm run test

# Database
npx prisma generate       # Regenerate Prisma client after schema changes
npx prisma migrate dev    # Run pending migrations
npm run db:reset          # Force reset database
```

**Environment:** Copy `.env.example` to `.env`. Set `ANTHROPIC_API_KEY` for real AI; omit it to use the mock provider (generates sample components without an API key).

## Architecture

### Request Flow

1. User submits a prompt → `ChatProvider` sends `POST /api/chat` with messages + serialized `VirtualFileSystem`
2. Server reconstructs the VFS, calls Claude (Haiku 4.5) with two tools: `str_replace_editor` and `file_manager`
3. AI streams text + tool calls back to the client
4. `FileSystemProvider` intercepts tool calls, updates the in-memory VFS, and propagates state to the editor/file tree
5. `PreviewFrame` watches VFS changes, compiles JSX via Babel Standalone, generates an import map (esm.sh CDN), and rerenders in a sandboxed iframe
6. On finish, if the user is authenticated, the server saves messages + serialized VFS to SQLite via Prisma

### Key Layers

| Layer | Key Files | Responsibility |
|---|---|---|
| API endpoint | `src/app/api/chat/route.ts` | Streaming AI responses, tool orchestration, project persistence |
| AI tools | `src/lib/tools/` | `str_replace_editor` (view/create/patch files), `file_manager` (rename/delete) |
| AI provider | `src/lib/provider.ts` | Selects Claude Haiku or `MockLanguageModel` based on env |
| System prompt | `src/lib/prompts/` | Instructs the model on component generation conventions |
| Virtual FS | `src/lib/file-system.ts` | In-memory file system; no disk I/O; serializable for DB storage and API transport |
| Transform | `src/lib/transform/` | Babel JSX→JS compilation, import map generation, preview HTML assembly |
| Contexts | `src/lib/contexts/` | `ChatProvider` (chat state, AI comms), `FileSystemProvider` (VFS state) |
| Auth | `src/lib/auth.ts`, `src/middleware.ts` | JWT in HTTP-only cookies (7 days), bcrypt passwords, anonymous project support |
| DB | `prisma/schema.prisma` | SQLite; `User` and `Project` models; `Project.data` stores serialized VFS as JSON string |
| UI | `src/app/main-content.tsx` | Split-panel layout: chat left, preview/code editor right |

### AI Integration Details

- Real model: `claude-haiku-4-5` via `@ai-sdk/anthropic`, max 10,000 tokens, max 40 steps, prompt caching enabled
- Mock model: generates Counter/Form/Card components; used when `ANTHROPIC_API_KEY` is absent
- Tool calls are handled client-side by `FileSystemProvider` — the server streams raw tool call deltas

### Path Aliases

`@/*` maps to `src/*` (configured in `tsconfig.json`).

### Testing

Tests use Vitest + React Testing Library. Test files live in `__tests__/` subdirectories alongside source. Run a single test file:

```bash
npx vitest run src/lib/transform/__tests__/transform.test.ts
```
