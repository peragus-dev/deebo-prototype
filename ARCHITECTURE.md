## Deebo Prototype: Architecture and Operations Guide

### Overview
Deebo is an agentic debugging copilot designed to augment AI coding agents via the Model Context Protocol (MCP). It orchestrates a “mother” agent that coordinates investigations and spawns multiple parallel “scenario” agents, each testing a specific hypothesis on isolated Git branches. Results are logged to a memory bank and surfaced through a status “pulse.”

Key components:
- Mother agent (`src/mother-agent.ts`): Coordinates the investigation loop, manages hypotheses, executes MCP tool calls, spawns scenario agents, and synthesizes a final solution when confident.
- Scenario agent (`src/scenario-agent.ts`): Focused investigator for a single hypothesis that uses tools, gathers evidence, and produces a structured report.
- MCP server (`src/index.ts`): Exposes tools to start, check, cancel sessions, and add observations; manages process lifecycle and memory bank folders.
- Utilities (`src/util/*`): Tool client connections, logging, memory bank updates, observations, reports, branch creation, LLM plumbing, and sanitization.
- Setup CLI (`packages/deebo-setup`): One-command installer that configures MCP clients (Cursor/Cline/Claude/VS Code), writes `.env`, clones and builds Deebo under `~/.deebo`.
- Doctor CLI (`packages/deebo-doctor`): Verifies environment, tools, and configuration.

Relevant docs:
- README with quick usage and guide
- `config/tools.json` describing how external MCP tools are launched

---

### Runtime Topology and Flow

- Deebo runs as an MCP server process (Node, ESM, TypeScript → built to `build/`).
- Entry is `src/index.ts`, which:
  - Loads env, validates `MOTHER_MODEL` and `SCENARIO_MODEL`.
  - Resolves tool paths for `npx` and `uvx` (and Windows-specific `npm` bin) and exposes them via `DEEBO_*` env vars.
  - Prepares `~/.deebo/memory-bank` structure under the repo-specific project hash.
  - Registers MCP tools: `start`, `check`, `cancel`, `add_observation`.
  - Maintains a per-session registry tracking the mother AbortController and scenario PIDs.

High-level lifecycle:
1. Start: Client calls `start(error, repoPath, context?, language?, filePath?)`.
   - A session ID is created and logged. The mother agent begins its OODA loop.
2. Investigate: Mother agent observes, orients, decides, and acts repeatedly:
   - Executes MCP tool calls (git/filesystem) requested by the LLM.
   - Extracts `<hypothesis>` tags and spawns one scenario agent per hypothesis on new Git branches.
   - Incorporates new “scientific observations” written by users/tools.
   - Continues until a `<solution>` is confidently produced or the session is cancelled.
3. Scenarios: Each scenario agent investigates one hypothesis:
   - Uses tools, collects evidence, and when ready emits a structured `<report>` that gets persisted.
4. Status: Client calls `check(sessionId)` to get a human-readable pulse including mother status, scenario summaries, links to logs/reports, and solution content if complete.
5. Cancellation: Client calls `cancel(sessionId)` to abort mother and terminate tracked scenario processes.
6. Observations: Client can `add_observation(observation, sessionId, agentId?)` which feeds the loop.

---

### MCP Server API (`src/index.ts`)

- Tools
  - `start`: Creates session directories (`logs`, `reports`), registers a mother AbortController and a Set of scenario PIDs, and launches `runMotherAgent(...)` in the background.
  - `check`: Reads `mother.log` and scenario logs/reports to build a pulse: status (in_progress/completed/cancelled/failed), session duration, last activity, mother stage, solution content (if any), and per-scenario outcomes. Includes `file://` links for quick navigation.
  - `cancel`: Aborts the mother agent via AbortController, sends SIGTERM to tracked scenario PIDs, updates the PID tombstone set, and cleans registry.
  - `add_observation`: Appends a JSONL observation to the agent-specific observations log after deriving `repoPath` from the agent’s first log entry.

- Tool Path Resolution
  - Non-Windows: uses `which npx` and `which uvx`.
  - Windows: uses `where npx.cmd` and `where uvx.exe` and derives `%APPDATA%/npm` for `.cmd` shims.
  - Exposes `DEEBO_NPX_PATH`, `DEEBO_UVX_PATH`, and `DEEBO_NPM_BIN` for child processes and tool clients.

- Memory Bank Root
  - `DEEBO_ROOT` resolved as project root; memory bank under `DEEBO_ROOT/memory-bank`.
  - Project ID derived from SHA-256 of `repoPath` first 12 chars.

---

