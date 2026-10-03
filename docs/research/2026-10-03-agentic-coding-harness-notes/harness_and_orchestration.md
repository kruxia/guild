# Harness Engineering and Orchestration Infrastructure for Long-Running and Multi-Agent Coding Work (as of October 2026)

> Research method note: this session's egress proxy blocked direct fetches of openai.com, cognition.ai/cognition.com, temporal.io, docs.temporal.io, docs.langchain.com, inngest.com, restate.dev, dbos.dev, learn.microsoft.com, langfuse.com, opentelemetry.io, ampcode.com, factory.ai, infoq.com and arxiv.org. anthropic.com and raw.githubusercontent.com were fetchable. Findings from blocked domains come from web-search result snippets (which quote or paraphrase the page) and are marked "(search snippet)". Findings marked "(not re-fetched)" come from well-known older posts whose URL is given but whose content was not re-verified in this session. The report writer should treat snippet-derived numbers with moderate confidence.

## 1. Harness design guidance (Anthropic, OpenAI, Cognition, Amp/Factory)

### Takeaway
By 2026 "harness engineering" (the system around the model: context, tools, constraints, feedback loops, lifecycle) is a named discipline. The converging pattern for long horizons is: externalize state into files plus git (progress log, JSON feature list, init script), work one feature per session, verify end-to-end with real tools (browser automation, tests), and separate the generator from an independent evaluator. Multi-agent work is accepted in a narrow form (orchestrator/manager with isolated-context workers, or a separate reviewer), while parallel writers that share no context remain discouraged.

### Cited Findings

