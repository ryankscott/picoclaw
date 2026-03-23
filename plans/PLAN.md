# Plan: Picoclaw — TUI AI Agent with Vercel AI SDK

## TL;DR

Build a lightweight clone of [nanoclaw](https://github.com/qwibitai/nanoclaw) that replaces external messaging channels (WhatsApp, Telegram, etc.) with an Ink-based TUI chat, and swaps the Claude Agent SDK for the Vercel AI SDK to enable any model provider. No containers — tools run directly in-process. Keeps nanoclaw's core: SQLite storage, markdown-based memory, scheduled tasks, and conversation isolation.

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│                 Ink TUI (React)                   │
│  ┌────────────┐ ┌─────────────────────────────┐  │
│  │ Sidebar    │ │ Chat View (messages stream) │  │
│  │ (convos)   │ │                             │  │
│  │            │ │                             │  │
│  │            │ ├─────────────────────────────┤  │
│  │            │ │ Input + Status Bar          │  │
│  └────────────┘ └─────────────────────────────┘  │
├──────────────────────────────────────────────────┤
│              Vercel AI SDK (streamText)           │
│  Model: openai:gpt-4o / anthropic:claude-sonnet-4-20250514 / ...  │
│  Tools: file ops, bash, web, MCP                 │
├──────────────────────────────────────────────────┤
│  SQLite (messages, conversations, tasks)         │
│  Memory (markdown files per conversation)        │
│  Scheduler (cron/interval/once tasks)            │
└──────────────────────────────────────────────────┘
```

**Key differences from nanoclaw:**
- No channels, no channel registry, no JID routing — single TUI interface
- No containers — tools execute directly in the host process
- No Claude Agent SDK — Vercel AI SDK with model registry for provider flexibility
- No IPC filesystem — direct function calls
- Conversations replace "groups" — local-only, no external chat IDs

---

## Steps

### Phase 1: Project Scaffolding

1. **Initialize the project** — `npm init`, TypeScript config, ESM module setup
   - `package.json` with `"type": "module"`, scripts: `build`, `start`, `dev`
   - `tsconfig.json` targeting ES2022, NodeNext module resolution

2. **Install core dependencies**
   - `ai` (Vercel AI SDK core)
   - `@ai-sdk/openai`, `@ai-sdk/anthropic`, `@ai-sdk/google` (provider packages)
   - `@ai-sdk/mcp` (MCP client support)
   - `ink`, `react` (TUI framework)
   - `better-sqlite3` (SQLite)
   - `cron-parser` (scheduled tasks)
   - `pino` (logging)
   - `zod` (tool parameter schemas)
   - `dotenv` (env config)
   - Dev: `typescript`, `tsx`, `@types/react`, `@types/better-sqlite3`, `vitest`

### Phase 2: Core Infrastructure

3. **Config module** (`src/config.ts`) — Load from `.env`:
   - `AI_MODEL` — model identifier string, e.g. `openai:gpt-4o`
   - `AI_API_KEY` — API key for the active provider (or provider-specific vars like `OPENAI_API_KEY`)
   - `ASSISTANT_NAME` — default "Pico"
   - `STORE_DIR`, `CONVERSATIONS_DIR`, `DATA_DIR` — paths
   - `SCHEDULER_POLL_INTERVAL` — default 60s

4. **SQLite database** (`src/db.ts`) — Adapted from nanoclaw's `db.ts`:
   - Tables: `messages`, `conversations`, `sessions`, `scheduled_tasks`, `task_run_logs`
   - `conversations` replaces `registered_groups` — fields: `id`, `name`, `folder`, `created_at`, `system_prompt`
   - `messages` — `id`, `conversation_id`, `role` (user/assistant/system/tool), `content`, `timestamp`
   - Reuse nanoclaw's task tables verbatim

5. **Types** (`src/types.ts`) — Simplified from nanoclaw:
   - `Conversation` (replaces `RegisteredGroup`)
   - `Message` (simplified, no JID/sender for single-user)
   - `ScheduledTask`, `TaskRunLog` (kept from nanoclaw)

6. **Memory system** (`src/memory.ts`):
   - Global memory: `memory/MEMORY.md` — loaded as system prompt prefix for all conversations
   - Per-conversation memory: `conversations/{name}/MEMORY.md` — loaded for that conversation
   - Read/write helpers for the AI to use as tools

### Phase 3: AI Agent Layer

7. **Model provider registry** (`src/ai/provider.ts`):
   - Parse model strings like `openai:gpt-4o`, `anthropic:claude-sonnet-4-20250514`, `google:gemini-2.5-pro`
   - Use `createOpenAI()`, `createAnthropic()`, etc. based on prefix
   - Support `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GOOGLE_GENERATIVE_AI_API_KEY` env vars
   - Also support custom base URLs via `*_BASE_URL` vars (for Ollama, etc.)
   - Export `getModel(modelString)` → returns a language model instance

8. **Tool definitions** (`src/ai/tools/`): *parallel with step 7*
   - `file-ops.ts` — `readFile`, `writeFile`, `editFile`, `glob`, `grep` tools using `zod` schemas
   - `bash.ts` — `bash` tool: spawns child process, captures stdout/stderr, with timeout and working directory scoped to conversation folder
   - `web.ts` — `webFetch` (fetch URL content), `webSearch` (if search API configured)
   - `memory.ts` — `readMemory`, `writeMemory` (read/write MEMORY.md files)
   - `scheduler.ts` — `scheduleTask`, `listTasks`, `pauseTask`, `resumeTask`, `cancelTask`
   - Each tool defined using Vercel AI SDK's `tool()` with zod parameter schemas

9. **MCP client** (`src/ai/mcp.ts`): *parallel with steps 7-8*
   - Load MCP server configs from `.mcp.json` or config
   - Use `@ai-sdk/mcp` `createMCPClient()` to connect to configured MCP servers
   - Merge MCP tools with built-in tools for each agent call

10. **Agent runner** (`src/ai/agent.ts`) — *depends on steps 7-9*:
    - `runAgent(conversation, messages, onChunk)` function
    - Uses `streamText()` from Vercel AI SDK with:
      - Model from provider registry
      - System prompt: global memory + conversation memory + conversation system prompt
      - Message history from SQLite
      - All tools (built-in + MCP)
      - `maxSteps` for multi-step tool use (agentic loop)
    - Streams text chunks to TUI via callback
    - Stores complete response in SQLite when done

### Phase 4: Scheduled Tasks

11. **Task scheduler** (`src/scheduler.ts`) — Adapted from nanoclaw:
    - Timer that checks SQLite for due tasks
    - When a task fires: load conversation context, call `runAgent()` with the task prompt
    - Results stored in `task_run_logs`
    - Support cron (via `cron-parser`), interval, and one-time schedules
    - Tasks are scoped to conversations (like nanoclaw groups)

### Phase 5: TUI Interface

12. **Ink app entry point** (`src/index.tsx`):
    - Initialize database, load config
    - Start scheduler in background
    - Render `<App />` component

13. **App component** (`src/components/app.tsx`) — *depends on step 12*:
    - Layout: sidebar (conversation list) + main area (chat + input)
    - State: active conversation, messages, streaming status
    - Keyboard shortcuts: Ctrl+N (new conversation), Ctrl+D/Q (quit), Tab (switch focus), Up/Down (navigate conversations)

14. **Chat components** (`src/components/`) — *parallel with step 13*:
    - `chat.tsx` — Scrollable message list, renders user/assistant messages with markdown
    - `input.tsx` — Multi-line text input at bottom, Enter to send
    - `message.tsx` — Single message bubble (role indicator, content, timestamp)
    - `status-bar.tsx` — Shows current model, conversation name, streaming indicator
    - `sidebar.tsx` — List of conversations, create new, switch active

15. **Markdown rendering** — Use `ink-markdown` or `marked-terminal` for rendering AI responses with formatting in the terminal

### Phase 6: Wiring & Polish

16. **Wire TUI to agent** — *depends on steps 10, 13-14*:
    - On user submit: store message in SQLite, call `runAgent()`, stream response to chat view
    - On conversation switch: load message history from SQLite
    - On new conversation: create folder + MEMORY.md, insert into SQLite

17. **Model switching** — Ctrl+M or `/model` command to change active model at runtime

18. **Slash commands**: `/new [name]`, `/switch [name]`, `/model [model]`, `/memory [text]`, `/tasks`, `/quit`

---

## File Structure

```
picoclaw/
├── package.json
├── tsconfig.json
├── .env.example
├── src/
│   ├── index.tsx               # Entry: init DB, start scheduler, render Ink app
│   ├── config.ts               # Configuration from .env
│   ├── db.ts                   # SQLite schema & queries
│   ├── types.ts                # TypeScript interfaces
│   ├── memory.ts               # Read/write markdown memory files
│   ├── scheduler.ts            # Cron/interval/once task runner
│   ├── logger.ts               # Pino logger setup
│   ├── ai/
│   │   ├── provider.ts         # Model registry (parse "openai:gpt-4o" etc.)
│   │   ├── agent.ts            # streamText agent loop with tools
│   │   ├── mcp.ts              # MCP client for external tool servers
│   │   └── tools/
│   │       ├── index.ts        # Aggregate all tools
│   │       ├── file-ops.ts     # readFile, writeFile, editFile, glob, grep
│   │       ├── bash.ts         # Shell command execution
│   │       ├── web.ts          # webFetch, webSearch
│   │       ├── memory.ts       # readMemory, writeMemory
│   │       └── scheduler.ts    # scheduleTask, listTasks, etc.
│   └── components/
│       ├── app.tsx             # Root layout component
│       ├── chat.tsx            # Scrollable message list
│       ├── input.tsx           # Text input
│       ├── message.tsx         # Single message render
│       ├── status-bar.tsx      # Model/status info
│       └── sidebar.tsx         # Conversation list
├── memory/
│   └── MEMORY.md              # Global memory file
├── conversations/              # Per-conversation folders (created at runtime)
├── store/                      # SQLite database (gitignored)
└── logs/                       # Runtime logs (gitignored)
```

---

## Relevant Files (from nanoclaw, for reference)

- `src/db.ts` — Reuse schema patterns for SQLite tables (messages, tasks, sessions)
- `src/config.ts` — Adapt configuration constants pattern
- `src/types.ts` — Simplify interfaces (remove Channel, JID, container concepts)
- `src/router.ts` — Reuse `formatMessages()` XML format for conversation history, `stripInternalTags()` for output
- `src/task-scheduler.ts` — Adapt scheduler loop (remove container spawning, call agent directly)
- `src/group-queue.ts` — Simplify to a basic async queue (no container process management)
- `container/agent-runner/src/index.ts` — Reference for tool list and agent invocation pattern

---

## Verification

1. **Unit tests** (vitest): test db operations, tool implementations, provider parsing, memory read/write, scheduler logic
2. **Manual testing**: run `npm run dev`, verify:
   - TUI renders with sidebar + chat + input
   - Can create conversation, type message, get streamed AI response
   - `/model openai:gpt-4o` switches model
   - `/tasks` shows scheduled tasks
   - Memory files are created and read correctly
   - Bash tool executes commands and returns output
   - File tools can read/write within conversation directory
3. **Model flexibility test**: configure different providers (OpenAI, Anthropic) via `.env` and verify both work
4. **MCP test**: configure a test MCP server in `.mcp.json`, verify tools appear and are callable

---

## Decisions

- **No containers** — Tools run in-process for simplicity. Bash tool should scope working directory to conversation folder but does NOT provide OS-level isolation.
- **No channel system** — Single TUI replaces all channels. No JIDs, no channel registry.
- **Model via string** — `AI_MODEL=openai:gpt-4o` pattern for easy switching. Multiple provider API keys can coexist in `.env`.
- **Conversations replace groups** — Each conversation gets its own folder, memory file, and message history. No "main" vs "other" distinction.
- **Scheduled tasks kept** — Tasks are scoped to conversations and run the agent with the task prompt when due.
- **MCP via @ai-sdk/mcp** — Leverage Vercel AI SDK's built-in MCP support rather than reimplementing.

## Further Considerations

1. **Bash safety** — Without container isolation, bash tool has full host access. Recommendation: scope working directory to conversation folder, add configurable command blocklist, and require user confirmation for destructive commands (or add a `--dangerously-skip-permissions` flag like Claude Code).
2. **Streaming UX** — Vercel AI SDK's `streamText` returns an async iterable of text deltas. Ink can re-render on each chunk for real-time streaming. Need to handle tool call rendering (show tool name + args while executing).
3. **Message history window** — For long conversations, sending all messages exceeds context limits. Recommendation: implement a sliding window (last N messages) or token-based truncation, with older messages summarized.