### Tooling Integration (`src/util/mcp.ts`, `config/tools.json`)

- `connectRequiredTools(agentName, sessionId, repoPath)` returns two MCP clients:
  - `git-mcp`: launched via `uvx mcp-server-git --repository {repoPath}`.
  - `desktopCommander`: launched via `npx @wonderwhy-er/desktop-commander` (or `desktop-commander.cmd` on Windows).
- Placeholder substitution in command and args:
  - `{npxPath}`, `{uvxPath}`, `{repoPath}`, `{memoryPath}`, `{memoryRoot}` are expanded before process spawn.
- Critical env passed to tool processes: `NODE_ENV`, `USE_MEMORY_BANK`, `MOTHER_*`, `SCENARIO_*`, `OPENROUTER_API_KEY`.

---

### Mother Agent (`src/mother-agent.ts`)

Purpose:
- Drive the OODA loop (Observe, Orient, Decide, Act) and converge on a high-confidence solution tagged as `<solution>...</solution>`.

Key behaviors:
- LLM configuration chosen by env: `MOTHER_HOST` in {openai, openrouter, anthropic, gemini} and `MOTHER_MODEL`.
- Initial context includes error, repo path, language, file path, session/project IDs, and a rich system prompt encouraging multiple hypotheses and memory-bank usage.
- Tool vs hypothesis precedence in a single LLM turn:
  - If a response contains both `<use_mcp_tool>` and `<hypothesis>`, Deebo executes the tools first and defers hypotheses to a later turn. Logging clarifies this precedence.
  - If a response contains only hypotheses, scenarios are spawned immediately.
  - If a response contains only tools, tools are executed.
  - If it contains only a `<solution>` (without hypotheses), the loop ends.
- Spawning scenarios:
  - Creates a unique `debug-<session>-<counter>` branch via `simple-git`.
  - Spawns `node build/scenario-agent.js --id ... --session ... --repo ... --branch ...` with env inherited.
  - Tracks PID in a shared Set for the session; applies a per-scenario timeout with SIGTERM/SIGKILL cleanup.
- Observations: Periodically polls agent-specific observations to augment message history with “Scientific observation: …”.
- Robustness:
  - Retries and backoff on empty/malformed LLM replies; logs structured events.
  - Gentle parsing for MCP tool requests; malformed XML tool calls are logged and skipped, not fatal.
  - Cooperative cancellation via AbortSignal; ensures tracked PIDs are terminated on exit.
- Memory bank updates (optional via `USE_MEMORY_BANK=true`):
  - Appends hypothesis records to `activeContext.md` and session progress to `progress.md`.

---

### Scenario Agent (`src/scenario-agent.ts`)

Purpose:
- Validate or falsify a single hypothesis on an isolated branch and produce a structured report.

Key behaviors:
- CLI-like process that parses args (`--id --session --error --context --hypothesis --language --file --repo --branch`).
- Connects to `git-mcp` and `desktop-commander` for Git and filesystem/terminal actions.
- Conversation seed:
  - System prompt mandates using tools, respecting provided context (what was already tried), and producing a final `<report>`.
  - User message includes the hypothesis and investigation context.
- Loop:
  - Executes `<use_mcp_tool>` calls; on errors, logs and feeds them back as user messages for recovery.
  - If a `<report>` tag is present without tools, it writes the report and exits.
  - Retries/backoff on malformed/empty LLM replies.
- Output:
  - Writes JSON report to `memory-bank/<project>/sessions/<session>/reports/<scenarioId>.json`.
  - Logs JSONL events to `logs/scenario-<id>.log`.

---

### Logging and Memory Bank (`src/util/logger.ts`, `src/util/membank.ts`, `src/util/observations.ts`, `src/util/reports.ts`)

- Directory structure (under `DEEBO_ROOT/memory-bank/<projectId>/sessions/<sessionId>/`):
  - `logs/`
    - `mother.log`: JSONL entries with `{timestamp, agent, level, message, data}`.
    - `scenario-<id>.log`: per-scenario JSONL logs.
  - `reports/`
    - `<scenarioId>.json`: scenario’s final report.
  - `observations/`
    - `<agentId>.log`: JSONL “scientific observations” appended by clients/tools.
- Project-level files (under `DEEBO_ROOT/memory-bank/<projectId>/`):
  - `activeContext.md`: running notes; mother appends hypothesis records here.
  - `progress.md`: appended records of session-level progress and outcomes.
- Write helpers:
  - `log(...)` appends JSONL; `writeReport(...)` pretty-prints JSON; `writeObservation(...)` appends JSONL; `updateMemoryBank(...)` appends markdown to active/progress files.

