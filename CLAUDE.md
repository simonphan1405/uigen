# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude AI (via Anthropic API) to generate React components through a chat interface, displays them in real-time in a virtual file system, and provides an in-browser preview using Babel transformation and ES module imports.

## Common Commands

### Setup
```bash
npm run setup  # Install dependencies, generate Prisma client, and run migrations
```

### Development
```bash
npm run dev              # Start development server with Turbopack
npm run dev:daemon       # Start dev server in background, logs to logs.txt
```

### Testing & Linting
```bash
npm test                 # Run Vitest tests
npm run lint            # Run ESLint
```

### Database
```bash
npx prisma generate     # Generate Prisma client after schema changes
npx prisma migrate dev  # Create and apply migrations
npm run db:reset        # Reset database (WARNING: deletes all data)
```

### Build & Production
```bash
npm run build           # Build for production
npm start               # Start production server
```

## Architecture

### Core Concept: Virtual File System

The application uses an **in-memory virtual file system** (`VirtualFileSystem` class in `src/lib/file-system.ts`) that stores generated components without writing to disk. This is the heart of the application.

- Files exist only in memory during runtime
- File structure mirrors a real filesystem with paths, directories, and files
- Serializes to/from JSON for database persistence
- Provides text-editor-like operations: `viewFile`, `replaceInFile`, `insertInFile`

### AI Tool Integration

The AI component generation works through a tool-calling pattern:

1. **Chat API** (`src/app/api/chat/route.ts`):
   - Receives messages and serialized file system state
   - Uses Vercel AI SDK's `streamText` with Claude (or mock provider if no API key)
   - Provides two tools to the AI:
     - `str_replace_editor`: Create, edit files (str_replace, insert commands)
     - `file_manager`: Rename, delete files
   - Saves messages and file state to database on completion

2. **Context Providers**:
   - `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`): Manages virtual file system, handles tool calls, triggers UI refreshes
   - `ChatProvider` (`src/lib/contexts/chat-context.tsx`): Wraps Vercel AI SDK's `useChat`, connects file system state to API, tracks anonymous work

3. **Tool Call Flow**:
   ```
   User message → ChatContext → API endpoint → Claude + Tools → Tool calls stream back →
   FileSystemContext.handleToolCall() → VirtualFileSystem updates → UI refresh
   ```

### Client-Side Preview System

The preview system transforms and executes code in the browser:

1. **JSX Transformation** (`src/lib/transform/jsx-transformer.ts`):
   - Uses `@babel/standalone` to transform JSX/TSX to plain JavaScript
   - Handles TypeScript and React JSX transform
   - Creates blob URLs for each transformed module
   - Builds an import map for ES module resolution

2. **Import Map Resolution**:
   - Third-party packages: Resolved to `https://esm.sh/{package}`
   - Local files: Blob URLs pointing to transformed code
   - Supports `@/` path alias (maps to root `/`)
   - Handles missing files by creating placeholder modules

3. **Preview HTML Generation** (`createPreviewHTML`):
   - Generates standalone HTML with embedded import map
   - Includes Tailwind CSS CDN
   - Creates error boundary for runtime errors
   - Displays syntax errors prominently if transformation fails
   - Dynamically imports entry point (usually `/App.jsx`) and renders to `#root`

### Authentication & Projects

- **Anonymous users**: Can use the app, but work is stored in `localStorage` only
- **Registered users**: Work persists to SQLite database via Prisma
- **Database models** (`prisma/schema.prisma`):
  - `User`: id, email, password (bcrypt hashed)
  - `Project`: id, name, userId, messages (JSON), data (serialized file system JSON)
- **Auth** (`src/lib/auth.ts`): Custom JWT-based session using `jose` library
- **Actions** (`src/actions/`): Server actions for creating/fetching projects

### Mock Provider Mode

When `ANTHROPIC_API_KEY` is not set, a `MockLanguageModel` (`src/lib/provider.ts`) is used:
- Returns static component code (Counter, ContactForm, or Card)
- Simulates streaming and tool calls
- Useful for development/testing without API costs

## Key Files

- `src/lib/file-system.ts`: Virtual file system implementation
- `src/lib/transform/jsx-transformer.ts`: Babel transformation and import map generation
- `src/app/api/chat/route.ts`: Main API endpoint for AI interactions
- `src/lib/contexts/file-system-context.tsx`: Client-side file system management
- `src/lib/contexts/chat-context.tsx`: Chat state management
- `src/lib/provider.ts`: AI model provider (real or mock)
- `src/lib/prompts/generation.ts`: System prompt for component generation (if exists)

## Important Notes

- **Node compatibility**: Uses `node-compat.cjs` to polyfill Node.js modules in Next.js edge runtime
- **Turbopack**: Development uses Turbopack for faster builds
- **Path alias**: `@/` maps to `src/` directory via `tsconfig.json`
- **Prisma output**: Client is generated to `src/generated/prisma` (not default location)
- **React 19**: Using latest React with automatic JSX runtime
- **Tailwind v4**: Uses new PostCSS-based Tailwind CSS v4
