# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup
npm install && npx prisma generate && npx prisma migrate dev

# Development (requires node-compat.cjs shim for Babel standalone)
npm run dev          # Turbopack dev server
npm run dev:daemon   # Background server, logs → logs.txt

# Build & production
npm run build
npm start

# Testing
npm test             # Vitest (watch mode)
npx vitest run       # Single run
npx vitest run src/lib/__tests__/file-system.test.ts  # Single file

# Lint
npm run lint

# Database
npx prisma studio    # GUI browser
npm run db:reset     # Drop + recreate all tables (destructive)
```

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in chat; Claude generates code via tool calls that mutate a virtual file system; the preview iframe rerenders in real time.

### Request lifecycle

1. **User message** → `MessageInput` → `ChatContext` (wraps Vercel AI SDK `useChat`) → `POST /api/chat`
2. **`/api/chat`** deserializes the file system from the request, calls Claude with `str_replace_editor` and `file_manager` tools, streams the response
3. **Tool call callbacks** in `ChatContext` (`onToolCall`) forward tool results to `FileSystemContext`, which mutates the in-memory `VirtualFileSystem`
4. **`PreviewFrame`** watches the file system; on change it calls `JSXTransformer.generatePreviewHTML()`, which Babel-transforms all files, builds an ESM import map (third-party packages → CDN), and sets `iframe.srcdoc`
5. **On finish**, if the user is authenticated, the serialized file system + messages are persisted to SQLite via Prisma

### Key modules

| Path | Role |
|------|------|
| `src/app/api/chat/route.ts` | Streaming chat endpoint; tool definitions; DB persistence |
| `src/lib/contexts/FileSystemContext.tsx` | Shared virtual FS state; tool callback handler |
| `src/lib/contexts/ChatContext.tsx` | `useChat` wrapper; streams tool call results into FS |
| `src/lib/file-system.ts` | In-memory VirtualFileSystem (CRUD + rename) |
| `src/lib/tools/str-replace.ts` | `str_replace_editor` tool — create / str_replace / insert / view |
| `src/lib/tools/file-manager.ts` | `file_manager` tool — rename / delete |
| `src/lib/transform/jsx-transformer.ts` | Babel JSX → ESM; import map; blob URLs; preview HTML |
| `src/lib/provider.ts` | Anthropic client init; falls back to `MockLanguageModel` if no API key |
| `src/lib/prompts/generation.tsx` | System prompt (with Anthropic prompt-cache headers) |
| `src/lib/auth.ts` | JWT session via httpOnly cookie |
| `src/actions/` | Server actions: auth, create/get projects |
| `prisma/schema.prisma` | User → Project (messages JSON + data JSON) |

### Preview pipeline detail

`JSXTransformer` (`src/lib/transform/jsx-transformer.ts`):
- Transforms every `.tsx`/`.jsx`/`.ts`/`.js` file with `@babel/standalone`
- Rewrites bare specifier imports (e.g. `react`, `lucide-react`) to ESM CDN URLs (esm.sh)
- Rewrites `@/` and relative imports to blob URLs of transformed sibling files
- Inlines CSS files as `<style>` tags
- Produces a full HTML document with an import map set as `iframe.srcdoc`

The `node-compat.cjs` shim (required via `NODE_OPTIONS`) polyfills Node APIs that `@babel/standalone` needs at build time.

### AI integration

- Model: `claude-haiku-4-5` (configured in `src/lib/provider.ts`)
- Prompt caching is applied to the system prompt via `providerOptions: { anthropic: { cacheControl: { type: 'ephemeral' } } }`
- API route has `export const maxDuration = 120` for long generations
- Mock provider (`MockLanguageModel`) returns a static component when `ANTHROPIC_API_KEY` is absent

### Path aliases

`@/*` maps to `src/*` (configured in `tsconfig.json` and `vitest.config.mts`).