---

### Status Pulse (`check` tool)

- Reads mother and scenario logs/reports to compute:
  - Overall status: in_progress / completed / cancelled / failed.
  - Session duration and last activity.
  - Solution block (between `<solution>...</solution>`) if present.
  - Scenario summaries: Reported outcomes include extracted hypothesis, confirmation status, and investigation summary; unreported list shows running/terminated state derived from PID mapping.
  - Clickable `file://` links to `progress.md`, `mother.log`, per-scenario logs and reports.
- Tracks terminated PIDs across lines indicating “Spawned/Removed/Terminated/Cancelled Scenario … PID <n>” to classify scenario status.

---

### Git Branching Strategy (`src/util/branch-manager.ts`)

- Each scenario runs on a unique branch: `debug-<session>-<counter>`.
- Uses `simple-git` to `checkoutLocalBranch(...)`. The scenario agent should not attempt to create branches directly (guarded by a check).

---

### LLM Providers and API Integration (`src/util/agent-utils.ts`)

- Supported providers via `MOTHER_HOST`, `SCENARIO_HOST`:
  - `openai` (requires `OPENAI_API_KEY` and `OPENAI_BASE_URL`)
  - `openrouter` (requires `OPENROUTER_API_KEY`)
  - `anthropic` (requires `ANTHROPIC_API_KEY`)
  - `gemini` (requires `GEMINI_API_KEY`)
- Uniform chat API wrapper `callLlm(messages, config)` chooses client and maps message formats:
  - OpenAI-compatible: `openai.chat.completions.create` with `baseURL`.
  - OpenRouter: OpenAI-compatible endpoint at `https://openrouter.ai/api/v1`.
  - Gemini: `@google/generative-ai` with messages mapped to `Content[]`.
  - Anthropic: `@anthropic-ai/sdk` with `MessageParam[]` messages.
- Prompts:
  - Mother: mandates multiple hypotheses, prioritizes tool execution when mixed with hypotheses, uses memory bank guidance, and requires ≥96% confidence for `<solution>`.
  - Scenario: mandates tool usage and a structured `<report>` with HYPOTHESIS/CONFIRMED/INVESTIGATION/CHANGES MADE/CONFIDENCE sections.

---

### CLI Packages

#### deebo-setup (`packages/deebo-setup`)
- Purpose: One-command installer and configurator for Deebo.
- Behavior:
  - Prompts for LLM hosts/models for mother/scenario and API keys.
  - Verifies prerequisites: Node≥18, git, ripgrep, `uvx` (and installs when possible with platform-aware instructions).
  - Creates `~/.deebo`, clones the repository, installs dependencies, and builds the project.
  - Writes `~/.deebo/.env` with selected hosts, models, API keys, and `USE_MEMORY_BANK=true`.
  - Updates MCP configuration for detected clients (Cline, Claude Desktop, Cursor, VS Code settings) without overwriting other servers.
  - Windows notes: ensures VS Code/Cursor directories are created and advises installing DesktopCommander globally.

#### deebo-doctor (`packages/deebo-doctor`)
- Purpose: Verify environment health and configuration.
- Checks:
  - Node version (v18, v20, v22 accepted), Git present.
  - Tool paths (`node`, `npm`, `npx`, `uvx`, `git`, `rg`) and PATH sanity.
  - MCP tools: `uvx mcp-server-git` and `desktop-commander` availability (Windows `.cmd` checked via `%APPDATA%/npm`).
  - Configuration files: presence and validity of `~/.deebo/.env`, `~/.deebo/config/tools.json`, MCP configs in Cline/Claude/Cursor/VS Code.
  - API key heuristics by prefix per provider.
- Output: colored pass/warn/fail with optional `--verbose` details.

---

### Environment, Configuration, and Paths

Required env at server start:
- `MOTHER_MODEL`, `SCENARIO_MODEL` (hard-required by `index.ts`).
- Hosts: `MOTHER_HOST`, `SCENARIO_HOST` (used by agents).
- Provider keys: `OPENROUTER_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY` depending on hosts.
- Optional: `OPENAI_BASE_URL` for OpenAI-compatible endpoints.

Tool discovery and substitution:
- `findToolPaths()` sets `DEEBO_NPX_PATH`, `DEEBO_UVX_PATH`, and `DEEBO_NPM_BIN` (Windows)
- `config/tools.json` controls spawned MCP tools with placeholders replaced at connect time.

Project identification:
- `getProjectId(repoPath)`: SHA-256 of absolute repo path (first 12 chars) used for namespacing memory bank directories.

---

### Safety and Operational Considerations

