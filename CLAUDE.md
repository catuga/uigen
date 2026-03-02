# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server with Turbopack on localhost:3000
npm run dev:daemon   # Run dev server in background, logs to logs.txt
npm run build        # Production build
npm run lint         # Run ESLint
npm run test         # Run Vitest unit tests
npm run setup        # Install deps + Prisma client generation + migrations
npm run db:reset     # Reset SQLite database (dev only)
```

## Architecture

UIGen is an AI-powered React component generator built with Next.js 15 App Router. Users describe components in natural language; Claude generates them with live preview and code editing.

### Key Architectural Concepts

**Virtual File System** (`/src/lib/file-system.ts`): All generated code lives in an in-memory virtual file system — no files are written to disk. It serializes to JSON for persistence in the database. Both the Monaco editor UI and AI tool calls operate on this virtual FS.

**AI Tool Calling Flow**: The chat API (`/src/app/api/chat/route.ts`) receives messages, provides Claude with the current virtual FS state, and Claude responds using `str_replace_editor` and `file_manager` tools (defined in `/src/tools/`). Tool effects stream back to the client and update the virtual FS via `FileSystemProvider` context.

**In-Browser JSX Compilation**: `PreviewFrame` (`/src/components/preview/PreviewFrame.tsx`) uses `@babel/standalone` to compile JSX in the browser, creates blob URLs for ES modules, and renders them in a sandboxed iframe. The entry point is auto-detected (App.jsx/index.jsx). See `/src/transform/jsx-transformer.ts`.

**Anonymous Mode**: Users can generate components without signing in. State is ephemeral (lost on refresh). Authenticated users get project persistence via Prisma/SQLite.

**Mock Provider**: If `ANTHROPIC_API_KEY` is not set, `/src/lib/provider.ts` falls back to a `MockLanguageModel` that returns a hardcoded component.

### State Management

Two main React contexts:
- `ChatProvider` (`/src/contexts/chat-context.tsx`): Wraps Vercel AI SDK's `useChat`, manages chat state, triggers project auto-save on completion
- `FileSystemProvider` (`/src/contexts/file-system-context.tsx`): Manages virtual FS state, handles incoming tool call results from AI responses

### Tech Stack

- **Framework**: Next.js 15, React 19, TypeScript 5
- **AI**: Vercel AI SDK (`ai`) + `@ai-sdk/anthropic` → Claude Haiku 4.5
- **Database**: Prisma ORM with SQLite (`/prisma/dev.db`); client generated to `/src/generated/prisma`
- **Auth**: JWT sessions via `jose`, passwords hashed with `bcrypt`; middleware at `/src/middleware.ts` protects `/api/projects` and `/api/filesystem`
- **Editor**: Monaco Editor (`@monaco-editor/react`)
- **Styling**: Tailwind CSS v4 + Radix UI primitives (Shadcn/ui)
- **Testing**: Vitest + Testing Library + jsdom

### Database Schema

Defined in `prisma/schema.prisma` — reference it whenever you need to understand the structure of data stored in the database. Key notes:
- `Project.messages` and `Project.data` are JSON strings (not native JSON columns, SQLite limitation)
- `Project.data` holds the serialized virtual file system
- `Project.userId` is nullable to support anonymous projects

Projects are loaded via Server Actions (`/src/actions/`) and saved automatically on chat completion.

### AI System Prompt

The system prompt in `/src/prompts/generation.tsx` instructs Claude to always use `App.jsx` as the entry point and `@/` imports for cross-file references. Anthropic's prompt caching is enabled for cost efficiency.
