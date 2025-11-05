# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It allows users to describe React components in natural language, generates them using Claude AI (or mock responses), and displays them in a real-time preview using a virtual file system.

**Tech Stack:**
- Next.js 15 with App Router
- React 19
- TypeScript
- Tailwind CSS v4
- Prisma with SQLite
- Anthropic Claude AI via Vercel AI SDK
- Vitest for testing

## Development Commands

### Initial Setup
```bash
npm run setup
```
This installs dependencies, generates Prisma client, and runs database migrations.

### Development
```bash
npm run dev              # Start dev server with Turbopack
npm run dev:daemon       # Start dev server in background, logs to logs.txt
```
The app runs at http://localhost:3000

### Testing
```bash
npm test                 # Run tests with Vitest
```

### Building
```bash
npm run build           # Production build
npm start               # Run production build
npm run lint            # Run ESLint
```

### Database
```bash
npx prisma generate      # Generate Prisma client
npx prisma migrate dev   # Run migrations
npm run db:reset         # Reset database (force)
```

## Architecture

### Virtual File System

The core innovation is the `VirtualFileSystem` class (src/lib/file-system.ts) which manages an in-memory file tree. No files are written to disk during component generation. The VFS:

- Stores files in a `Map<string, FileNode>` structure
- Supports standard file operations: create, read, update, delete, rename
- Serializes to/from JSON for persistence in the database
- Used by AI tools to manipulate generated components

### AI Integration

The chat endpoint (src/app/api/chat/route.ts) uses Vercel AI SDK's `streamText` with two custom tools:

1. **str_replace_editor** - File viewing and editing (create, view, str_replace, insert)
2. **file_manager** - File system operations (list, move, delete directories)

Both tools operate on the `VirtualFileSystem` instance, which is:
- Reconstructed from serialized data on each request
- Passed to AI tools via `buildStrReplaceTool()` and `buildFileManagerTool()`
- Persisted to database on completion (if user is authenticated)

The system prompt (src/lib/prompts/generation.tsx) instructs the AI to:
- Create React components with Tailwind CSS
- Always start with /App.jsx as the entry point
- Use '@/' import alias for local files
- Operate on the root route '/' of the virtual FS

### Mock Provider Fallback

If no `ANTHROPIC_API_KEY` is set, a `MockLanguageModel` (src/lib/provider.ts) generates static component code instead of calling the API. This allows the app to function without API credentials for demonstration purposes.

### Authentication & Data Persistence

- JWT-based authentication (src/lib/auth.ts) with session verification
- Middleware (src/middleware.ts) protects API routes requiring authentication
- Anonymous users can work without logging in; their work is stored in sessionStorage via `anon-work-tracker.ts`
- Authenticated users have projects persisted to SQLite via Prisma:
  - `messages` field stores conversation history (JSON)
  - `data` field stores serialized VFS state (JSON)

### Database Schema

```prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  password  String
  projects  Project[]
}

model Project {
  id        String   @id @default(cuid())
  name      String
  userId    String?  # Nullable to support anonymous projects
  messages  String   @default("[]")  # JSON-serialized chat history
  data      String   @default("{}")  # JSON-serialized VFS state
  user      User?    @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

Note: Prisma client is generated to `src/generated/prisma` (not the default location).

### Component Preview

Generated React components are rendered in an iframe using Babel Standalone for runtime JSX transpilation. The preview system:
- Transforms JSX to executable JavaScript in the browser
- Injects Tailwind CSS
- Supports hot-reloading when files change
- Located in src/components/preview

### Testing

Tests use Vitest with React Testing Library and jsdom. Test files are colocated in `__tests__` directories next to their source files.

## Important Notes

- All component generation happens through AI tool calls; the VFS is the single source of truth
- The `/App.jsx` file is the required entry point for every generated project
- Import aliases use '@/' prefix (configured via tsconfig paths)
- Session persistence differs: sessionStorage for anonymous users, database for authenticated users
- The system operates with or without an Anthropic API key (falls back to mock provider)
- **Node.js v25+**: The dev script disables experimental Web Storage (`--no-experimental-webstorage`) to prevent conflicts with client-side sessionStorage usage