- Tool Execution:
  - `desktop-commander` exposes terminal and filesystem actions; Deebo’s prompts constrain usage, but MCP client/server policies should enforce safe command blocking.
  - Scenario agent explicitly blocks `git_create_branch` tool usage to enforce branch creation only by mother.
- Process Management:
  - PIDs of scenario agents are tracked and terminated on `cancel` or when timeouts occur.
- Memory Bank Integrity:
  - Prompts instruct editing memory files via `edit_file` (surgical diffs) rather than `write_file` to avoid overwrites.
- Windows Specifics:
  - Uses `desktop-commander.cmd` shim to ensure proper stdio attachment; `npm` roaming path is discovered via `%APPDATA%` substitute if VS Code strips env.

---

### Building, Running, and Using Deebo

Build from source:
- Node 18+, `npm install`, `npm run build` (outputs to `build/`).

Run as MCP server (direct):
- `node --experimental-specifier-resolution=node --experimental-modules --max-old-space-size=4096 build/index.js`

Recommended install (automated):
- `npx deebo-setup@latest`
  - Follow prompts; then restart your MCP client and ensure the `deebo` server appears in MCP configuration.

Basic usage in an MCP client:
- Start a session: call `deebo.start` with `error`, `repoPath`, optional `context`, `language`, `filePath`.
- Check status: call `deebo.check` with `sessionId` (wait ~30s initially).
- Add observation: call `deebo.add_observation` with `observation`, `sessionId`, and optional `agentId`.
- Cancel: call `deebo.cancel` with `sessionId`.

Diagnostics:
- `npx deebo-doctor` (use `--verbose` for details) to verify environment and configuration.

Memory bank navigation:
- `~/.deebo/memory-bank/<projectId>/sessions/<sessionId>/logs` for JSONL logs.
- `~/.deebo/memory-bank/<projectId>/sessions/<sessionId>/reports` for scenario outcomes.
- `~/.deebo/memory-bank/<projectId>/activeContext.md` and `progress.md` for running notes and history.

---

### Extensibility

- Adding tools:
  - Extend `config/tools.json` with new tool definitions (e.g., code analyzers, test runners). Use placeholders for paths; ensure env passed is sufficient.
  - Update prompts to teach tools, or detect new `<use_mcp_tool>` tags at runtime.
- New providers/models:
  - `callLlm` already supports OpenAI/OpenRouter/Anthropic/Gemini; adapt to additional providers by extending the switch.
- Custom orchestration:
  - Adjust OODA loop thresholds, timeouts, and branching strategy.
  - Modify `check` pulse rendering or add new computed metrics.

---

### Key Files Map

- Server and agents
  - `src/index.ts`: MCP server, tools `start|check|cancel|add_observation`, process registry, path discovery, memory root.
  - `src/mother-agent.ts`: Mother OODA loop, tool/hypothesis precedence, scenario spawning, memory updates, cancellation.
  - `src/scenario-agent.ts`: Hypothesis-focused investigation loop and report writing.
- Utilities
  - `src/util/mcp.ts`: Connects `git-mcp` and `desktop-commander`, placeholder/env handling, Windows shim.
  - `src/util/logger.ts`: JSONL logging; `src/util/reports.ts`: report writer; `src/util/observations.ts`: observation read/write.
  - `src/util/membank.ts`: appenders for `activeContext.md` and `progress.md`.
  - `src/util/branch-manager.ts`: creates `debug-...` branches via `simple-git`.
  - `src/util/agent-utils.ts`: prompts and provider-agnostic `callLlm`.
  - `src/util/sanitize.ts`: stable project ID hashing from `repoPath`.
- Configuration and assets
  - `config/tools.json`: external tool launch definitions with placeholders.
  - `tsconfig.json`, `package.json`: build/CLI exposure (`bin.deebo`, main).
- CLI packages
  - `packages/deebo-setup`: installation/configuration wizard.
  - `packages/deebo-doctor`: environment/configuration diagnostics.

---

### Notes and Caveats

- Ensure `MOTHER_MODEL` and `SCENARIO_MODEL` are set; otherwise the MCP server will throw on startup.
- `OPENAI_BASE_URL` must be provided for `openai` host usage.
- The first `check` often needs ~30 seconds after `start` for meaningful status.
- Scenario timeouts default to 5 minutes; mother agent max runtime defaults to 60 minutes.
- Report files are JSON-formatted; the scenario prints report text to stdout as well for mother ingestion.

This document summarizes how Deebo is structured and how its agents coordinate investigations via MCP tools, Git isolation, and memory bank logs to accelerate debugging workflows.