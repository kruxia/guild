# Claude Code and the Claude Agent SDK as a substrate for a staged dev harness (state as of 2026-10-03)

Version anchors at time of research:
- Claude Code CLI: `latest`/`next` = **2.1.288** (published 2026-10-02), `stable` = **2.1.285**. 2.0.0 shipped 2025-09-29 and 2.1.0 shipped 2026-01-07. Source: [npm registry @anthropic-ai/claude-code](https://registry.npmjs.org/@anthropic-ai/claude-code); [CHANGELOG](https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md). The CHANGELOG has no dates, so dates come from npm publish times.
- TypeScript Agent SDK `@anthropic-ai/claude-agent-sdk`: **0.3.288** (2026-10-02). Source: [npm registry](https://registry.npmjs.org/@anthropic-ai/claude-agent-sdk)
- Python Agent SDK `claude-agent-sdk`: **0.2.163** (2026-09-30). Source: [PyPI](https://pypi.org/pypi/claude-agent-sdk/json)
- The Claude Code SDK was renamed the Claude Agent SDK. The Python package moved from `claude-code-sdk` to `claude-agent-sdk`. Source: [Migration guide](https://code.claude.com/docs/en/agent-sdk/migration-guide)
- Caveat: Claude Code ships almost daily. Many behaviors below carry "requires v2.1.2xx" notes. Several features are labeled research preview, experimental or beta (routines, agent view, channels, agent teams, projects, sandbox runtime), and may change.

## 1. Programmatic invocation: headless CLI (`claude -p`) and the Agent SDK

### Takeaway
Claude Code can run as a fully scriptable, non-interactive job. Headless runs support JSON and stream-JSON output, JSON-Schema-validated structured output, session IDs with resume and fork, turn and dollar caps, fine-grained tool allow and deny rules, permission modes, system-prompt injection, and per-run cost estimates. The Python and TypeScript Agent SDKs wrap the same CLI binary as a subprocess. They add in-process custom tools and hooks, a `canUseTool` approval callback, pluggable transcript storage (`SessionStore`), and a "defer" mechanism that lets a process exit at a tool call and resume days later. Together these are the key primitives for "run a stage unattended, then pause for a human."

### Cited Findings

**Headless CLI basics**
- `claude -p` (`--print`) runs non-interactively. It exits 0 on success and non-zero on failure. Failures inside the run, such as missing auth, are printed as the result on stdout. — [Run Claude Code programmatically](https://code.claude.com/docs/en/headless)
- `--output-format` takes `text`, `json` or `stream-json`. JSON output includes `result`, `session_id`, metadata, `total_cost_usd` and a per-model cost breakdown. With `--continue` or `--resume`, cost totals include earlier runs' spend. All cost figures are client-side estimates. — [headless](https://code.claude.com/docs/en/headless)
- `--json-schema '<schema>'` with `--output-format json` returns schema-conforming output in the `structured_output` field. An invalid schema exits with an error; before v2.1.205 an invalid schema was silently ignored. — [headless](https://code.claude.com/docs/en/headless); [CLI reference](https://code.claude.com/docs/en/cli-reference)
- `stream-json` (with `--verbose`, and optionally `--include-partial-messages`) emits newline-delimited events. The last line is a `result` message. Subagent messages carry `parent_tool_use_id`. `--forward-subagent-text` (v2.1.211+) forwards subagent text and thinking blocks so subagent transcripts can be rebuilt. — [headless](https://code.claude.com/docs/en/headless)
- The stream emits `system/api_retry` events (attempt, max_retries, retry_delay_ms, error category such as `rate_limit`, `overloaded`, `billing_error`). It also emits `system/init` with model, tools, `mcp_servers`, `mcp_server_errors`, `plugins`, `plugin_errors` and a `capabilities` array for feature detection. This lets CI fail when a plugin or MCP server didn't load. — [headless](https://code.claude.com/docs/en/headless)
- `--input-format stream-json` allows multi-turn input over stdin. `--replay-user-messages` echoes user messages back. — [CLI reference](https://code.claude.com/docs/en/cli-reference)
- **Session controls**:
  - `--continue`/`-c` picks up the most recent conversation in the directory.
  - `--resume`/`-r <id|name|path-to-.jsonl>` picks up a specific session.
  - `--session-id <uuid>` presets the ID.
  - `--fork-session` branches when resuming.
  - `--no-session-persistence` keeps the session off disk.
  - Since v2.1.223, a session ID resolves from any project directory on the same machine.
  — [CLI reference](https://code.claude.com/docs/en/cli-reference); [headless](https://code.claude.com/docs/en/headless)
- **Caps**:
  - `--max-turns N` limits agentic turns (print mode). It exits with an error at the limit. There is no limit by default.
  - `--max-budget-usd X` stops at a dollar cap (print mode). Subagent spend counts toward the cap.
  - When the cap is reached, spawning another subagent fails with `Budget limit reached` and running background subagents are stopped (v2.1.217+).
  - Totals restored by `--resume` don't count toward the cap.
  — [CLI reference](https://code.claude.com/docs/en/cli-reference)
- **Tool and permission controls**:
  - `--allowedTools` auto-approves tools using rule syntax such as `Bash(git diff *)`.
  - `--disallowedTools` denies or removes tools.
  - `--tools` restricts the built-in set.
  - `--permission-mode` accepts `default`/`manual`, `acceptEdits`, `plan`, `auto`, `dontAsk` or `bypassPermissions`.
  - `--dangerously-skip-permissions` is equivalent to `bypassPermissions`.
  - `--permission-prompt-tool <mcp tool>` delegates permission prompts to an MCP tool.
  - `--permission-prompts none` (v2.1.259+) denies anything that would prompt and tells Claude not to retry. Denials appear as `permission_denied` events and in `permission_denials` on the result.
  — [CLI reference](https://code.claude.com/docs/en/cli-reference); [headless](https://code.claude.com/docs/en/headless)
- `dontAsk` mode denies every call that would otherwise prompt. It is described as "useful for locked-down CI runs." `auto` mode uses a classifier model to review actions instead of a human. — [headless](https://code.claude.com/docs/en/headless); [Permission modes](https://code.claude.com/docs/en/permission-modes)
- **System prompt flags**:
  - `--append-system-prompt` and `--append-system-prompt-file` add to the default prompt.
  - `--system-prompt` and `--system-prompt-file` replace it.
  - `--append-subagent-system-prompt` (v2.1.205+, `-p` only) appends to every subagent's prompt.
  - A line containing `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` splits the cached static part from per-run context (v2.1.275+).
  - The system prompt is recorded on a conversation's first request and reused on resume until compaction.
  — [CLI reference](https://code.claude.com/docs/en/cli-reference)
- `--bare` skips auto-discovery of hooks, skills, commands, subagents, plugins, MCP servers, auto memory and CLAUDE.md. It uses API-key auth only, and you pass context explicitly (`--settings`, `--mcp-config`, `--agents`, `--plugin-dir`). Docs: "`--bare` is the recommended mode for scripted and SDK calls, and will become the default for `-p` in a future release." — [headless](https://code.claude.com/docs/en/headless)
- **Security note for CI**: without `--bare`, a `-p` session runs the project's `.claude/settings.json` hooks and connects `.mcp.json` servers "even in a folder you've never trusted". There is no trust dialog. — [headless](https://code.claude.com/docs/en/headless)
- `--restricted` (v2.1.248+) is for evaluation harnesses on shared machines. It removes command-running tools and WebFetch, confines file tools to the working directories, loads only managed settings and `--settings`, and refuses `bypassPermissions`. — [CLI reference](https://code.claude.com/docs/en/cli-reference)
- **Process lifecycle**:
  - A background Bash task started during `-p` is killed about 5 seconds after the result.
  - `-p` stays open while background subagents or workflows run, up to a 10-minute idle ceiling (`CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS`).
  - SIGTERM exits with code 143, leaves the turn unfinished, kills Bash process trees and runs `SessionEnd` hooks.
  - `CLAUDE_CODE_RESUME_INTERRUPTED_TURN=1` makes resume continue an interrupted turn.
  — [headless](https://code.claude.com/docs/en/headless)
- Skills and custom commands work in `-p` when you put `/skill-name` in the prompt. Terminal-only commands don't. — [headless](https://code.claude.com/docs/en/headless)
- `claude setup-token` generates a long-lived OAuth token for CI and scripts (requires a subscription). `claude auth status` prints JSON. — [CLI reference](https://code.claude.com/docs/en/cli-reference)

**Agent SDK (Python and TypeScript)**
- The SDK "runs the Claude Code binary" as a library, with built-in tools, hooks, subagents, MCP, permissions, sessions, skills, commands, memory and plugins. For other languages, "run the CLI as a subprocess with the `-p` flag and `--output-format json`." — [Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview)
- Each `query()` spawns a separate `claude` CLI subprocess talking over stdio. "One agent session maps to one subprocess." — [Hosting the Agent SDK](https://code.claude.com/docs/en/agent-sdk/hosting)
- **Licensing and auth**: governed by Anthropic Commercial Terms. "Unless previously approved, Anthropic does not allow third party developers to offer claude.ai login or rate limits for their products, including agents built on the Claude Agent SDK. Use the API key authentication methods." — [Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview)
- **Session APIs**:
  - Python `ClaudeSDKClient` keeps a session across turns.
  - Session options are `resume`, `continue`/`continue_conversation`, `fork_session`/`forkSession` and `persistSession: false` (TS only).
  - `listSessions`/`getSessionMessages`/`getSessionInfo`/`renameSession`/`tagSession` (and Python equivalents) operate on sessions.
  - The TS V2 session API (`createSession()`) was removed in TS SDK 0.3.142.
  — [Work with sessions](https://code.claude.com/docs/en/agent-sdk/sessions)
- Sessions persist the conversation, not the filesystem. Transcripts live at `~/.claude/projects/<encoded-cwd>/*.jsonl` and are local to the machine. To resume on another host, use a `SessionStore` adapter, move the JSONL file, or "don't rely on session resume" and pass application state into a fresh session. Docs: the last option is "often more robust." — [sessions](https://code.claude.com/docs/en/agent-sdk/sessions)
- **`SessionStore`**:
  - The adapter has required `append`/`load` and optional `listSessions`, `listSessionSummaries`, `delete` and `listSubkeys`.
  - It mirrors transcripts (including subagent transcripts under `subpath`) to your backend.
  - Reference adapters for S3, Redis and **Postgres** live in both SDK repos. A conformance test suite ships too.
  - Mirror writes are best-effort: up to 3 attempts, then a `mirror_error` system message and the batch is dropped. Dedupe by `entry.uuid`.
  - It is incompatible with file checkpointing and `persistSession:false`.
  - The `projectKey` derives from cwd, so you must resume from a matching cwd.
  - It stores transcripts only, not CLAUDE.md or working-directory artifacts.
  — [Persist sessions to external storage](https://code.claude.com/docs/en/agent-sdk/session-storage); [Hosting](https://code.claude.com/docs/en/agent-sdk/hosting)
- **Human approval callback**:
  - `canUseTool` (TS) / `can_use_tool` (Python) receives `(toolName, input, {signal, suggestions})`.
  - It returns allow (optionally with `updatedInput` and `updatedPermissions`) or deny (with `message`).
  - It also handles `AskUserQuestion`: 1–4 questions with 2–4 options each, answered via `updatedInput.answers`.
  - It never fires for auto-approved tools; use a `PreToolUse` hook for logic that must see every call.
  - `AskUserQuestion` is not available in subagents.
  — [Handle approvals and user input](https://code.claude.com/docs/en/agent-sdk/user-input)
- "The callback can stay pending indefinitely… If a user might take longer to respond than your process can reasonably stay running, register a `PreToolUse` hook that returns the `defer` decision… so the process can exit and resume later from the persisted session." — [user-input](https://code.claude.com/docs/en/agent-sdk/user-input)
- **The `defer` decision**:
  1. A `PreToolUse` hook returns `permissionDecision: "defer"`. This is honored only in `-p`/SDK mode.
  2. The tool doesn't execute. The process exits with `stop_reason: "tool_deferred"` and a `deferred_tool_use` payload (`id`, `name`, `input`).
  3. The caller later runs `claude -p --resume <session-id>`. The same tool call fires `PreToolUse` again.
  4. The hook returns `allow` with the answer in `updatedInput`, or `deny`.

  Constraints:
  - "There is no timeout or retry limit". The session survives until `cleanupPeriodDays` (30 days default) deletes it.
  - Defer works only when Claude makes a single tool call in the turn. Parallel calls make defer get ignored.
  - If the tool is missing on resume, the process exits with `tool_deferred_unavailable`.
  - Resuming with `-p` doesn't restore the stored permission mode; pass it again.

  — [Hooks reference: Defer a tool call for later](https://code.claude.com/docs/en/hooks#defer-a-tool-call-for-later)
- **Custom tools**: define functions with `tool()` and wrap them in `createSdkMcpServer`/`create_sdk_mcp_server`, an MCP server that "runs in-process inside your application", passed via `mcpServers`. — [Give Claude custom tools](https://code.claude.com/docs/en/agent-sdk/custom-tools)
- **Structured output in the SDK**: the option is `outputFormat: {type: "json_schema", schema}` (TS) / `output_format` (Python). The failure subtype is `error_max_structured_output_retries`. — [Structured outputs](https://code.claude.com/docs/en/agent-sdk/structured-outputs)
- **SDK hooks** are callbacks for essentially the same events as CLI hooks. `PreToolUse`, `PostToolUse`, `Stop`, `SubagentStart`/`SubagentStop`, `PreCompact`, `PermissionRequest` and `Notification` are available in both Python and TS. Others such as `SessionStart`/`SessionEnd`, `TaskCreated`, `TeammateIdle`, `WorktreeCreate` and `PostToolBatch` are TS-only. — [SDK hooks](https://code.claude.com/docs/en/agent-sdk/hooks)
- **Subagents in the SDK**: `AgentDefinition` fields include `maxTurns` (partial output, resumable).
  - Depth cap `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`: default 3.
  - Concurrency cap `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`: default 20. At the cap the spawn fails with `Concurrent subagent limit reached`.
  - Spend cap `maxBudgetUsd`/`max_budget_usd`: ends the query with the `error_max_budget_usd` subtype.
  - These limits require TS SDK ≥0.3.219 / Python ≥0.2.127.
  — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- **Cost tracking**:
  - `total_cost_usd` and per-model `modelUsage` include subagents; `usage` excludes them.
  - Per-step `output_tokens` is a placeholder.
  - Since v2.1.277, resumed sessions restore and accumulate prior totals.
  - After a crash, an `error_during_execution` result may carry zeroed costs.
  - Docs: the fields are "client-side estimates, not authoritative billing data… Do not bill end users or trigger financial decisions from these fields". Use the Usage and Cost API instead.
  — [Track cost and usage](https://code.claude.com/docs/en/agent-sdk/cost-tracking)
- **File checkpointing in the SDK**: `enableFileCheckpointing` plus `rewindFiles()`/`rewind_files()` restores files. It doesn't rewind the conversation. — [SDK file checkpointing](https://code.claude.com/docs/en/agent-sdk/file-checkpointing)
- **Hosting patterns**: ephemeral, long-running (TS `streamInput()`/`startup()`/`prewarm()`, Python `ClaudeSDKClient`), and hybrid (ephemeral containers that hydrate from a `SessionStore`; "the store is required for this pattern"). Suggested start: 1 GiB RAM, 5 GiB disk and 1 CPU per agent. — [Hosting](https://code.claude.com/docs/en/agent-sdk/hosting)
- **Known limitations**: "No top-level session timeout" (use `maxTurns`). Memory grows over long sessions. Wide subagent fan-outs hit rate limits. There is "no per-subagent wall-clock deadline" (`CLAUDE_ASYNC_AGENT_STALL_TIMEOUT_MS` is only a stall watchdog). — [Hosting](https://code.claude.com/docs/en/agent-sdk/hosting)
- Anthropic separately offers **Managed Agents**, a hosted harness configured through the Claude API with sessions in Anthropic-managed or self-hosted sandboxes. — [Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview). Its engineering post (2026-04-08) describes an append-only session log outside the harness, with `wake(sessionId)`/`getSession(id)` crash recovery. — [Scaling Managed Agents](https://www.anthropic.com/engineering/managed-agents)

### Inferences
- An external orchestrator can drive each stage as `claude -p --bare --output-format json --json-schema <stage-output-schema> --max-turns N --max-budget-usd X --permission-mode dontAsk|auto --allowedTools ...`. It persists `session_id`, `structured_output`, `total_cost_usd` and `permission_denials` into its own database. Feedback after human review becomes a new `--resume <id>` (or `--fork-session`) turn.
- PreToolUse `defer` plus `--resume` is the native "pause for human, exit process, resume later" primitive. It is limited to single-tool-call turns and 30-day transcript retention. A durable orchestrator must own the wait state and the eventual answer.
- Because session transcripts are host-local JSONL, a multi-host orchestrator needs either a `SessionStore` (Postgres reference adapter exists) or the "pass state into a fresh session" pattern. Anthropic itself calls the latter more robust.
- Cost data is advisory. Cross-session cost governance (per stage, per workflow, per tenant) must be computed by the orchestrator, and authoritative reconciliation must come from Anthropic's Usage and Cost API.

### Gaps
- I did not verify the exact current shape of every field in the TS `SDKResultMessage` (for example `num_turns`, `duration_ms`). Only fields mentioned in the docs pages read above are cited.
- I couldn't fetch GitHub repo pages directly (session GitHub access was restricted). SDK CHANGELOG contents are not summarized beyond what the docs cite.

## 2. Extension points (CLAUDE.md, subagents, skills, plugins, commands, hooks, output styles, MCP, settings)

### Takeaway
Claude Code has a rich declarative extension surface. All of it is file-based and checkable into a repo, so it can encode a stage-specific "agent persona and policy" per pipeline stage:
- CLAUDE.md/AGENTS.md memory and `.claude/rules/`
- subagents in `.claude/agents/*.md`
- skills in `.claude/skills/*/SKILL.md`
- saved workflows in `.claude/workflows/*.js`
- plugins and marketplaces bundling all of the above
- about 34 hook events with five handler types
- output styles
- MCP servers and in-process SDK tools
- layered `settings.json`, including org-managed settings

### Cited Findings
- **Memory**: CLAUDE.md or AGENTS.md files give persistent instructions. `.claude/rules/` scopes rules to file types. "Auto memory" is notes Claude writes itself from corrections. Subagents can keep their own memory. — [How Claude remembers your project](https://code.claude.com/docs/en/memory)
- **Subagents** are Markdown files with frontmatter.
  - Fields: `name`, `description`, `tools`, `disallowedTools`, `model`, `permissionMode`, `maxTurns`, `skills` (preloaded), `mcpServers`, `hooks`, `memory` (`user`/`project`/`local`), `background`, `omitClaudeMd`, `effort`, `isolation: worktree`, `initialPrompt` and `experimental.cacheTtl`.
  - They can also be defined via `--agents` JSON or SDK `AgentDefinition`.
  - `--agent <name>` runs a subagent as the main session agent.
  — [Create custom subagents](https://code.claude.com/docs/en/sub-agents); [CLI reference](https://code.claude.com/docs/en/cli-reference)
- **Skills** are `SKILL.md` files with frontmatter.
  - Fields: `name`, `description`, `arguments`, `disable-model-invocation`, `user-invocable`, `allowed-tools`, `disallowed-tools`, `model`, `effort`, `context: fork` (runs in a forked subagent), `agent`, `background`, `hooks`, `paths` (glob activation), `shell` and `metadata`.
  - Skills double as slash commands.
  — [Extend Claude with skills](https://code.claude.com/docs/en/skills)
- **Plugins** are directories with `.claude-plugin/plugin.json` bundling skills, agents, hooks, a JS "hooks module" (a "mod"), MCP servers and workflows. They are distributed via marketplaces, loaded per session with `--plugin-dir`/`--plugin-url`, and managed with `claude plugin install name@marketplace`. — [Plugins overview](https://code.claude.com/docs/en/plugins/overview); [CLI reference](https://code.claude.com/docs/en/cli-reference); [Workflows](https://code.claude.com/docs/en/workflows)
- **Hook events**:
  - `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`
  - `PermissionRequest`, `PermissionDenied`
  - `UserPromptSubmit`, `UserPromptExpansion`
  - `Stop`, `StopFailure`, `SubagentStart`, `SubagentStop`
  - `SessionStart`, `SessionEnd`, `Setup`
  - `PreCompact`, `PostCompact`
  - `Notification`, `MessageDisplay`
  - `TaskCreated`, `TaskCompleted`, `TeammateIdle`
  - `WorktreeCreate`, `WorktreeRemove`
  - `ConfigChange`, `InstructionsLoaded`, `FileChanged`, `CwdChanged`, `DirectoryAdded`
  - `Elicitation`, `ElicitationResult`
  - `PreModelSwitch`, `PostModelSwitch`

  — [Hooks reference](https://code.claude.com/docs/en/hooks). The event table was obtained via a summarizing fetch; the defer and PreToolUse decision sections were verified against the raw page.
- **Hook handler types**: `command`, `http` (POST JSON to a URL), `mcp_tool`, `prompt` (single-turn model judgment) and `agent` (a subagent with Read/Grep/Glob; experimental). Hooks can run `async: true`, or `asyncRewake: true` to wake Claude on exit code 2. — [Hooks reference](https://code.claude.com/docs/en/hooks)
- **Hook decision control**:
  - Exit code 2 blocks.
  - `decision: "block"` works on `Stop`, `SubagentStop`, `PostToolUse`, `UserPromptSubmit` and others.
  - `PreToolUse` `permissionDecision` takes allow/deny/ask/defer, with precedence deny > defer > ask > allow, plus `updatedInput` and `additionalContext`.
  - `continue: false` with `stopReason` halts Claude.
  - Blocking a `Stop` hook forces Claude to keep working.
  — [Hooks reference](https://code.claude.com/docs/en/hooks)
- `Notification` hook matchers include `permission_prompt`, `idle_prompt`, `agent_needs_input` and `agent_completed`. These are useful for paging a human when an agent blocks. — [Hooks reference](https://code.claude.com/docs/en/hooks); [Agent view](https://code.claude.com/docs/en/agent-view)
- **Output styles** set role, tone and format for every response in a session. There are four built-ins besides the default, plus custom styles. `/output-style <style>` works in `-p` since v2.1.269. — [Output styles](https://code.claude.com/docs/en/output-styles); [headless](https://code.claude.com/docs/en/headless)
- **MCP**: `--mcp-config` and `--strict-mcp-config` control servers. `-p` waits for pending servers up to `MCP_TIMEOUT` (30s). claude.ai connectors are also usable. — [CLI reference](https://code.claude.com/docs/en/cli-reference)
- **Settings**:
  - `--settings <file|json>` overrides file settings.
  - `--setting-sources user,project,local` selects which sources load.
  - Managed (org) settings can force policy, for example `disableWorkflows`.
  — [CLI reference](https://code.claude.com/docs/en/cli-reference); [Workflows](https://code.claude.com/docs/en/workflows)
- **OpenTelemetry**: `OTEL_METRICS_EXPORTER`/`OTEL_LOGS_EXPORTER` (otlp/prometheus/console) and an OTLP endpoint export usage metrics and logs. — [Monitoring](https://code.claude.com/docs/en/monitoring-usage)

### Inferences
- Each harness stage can be packaged as a subagent or skill with its own tools, model, permission mode, `maxTurns`, hooks and preloaded skills. A plugin can version and ship the entire stage library to every repo or CI runner.
- `Stop`/`SubagentStop` hooks of type `prompt`/`agent`/`command` act as in-session quality gates (for example "tests must pass before stopping"). `http` hooks can report lifecycle events to an external orchestrator. They are not a durable control plane.

### Gaps
- I did not read the full plugin manifest or marketplace reference. Details such as plugin versioning and dependency resolution are not captured beyond "unsatisfied dependency versions" appearing in `plugin_errors` ([headless](https://code.claude.com/docs/en/headless)).

## 3. Built-in orchestration features and where durability ends

### Takeaway
As of October 2026, Claude Code includes substantial built-in orchestration:
- subagents with nesting and worktree isolation
- **dynamic workflows** (JS scripts orchestrating up to 1,000 agents per run)
- **agent view / background sessions** under a local supervisor daemon
- experimental **agent teams**
- `/goal` and `/loop` (session-scoped scheduling)
- **cross-session messaging**
- **channels** for pushing events into a session
- **cloud sessions** and **Projects** (public beta) that coordinate parallel cloud threads with a "Waiting on you" pane
- **routines** (cloud cron, API and GitHub-event triggers; research preview)
- GitHub Actions
- checkpoints and rewind

Durability is explicitly bounded:
- Workflows have "no mid-run user input" and resume only within a session.
- Background sessions are local and stop on machine shutdown.
- `/loop` tasks expire after 7 days.
- Agent teams can't resume teammates.
- Routines are fire-and-forget sessions with hourly caps.
- Cost data is estimated per session.

There is no first-class durable, multi-day, multi-stage pipeline with human approval gates and cross-session budget governance. That remains the orchestrator's job.

### Cited Findings

**Subagents (Agent/Task tool)**
- Subagents nest up to 3 layers by default, with up to 20 concurrent by default. They can run foreground or background, and `isolation: worktree` gives each its own git worktree. — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents); [Create custom subagents](https://code.claude.com/docs/en/sub-agents)
- Anthropic's own comparison lists five parallel approaches: subagents, agent view (research preview), agent teams (experimental, disabled by default), projects (public beta on Pro/Max) and dynamic workflows. `/batch` splits a change into 5–30 worktree-isolated subagents. — [Run agents in parallel](https://code.claude.com/docs/en/agents)

**Dynamic workflows**
- A workflow is a JavaScript script, usually Claude-written, run by a background runtime. It uses `agent()` (with optional JSON `schema`), `pipeline()`, `parallel()`, `phase()`, `log()` and an `args` global. — [Dynamic workflows](https://code.claude.com/docs/en/workflows)
- Workflows can be saved to `.claude/workflows/` or `~/.claude/workflows/` and run as `/<name>`. They can be distributed in plugins. They are triggered by the `ultracode` keyword, by `/effort ultracode`, or by asking. — [Workflows](https://code.claude.com/docs/en/workflows)
- **Limits**:
  - "No mid-run user input — A run pauses on its own only for agent permission prompts and a usage-limit wait. For sign-off between stages, run each stage as its own workflow."
  - No filesystem or shell access from the script itself, and no `import()`.
  - 16 concurrent agents by default (configurable 1–256 via `CLAUDE_CODE_WORKFLOW_MAX_CONCURRENT_AGENTS`).
  - At most 4,096 items per `parallel`/`pipeline` call and 1,000 agents per run.
  — [Workflows](https://code.claude.com/docs/en/workflows)
- **Determinism and resume**:
  - `Date.now()`, `Math.random()` and `new Date()` throw inside scripts so replays repeat the same `agent()` calls.
  - Resume replays in start order. Completed agents return cached results. The first changed or failed agent and every agent after it re-run.
  - Resume works "within the same Claude Code session". It carries over to a backgrounded session.
  - Saved results let `claude --resume` sessions relaunch the run.
  - Cloud sessions keep results across VM reclamation.
  — [Workflows](https://code.claude.com/docs/en/workflows)
- **Cost guardrails**: a `Large workflow` warning appears above 25 agents or 1.5M projected tokens. It is advisory only. A size guideline (`workflowSizeGuideline`: small <5, medium <10, large <50 agents) is advice, not a cap. Usage-limit pause/auto-resume works only in interactive subscription sessions, not `-p`/SDK/background. — [Workflows](https://code.claude.com/docs/en/workflows)
- In `-p`/SDK there is no approval prompt. The `Workflow` tool goes through normal permissions: allow `Workflow` or `Workflow(<name>)`, auto mode, a PreToolUse hook or `canUseTool`. The `ultracode` keyword doesn't trigger from `-p`, scheduled prompts or webhook payloads. — [Workflows](https://code.claude.com/docs/en/workflows)

**Agent view / background sessions (research preview)**
- `claude --bg "<task>"` starts a background session under a supervisor daemon.
  - Management commands: `claude agents [--json]`, `claude attach`, `claude logs`, `claude stop`, `claude respawn [--all]`, `claude rm` and `claude daemon status|stop`.
  - `claude agents --json` reports `state` as working, blocked, done, failed or stopped, with `waitingFor`.
  — [Agent view](https://code.claude.com/docs/en/agent-view); [CLI reference](https://code.claude.com/docs/en/cli-reference)
- **Durability**:
  - The supervisor restarts crashed session processes.
  - Sessions survive auto-updates, supervisor restarts and sleep. "Shutting down still stops running sessions."
  - After a reboot, sessions under 48h show as `failed` (attach to resume) and older ones as `stopped`.
  - Idle unattached sessions stop after about 1 hour unless pinned.
  - Limitation: "Sessions are local… stop if the machine shuts down."
  — [Agent view](https://code.claude.com/docs/en/agent-view)
- Dispatched sessions move into `.claude/worktrees/<id>` before their first edit. `claude --bg` from the shell defaults to the directory's settings, falling back to `plan` mode. `--bg` cannot combine with `-p`. — [Agent view](https://code.claude.com/docs/en/agent-view); [headless](https://code.claude.com/docs/en/headless)

**Agent teams (experimental)**
- Agent teams are enabled with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. A lead supervises peers with a shared task list and messaging.
- Limitations:
  - "No session resumption with in-process teammates"
  - task status can lag
  - one team per session
  - no nested teams
  - the lead is fixed
- — [Agent teams](https://code.claude.com/docs/en/agent-teams)

**Session continuation and scheduling**
- `/goal` sets a completion condition. After each turn an evaluator model checks it, and Claude keeps going until the condition is met or judged impossible. It is session-scoped. Stop hooks are the settings-file equivalent. — [Keep Claude working toward a goal](https://code.claude.com/docs/en/goal)
- `/loop` and in-session scheduled tasks (CronCreate) are session-scoped. Minimum interval is 1 minute. Recurring tasks auto-expire after 7 days. They are restored on `--resume`. Docs point to routines, Desktop scheduled tasks or GitHub Actions for "scheduling that survives independently of any session". — [Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks)
- Desktop local scheduled tasks fire only while the app is open and the computer is awake. — [Desktop scheduled tasks](https://code.claude.com/docs/en/desktop-scheduled-tasks)

**Routines (research preview, claude.ai subscription plans)**
- A routine is a saved prompt plus repos, environment and connectors.
  - It runs as a full cloud session with "no permission-mode picker". It runs shell commands and connector writes without approval.
  - Triggers are schedule (minimum 1 hour, or one-off), API (`POST https://api.anthropic.com/v1/claude_code/routines/<trig_id>/fire` with a bearer token and beta header `experimental-cc-routine-2026-04-01`; returns `claude_code_session_id` and URL) and GitHub events (pull_request, release; with filters).
  - Fire `text` arrives wrapped as untrusted `<routine-fire-payload>`.
  — [Routines](https://code.claude.com/docs/en/routines)
- **Routine limits**:
  - 100 scheduled runs per hour per account.
  - 30 Run-now/API fires per hour per routine.
  - 100 API fires per hour per account.
  - GitHub events are capped per routine and account; excess events are dropped.
  - Runs draw subscription usage.
  - "A green status… does not mean the task in your prompt succeeded."
  - Each GitHub event creates a new independent session; there is no session reuse.
  - The `/fire` endpoint is "available to claude.ai users only and is not part of the Claude Platform API surface."
  — [Routines](https://code.claude.com/docs/en/routines)

**Cloud sessions and Projects**
- Cloud sessions run in Anthropic-managed VMs or self-hosted environments (`claude self-hosted-runner`, `--environment ccpool_...`).
  - They are created via `claude --cloud "<task>"`.
  - `claude -p "msg" --cloud <session-id>` queues a follow-up and exits; `--output-format json` returns `{ok, session_id, url}`.
  - `--teleport` pulls a cloud session local.
  - VMs are reclaimed after inactivity. Reopening restores the conversation but not background subagents or shell jobs.
  - Cloud sessions require a claude.ai subscription, not API keys or Bedrock/Vertex/Foundry.
  — [Use Claude Code in the cloud](https://code.claude.com/docs/en/claude-code-on-the-web); [CLI reference](https://code.claude.com/docs/en/cli-reference)
- Auto-fix PRs: Claude subscribes to PR CI failures and review comments and pushes fixes. It can't react to base-branch merge conflicts (no webhook). — [Cloud sessions](https://code.claude.com/docs/en/claude-code-on-the-web)
- **Projects** (public beta, Pro/Max only, not Team/Enterprise yet) are one conversation where Claude spawns parallel "threads", mostly cloud sessions.
  - An Overview pane shows "Ready for review", "Waiting on you" and similar states.
  - Project memory and instructions are shared across threads.
  - Threads run in auto mode, and a permission prompt waits inside the thread.
  - Thread limits stated in conversation "aren't enforced settings… not a hard cap".
  - Usage hits plan limits, and threads wait for limit reset.
  — [Projects](https://code.claude.com/docs/en/claude-projects)
- **Cross-session messaging** (v2.1.224+): `ListAgents`/`SendMessage` deliver text between your sessions on the same machine, other machines or the cloud. It moves text only, not history. — [Cross-session messaging](https://code.claude.com/docs/en/cross-session-messaging)
- **Channels** (research preview): MCP servers that push events (Telegram, Discord, iMessage, custom) into a running session. "Events only arrive while the session is open." — [Channels](https://code.claude.com/docs/en/channels)
- **Remote Control**: `claude remote-control` / `--rc` lets you steer a local session from claude.ai or mobile. — [CLI reference](https://code.claude.com/docs/en/cli-reference)

**GitHub Actions**
- `anthropics/claude-code-action@v1` is built on the Agent SDK.
  - Interactive mode responds to `@claude` in issues and PRs. Automation mode runs on any event, including cron, when given a `prompt`.
  - `claude_args` passes any CLI flag (`--max-turns`, `--model`, `--allowedTools`).
  - Auth options: API key, `CLAUDE_CODE_OAUTH_TOKEN`, OIDC workload identity federation, Bedrock/Vertex/Foundry.
  - Write-access and human-actor checks guard triggering.
  - Cost guidance: `--max-turns`, workflow timeouts and GitHub concurrency controls.
  — [Claude Code GitHub Actions](https://code.claude.com/docs/en/github-actions)

**Plan mode, todos and checkpoints**
- Plan mode is read-only exploration, with classifier-approved commands where auto mode is available. It pairs with "plan locally, execute in the cloud". — [Permission modes](https://code.claude.com/docs/en/permission-modes); [Cloud sessions](https://code.claude.com/docs/en/claude-code-on-the-web)
- Task tracking uses the TaskCreate/TaskUpdate tools, with `TaskCreated`/`TaskCompleted` hooks. — [Hooks reference](https://code.claude.com/docs/en/hooks); [CLI reference](https://code.claude.com/docs/en/cli-reference)
- **Checkpoints and `/rewind`**: file edits are tracked automatically and saved with the conversation. Snapshots are deleted about 30 days later. Bash-made and external changes are not tracked. "Not a replacement for version control." — [Checkpointing](https://code.claude.com/docs/en/checkpointing)

**Cost governance**
- Org-level controls are spend limits (Team/Enterprise admin, Console workspace spend limits), usage credits, analytics and the Claude Code Analytics API. Per-session caps are `--max-budget-usd`/`maxBudgetUsd`. — [Manage costs](https://code.claude.com/docs/en/costs); [CLI reference](https://code.claude.com/docs/en/cli-reference)

### Inferences
- **What Claude Code itself can orchestrate**:
  - intra-session fan-out (subagents, workflows with schema-typed agent outputs and adversarial verification)
  - local fleets (agent view)
  - cloud fleets with a human "waiting on you" inbox (Projects)
  - trigger-to-session automation (routines, GitHub Actions)
- **What it lacks for a staged, human-gated harness**:
  1. A durable state machine spanning stages and days. Workflows forbid mid-run human input and resume only per session. Routines create independent sessions per trigger.
  2. Crash recovery independent of one machine. Background sessions die on shutdown; host-local transcripts need a `SessionStore`.
  3. Explicit human approval gates as first-class objects. The available proxies are permission prompts, `AskUserQuestion`, `defer`, the Projects "Waiting on you" view, and PR review.
  4. Cross-session or cross-workflow budget enforcement. Caps are per `-p` invocation or query and estimated client-side; Projects thread limits are "not a hard cap".
  5. An auditable artifact lineage between stages beyond git branches and transcripts.
- Anthropic's own docs suggest the staged pattern directly: "For sign-off between stages, run each stage as its own workflow." This maps to an external orchestrator invoking one Claude Code job per stage and persisting gates between them.
- Routines, Projects and cloud sessions are tied to claude.ai subscription auth, while the SDK forbids third-party products from offering claude.ai login. An external product-grade orchestrator would likely use the CLI/SDK with API keys (or Bedrock/Vertex/Foundry) and its own scheduling, not routines.

### Gaps
- I didn't find a documented public API to create or list cloud sessions programmatically beyond `claude --cloud`, `--cloud <id> -p` and the routine `/fire` endpoint. A Claude Platform API reference for routines is linked ([Trigger a routine via API](https://platform.claude.com/docs/en/api/claude-code/routines-fire)) but was not read.
- Projects internals (whether thread state or approvals are exposed via any API) were not documented in the portions read.
- I found no Anthropic doc describing a built-in multi-day human-approval gate primitive beyond those listed. Absence is inferred from the docs read, not explicitly stated.

## 4. Recommended patterns from Anthropic for CI/automation and long-running tasks

### Takeaway
Anthropic's guidance for automation:
- use `-p` with `--bare`, JSON/schema output, explicit tool allowlists, a locked-down permission mode (`dontAsk`/`auto` with `--permission-prompts none`), and turn and budget caps
- persist session IDs and resume
- prefer passing explicit state into fresh sessions over shipping transcripts

For long-running work, its engineering posts recommend structured external artifacts (feature lists, progress files, git commits, test gates), separate generator and evaluator agents, and sprint contracts. Those artifacts are exactly what a staged harness would checkpoint between human reviews.

### Cited Findings
- In CI, add `--bare` so the run doesn't load host hooks, plugins, memory or CLAUDE.md. "`--bare` is the recommended mode for scripted and SDK calls." — [headless](https://code.claude.com/docs/en/headless)
- Use `--permission-prompts none` "when nobody is available to answer permission prompts, for example in a scheduled job." An example pairs it with `--permission-mode auto`. — [headless](https://code.claude.com/docs/en/headless)
- Capture `session_id` from JSON output and `--resume` it for follow-ups. — [headless](https://code.claude.com/docs/en/headless)
- For cross-host resume, "Capture the results you need (analysis output, decisions, file diffs) as application state and pass them into a fresh session's prompt. This is often more robust than shipping transcript files around." — [sessions](https://code.claude.com/docs/en/agent-sdk/sessions)
- For users who may take longer to respond than the process can live, use `PreToolUse` `defer` and resume later. — [user-input](https://code.claude.com/docs/en/agent-sdk/user-input)
- Bound sessions with `maxTurns` (there's no session timeout), recycle subprocesses for memory, batch wide fan-outs, alert on `mirror_error`, and route egress through an allowlisting proxy. — [Hosting](https://code.claude.com/docs/en/agent-sdk/hosting)
- In GitHub Actions, put standards in CLAUDE.md, set `--max-turns`, set workflow timeouts and use concurrency controls. — [GitHub Actions](https://code.claude.com/docs/en/github-actions)
- Routine prompts "must be self-contained and explicit about what to do and what success looks like". Run status doesn't indicate task success, so read the transcript. — [Routines](https://code.claude.com/docs/en/routines)
- "Plan locally, execute in the cloud": collaborate in plan mode, commit the plan file, then `claude --cloud "Execute the migration plan in docs/migration-plan.md"`. — [Cloud sessions](https://code.claude.com/docs/en/claude-code-on-the-web)
- **"Effective harnesses for long-running agents"** (2025-11-26):
  - An initializer agent creates `init.sh`, `claude-progress.txt`, an initial git commit, and a JSON feature list of 200+ features marked passing or failing.
  - A coding agent works one feature per session. It starts by reading git logs and the progress file and running a smoke test.
  - It is told "It is unacceptable to remove or edit tests".
  - Observed failure modes: doing too much at once, prematurely declaring completion, and marking features done without testing.
  — [Anthropic Engineering](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- **"Harness design for long-running application development"** (2026-03-24):
  - Planner, generator and evaluator agents are built on the Agent SDK. The evaluator uses Playwright MCP.
  - Generator and evaluator negotiate "sprint contracts" with testable criteria. Agents communicate via structured files.
  - Opus 4.5 needed context resets; Opus 4.6 ran continuously with compaction.
  - Example costs: solo run 20 min / $9 vs. full harness 6 h / $200; a later DAW build 3 h 50 min / $124.70.
  - Fully autonomous, with no human checkpoints during runs.
  — [Anthropic Engineering](https://www.anthropic.com/engineering/harness-design-long-running-apps). A secondary summary of the same post: "tuning a standalone evaluator to be skeptical turned out to be far more tractable than making a generator critical of its own work." — via search snippet of the same URL.
- Dynamic workflows codify "adversarially review each other's findings" and "draft a plan from several angles" as repeatable quality patterns. — [Workflows](https://code.claude.com/docs/en/workflows)

### Inferences
- The engineering posts' handoff artifacts (spec, feature list, progress log, sprint contract, evaluator verdict) are natural review boundaries for human checkpoints. Anthropic's reference harnesses run them autonomously; inserting human gates is the gap a staged harness fills.
- Dollar figures from Anthropic's own harness runs ($100–$200 per multi-hour build) show why cross-stage budget governance matters.

### Gaps
- I couldn't fetch the Claude blog post "A harness for every task: dynamic workflows in Claude Code" (claude.dev egress blocked). Its contents are not summarized.
- I did not review the "Claude Code best practices" engineering post. Its content may overlap with the above.

## 5. Sandboxing and isolation options

### Takeaway
Claude Code offers a graded set of isolation options:
- the OS-level sandboxed Bash tool (Seatbelt/bubblewrap)
- the `@anthropic-ai/sandbox-runtime` wrapping the whole process (beta research preview)
- dev containers
- custom containers and VMs
- git worktrees for file-level parallelism
- Anthropic-hosted or self-hosted cloud sessions
- `--restricted` mode for eval harnesses

Anthropic's guidance is that unattended `--dangerously-skip-permissions` or auto-mode runs belong in a container, VM or the sandbox runtime.

### Cited Findings
- **Isolation options compared**:

  | Option | What it isolates | Docker needed |
  | :- | :- | :- |
  | Sandboxed Bash tool | Bash/PowerShell/Monitor commands and children | No |
  | Sandbox runtime | Whole process, including file tools, MCP servers and hooks | No |
  | Dev container | Full environment | Yes |
  | Custom container | Full environment | Yes |
  | VM | Full OS | No |
  | Cloud sessions | Full OS, hosted by Anthropic | No |

  — [Choose a sandbox environment](https://code.claude.com/docs/en/sandbox-environments)
- "Always run `--dangerously-skip-permissions` sessions inside a container, a VM, or the sandbox runtime, so that file tools, MCP servers, and hooks are also inside the boundary." Claude Code refuses this flag as root on Linux and macOS. — [Sandbox environments](https://code.claude.com/docs/en/sandbox-environments)
- The Bash sandbox is OS-enforced file and network limits on shell commands. It allows running sandboxed commands without prompts. File tools, MCP servers and hooks run outside it. — [Configure the sandboxed Bash tool](https://code.claude.com/docs/en/sandboxing)
- `@anthropic-ai/sandbox-runtime` wraps an entire process in Seatbelt/bubblewrap. It is a "beta research preview, and its configuration format may change". — [Sandbox environments](https://code.claude.com/docs/en/sandbox-environments)
- Dev containers cover a team-standard isolated environment, persisted auth, org policy, network egress restriction and running without prompts. — [Development containers](https://code.claude.com/docs/en/devcontainer)
- **Worktrees**:
  - `claude -w <name>` creates `.claude/worktrees/<name>/` on branch `worktree-<name>`; `#<PR>` or a PR URL branches from that PR.
  - Subagent `isolation: worktree` gives a subagent its own worktree.
  - Background sessions auto-isolate.
  - `WorktreeCreate`/`WorktreeRemove` hooks allow custom (for example non-git) workspace provisioning.
  — [Worktrees](https://code.claude.com/docs/en/worktrees); [CLI reference](https://code.claude.com/docs/en/cli-reference); [Hooks reference](https://code.claude.com/docs/en/hooks)
- **Cloud session security**:
  - isolated VMs
  - network allowlist by default ("Trusted") with Custom and Full options
  - GitHub credentials kept outside the VM via a proxy
  - self-hosted environments, where isolation is the deployer's responsibility
  - With network disabled, the Anthropic API is still reachable.
  — [Cloud sessions](https://code.claude.com/docs/en/claude-code-on-the-web)
- For SDK hosting, "Container-based sandboxing" and an egress proxy enforcing domain allowlists are recommended. — [Hosting](https://code.claude.com/docs/en/agent-sdk/hosting)
- `--restricted` removes command and code tools and WebFetch, confines file tools, ignores user and project settings and refuses bypass mode. It is designed for evaluation harnesses on shared machines. — [CLI reference](https://code.claude.com/docs/en/cli-reference)

### Inferences
- A practical stage runner is one container (or sandbox-runtime-wrapped process) per stage job, with a dedicated git worktree or branch per workflow and an egress proxy. Hooks and MCP servers must be inside that boundary, which the Bash sandbox alone does not provide.

### Gaps
- I did not read the detailed sandbox configuration schema (filesystem and network rule syntax) or the org-wide "enforce isolation" section in depth.
