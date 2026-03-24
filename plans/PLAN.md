# Plan: Picoclaw — TUI AI Agent with Vercel AI SDK

## TL;DR

Build a lightweight clone of [nanoclaw](https://github.com/qwibitai/nanoclaw) that replaces external messaging channels (WhatsApp, Telegram, etc.) with an Ink-based TUI chat, and swaps the Claude Agent SDK for the Vercel AI SDK to enable any model provider. No containers — tools run directly in-process. Keeps nanoclaw's core: SQLite storage, markdown-based memory, scheduled tasks, and conversation isolation.

Adds two new first-class concepts: **Skills** (markdown-defined domain expertise with tool restrictions) and **Agents** (named personas that compose multiple skills and can run autonomously on a schedule). Ships with default MCP integrations for Slack and Atlassian (Jira/Confluence), with a simple config-driven pattern for adding more.

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│                 Ink TUI (React)                   │
│  ┌────────────┐ ┌─────────────────────────────┐  │
│  │ Sidebar    │ │ Chat View (messages stream) │  │
│  │ (convos +  │ │                             │  │
│  │  agents)   │ │                             │  │
│  │            │ ├─────────────────────────────┤  │
│  │            │ │ Input + Status Bar          │  │
│  └────────────┘ └─────────────────────────────┘  │
├──────────────────────────────────────────────────┤
│         Agent Runner (resolves agent → skills)   │
│  Vercel AI SDK (streamText) + tool scoping       │
│  Model: openai:gpt-4o / anthropic:claude-sonnet-4-20250514 / ...  │
├──────────────────────────────────────────────────┤
│  Skills        │ Built-in Tools  │ MCP Tools     │
│  (markdown     │ file ops, bash, │ Slack,        │
│   definitions) │ web, memory,    │ Atlassian,    │
│                │ scheduler       │ user-defined  │
├──────────────────────────────────────────────────┤
│  SQLite (messages, conversations, tasks, agents) │
│  Memory (markdown files per conversation)        │
│  Scheduler (cron/interval/once → agent runs)     │
└──────────────────────────────────────────────────┘
```

**Key differences from nanoclaw:**
- No channels, no channel registry, no JID routing — single TUI interface
- No containers — tools execute directly in the host process
- No Claude Agent SDK — Vercel AI SDK with model registry for provider flexibility
- No IPC filesystem — direct function calls
- Conversations replace "groups" — local-only, no external chat IDs
- Adds skills (markdown-defined expertise) and agents (skill compositors) — concepts not in nanoclaw
- Default MCP integrations (Slack, Atlassian) ship out of the box

---

## Steps

### Phase 1: Project Scaffolding

1. **Initialize the project** — `bun init`, TypeScript config
   - `package.json` with scripts: `build`, `start`, `dev`
   - `tsconfig.json` — Bun handles TypeScript natively, minimal config needed

2. **Install core dependencies**
   - `ai` (Vercel AI SDK core)
   - `@ai-sdk/openai`, `@ai-sdk/anthropic`, `@ai-sdk/google` (provider packages)
   - `@ai-sdk/mcp` (MCP client support)
   - `ink`, `react` (TUI framework)
   - `cron-parser` (scheduled tasks)
   - `pino` (logging)
   - `zod` (tool parameter schemas)
   - `gray-matter` (parse YAML frontmatter from skill/agent markdown files)
   - `open` (open browser for OAuth authorization URLs)
   - Built-in (no install): `bun:sqlite` (SQLite), `bun:test` (test runner), `.env` loading, TypeScript execution
   - Dev: `@types/react`, `@types/bun`

### Phase 2: Core Infrastructure

3. **Config module** (`src/config.ts`) — Load from `.env`:
   - `AI_MODEL` — model identifier string, e.g. `openai:gpt-4o`
   - `AI_API_KEY` — API key for the active provider (or provider-specific vars like `OPENAI_API_KEY`)
   - `ASSISTANT_NAME` — default "Pico"
   - `STORE_DIR`, `CONVERSATIONS_DIR`, `DATA_DIR` — paths
   - `SKILLS_DIR` — default `skills/`
   - `AGENTS_DIR` — default `agents/`
   - `DEFAULT_AGENT` — agent name to use when none specified, e.g. `"general"`
   - `SCHEDULER_POLL_INTERVAL` — default 60s
   - `OAUTH_REDIRECT_PORT` — port for local OAuth callback server, default `9876`
   - `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET` — OAuth app credentials for Slack (optional, enables Slack MCP)
   - `ATLASSIAN_CLIENT_ID`, `ATLASSIAN_CLIENT_SECRET` — OAuth app credentials for Atlassian (optional, enables Atlassian MCP)

4. **SQLite database** (`src/db.ts`) — Uses `bun:sqlite` (built-in, no external dependency). Adapted from nanoclaw's `db.ts`:
   - Tables: `messages`, `conversations`, `sessions`, `scheduled_tasks`, `task_run_logs`, `oauth_tokens`
   - `conversations` replaces `registered_groups` — fields: `id`, `name`, `folder`, `created_at`, `system_prompt`, `agent_name` (nullable — bound agent for this conversation)
   - `messages` — `id`, `conversation_id`, `role` (user/assistant/system/tool), `content`, `timestamp`
   - `scheduled_tasks` — `id`, `conversation_id`, `agent_name`, `prompt` (what to tell the agent), `schedule_type` (`cron` | `interval` | `once`), `schedule_value` (cron expression like `0 9 * * 1-5`, interval in seconds, or ISO timestamp), `status` (`active` | `paused` | `completed` | `cancelled`), `last_run_at`, `next_run_at`, `created_at`
   - `oauth_tokens` — `provider` (e.g. `slack`, `atlassian`), `access_token`, `refresh_token`, `expires_at`, `scopes`, `metadata` (JSON — workspace/site info). Tokens encrypted at rest.
   - Reuse nanoclaw's task tables with the above addition

5. **Types** (`src/types.ts`) — Simplified from nanoclaw:
   - `Conversation` (replaces `RegisteredGroup`, includes optional `agent_name`)
   - `Message` (simplified, no JID/sender for single-user)
   - `ScheduledTask` — `{ id, conversationId, agentName, prompt, scheduleType: 'cron' | 'interval' | 'once', scheduleValue: string, status, lastRunAt?, nextRunAt, createdAt }`
   - `TaskRunLog` (kept from nanoclaw)
   - `Skill` — `{ name, description, systemPrompt, tools: string[], model?: string, temperature?: number, maxSteps?: number }`
   - `AgentDef` — `{ name, description, skills: string[], model?: string, systemPrompt, toolFilter?: string[], scheduleEligible: boolean }`
   - `OAuthToken` — `{ provider, accessToken, refreshToken?, expiresAt?, scopes, metadata? }`
   - `MCPServerConfig` — `{ command?, args?, env?, url?, headers?, oauth?: { authUrl, tokenUrl, scopes, clientIdEnv, clientSecretEnv, tokenEnvVar } }` (matches VS Code/Cursor format + optional OAuth block)

6. **Memory system** (`src/memory.ts`):
   - Global memory: `memory/MEMORY.md` — loaded as system prompt prefix for all conversations
   - Per-conversation memory: `conversations/{name}/MEMORY.md` — loaded for that conversation
   - Read/write helpers for the AI to use as tools

### Phase 3: Skills & Agents

7. **Skill definitions** (`src/skills/`):
   - Skills are markdown files in `skills/` with YAML frontmatter + body instructions
   - Frontmatter fields: `name`, `description`, `tools` (allowed tool names), `model` (optional override), `temperature`, `maxSteps`
   - Body is the system prompt content injected when the skill is active
   - Example `skills/research.md`:
     ```markdown
     ---
     name: research
     description: Deep web research and summarization
     tools: [webFetch, webSearch, readFile, writeFile, writeMemory]
     ---

     You are a thorough researcher. When given a topic, search multiple sources,
     cross-reference findings, and produce a well-structured summary with citations.
     Always save your findings to a file before responding.
     ```
   - `src/skills/loader.ts` — Parse skill files using `gray-matter` (YAML frontmatter + markdown body)
   - `src/skills/registry.ts` — Load all skills from `SKILLS_DIR` at startup, index by name, provide `getSkill(name)` and `listSkills()` accessors
   - `src/skills/types.ts` — `Skill` interface (re-exported from `src/types.ts`)

8. **Agent definitions** (`src/agents/`): *parallel with step 7*
   - Agents are markdown files in `agents/` with YAML frontmatter + persona prompt body
   - Frontmatter fields: `name`, `description`, `skills` (array of skill names to compose), `model` (optional override), `scheduleEligible` (boolean — can scheduler invoke this agent?)
   - Body is the agent persona / system prompt prepended before skill instructions
   - Example `agents/analyst.md`:
     ```markdown
     ---
     name: analyst
     description: Scheduled data analyst — researches topics and posts summaries to Slack
     skills: [research, slack-comms]
     model: anthropic:claude-sonnet-4-20250514
     scheduleEligible: true
     ---

     You are a data analyst. When a scheduled task fires, you research the given
     topic using the research skill, then post a concise summary to the
     configured Slack channel. Include key metrics and trends.
     ```
   - Example `agents/general.md` (default agent):
     ```markdown
     ---
     name: general
     description: General-purpose assistant with all tools available
     skills: []
     scheduleEligible: false
     ---

     You are Pico, a helpful assistant. You have access to all available tools.
     ```
   - `src/agents/loader.ts` — Parse agent files using `gray-matter`
   - `src/agents/registry.ts` — Load all agents from `AGENTS_DIR` at startup, index by name, provide `getAgent(name)` and `listAgents()`, validate that referenced skills exist
   - `src/agents/types.ts` — `AgentDef` interface (re-exported from `src/types.ts`)

### Phase 4: AI Layer

9. **Model provider registry** (`src/ai/provider.ts`):
   - Parse model strings like `openai:gpt-4o`, `anthropic:claude-sonnet-4-20250514`, `google:gemini-2.5-pro`
   - Use `createOpenAI()`, `createAnthropic()`, etc. based on prefix
   - Support `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GOOGLE_GENERATIVE_AI_API_KEY` env vars
   - Also support custom base URLs via `*_BASE_URL` vars (for Ollama, etc.)
   - Export `getModel(modelString)` → returns a language model instance

10. **Tool definitions** (`src/ai/tools/`): *parallel with step 9*
    - `file-ops.ts` — `readFile`, `writeFile`, `editFile`, `glob`, `grep` tools using `zod` schemas
    - `bash.ts` — `bash` tool: uses `Bun.spawn()`, captures stdout/stderr, with timeout and working directory scoped to conversation folder
    - `web.ts` — `webFetch` (fetch URL content), `webSearch` (if search API configured)
    - `memory.ts` — `readMemory`, `writeMemory` (read/write MEMORY.md files)
    - `scheduler.ts` — `scheduleTask` (create a new scheduled task with agent, prompt, and schedule), `listTasks`, `pauseTask`, `resumeTask`, `cancelTask`
    - `scheduleTask` parameters: `agentName` (which agent runs), `prompt` (what to tell it), `scheduleType` (`cron` | `interval` | `once`), `scheduleValue` (e.g. `"0 9 * * 1-5"` for weekday 9am, `"3600"` for hourly, or ISO timestamp for one-shot)
    - Each tool defined using Vercel AI SDK's `tool()` with zod parameter schemas
    - `index.ts` — Aggregate all tools into a named map; export `filterTools(toolNames: string[])` to return a subset

11. **MCP client** (`src/ai/mcp.ts`): *parallel with steps 9-10*
    - Uses the **standard VS Code / Cursor `.mcp.json` format**:
      ```json
      {
        "mcpServers": {
          "slack": {
            "command": "bunx",
            "args": ["@anthropic/mcp-slack"],
            "env": {
              "SLACK_BOT_TOKEN": "{{oauth:slack}}"
            }
          },
          "atlassian": {
            "command": "bunx",
            "args": ["@anthropic/mcp-atlassian"],
            "env": {
              "ATLASSIAN_API_TOKEN": "{{oauth:atlassian}}"
            }
          },
          "custom-server": {
            "command": "node",
            "args": ["./my-server.js"],
            "env": { "API_KEY": "sk-..." }
          },
          "remote-server": {
            "url": "https://example.com/mcp/sse",
            "headers": { "Authorization": "Bearer ..." }
          }
        }
      }
      ```
    - **Two transports** (same as VS Code/Cursor):
      - **stdio** — `command` + `args` + `env`: spawns a local process, communicates via stdin/stdout
      - **SSE / HTTP** — `url` + `headers`: connects to a remote MCP server
    - **Config layering**: `src/ai/mcp-defaults.ts` defines a default `mcpServers` object with Slack and Atlassian entries. User `.mcp.json` is deep-merged on top — user entries override defaults with the same key, and new entries are added.
    - **OAuth token interpolation**: env values containing `{{oauth:<provider>}}` are resolved at connection time by calling `getToken(provider)` from the OAuth module. This injects the live access token into the MCP server's environment. If the token is missing or the user hasn't connected yet, the server is skipped with a warning.
    - Default MCPs ship as part of `mcp-defaults.ts` but use the same `mcpServers` schema — no special format. Adding a new default = adding one entry to the defaults object.
    - An MCP server with `{{oauth:...}}` placeholders is *available* when its OAuth client credentials exist in `.env`, and *active* only after the user completes the OAuth flow via `/connect <provider>`.
    - Servers without OAuth placeholders (static `env` values, or `url`+`headers`) connect immediately at startup.
    - Use `@ai-sdk/mcp` `createMCPClient()` to connect to each server
    - Merge MCP tools with built-in tools for each agent call
    - MCP tools are namespaced by server key: `slack_send_message`, `atlassian_create_issue`, etc.

    **OAuth module** (`src/ai/oauth.ts`):
    - Handles the `{{oauth:<provider>}}` token resolution
    - `startOAuthFlow(provider)` — Spins up a temporary local HTTP server on `OAUTH_REDIRECT_PORT` (default `9876`), constructs the authorization URL with the provider's `authUrl`, `scopes`, `clientId`, and `redirect_uri=http://localhost:{port}/callback`, opens the user's browser via `open` package
    - OAuth provider configs (endpoints, scopes, client credential env var names) are defined alongside the default MCP configs in `mcp-defaults.ts` as a separate `oauthProviders` map
    - Local server handles the callback, exchanges the authorization code for tokens via the provider's `tokenUrl`, stores `access_token`, `refresh_token`, `expires_at` in the `oauth_tokens` SQLite table (encrypted at rest), then shuts down
    - `getToken(provider)` — Reads stored token, auto-refreshes if expired using `refresh_token`, updates DB. Returns valid access token or `null` if not connected.
    - `revokeToken(provider)` — Calls provider's revocation endpoint if available, deletes from DB

12. **Agent runner** (`src/ai/agent.ts`) — *depends on steps 7-11*:
    - `runAgent({ agentName?, conversation, messages, onChunk })` function
    - Resolves agent definition from registry (defaults to `DEFAULT_AGENT` config)
    - Resolves all skills referenced by the agent
    - Builds composite system prompt: global memory + conversation memory + agent persona + concatenated skill instructions
    - Selects model: agent `model` override → skill `model` override → conversation default → global `AI_MODEL` config
    - Filters tools: if agent/skills declare `tools` lists, only include those tools (union of all skill tool lists). If no restrictions, include all tools.
    - Uses `streamText()` from Vercel AI SDK with the above configuration
    - `maxSteps` for multi-step tool use (agentic loop)
    - Streams text chunks to TUI via callback
    - Stores complete response in SQLite when done

### Phase 5: Scheduled Tasks

13. **Task scheduler** (`src/scheduler.ts`) — Adapted from nanoclaw:
    - Timer that polls SQLite every `SCHEDULER_POLL_INTERVAL` for tasks where `next_run_at <= now` and `status = 'active'`
    - When a task fires: resolve `agent_name` from the task row → load agent definition → call `runAgent()` with the agent and the task's `prompt`
    - After run: update `last_run_at`, compute `next_run_at` based on schedule type:
      - **cron** — parse `schedule_value` with `cron-parser`, compute next occurrence. Supports standard 5-field cron syntax (e.g. `0 9 * * 1-5` = weekdays at 9am, `*/30 * * * *` = every 30 min)
      - **interval** — `schedule_value` is seconds (e.g. `3600` = hourly). `next_run_at = last_run_at + interval`
      - **once** — `schedule_value` is ISO timestamp. After firing, set `status = 'completed'`
    - Each task row is independent — different tasks can use different agents, prompts, and schedules
    - Only agents with `scheduleEligible: true` can be assigned to scheduled tasks (validated at task creation)
    - Results stored in `task_run_logs` (task_id, run_at, success, output/error)
    - Tasks are scoped to conversations (like nanoclaw groups)
    - Example: two tasks in the same conversation — one runs `analyst` agent daily at 9am (`cron: 0 9 * * *`), another runs `ticket-bot` every 2 hours (`interval: 7200`)

### Phase 6: TUI Interface

14. **Ink app entry point** (`src/index.tsx`):
    - Initialize database, load config
    - Load skill and agent registries from disk
    - Initialize MCP clients (default + user-defined)
    - Start scheduler in background
    - Render `<App />` component

15. **App component** (`src/components/app.tsx`) — *depends on step 14*:
    - Layout: sidebar (conversation list + active agent) + main area (chat + input)
    - State: active conversation, active agent, messages, streaming status
    - Keyboard shortcuts: Ctrl+N (new conversation), Ctrl+D/Q (quit), Tab (switch focus), Up/Down (navigate conversations)

16. **Chat components** (`src/components/`) — *parallel with step 15*:
    - `chat.tsx` — Scrollable message list, renders user/assistant messages with markdown
    - `input.tsx` — Multi-line text input at bottom, Enter to send
    - `message.tsx` — Single message bubble (role indicator, content, timestamp)
    - `status-bar.tsx` — Shows current model, conversation name, active agent name, streaming indicator
    - `sidebar.tsx` — List of conversations, create new, switch active

17. **Markdown rendering** — Use `ink-markdown` or `marked-terminal` for rendering AI responses with formatting in the terminal

### Phase 7: Wiring & Polish

18. **Wire TUI to agent** — *depends on steps 12, 15-16*:
    - On user submit: store message in SQLite, call `runAgent()` with active agent, stream response to chat view
    - On conversation switch: load message history from SQLite, restore bound agent
    - On new conversation: create folder + MEMORY.md, insert into SQLite, optionally bind an agent

19. **Model switching** — Ctrl+M or `/model` command to change active model at runtime

20. **Slash commands**:
    - `/new [name]` — create new conversation
    - `/switch [name]` — switch to conversation
    - `/model [model]` — change active model
    - `/memory [text]` — append to conversation memory
    - `/agent [name]` — switch active agent for current conversation (persisted in DB)
    - `/agents` — list available agents with their skills
    - `/skills` — list available skills
    - `/tasks` — list scheduled tasks (shows agent, schedule, next run time, status)
    - `/task create` — create a scheduled task interactively (or the AI can use the `scheduleTask` tool)
    - `/connect [provider]` — start OAuth flow for an MCP provider (e.g. `/connect slack`). Opens browser, completes auth, stores tokens.
    - `/disconnect [provider]` — revoke and remove stored OAuth tokens for a provider
    - `/connections` — list MCP providers and their connection status (available / connected / token expired)
    - `/quit` — exit

---

## File Structure

```
picoclaw/
├── package.json
├── tsconfig.json
├── .env.example
├── .mcp.json                   # User-defined MCP server overrides (optional)
├── src/
│   ├── index.tsx               # Entry: init DB, registries, MCP, scheduler, render Ink app
│   ├── config.ts               # Configuration from .env
│   ├── db.ts                   # SQLite schema & queries
│   ├── types.ts                # TypeScript interfaces (Conversation, Skill, AgentDef, etc.)
│   ├── memory.ts               # Read/write markdown memory files
│   ├── scheduler.ts            # Cron/interval/once task runner → resolves agent per task
│   ├── logger.ts               # Pino logger setup
│   ├── skills/
│   │   ├── loader.ts           # Parse skill .md files (gray-matter frontmatter + body)
│   │   ├── registry.ts         # Skill index, getSkill(), listSkills()
│   │   └── types.ts            # Skill interface (re-export)
│   ├── agents/
│   │   ├── loader.ts           # Parse agent .md files (gray-matter frontmatter + body)
│   │   ├── registry.ts         # Agent index, getAgent(), listAgents(), validate skill refs
│   │   └── types.ts            # AgentDef interface (re-export)
│   ├── ai/
│   │   ├── provider.ts         # Model registry (parse "openai:gpt-4o" etc.)
│   │   ├── agent.ts            # Agent runner: resolve agent → skills → tools → streamText
│   │   ├── mcp.ts              # MCP client manager (connect/disconnect/merge tools)
│   │   ├── mcp-defaults.ts     # Default mcpServers config + oauthProviders map
│   │   ├── oauth.ts            # OAuth flow: local callback server, token exchange, refresh, {{oauth:*}} resolution
│   │   └── tools/
│   │       ├── index.ts        # Aggregate all tools + filterTools() helper
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
│       ├── status-bar.tsx      # Model, agent, status info
│       └── sidebar.tsx         # Conversation list + active agent
├── skills/                     # User-defined skill markdown files
│   ├── research.md             # Example: web research & summarization
│   ├── slack-comms.md          # Example: Slack channel communication
│   ├── jira-triage.md          # Example: Jira issue triage & updates
│   └── code-review.md         # Example: code review & feedback
├── agents/                     # User-defined agent markdown files
│   ├── general.md              # Default agent (all tools, no skill restrictions)
│   ├── analyst.md              # Example: scheduled research + Slack reporting
│   └── ticket-bot.md          # Example: scheduled Jira triage
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

1. **Unit tests** (`bun test`): test db operations, tool implementations, provider parsing, memory read/write, scheduler logic, skill loader/registry, agent loader/registry, tool filtering
2. **Manual testing**: run `bun run dev`, verify:
   - TUI renders with sidebar + chat + input
   - Can create conversation, type message, get streamed AI response
   - `/model openai:gpt-4o` switches model
   - `/agent analyst` switches to a named agent; status bar updates
   - `/skills` lists loaded skills; `/agents` lists loaded agents
   - `/tasks` shows scheduled tasks
   - Memory files are created and read correctly
   - Bash tool executes commands and returns output
   - File tools can read/write within conversation directory
3. **Skills test**: create a custom skill `.md` in `skills/`, restart, verify it loads and its system prompt + tool restrictions are applied when used by an agent
4. **Agent test**: create an agent that composes two skills, verify the combined system prompt includes both skill instructions and tools are the union of both skill tool lists
5. **Scheduled agent test**: create a scheduled task bound to a `scheduleEligible` agent, verify task fires and the correct agent runs with its skills
6. **Model flexibility test**: configure different providers (OpenAI, Anthropic) via `.env` and verify both work
7. **OAuth test — Slack**: set `SLACK_CLIENT_ID` + `SLACK_CLIENT_SECRET` in `.env`, run `/connect slack`, verify browser opens to Slack authorization page, complete auth, verify token stored in DB and `{{oauth:slack}}` resolves in the default MCP config so Slack tools appear (e.g. `slack_send_message`). Run `/disconnect slack`, verify token removed and tools disappear.
8. **OAuth test — Atlassian**: same flow with `/connect atlassian`, verify Jira/Confluence tools.
9. **OAuth test — token refresh**: wait for token to expire (or manually set short expiry), verify `getToken()` auto-refreshes before agent run.
10. **MCP test — custom stdio**: add a custom `mcpServers` entry in `.mcp.json` with `command`+`args`+`env`, verify its tools are available alongside defaults
11. **MCP test — custom SSE**: add an entry with `url`+`headers`, verify remote connection works
12. **MCP test — override**: add a `"slack"` entry in `.mcp.json`, verify it overrides the default
13. **Connections command**: run `/connections`, verify it shows available providers, connected status, and token expiry

---

## Decisions

- **Bun runtime** — Use Bun as the runtime, package manager, and test runner. Eliminates `tsx`, `dotenv`, `better-sqlite3`, and `vitest` as dependencies — Bun provides native TypeScript execution, `.env` loading, `bun:sqlite`, and `bun test` out of the box.
- **No containers** — Tools run in-process for simplicity. Bash tool should scope working directory to conversation folder but does NOT provide OS-level isolation.
- **No channel system** — Single TUI replaces all channels. No JIDs, no channel registry.
- **Model via string** — `AI_MODEL=openai:gpt-4o` pattern for easy switching. Multiple provider API keys can coexist in `.env`.
- **Conversations replace groups** — Each conversation gets its own folder, memory file, and message history. No "main" vs "other" distinction.
- **Scheduled tasks kept** — Tasks are scoped to conversations and run a specific named agent with the task prompt when due.
- **MCP via @ai-sdk/mcp** — Leverage Vercel AI SDK's built-in MCP support rather than reimplementing.
- **Skills are markdown files** — YAML frontmatter for metadata + markdown body for system prompt instructions. Parsed with `gray-matter`. No code in skill files — skills only configure prompt + tool access, they don't define new tools.
- **Agents compose skills** — An agent references skills by name. The agent runner resolves them, unions their tool lists, concatenates their instructions, and injects it all into the `streamText` call. Agents are also markdown files with the same frontmatter+body pattern.
- **MCP config follows VS Code / Cursor format** — `.mcp.json` uses the standard `{ "mcpServers": { ... } }` schema with `command`/`args`/`env` for stdio servers and `url`/`headers` for remote servers. Default servers (Slack, Atlassian) are defined in `src/ai/mcp-defaults.ts` using the same schema and deep-merged under user configs. `{{oauth:<provider>}}` placeholders in `env` values are resolved at connection time from the OAuth token store.
- **Tool scoping via skills** — Skills declare which tools they need. When an agent activates its skills, only the union of declared tools is available. If no skills declare tool restrictions, all tools are available (the `general` agent default).

## Further Considerations

1. **Bash safety** — Without container isolation, bash tool has full host access. Recommendation: scope working directory to conversation folder, add configurable command blocklist, and require user confirmation for destructive commands (or add a `--dangerously-skip-permissions` flag like Claude Code).
2. **Streaming UX** — Vercel AI SDK's `streamText` returns an async iterable of text deltas. Ink can re-render on each chunk for real-time streaming. Need to handle tool call rendering (show tool name + args while executing).
3. **Message history window** — For long conversations, sending all messages exceeds context limits. Recommendation: implement a sliding window (last N messages) or token-based truncation, with older messages summarized.
4. **Skill/agent hot-reload** — Currently skills and agents are loaded at startup. A `/reload` slash command could re-scan the `skills/` and `agents/` directories without restarting the app. Not required for v1 but nice to have.
5. **Skill composition conflicts** — When an agent uses multiple skills that both define instructions, the concatenation order matters. Current approach: agent persona first, then skills in the order listed in the agent's `skills` array. If two skills give conflicting instructions, the later one wins in the LLM's attention. Document this behavior.
6. **MCP lifecycle** — MCP clients need to be connected at startup and disconnected on shutdown. If an MCP server goes down, the agent should degrade gracefully (tools from that server become unavailable, but the agent still works with remaining tools). Needs error handling in `mcp.ts`.
7. **OAuth security** — Tokens in the `oauth_tokens` table should be encrypted at rest using a key derived from a machine-specific secret or a user-provided passphrase. The local OAuth callback server should validate the `state` parameter to prevent CSRF. The callback server should bind to `127.0.0.1` only and shut down immediately after receiving the callback. Consider adding PKCE (Proof Key for Code Exchange) for providers that support it.
8. **OAuth for custom MCPs** — Any user-defined MCP in `.mcp.json` can use `{{oauth:<provider>}}` placeholders in its `env` values. If the referenced OAuth provider doesn't exist in `oauthProviders`, users can add one in a top-level `"oauthProviders"` key in `.mcp.json`. This keeps the same pattern extensible without code changes.
9. **Token storage portability** — Tokens are in SQLite which is local to the machine. If users want to run picoclaw on multiple machines, they'll need to re-authenticate. This is by design — tokens shouldn't be synced.