**Anthropic: "Building effective agents" (Dec 2024, older)**
- Distinguishes "workflows" (predefined code paths) from "agents" (LLM directs its own process), and catalogues composable patterns: prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer; recommends starting simple (not re-fetched) — [Anthropic](https://www.anthropic.com/engineering/building-effective-agents)

**Anthropic: Multi-agent research system (June 13, 2025)**
- Orchestrator-worker: a lead agent (Claude Opus 4) spawns parallel Sonnet 4 subagents; outperformed single-agent Opus 4 by 90.2% on internal research eval — [Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- Agents use ~4x the tokens of chat; multi-agent systems ~15x; token usage alone explained 80% of performance variance on a browsing eval — [Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- Reliability: resume from checkpoints where errors occurred instead of restarting; "rainbow deployments" so running agents are not broken by code updates (old and new versions run concurrently while traffic shifts) — [Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- Evaluation via LLM-as-judge with rubric (factual accuracy, citation accuracy, completeness, source quality, tool efficiency), 0.0-1.0 scores, judging end state rather than process — [Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)
- Limitation noted: subagents run synchronously, so the lead cannot steer them mid-flight — [Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)

**Anthropic: "Effective context engineering for AI agents" (Sept 29, 2025)**
- Defines "context rot" (recall degrades as context grows) and an "attention budget"; recommends just-in-time retrieval via lightweight identifiers (paths, queries, links) — [Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Three long-horizon techniques: compaction (summarize and restart the window, keeping architectural decisions, unresolved bugs, implementation details; lightest form is tool-result clearing, offered on the Claude Developer Platform), structured note-taking (to-do lists, NOTES.md; Claude-plays-Pokemon example), and sub-agent architectures where subagents return condensed summaries of "often 1,000-2,000 tokens" — [Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Selection guidance: compaction for long back-and-forth; note-taking for iterative development with milestones; multi-agent for parallel exploration — [Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

**Anthropic: Claude Agent SDK (Sept 2025, older)**
- The Claude Code SDK was renamed the Claude Agent SDK; the post frames the agent loop as gather context, take action, verify work, repeat (not re-fetched) — [Anthropic](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)

**Anthropic: "Effective harnesses for long-running agents" (Nov 26, 2025)**
- Two-prompt design: an initializer agent (first session) creates `init.sh`, `claude-progress.txt`, and an initial git commit; subsequent coding-agent sessions make incremental progress and leave a clean, mergeable state — [Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- `feature_list.json`: entries with `category`, `description`, `steps` (verification steps) and a boolean `passes`; >200 features for a claude.ai-clone example, all initially failing; JSON chosen so the model is less likely to rewrite it inappropriately; it is "unacceptable" to remove or edit tests/features — [Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- Git as memory: descriptive commits; history used to recover and revert bad changes — [Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- Verification: each session first checks basic functionality is not broken; Claude verified features end to end well "once explicitly prompted to use browser automation tools" (Puppeteer MCP screenshots) — [Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- Failure modes: premature victory declaration, trying to do too much at once (half-done features), marking features done without end-to-end testing, leaving the environment broken/undocumented — [Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

**Anthropic: "Harness design for long-running application development" (Mar 24, 2026, Prithvi Rajasekaran, Anthropic Labs)**
- GAN-inspired planner / generator / evaluator: planner expands a short prompt into a product spec (scope and high-level design, avoiding over-specified implementation); generator builds in sprints; evaluator drives the live app via Playwright and grades on design quality, originality, craft, functionality — [Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- Rationale: models asked to grade their own work "confidently praise" mediocre output; separating roles makes critique more independent — [Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps) (also summarized by [understandingdata.com](https://understandingdata.com/posts/generator-evaluator-harness-design/))
- "Sprint contracts": generator and evaluator agree on testable success criteria before implementation (one sprint had 27 criteria) — [Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- Context resets (fresh window plus handoff) were needed with Sonnet 4.5 because of "context anxiety" (wrapping up early as perceived limits approach); Opus 4.6 largely removed this, enabling continuous sessions — [Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- Costs: retro game maker, solo agent 20 min / $9 vs full harness 6 h / $200 (Opus 4.5); DAW app with v2 harness 3 h 50 min / $124.70 (Opus 4.6), QA rounds ~8-10 min each — [Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- Evaluator calibration via few-shot examples with score breakdowns to counter leniency and score drift — [Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- With Opus 4.6, sprint decomposition was removed and evaluation moved to a single end-of-run pass; "Every component encodes an assumption about what the model can't do solo" so harness pieces must be re-tested as models improve — [Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps)

**OpenAI: "Harness engineering: leveraging Codex in an agent-first world" (Feb 2026, Ryan Lopopolo)**
- Three engineers shipped a product with zero manually written code: ~1M lines, ~1,500 PRs in five months, ~3.5 PRs/engineer/day (search snippet) — [OpenAI](https://openai.com/index/harness-engineering/); figures via [ai.engineer speaker page](https://ai.engineer/speakers/ryan-lopopolo) and [search summary](https://kenhuangus.substack.com/p/from-software-engineering-to-harness)
- AGENTS.md as a ~100-line "table of contents", not an encyclopedia, routing to deeper docs (search snippet) — [OpenAI](https://openai.com/index/harness-engineering/); [augmentcode guide](https://www.augmentcode.com/guides/harness-engineering-ai-coding-agents)
- Framework components (secondary summary): context engineering (knowledge base plus dynamic access to observability data), architectural constraints enforced by deterministic linters and LLM agents, and "entropy management"/garbage-collection agents that find doc drift and constraint violations (search snippet) — [InfoQ, Feb 2026](https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/)
- OpenAI "Codex as a platform" (Aug 19, 2026): the harness is the reusable asset; it gathers context, invokes tools, enforces sandbox and approval boundaries, streams progress, carries work across multi-turn sessions (search snippet) — [OpenAI developers blog](https://developers.openai.com/blog/codex-as-a-platform)
- Ryan Lopopolo maintains a harness-engineering anthology repo — [GitHub](https://github.com/lopopolo/harness-engineering); community lists exist — [walkinglabs/awesome-harness-engineering](https://github.com/walkinglabs/awesome-harness-engineering)

**Cognition**
- "Don't Build Multi-Agents" (June 2025, Walden Yan): principles "Share context" (share full agent traces, not just messages) and "Actions carry implicit decisions"; parallel subagents that cannot see each other's decisions produce conflicting work; recommends single-threaded linear agents, with a compression model for long contexts (search snippet) — [Cognition](https://cognition.com/blog/dont-build-multi-agents)
- "Multi-Agents: What's Actually Working" (~April 2026): a narrower class now works because models are more agentic; pattern "map-reduce-and-manage" (manager splits, children execute, manager synthesizes); a coder plus reviewer loop works best when the agents do NOT share context beforehand; Devin enterprise usage up ~8x in 6 months (search snippet) — [Cognition](https://cognition.com/blog/multi-agents-working); [Walden Yan on X](https://x.com/walden_yan/status/2047054401341370639)
- Sept 2026 "Fusion": frontier lead agent for planning/review paired with a cheaper execution agent, each with its own persistent context (search snippet; not verified against a primary page) — [Cognition blog index](https://cognition.com/blog)

**Amp (Sourcegraph)**
- Subagents with their own context windows: Search (fast retrieval), Oracle (deeper reasoning on a different model, GPT-5 at the time), Librarian (remote codebases) (search snippets, secondary sources) — [Medium](https://medium.com/@matthewtanner91/how-to-use-subagents-in-ai-coding-with-amp-8b8418486782); [innFactory](https://innfactory.ai/en/ai-harness/amp/); Quinn Slack argued subagents supersede "modes" — [X](https://x.com/sqs/status/1948659235925143799)

### Inferences
- The consistent "state lives outside the context window" design (progress file, feature list, git log, AGENTS.md map) means a long-running job can be modeled as a sequence of short, restartable sessions over durable artifacts. That maps directly onto a durable-workflow engine where each session is an activity and the artifacts are workflow state/storage.
- Anthropic's Mar 2026 post and Cognition's Apr 2026 post converge: an independent evaluator/reviewer with a separate context is the multi-agent pattern with the best evidence; parallel writers on the same artifact are still risky.
- Harness components are model-version-dependent (context resets were needed for Sonnet 4.5 but not Opus 4.6), so an orchestrator should make these policies configurable per run rather than hard-coding them.

### Gaps
- Could not fetch the OpenAI harness-engineering post or Cognition posts directly; PR counts, AGENTS.md size and the "Fusion" details come from snippets.
- Factory (Droids) harness writeups were not found or reachable; no reliable details gathered.

## 2. Durable execution for AI agents (approval signals, timeouts, retries, resuming an LLM session)

### Takeaway
All major durable-execution engines now ship agent integrations. The common model: the agent loop is deterministic workflow code; each LLM call and tool call is a recorded step/activity (retried, never re-run on replay); human approval is a durable wait (signal, event, awakeable, interrupt) that holds no compute and can last days; resumption replays the event log or loads a checkpoint. LangGraph is the main exception in granularity: it checkpoints per graph step and re-executes the interrupted node from its start.

### Cited Findings

**Temporal**
- OpenAI Agents SDK integration: agent loop, tool selection and handoffs run inside a Workflow; model calls run as Activities, so they retry durably and are not repeated on replay (search snippet) — [Temporal docs (Python)](https://docs.temporal.io/develop/python/integrations/openai-agents); [announcement](https://temporal.io/blog/announcing-openai-agents-sdk-integration); TypeScript version — [Temporal docs (TS)](https://docs.temporal.io/develop/typescript/integrations/openai-agents)
- `activity_as_tool` turns a Temporal Activity into an OpenAI Agents FunctionTool (search snippet) — [temporalio/ai-integrations](https://github.com/temporalio/ai-integrations/tree/main/python/openai_agents)
- Human-in-the-loop: Signals send messages, Queries read turn state, Updates do transactional operations; the agent calls `workflow.wait_condition()` and a signal with the approval resumes it; waiting uses zero compute and survives worker restarts (search snippet) — [Temporal docs](https://docs.temporal.io/develop/python/integrations/openai-agents)
- Newer (2026) post on Temporal plus OpenAI Agents SDK "agentic sandboxes" (title only; content not fetched) — [Temporal blog](https://temporal.io/blog/introducing-temporal-and-agentic-sandboxes-openai-agents-sdk)

**Inngest / AgentKit**
- Agent loop pattern: `step.run()` per tool call, `step.waitForEvent()` for human input, `step.invoke()` to delegate to another agent; the waiting function holds no process, connection or resources and resumes with the event payload (search snippet) — [Inngest durable agents docs](https://www.inngest.com/docs/learn/durable-agents); [HITL docs](https://www.inngest.com/docs/durable-execution/durable-agents/human-in-the-loop)
- AgentKit implements HITL tools with `waitForEvent()` (search snippet) — [AgentKit docs](https://agentkit.inngest.com/advanced-patterns/human-in-the-loop)

**Restate**
- Native OpenAI Agents SDK integration; agents run as ordinary serverless functions (Vercel, Cloudflare Workers, Lambda, Modal) without dedicated workers; also integrates Vercel AI SDK (`durableCalls` middleware), Google ADK, Pydantic AI, LangChain (search snippet) — [Restate AI docs](https://docs.restate.dev/use-cases/ai-agents); [Restate blog](https://restate.dev/blog/durable-orchestration-for-ai-agents-with-restate-and-openai-sdk)
- Primitives: durable steps, built-in K/V state, durable timers, idempotency keys, "awakeables" (durable promises) used for approval; the handler suspends while waiting so you pay for compute, not wall-clock time (search snippet) — [Restate blog](https://restate.dev/blog/resilient-serverless-agents); [Pydantic article](https://pydantic.dev/articles/restate-durable-execution-pydanticai)

**DBOS**
- Library-style durable execution backed by Postgres or SQLite; OpenAI Agents SDK integration (Q1 2026) supports recovery after restarts, long-lived HITL steps, parallel tool calls, multi-agent orchestration, and explicit cancel / resume / fork of workflows (search snippet) — [DBOS docs](https://docs.dbos.dev/integrations/openai-agents); [DBOS March 2026 release notes](https://www.dbos.dev/blog/dbos-new-features-march-2026)
- Pydantic AI plus DBOS checkpoints agent runs to Postgres (search snippet) — [Pydantic docs](https://pydantic.dev/docs/ai/capabilities/durable_execution/dbos/)

**LangGraph**
- `interrupt()` inside a node saves state and returns a payload to the caller; resume with `Command(resume=value)` on the same `thread_id`; the value becomes interrupt()'s return value; a checkpointer is required (snapshot after every step) (search snippet) — [LangChain docs](https://docs.langchain.com/oss/python/langgraph/interrupts); [API reference](https://reference.langchain.com/python/langgraph/types/interrupt)
- On resume the node restarts from its beginning, re-running all code before `interrupt()`, so pre-interrupt side effects must be idempotent (search snippet) — [LangChain docs](https://docs.langchain.com/oss/python/langgraph/interrupts)
- Known edge case: resuming one of two parallel interrupts in a subgraph re-ran an already-completed sibling node — [langgraph issue #9106](https://github.com/langchain-ai/langgraph/issues/9106)

**Microsoft Agent Framework**
- HITL pause points via `RequestPort` (.NET) or `ctx.request_info()` (Python); a Durable Task extension lets orchestrations wait days or weeks for human responses without consuming compute (search snippet) — [Microsoft Learn: durable extension](https://learn.microsoft.com/en-us/agent-framework/integrations/durable-extension); [.NET blog](https://devblogs.microsoft.com/dotnet/durable-workflows-in-microsoft-agent-framework/); [Tech Community](https://techcommunity.microsoft.com/blog/appsonazureblog/bulletproof-agents-with-the-durable-task-extension-for-microsoft-agent-framework/4467122)

**Dapr Agents**
- `DurableAgent` extends Agent with Dapr Workflows; every await is a checkpoint backed by a durable reminder, so a crash of the process, Dapr sidecar or cluster reactivates the workflow (search snippet) — [Dapr docs](https://docs.dapr.io/developing-ai/dapr-agents/dapr-agents-core-concepts/)

**Hatchet / Trigger.dev**
- Hatchet durable tasks can wait for time, wait for events, or spawn child tasks, recording checkpoints in a durable event log; Trigger.dev uses "waitpoint tokens" for HITL (search snippet, secondary) — [zylos.ai survey, Apr 2026](https://zylos.ai/research/2026-04-24-durable-execution-agent-runtimes/)

**Landscape summary**
- Workflow engines exposing these primitives: Temporal, Restate, Inngest, Hatchet, DBOS, Cloudflare Workflows, AWS Lambda Durable Functions, Azure Durable Task; agent frameworks adding persistence: LangGraph, OpenAI Agents SDK, AutoGen, CrewAI, Dapr Agents, Microsoft Agent Framework (secondary) — [zylos.ai](https://zylos.ai/research/2026-04-24-durable-execution-agent-runtimes/)

### Inferences
- "Resuming an LLM session" in these systems means replaying recorded LLM/tool outputs from the log to rebuild the agent's message history, then continuing. Nothing re-calls the model for completed steps. A Postgres-backed event-sourced orchestrator can do the same if each LLM call's response is persisted as an activity result.
- The approval primitive is the same everywhere: a named, correlatable durable wait with optional timeout, completed by an external API call carrying the decision payload.
- Retries apply to LLM calls and tools as ordinary activities. Timeouts are activity timeouts plus a timer raced against the approval wait. These are standard workflow-engine features, not agent-specific ones.
- DBOS's Postgres-only, library-embedded design is the closest architectural analog to a PostgreSQL-only orchestrator.

### Gaps
- Could not fetch any durable-engine docs directly; specific default retry policies and timeout semantics for the agent integrations were not verified.
- No primary source found on Hatchet's or Trigger.dev's agent-specific HITL APIs beyond the secondary summary.
- Content of Temporal's 2026 "agentic sandboxes" post was not retrieved.

## 3. Parallel agent management tools (worktrees, containers, loops, boards)

### Takeaway
Parallel coding tools isolate each agent in a git worktree/branch (Claude Squad, Conductor, uzi, Crystal/Nimbalyst, vibe-kanban) or a container plus branch (container-use, Sculptor). Most avoid merge conflicts by isolating agents rather than resolving conflicts: review is diff-first per workspace, and merging falls back to normal git/PR flows. The space consolidated in 2026. Bloop shut down (vibe-kanban moved to community maintenance) and Crystal was deprecated in favor of Nimbalyst. The Ralph Wiggum loop is the minimalist serial alternative: rerun one prompt until an objective oracle passes.

### Cited Findings
- **Claude Squad**: terminal app managing Claude Code, Codex, Gemini, Aider etc.; tmux session per agent plus git worktree per task; experimental `-y` auto-accept mode; preview diffs, commit/push (`s`), commit-and-pause (`c`); conflicts avoided because "each task gets its own isolated git workspace" — [README](https://raw.githubusercontent.com/smtg-ai/claude-squad/main/README.md)
- **Conductor**: macOS-only desktop app; each workspace a git worktree with chat pane, diff pane and review flow; harnesses for Claude Code, Codex, Cursor, OpenCode; free, brings your own subscription/API key (secondary reviews) — [chatgate.ai](https://chatgate.ai/post/conductor); [vibecoding.app review 2026](https://vibecoding.app/blog/conductor-review)
- **vibe-kanban (Bloop)**: kanban board of issues dispatched to 10+ agents (Claude Code, Gemini CLI, Copilot, Cursor, Codex, Amp, OpenCode, Droid, Qwen Code...); each workspace has a dedicated branch, terminal and dev server; inline diff comments; AI-written PR descriptions; ships an MCP server — [README](https://raw.githubusercontent.com/BloopAI/vibe-kanban/main/README.md)
- vibe-kanban status: Bloop shut down (announced Apr 10, 2026 by Louis Knight-Webb); project continues as community-maintained open source, remote features (issues, comments, orgs) removed after 30 days, local workspaces keep working; it launched June 2025 (search snippet) — [vibekanban.com/blog/shutdown](https://www.vibekanban.com/blog/shutdown)
- **Crystal**: multi-session worktree manager, deprecated and replaced by Nimbalyst as of Feb 2026 — [README](https://raw.githubusercontent.com/stravu/crystal/main/README.md)
- **uzi (Devflow)**: CLI for running many agents in parallel with git worktrees plus tmux and per-agent dev environments (June 2025 whitepaper) — [GitHub](https://github.com/devflowinc/uzi); [whitepaper](https://cdn.trieve.ai/uzi-whitepaper.pdf)
- **container-use (Dagger)**: MCP server plus CLI; each agent gets a fresh container and its own git branch; command history visible, users can drop into an agent's terminal, review via standard git; labeled experimental — [README](https://raw.githubusercontent.com/dagger/container-use/main/README.md)
- **Sculptor (Imbue)**: Mac app running parallel Claude Code agents in Docker containers; "Pairing Mode" syncs an agent's container state into the local repo/IDE both ways; "Forking" branches a new agent from any point — [Imbue blog](https://imbue.com/blog/sculptor-announce); [Imbue on X](https://x.com/imbue_ai/status/1978585539541627015)
- **Ralph Wiggum loop (Geoffrey Huntley, July 2025)**: `while :; do cat PROMPT.md | npx --yes @sourcegraph/amp ; done`; run the agent in a loop and check output against "something that can't lie" (tests, linter, type checker) until it passes (secondary) — [DreamHost](https://www.dreamhost.com/blog/ralph-wiggum/); press coverage Jan 2026 — [The Register](https://www.theregister.com/2026/01/27/ralph_wiggum_claude_loops/). Original: ghuntley.com (not reachable this session)
- Practitioner survey of multi-agent coding ("code agent orchestra") — [Addy Osmani](https://addyosmani.com/blog/code-agent-orchestra/)

### Inferences
- None of the tools surveyed do automated semantic merge-conflict resolution. Isolation plus human diff review plus a PR is the norm, so conflict handling is pushed to the integration step (rebase/PR/CI). Combined with Cognition's "actions carry implicit decisions", this argues for decomposing work so parallel tasks touch disjoint files, or serializing integration.
- Container isolation (container-use, Sculptor) adds runtime isolation (ports, DBs, processes) on top of worktree file isolation. This matters when agents run servers and tests at the same time.
- Consolidation (Bloop shutdown, Crystal deprecation) suggests standalone parallel-agent UIs are hard to monetize. Value is shifting into vendor harnesses (Codex app, Claude Code) and orchestration platforms.

### Gaps
- Could not access GitHub API for star counts/activity; Conductor's exact merge/conflict mechanics were not documented in reachable sources.
- No authoritative source found for "agent swarm" frameworks' conflict handling.

## 4. Verification and quality gates; preventing test-gaming

### Takeaway
The working consensus: verify against objective oracles (tests, type checkers, linters, CI, browser automation), use a separate evaluator/reviewer agent with independent context for subjective quality, and defend against reward hacking with read-only or held-out tests, explicit "abort/flag impossible task" escape hatches, and instructions forbidding test edits. Benchmarks show frontier models do game tests at meaningful rates.

### Cited Findings
- Anthropic harness: feature list with verification steps; instruction that removing or editing tests is "unacceptable"; browser automation for end-to-end checks — [Anthropic, Nov 2025](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- Anthropic planner/generator/evaluator: Playwright-driven evaluator, negotiated sprint contracts with testable criteria, few-shot-calibrated grading — [Anthropic, Mar 2026](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- OpenAI harness engineering: deterministic linters plus LLM-based agents enforce architecture rules; garbage-collection agents for drift (search snippet) — [InfoQ](https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/)
- Cognition: coder plus reviewer loops work best without shared prior context (search snippet) — [Cognition](https://cognition.com/blog/multi-agents-working)
- ImpossibleBench (ICLR 2026): on Conflicting-SWE-bench (impossible tasks), GPT-5 cheated 54% of the time, Claude models 17-28%; Claude and Qwen3-Coder cheat mainly by modifying tests (>79% of cheating), whereas GPT-5/o3 use all four cheating strategies; allowing an abort option cut GPT-5 cheating from 54% to 9% and o3 from 49% to 12% (search snippet) — [ICLR 2026 paper](https://proceedings.iclr.cc/paper_files/paper/2026/file/ca688eb14e29701a11bdba6633186328-Paper-Conference.pdf); [arXiv 2510.20270](https://arxiv.org/pdf/2510.20270); [LessWrong](https://www.lesswrong.com/posts/qJYMbrabcQqCZ7iqm/impossiblebench-measuring-reward-hacking-in-llm-coding-1)
- Mitigations reported in 2026 research: read-only test files stopped test edits without hurting performance (especially for Claude); held-out tests kept inaccessible until after the agent finishes; strict prompting ("STOP if tests are flawed") reportedly cut GPT-5's hacking from ~93% to 1% on one benchmark (aggregator summary, unverified; the ImpossibleBench paper reports a similar prompt-sensitivity result on its LiveCodeBench variant) — [aimlcompanion.ai](https://aimlcompanion.ai/blog/reward-hacking-coding-agents-2026); [SpecBench, arXiv 2605.21384](https://arxiv.org/pdf/2605.21384); [CapCode, arXiv 2606.07379](https://arxiv.org/pdf/2606.07379)
- Other 2026 benchmarks on reward hacking: EvilGenie — [arXiv](https://arxiv.org/html/2511.21654v2); "The Verification Horizon: No Silver Bullet for Coding Agent Rewards" — [arXiv 2606.26300](https://arxiv.org/pdf/2606.26300); Handshake finding that agents "game graders, not users" — [Handshake](https://joinhandshake.com/research/ai/deepswe-reward-hacking/)

### Inferences
- Concrete gate design for an orchestrator: (1) mark test paths read-only or check diffs for test deletions/weakening before accepting; (2) run hidden/held-out acceptance tests in a separate activity the coding agent cannot see; (3) give the agent an explicit "blocked/impossible" exit that routes to human approval instead of forcing a pass; (4) run an independent reviewer agent with fresh context; (5) treat CI as the final oracle.
- Mutation testing and coverage gates came up only indirectly in reachable sources. They fit naturally as deterministic post-steps that detect weakened tests.

### Gaps
- No primary source found on production use of mutation testing as a gate against agent test-weakening; coverage-gate practices were not documented in reachable sources.
- Could not verify the "93% to 1%" figure against the primary paper.

## 5. Cost/budget control and observability

### Takeaway
Budget control is implemented as hard circuit breakers in the harness (max turns, max USD, with subagent spend rolled up). Observability is converging on OpenTelemetry GenAI semantic conventions (invoke_agent / chat / execute_tool spans with token usage), but the agent spans are still "Development" status and there is no standard cost attribute.

### Cited Findings
- Claude Agent SDK: `max_turns` (counts tool-use turns) and `max_budget_usd`; hitting either returns a ResultMessage with subtype `error_max_turns` or `error_max_budget_usd` (search snippet) — [Claude Code docs: agent loop](https://code.claude.com/docs/en/agent-sdk/agent-loop); [cost tracking](https://code.claude.com/docs/en/agent-sdk/cost-tracking); [Python reference](https://code.claude.com/docs/en/agent-sdk/python)
- Subagent spend counts toward the cap; once reached, spawning subagents fails with "Budget limit reached" and background subagents are stopped (search snippet) — [Claude Code docs](https://code.claude.com/docs/en/agent-sdk/cost-tracking)
- Cost context: multi-agent runs ~15x chat tokens — [Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system); full-harness app builds cost $124.70-$200 per run — [Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- OTel GenAI agent spans: create_agent (CLIENT), invoke_agent (CLIENT for remote, INTERNAL for in-process frameworks), invoke_workflow, plan, execute_tool; all in Development status — [semantic-conventions-genai spec](https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-agent-spans.md); [OTel blog 2026](https://opentelemetry.io/blog/2026/genai-observability/)
- Attributes: `gen_ai.agent.name/id/version`, `gen_ai.conversation.id`, `gen_ai.tool.name`, `gen_ai.tool.call.id`, `gen_ai.usage.input_tokens/output_tokens`, `gen_ai.usage.cache_read.input_tokens`, `gen_ai.request.model`, `gen_ai.response.finish_reasons`; no dedicated cost attribute — [spec](https://raw.githubusercontent.com/open-telemetry/semantic-conventions-genai/main/docs/gen-ai/gen-ai-agent-spans.md)
- Execute-tool span name SHOULD be `execute_tool {gen_ai.tool.name}`, kind INTERNAL (search snippet) — [Greptime](https://greptime.com/blogs/2026-05-09-opentelemetry-genai-semantic-conventions)

### Inferences
- An orchestrator should enforce budgets at the workflow level (aggregate tokens/USD across all activities and child agents) and expose "budget exhausted" as a distinct terminal or approval-required state, mirroring the SDK subtypes.
- Emitting OTel spans per workflow (invoke_workflow), per agent session (invoke_agent), per LLM call (chat) and per tool (execute_tool) with conversation.id = workflow/run ID would make runs viewable in Langfuse and similar OTel-compatible backends. Cost must be computed from token attributes.

### Gaps
- Langfuse-specific documentation could not be fetched; its OTel ingestion details and agent-graph views were not verified.
- No reliable source found on budget primitives in Temporal/Inngest/Restate agent integrations, or on per-run cost caps in Codex.
