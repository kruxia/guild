# obra/superpowers (Jesse Vincent / Prime Radiant): deep dive as of October 2026

Method note: the repository was cloned through the session's git proxy on 2026-10-03. HEAD is commit `8ca22db` (2026-09-25), tagged `v6.4.2`, with 683 commits. Every SKILL.md, script, manifest and the 102 KB `RELEASE-NOTES.md` cited below was read from that clone. Repo URLs point at the `v6.4.2` tag so they stay stable. The GitHub REST API refused the request for this repo (HTTP 403 "GitHub access to this repository is not enabled for this session"). The egress proxy blocked these sites: simonwillison.net, blog.fsck.com, news.ycombinator.com, dev.to, rywalker.com, gitstarclub.com, pelayoarbues.com and web.archive.org. Claims from those sites rest on search-engine snippets only and are marked **[snippet]**. github.com HTML pages were reachable through WebFetch. The Anthropic official marketplace repo (`anthropics/claude-plugins-official`) was also cloned and read directly.

## 1. Architecture and distribution: what it is, how it installs, which agents it supports, version history, and how it relates to Anthropic Skills and marketplaces

### Takeaway
Superpowers has three parts:
- an MIT-licensed library of 15 SKILL.md skills;
- a thin per-harness bootstrap that injects the `using-superpowers` skill at session start, so the other skills trigger automatically;
- packaging as a Claude Code plugin, which is also listed in Anthropic's official `claude-plugins-official` marketplace and in Jesse Vincent's own `superpowers-marketplace`.

As of v6.4.2 (2026-09-25) the same `skills/` tree ships to about 16 harnesses, including Claude Code, Codex, Cursor, OpenCode, Copilot CLI, Gemini CLI, Pi, Antigravity, Kimi, Devin, Hermes, Muse, Qwen, Factory Droid and Grok. Since October 2025 it has gone through six major versions (v1.0 to v6.4.2).

### Cited Findings
**What it is**
- The project describes itself as "a complete software development methodology for your coding agents, built on top of a set of composable skills and some initial instructions that make sure your agent uses them." — [README](https://github.com/obra/superpowers/blob/v6.4.2/README.md)
- It is built by Jesse Vincent and "the rest of the folks at Prime Radiant". Prime Radiant offers commercial support at sales@primeradiant.com. — [README](https://github.com/obra/superpowers/blob/v6.4.2/README.md)
- The license is MIT, "Copyright (c) 2025 Jesse Vincent". `plugin.json` declares `"license": "MIT"`. — [LICENSE](https://github.com/obra/superpowers/blob/v6.4.2/LICENSE); [.claude-plugin/plugin.json](https://github.com/obra/superpowers/blob/v6.4.2/.claude-plugin/plugin.json)
- The skills directory at v6.4.2 holds 15 skills:
  - brainstorming
  - diagnosing-superpowers
  - dispatching-parallel-agents
  - executing-plans
  - finishing-a-development-branch
  - receiving-code-review
  - requesting-code-review
  - subagent-driven-development
  - systematic-debugging
  - test-driven-development
  - using-git-worktrees
  - using-superpowers
  - verification-before-completion
  - writing-plans
  - writing-skills

  Source: [skills/](https://github.com/obra/superpowers/tree/v6.4.2/skills)

**Bootstrap mechanism**
- `hooks/hooks.json` registers a Claude Code `SessionStart` hook with matcher `startup|clear|compact`. The hook runs `hooks/session-start`. That script reads `skills/using-superpowers/SKILL.md` in full and emits it as `additionalContext`, wrapped in `<EXTREMELY_IMPORTANT>You have superpowers...`. The JSON field name changes by platform: Cursor uses `additional_context`, Claude Code uses `hookSpecificOutput.additionalContext`, and Copilot CLI and others use top-level `additionalContext`. — [hooks/hooks.json](https://github.com/obra/superpowers/blob/v6.4.2/hooks/hooks.json); [hooks/session-start](https://github.com/obra/superpowers/blob/v6.4.2/hooks/session-start)
- `using-superpowers` contains the "1% rule": "If you think there is even a 1% chance a skill might apply ... you ABSOLUTELY MUST invoke the skill". It also has a `<SUBAGENT-STOP>` block, so dispatched subagents ignore the bootstrap. It sets precedence as user instructions (CLAUDE.md/AGENTS.md) > skills > default system prompt. — [skills/using-superpowers/SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/using-superpowers/SKILL.md)
- The porting guide says every integration needs three pieces: harness-agnostic skills, a bootstrap at session start, and per-harness tool mapping. Its "one rule" is to load the bootstrap at session start. The acceptance test is a clean session where the message "Let's make a react todo list" must auto-trigger `brainstorming`. — [docs/porting-to-a-new-harness.md](https://github.com/obra/superpowers/blob/v6.4.2/docs/porting-to-a-new-harness.md); [AGENTS.md](https://github.com/obra/superpowers/blob/v6.4.2/AGENTS.md)

**Installation by harness (v6.4.2 README)**
- Claude Code:
  - `/plugin install superpowers@claude-plugins-official` (Anthropic official marketplace)
  - or `/plugin marketplace add obra/superpowers-marketplace`, then `/plugin install superpowers@superpowers-marketplace`
- Codex App and CLI: via OpenAI's official Codex plugin marketplace.
- Cursor: `/add-plugin superpowers`.
- Gemini CLI: `gemini extensions install https://github.com/obra/superpowers`.
- GitHub Copilot CLI: `copilot plugin install superpowers@superpowers-marketplace`.
- OpenCode: "Fetch and follow instructions from .../.opencode/INSTALL.md".
- Pi: `pi install git:github.com/obra/superpowers`.
- Also documented: Antigravity (`agy plugin install`), Devin CLI, Factory Droid, Grok Build CLI (`superpowers@xai-official`), Kimi Code, Qwen Code, Hermes Agent and Muse.

Source: [README](https://github.com/obra/superpowers/blob/v6.4.2/README.md)

- The repo root carries manifests for each harness: `.claude-plugin/` (plugin.json and a dev marketplace.json), `.codex-plugin/`, `.cursor-plugin/`, `.devin-plugin/`, `.hermes-plugin/`, `.kimi-plugin/`, `.muse-plugin/`, `.opencode/`, `.pi/`, `.agents/plugins/` and `gemini-extension.json`. `.version-bump.json` keeps 11 version fields in sync. — [.version-bump.json](https://github.com/obra/superpowers/blob/v6.4.2/.version-bump.json)

**Relationship to Anthropic's marketplace and Skills**
- Anthropic's official marketplace lists `superpowers` (category "development") with source `https://github.com/obra/superpowers.git`, pinned to sha `5bf4e780...`. That sha is the v6.4.1 release commit of 2026-09-18, so the official listing trailed v6.4.2 by one patch on the day checked. The listing's description reads: "Superpowers teaches Claude brainstorming, subagent driven development with built in code review, systematic debugging, and red/green TDD..." — [anthropics/claude-plugins-official marketplace.json](https://github.com/anthropics/claude-plugins-official/blob/main/.claude-plugin/marketplace.json)
- Superpowers was accepted into the official marketplace on 2026-01-15 **[snippet]**. Jesse Vincent posted on Threads: "Oh. Anthropic is now including Superpowers in the official plugins marketplace. Rock on." — [pasqualepillitteri.it](https://pasqualepillitteri.it/en/news/215/superpowers-claude-code-complete-guide); [Threads @obrajesse](https://www.threads.com/@obrajesse/post/DTj1uJFlVms/oh-anthropic-is-now-including-superpowers-in-the-official-plugins-marketplace)
- Jesse's own `obra/superpowers-marketplace` also carries several related plugins:
  - superpowers-chrome
  - elements-of-style
  - episodic-memory ("memory that persists between sessions")
  - superpowers-lab
  - superpowers-developing-for-claude-code
  - superpowers-dev
  - claude-session-driver ("Launch, control, and monitor other Claude Code sessions as workers via tmux")
  - private-journal-mcp
  - double-shot-latte ("Stop 'Would you like me to continue?' interruptions")

  Source: [obra/superpowers-marketplace marketplace.json](https://github.com/obra/superpowers-marketplace/blob/main/.claude-plugin/marketplace.json)
- Timing relative to Anthropic Skills:
  - The initial commit "Superpowers plugin v1.0.0" is dated 2025-10-09, the same day as the announcement post. — [git history](https://github.com/obra/superpowers/commits/v6.4.2); [blog.fsck.com/2025/10/09/superpowers](https://blog.fsck.com/2025/10/09/superpowers/)
  - Anthropic announced Agent Skills on 2025-10-16 **[snippet]**. — [Anthropic engineering](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills); [howaiworks.ai](https://howaiworks.ai/blog/anthropic-agent-skills-announcement)
  - Snippets credit Vincent with using the SKILL.md format about a week before Anthropic's native framework shipped "using the same format" **[snippet]**. That snippet also misdates Superpowers to "October 2024". — [tomrochette.com](https://tomrochette.com/agents/people-and-publications/jesse-vincent/)
  - Anthropic made Agent Skills an open standard on 2025-12-18 **[snippet]**. — [SiliconANGLE](https://siliconangle.com/2025/12/18/anthropic-makes-agent-skills-open-standard/)
  - Since v5.0.1 ("Agentskills Compliance"), writing-skills has pointed to the agentskills.io spec for frontmatter fields. — [RELEASE-NOTES v5.0.1/v5.0.6](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- `writing-skills` ships Anthropic's own skill-authoring guidance as `anthropic-best-practices.md` (46 KB) alongside its own TDD-for-skills method. — [skills/writing-skills/](https://github.com/obra/superpowers/tree/v6.4.2/skills/writing-skills)

**Version history (tag commit dates from git; release-notes headers in parentheses where they differ)**
- **v1.0.0 to v3.x (October to December 2025; older):**
  - Initial plugin on 2025-10-09. v3.1.0 on 2025-10-17.
  - v3.3.0 (2025-10-28) added experimental Codex support via a `superpowers-codex` Node CLI.
  - v3.5.0 (2025-11-23) added OpenCode.
  - In this era, skills briefly lived in a separate `obra/superpowers-skills` repo.

  Source: [RELEASE-NOTES](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- **v4.0.0 (2025-12-17):**
  - Two-stage review in subagent-driven development (SDD): a spec-compliance reviewer, then a code-quality reviewer.
  - DOT/Graphviz flowcharts became "executable specifications".
  - "The Description Trap" finding: descriptions must say when to trigger and must not summarize the workflow.
  - Skill consolidation, for example root-cause-tracing folded into systematic-debugging.

  Source: [RELEASE-NOTES v4.0.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- **v4.1 and v4.2 (January to February 2026):** OpenCode and Codex moved to native skill discovery. — [RELEASE-NOTES](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- **v5.0.0 (2026-03-09):**
  - Specs moved to `docs/superpowers/specs/` and plans to `docs/superpowers/plans/`.
  - SDD became mandatory on harnesses with subagents.
  - executing-plans stopped batching ("execute 3 tasks then stop").
  - Slash commands were deprecated.
  - Added the visual brainstorming companion, implementer status codes (DONE / DONE_WITH_CONCERNS / BLOCKED / NEEDS_CONTEXT), and model-selection guidance.

  Source: [RELEASE-NOTES v5.0.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- **v5.0.6 (2026-03-24/25):** subagent spec and plan review loops were replaced with inline self-review. The loops "doubled execution time (~25 min overhead) without measurably improving plan quality". — [RELEASE-NOTES v5.0.6](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- **v5.1.0 (notes 2026-04-30, tag and blog 2026-05-04):**
  - Removed the `/brainstorm`, `/write-plan` and `/execute-plan` commands and the named `code-reviewer` agent.
  - Rewrote the worktree skills; they now ask consent before creating a worktree.
  - Added AI-contributor rules after "a 94% rejection rate driven by AI-generated slop".

  Source: [RELEASE-NOTES v5.1.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md); [blog.fsck.com/2026/05/04/superpowers-5.1](https://blog.fsck.com/2026/05/04/superpowers-5.1/) **[snippet]**
- **v6.0.0 (2026-06-16):**
  - SDD review rewritten: one `task-reviewer-prompt.md` returning two verdicts, diffs handed over as files (`task-brief` and `review-package` scripts), required model naming, read-only reviewers, a progress ledger, and a final whole-branch review on the most capable model.
  - Plans gained a Global Constraints block and per-task Interfaces blocks.
  - Added Kimi, Pi and Antigravity.
  - Skills rewritten in vendor-neutral vocabulary.
  - The evals moved into a separate "drill" harness.
  - Claimed "roughly twice as fast and ... almost 50% fewer tokens" in their evals.

  Source: [RELEASE-NOTES v6.0.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- **v6.0.3 (2026-06-18):** SDD scratch moved from `.git/sdd/` to a self-ignoring `.superpowers/sdd/`, because Claude Code blocks agent writes under `.git/`. — [RELEASE-NOTES v6.0.3](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- **v6.1.0 (2026-06-30):** the bootstrap was trimmed to lower per-session token cost. Gemini CLI was removed, citing "Google EOLed the Gemini CLI on 2026-06-18". — [RELEASE-NOTES v6.1.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- **v6.2.0 (2026-07-23):**
  - SDD workspace became plan-scoped: `.superpowers/sdd/<plan-basename>/`.
  - The fix loop resumes the original implementer, with a five-round circuit breaker.
  - `testing-anti-patterns.md` became `writing-good-tests.md`.
  - The finishing menu no longer offers "discard".
  - Gemini support was restored ("removal ... was premature").

  Source: [RELEASE-NOTES v6.2.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- **v6.3.0 (2026-08-12):**
  - Brainstorming classifies each request as spike, bounded or architectural.
  - SDD makes "rulings, not stalls": a donated session "had sat blocked for almost nine hours on a question the controller could have decided".
  - Same-shape tasks are batched into one dispatch.
  - Implementers and reviewers may not spawn subagents.
  - Plans carry a `Spec:` pointer.
  - Added Devin, Hermes and Grok.

  Source: [RELEASE-NOTES v6.3.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- **v6.4.1 (2026-09-18; v6.4.0 never shipped):**
  - Added the `diagnosing-superpowers` skill.
  - `executing-plans` rebuilt as "Native" inline execution with one final review; it "no longer stops every few tasks to check in".
  - The user must review the saved plan before anything runs.
  - Plans carry a Review Focus section.
  - Added OpenCode 2.0, Muse and Qwen.

  Source: [RELEASE-NOTES v6.4.1](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md); [blog.fsck.com/2026/09/21/superpowers-6.4](https://blog.fsck.com/2026/09/21/superpowers-6.4/) **[snippet]**
- **v6.4.2 (2026-09-25):**
  - Plans now "record decisions", not code transcripts.
  - The cause given: some frontier models, "including Opus 5.5 ... would sometimes try to implement the entire project while designing the plan".
  - Plans took "a quarter of the time and about a third of the tokens".
  - CLAUDE.md was removed in favour of AGENTS.md.

  Source: [RELEASE-NOTES v6.4.2](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)

### Inferences
- "Plugin" and "skills library" are not separate things. The skills are the product, and the per-harness plugin wrappers mostly exist to inject one bootstrap skill at session start. The skills would still be usable as plain SKILL.md content without the bootstrap. They would just not trigger automatically.
- The release cadence is high: 36 tags in under 12 months, with frequent behavioural rewrites driven by evals. Anyone vendoring the content should pin a tag or sha, as Anthropic's marketplace does. They should also expect upstream semantics to drift, for example who gates plan approval and whether executing-plans pauses.

### Gaps
- The full text of Jesse Vincent's blog posts could not be read (blog.fsck.com was blocked). That includes the v6.0 announcement and any post on design philosophy.
- I could not confirm from a primary Anthropic source the exact date Superpowers entered `claude-plugins-official`. The repo clone was shallow, so the commit history of the listing was not inspected.

## 2. The workflow and skills: what each skill enforces, which artifacts it writes, where it needs human approval, and how it handles TDD, verification, fresh reviewers and parallel agents

### Takeaway
The pipeline runs:
1. brainstorming: an intent interview ending in a spec in `docs/superpowers/specs/`
2. using-git-worktrees
3. writing-plans: a plan in `docs/superpowers/plans/`
4. subagent-driven-development or executing-plans, both keeping a ledger in `.superpowers/sdd/<plan>/`
5. TDD and verification on every task, plus code review
6. finishing-a-development-branch

Human approval is required at fixed in-chat points: the path classification, design sections, the written spec, the written plan plus choice of execution method, worktree consent, and the final merge/PR choice. Once execution starts, the skills deliberately avoid stopping. They stop only for destructive, security-sensitive or outside-the-worktree actions, or for a plan so broken that every path forward is a guess. Enforcement is entirely by prompt: "Iron Laws", rationalization tables and red-flag lists. Nothing is mechanically gated, apart from small helper scripts. One example is `task-done`, which refuses to write to the ledger if the tests fail.

### Cited Findings
**brainstorming** — [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/brainstorming/SKILL.md)
- Classification: it first sorts the request into **Spike**, **Bounded** or **Architectural** and says the classification aloud. "When in doubt ... take the heavier one." The ratchet only goes up.
- Intent: it discovers intent, writes back its understanding, and asks one question per message, preferring multiple choice. It then proposes 2–3 approaches and presents the design in sections of up to 200–300 words, getting approval after each section.
- `<HARD-GATE>`: there is no implementation action until the path's prerequisites are met. For Architectural work the human approves the written spec, then reviews the written implementation plan and selects its execution method. Quote: "Conversational design approval only permits writing the spec; written-spec approval only permits invoking writing-plans."
- Artifact: `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`, committed to git ("User preferences for spec location override this default"). An inline Spec Self-Review follows (placeholders, consistency, scope, ambiguity), then the gate: "Wait for the user's response."
- Spike and Bounded paths produce no spec file. Bounded presents a short in-chat design, then "STOP" until an explicit yes.
- The terminal state for Architectural is invoking writing-plans only.
- Optional "visual companion": a local WebSocket server with a per-session key. It is offered just-in-time. Its logo load doubles as opt-out telemetry (`SUPERPOWERS_DISABLE_TELEMETRY`). — [README](https://github.com/obra/superpowers/blob/v6.4.2/README.md); [RELEASE-NOTES v6.0.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)

**using-git-worktrees** — [SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/using-git-worktrees/SKILL.md)
- It detects existing isolation (`GIT_DIR != GIT_COMMON`, with a submodule guard) and asks consent before creating a worktree.
- It prefers native tools (for example `EnterWorktree`). The fallback is `git worktree add` under `.worktrees/`, after verifying the directory is git-ignored.
- It runs project setup (npm, cargo, pip/poetry, go) and a clean test baseline. If the baseline fails, it asks the human.

**writing-plans** — [SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/writing-plans/SKILL.md)
- Artifact: `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`; user preference overrides the location.
- Mandatory header names the REQUIRED SUB-SKILL (SDD or executing-plans) and gives Goal, Architecture, Tech Stack and a `Spec:` path. Then come **Global Constraints** (exact values copied verbatim) and **Review Focus** (up to five spec-implied failure modes, each pinned by a test).
- Tasks carry Files (Create/Modify/Test with line ranges), **Interfaces** (Consumes/Produces with exact signatures) and checkbox steps: write failing test, run it and see FAIL, implement a signature, run it and see PASS, commit.
- Since v6.4.2, a code step gives "the exact signature ... the file ... the specific values"; bodies appear only for algorithms. A self-review covers spec coverage, step scan, type consistency, Review Focus and proportion.
- Handoff: "Please review the plan. Which execution approach would you prefer?" The options are Subagent-driven and Native, and the skill recommends one.

**subagent-driven-development (SDD)** — [SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/subagent-driven-development/SKILL.md); [implementer-prompt.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/subagent-driven-development/implementer-prompt.md); [task-reviewer-prompt.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/subagent-driven-development/task-reviewer-prompt.md)
- Continuous execution: "Do not pause to check in with your human partner between tasks."
- "Rulings, not stalls": conflicts and ambiguities are decided by the controller and ledgered as `Ruling: <what> — <why> — <what it costs if wrong>`. There are four stop conditions:
  - an irreversible or destructive operation;
  - a security-sensitive action;
  - a side effect outside the worktree, such as a merge, a push to a shared branch or a publish;
  - a plan so broken that every path forward is a guess.
- Setup: a worktree, then `bash scripts/sdd-workspace PLAN_FILE`, which resolves `.superpowers/sdd/<plan-basename>/`. Next comes the ledger `progress.md` (first line `# SDD ledger — plan: <path>`), reading the plan and spec, and a pre-flight conflict scan written as a table into the ledger.
- Per task:
  - Record BASE.
  - `task-brief` extracts the task text to a file.
  - Dispatch a fresh implementer, always with an explicit model.
  - The implementer writes a report file that includes RED/GREEN TDD evidence (command plus failing output, then command plus passing output).
  - `review-package PLAN BASE HEAD` writes the diff to a file.
  - A fresh, read-only **task reviewer** returns a spec-compliance verdict (✅/❌/⚠️ "cannot verify from diff") and a quality verdict, grading issues Critical, Important or Minor.
- Fix loop: up to 5 rounds. Rounds 1–3 resume the same implementer; rounds 4–5 use a fresh implementer on a more capable model. Each round gets a scoped re-review. When the breaker trips, the controller adjudicates and parks findings with rulings.
- Rules: the controller may never pre-judge findings for the reviewer ("do not flag", "at most Minor"). It must "Never fix findings yourself in the controller session". It must "Never dispatch multiple implementation subagents in parallel (conflicts)". Implementers and reviewers may not spawn subagents.
- End of run: a final whole-branch review on the most capable model using `requesting-code-review/code-reviewer.md`, then ONE fix dispatch and one re-review.
- The final message lists "Rulings I made". The workspace is then deleted ("git history is the record now"), followed by finishing-a-development-branch.

**executing-plans ("Native")** — [SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/executing-plans/SKILL.md); [scripts/task-start](https://github.com/obra/superpowers/blob/v6.4.2/skills/executing-plans/scripts/task-start); [scripts/task-done](https://github.com/obra/superpowers/blob/v6.4.2/skills/executing-plans/scripts/task-done)
- The session implements every task itself under the same ledger, workspace and stop rules as SDD, followed by one fresh whole-branch review.
- `task-done PLAN N BASE -- <test cmd>` runs the tests and appends `Task N: complete (commits a..b, tests: <cmd> → <last line>)` to the ledger only when the exit status is 0.
- The workspace and ledger are shared with SDD, "so a plan can change executors mid-flight".

**test-driven-development** — [SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/test-driven-development/SKILL.md)
- Iron Law: "NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST. Write code before the test? Delete it. Start over."
- Verify RED is "MANDATORY. Never skip." Exceptions (throwaway prototypes, generated code, configuration) require asking the human partner.
- Since v6.4.1, "The project's suite defines green, not just your test file." — [RELEASE-NOTES v6.4.1](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- The reference doc `writing-good-tests.md` covers falsifiability and mutation checks and names the "string-presence trap". — [RELEASE-NOTES v6.2.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)

**verification-before-completion** — [SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/verification-before-completion/SKILL.md)
- Iron Law: "NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE."
- Gate function: IDENTIFY the command, RUN it in full, READ the output and exit code, VERIFY, and only then claim.
- Its table maps each claim to the evidence it needs:
  - "tests pass" needs 0 failures in the output;
  - "linter clean" needs linter output with 0 errors;
  - "build succeeds" needs exit 0;
  - "agent completed" needs a VCS diff, not the agent's report;
  - "requirements met" needs a line-by-line checklist.

**requesting-code-review** — [SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/requesting-code-review/SKILL.md); [code-reviewer.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/requesting-code-review/code-reviewer.md)
- It dispatches a `general-purpose` subagent with a template taking `{DESCRIPTION}`, `{PLAN_OR_REQUIREMENTS}`, `{BASE_SHA}` and `{HEAD_SHA}`: "never your session's history".
- Review is mandatory after each SDD task, after a major feature, and before merging to main. Critical issues are fixed immediately and Important ones before proceeding.

**receiving-code-review** — [SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/receiving-code-review/SKILL.md)
- Steps: READ, UNDERSTAND, VERIFY against the codebase, EVALUATE, RESPOND, then IMPLEMENT one item at a time.
- It forbids "You're absolutely right!" and other performative agreement. If any item is unclear, the agent stops and asks before implementing anything.

**dispatching-parallel-agents** — [SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/dispatching-parallel-agents/SKILL.md)
- "Dispatch one agent per independent problem domain." It is meant for 2+ independent failures or subsystems with no shared state. It is about investigation and debugging fan-out, not plan execution. SDD forbids parallel implementers.

**finishing-a-development-branch** — [SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/finishing-a-development-branch/SKILL.md)
- It verifies the full suite first. It then presents exactly three options: merge locally, push and create a PR (any forge), or keep as-is. On a detached HEAD there are two. "The integration decision is theirs."
- Discard only happens on explicit request, with typed confirmation. Cleanup is provenance-based and touches only `.worktrees/`.

**systematic-debugging** — [SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/systematic-debugging/SKILL.md)
- Iron Law: "NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST." It has four phases and bundles `root-cause-tracing.md`, `defense-in-depth.md`, `condition-based-waiting.md` and `find-polluter.sh`.

**writing-skills** — [SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/writing-skills/SKILL.md)
- "Writing skills IS Test-Driven Development applied to process documentation". It uses pressure scenarios with subagents: watch the agent fail without the skill, write the skill, then close loopholes.

**diagnosing-superpowers** (new in v6.4.1) — [SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/diagnosing-superpowers/SKILL.md)
- It reads session transcripts on disk and reports findings that each cite `path:line`. Optionally it builds a scrubbed bundle or drafts a GitHub issue.

**Persuasion-based enforcement**
- Rules are enforced through persuasion-based prompt design. `writing-skills/persuasion-principles.md` is part of the library. — [skills/writing-skills/](https://github.com/obra/superpowers/tree/v6.4.2/skills/writing-skills)
- Commentators link this to Cialdini-style principles **[snippet]**. — [DEV: "The Technology to 'Persuade' AI Agents"](https://dev.to/tumf/superpowers-the-technology-to-persuade-ai-agents-why-psychological-principles-change-code-quality-2d2f)

| Skill                          | Artifact (default path)                                            | Human gate                                                     |
| ------------------------------ | ------------------------------------------------------------------ | -------------------------------------------------------------- |
| brainstorming                  | `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` (committed)  | Path classification, design sections, written-spec approval    |
| using-git-worktrees            | `.worktrees/<branch>` (or native tool)                             | Consent to create; proceed past failing baseline               |
| writing-plans                  | `docs/superpowers/plans/YYYY-MM-DD-<feature>.md`                   | Review saved plan and choose Subagent-driven vs Native         |
| subagent-driven-development    | `.superpowers/sdd/<plan>/` ledger, briefs, reports, review diffs   | None mid-run except 4 stop classes; "Rulings I made" at end    |
| executing-plans                | Same workspace/ledger; `task-N-tests.log`                          | Same as SDD                                                    |
| finishing-a-development-branch | Merge, PR or kept branch; workspace deleted                        | Choose merge / PR / keep; confirm base branch                  |

### Inferences
- Superpowers already encodes an "evidence contract" in prose:
  - RED/GREEN evidence in the implementer report;
  - `task-done` ledger lines with the test command and its last output line;
  - reviewer verdicts with file:line citations;
  - verification-before-completion's claim→evidence table.

  These map closely onto a harness evidence schema, but as markdown rather than structured data.
- The skills cover brief/spec, plan and implementation. They have no separate "design", "stories" or "release" stages. Design is folded into the brainstorming spec. Stories roughly correspond to plan tasks. Release stops at merge/PR.
- Lint and docs are not first-class gates. Docs are folded into tasks ("fold ... documentation steps into the task whose deliverable needs them"). Lint shows up only in verification-before-completion's evidence table.

### Gaps
- I did not read the full `implementer-prompt.md`, `task-reviewer-prompt.md`, `code-reviewer.md` or `writing-good-tests.md`; only excerpts were grepped. A harness integrator should read them in full before reusing them.
- There is no quantitative independent evaluation of skill compliance rates. The only numbers are the project's own eval claims in its release notes.

## 3. Autonomy and durability: unattended runs, state between sessions, recovery after a session dies, and long-running modes

### Takeaway
Superpowers is designed for one interactive session with a human present at the front (brainstorm, spec, plan approval) and at the back (merge/PR choice). Between those points it runs autonomously for "a couple hours". State is kept in files and git, not in an engine:
- spec and plan markdown, with checkbox steps;
- a git-ignored per-plan ledger at `.superpowers/sdd/<plan>/progress.md`;
- briefs, reports and review diffs;
- commits.

Resuming after compaction or a crash relies on the agent re-reading the ledger and `git log`. There is no scheduler, no durable queue, no external approval mechanism and no supported headless or unattended mode in the core plugin. Adjacent marketplace plugins (claude-session-driver, double-shot-latte) go toward multi-session control and continuation.

### Cited Findings
- The README claims: "It's not uncommon for your agent to work autonomously for a couple hours at a time without deviating from the plan you put together." — [README](https://github.com/obra/superpowers/blob/v6.4.2/README.md)
- SDD on context loss:
  - "Conversation memory does not survive compaction. In real sessions, controllers that lost their place have re-dispatched entire completed task sequences — the single most expensive failure observed. Track progress in a ledger file, not only in todos."
  - "After compaction, trust the ledger and `git log` over your own recollection."
  - Resume rule: tasks with `Task <N>: complete` are DONE, and a task whose last line is a fix round resumes at the next round.

  Source: [subagent-driven-development/SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/subagent-driven-development/SKILL.md)
- Fragility: "`git clean -fdx` will destroy the workspace (it's git-ignored scratch); if that happens, recover from `git log`." The workspace is deleted when the final review is clean. — [SDD SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/subagent-driven-development/SKILL.md); [RELEASE-NOTES v6.0.3](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- Plan-scoping (v6.2.0) was introduced because "a follow-up plan in the same working tree could read the previous plan's ledger as its own progress". — [RELEASE-NOTES v6.2.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- The bootstrap is re-injected on `startup|clear|compact`. On Hermes, which has no post-compaction hook, "a very long session that compacts over its first turn loses the bootstrap — start a fresh session". — [hooks/hooks.json](https://github.com/obra/superpowers/blob/v6.4.2/hooks/hooks.json); [README](https://github.com/obra/superpowers/blob/v6.4.2/README.md)
- Plans use checkbox syntax "for progress tracking" (since v5.0.0). Harness todos are a "live view; the ledger is the record". — [RELEASE-NOTES v5.0.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md); [executing-plans/SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/executing-plans/SKILL.md)
- Movement toward fewer stops:
  - v5.0.0 removed "execute 3 tasks then stop".
  - v5.1.0 removed "pause every 3 tasks" from SDD.
  - v6.3.0 introduced "rulings, not stalls", citing a session "blocked for almost nine hours".
  - v6.4.1 says executing-plans "no longer stops every few tasks".

  Source: [RELEASE-NOTES](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- On Claude Code, the SDD controller can opt in to running "one layer down, as a nested subagent on a mid-tier model". This measured "about half the cost and wall clock". — [RELEASE-NOTES v6.4.1](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- The repo's skills contain no headless or `claude -p` mode. A grep for headless, unattended and autonomous found only the README sentence above and a browser-launch fallback. The project uses `claude -p` only in its own tests. — [RELEASE-NOTES v4.0.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md) (tests/claude-code uses `claude -p`)
- Related marketplace plugins: `claude-session-driver` ("Launch, control, and monitor other Claude Code sessions as workers via tmux"), `double-shot-latte` ("Stop 'Would you like me to continue?' interruptions ... Claude-judged decision making") and `episodic-memory` (cross-session conversation search). — [superpowers-marketplace](https://github.com/obra/superpowers-marketplace/blob/main/.claude-plugin/marketplace.json)
- Third-party framing **[snippet]**: "Superpowers optimizes for hands-off execution where you brainstorm, plan, then walk away while subagents implement and review", in contrast with Addy Osmani's Agent Skills, which put "a human checkpoint at every phase". — [rywalker.com](https://rywalker.com/research/agentic-skills-frameworks)

### Inferences
- If a session dies mid-plan, the committed spec, the plan and the per-task commits survive. The git-ignored ledger survives on disk unless the worktree or container is lost. In an ephemeral container per activity, the ledger disappears with the container unless the harness persists `.superpowers/sdd/` or reconstructs progress from `git log`.
- The plan → per-task ledger → commit structure is effectively a hand-rolled, file-based checkpoint log. A Kruxia Flow workflow could take this over as engine state, with one activity per task, and keep the ledger as a secondary human-readable record.
- In-flight human questions (BLOCKED, the four stop classes) assume a person is in the chat. Under unattended `claude -p` they would end the session or block it. A harness needs to turn them into structured "needs human" activity outcomes.

### Gaps
- I found no official documentation for running Superpowers non-interactively, for example with the Agent SDK or `claude -p` in CI, or for resuming SDD in a brand-new session from the ledger. The SKILL.md resume rules imply it works but give no test or guide.
- I did not inspect whether `claude-session-driver` or `double-shot-latte` are maintained or how they interact with SDD.

## 4. Adoption and reception: verified star count, endorsements and critiques, and comparison with Spec Kit, BMAD and Anthropic's feature-dev plugin

### Takeaway
The roughly 295k-star figure appears to be real, not a rendering artifact:
- A WebFetch of github.com/obra/superpowers on 2026-10-03 shows "294.8k stars", "26.3k forks", "1.1k watching" and "683 Commits".
- The commit count matches the cloned repo exactly, which shows the page was current.
- Third-party snippets give a steep, consistent trajectory: about 57.5k (February 2026), about 150k (April), 224.7k (June), 276.3k (August 23) and about 295k.

The API could not be queried, so the count is not confirmed from a second primary source. Simon Willison's October 2025 endorsement is the best-known early one. The main critique is token and time cost on small tasks.

### Cited Findings
**Star count**
- The GitHub page fetched 2026-10-03 reads "**294.8k** stars", "**26.3k** forks", "**1.1k** watching", "683 Commits". — [github.com/obra/superpowers](https://github.com/obra/superpowers)
  - The 683 commits equal `git rev-list --count HEAD` on the clone at `v6.4.2`.
  - `api.github.com` returned 403 through the session proxy, so the count could not be confirmed via the API.
- Trajectory from third parties, all **[snippet]**:
  - 57.5K in February 2026 and 224.7K as of June 2026, plus "787K installs through Anthropic's official plugin marketplace" and eight harnesses. — [rywalker.com](https://rywalker.com/research/agentic-skills-frameworks)
  - "150,000 stars, 13,000 forks, 28 contributors" as of April 2026. — [Medium, Anil Mathew](https://medium.com/@anilmathewm/i-gave-claude-code-a-brain-its-called-superpowers-and-it-has-150-000-github-stars-for-a-reason-16c4074a9209)
  - 211K stars, cited in a third-party issue. — [ksimback/hermes-ecosystem#293](https://github.com/ksimback/hermes-ecosystem/issues/293)
  - 276.3k stars as of August 23, 2026, and "about 295,000". — [gitstarclub.com](https://gitstarclub.com/obra/superpowers)
- The blog calls it "the most-used Claude Code plugin in the world" **[snippet]**. This is a self-description. — [blog.fsck.com](https://blog.fsck.com/2026/09/21/superpowers-6.4/)

**Endorsements**
- **Simon Willison**, 2025-10-10, "Superpowers: How I'm using coding agents in October 2025" **[snippet]**. He calls Jesse "one of the most creative users of coding agents". He notes the use of red/green TDD, planning, self-updating memory notes and a "feelings journal", and that Graphviz DOT graphs act as process instructions. He says the system is "very token light", pulling in fewer than 2k tokens of docs and using subagents for heavy work: "approximately 100k tokens for a complete project". — [simonwillison.net/2025/Oct/10/superpowers](https://simonwillison.net/2025/Oct/10/superpowers/); HN thread [45547344](https://news.ycombinator.com/item?id=45547344)
  - This describes v1-era behaviour (October 2025). Token use changed a lot later.
- HN "A Rave Review of Superpowers (For Claude Code)" exists. Its content was not readable. — [news.ycombinator.com/item?id=47623101](https://news.ycombinator.com/item?id=47623101)
- Jesse Vincent appeared on the Heavybit "Open Source Ready" podcast (ep. 36) **[snippet]**. — [heavybit.com](https://www.heavybit.com/library/podcasts/open-source-ready/ep-36-managing-ai-coding-agents-with-jesse-vincent)

**Critiques**
- Cost and overhead **[snippet]**:
  - "Superpowers fixes Claude Code. Then it bills you for every two-line fix." — [dev.to](https://dev.to/aidiveyt/superpowers-fixes-claude-code-then-it-bills-you-for-every-two-line-fix-6dg)
  - Users report it "burned through all my max plan" and that simple fixes "take literally an hour with all the verification". — [joanmedia.dev](https://www.joanmedia.dev/ai-blog/the-honest-tradeoffs-of-superpowers-token-costs-overkill-and-the-alternatives); [mcp.directory](https://mcp.directory/blog/superpowers-skill-worth-it-2026)
  - One HN commenter says "eventually, Plan mode became enough". — [HN 47418626](https://news.ycombinator.com/item?id=47418626)
- The project itself acknowledges cost:
  - v5.0.6 dropped subagent doc-review loops for being slow.
  - v6.0.0 claims about 50% fewer tokens.
  - v6.1.0 trimmed the bootstrap "because its size is paid for constantly".
  - v6.3.0 says "small tasks skip the two-document ritual".
  - v6.4.2 says plan writing over-generated with Opus 5.5.

  Source: [RELEASE-NOTES](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- Contributor-quality friction: an audit found "a 94% rejection rate driven by AI-generated slop" across the last 100 closed PRs. AGENTS.md now requires disclosure of model, harness and plugins, and does "not generally accept contributions of new skills". — [RELEASE-NOTES v5.1.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md); [README](https://github.com/obra/superpowers/blob/v6.4.2/README.md); [AGENTS.md](https://github.com/obra/superpowers/blob/v6.4.2/AGENTS.md)

**Comparisons** (third-party, **[snippet]**)
- "Spec Kit plans WHAT to build, while Superpowers controls HOW it gets built."
  - Spec Kit centres on a constitution/spec artifact reviewed through GitHub PRs, built for stakeholders who never open the agent.
  - BMAD splits the lifecycle into phases run by named role-agents (PM, architect, UX, dev, QA).
  - Suggested choice: Spec Kit if the spec must live long-term in Git or be read by non-agent users; BMAD when several roles or a PM stakeholder are in the loop; otherwise Superpowers.

  Source: [dev.to Spec Kit vs Superpowers](https://dev.to/truongpx396/spec-kit-vs-superpowers-a-comprehensive-comparison-practical-guide-to-combining-both-52jj); [claude-codex.fr methodologies](https://claude-codex.fr/en/advanced/methodologies-ecosystem/); [felipefontoura.com (Sep 2026)](https://felipefontoura.com/articles/spec-driven-development-tools-compared/); [docs.bswen.com (2026-08-07)](https://docs.bswen.com/blog/2026-08-07-ai-spec-frameworks-compared/)
  - One snippet claims Superpowers "requires coverage > 80% or the PR is blocked". **No such rule exists in the v6.4.2 skills.** A grep for "coverage" finds only reviewer wording. Treat that aggregator claim as inaccurate.
- Superpowers is "recommended for solo/small teams wanting full methodology enforcement, BMAD for ... agile lifecycle coverage, Spec Kit for spec-driven greenfield, OpenSpec for brownfield". Superpowers and Matt Pocock's skills sit at the "composable end": you can use worktree isolation without TDD enforcement. — [rywalker.com](https://rywalker.com/research/agentic-skills-frameworks)
- **Anthropic feature-dev** (official marketplace plugin, read directly) **[primary]**:
  - It is a single `/feature-dev` command with 7 phases: Discovery, Codebase Exploration (2–3 parallel `code-explorer` agents), Clarifying Questions, Architecture Design (`code-architect` agents; "Ask user which approach they prefer"), Implementation ("DO NOT START WITHOUT USER APPROVAL"), Quality Review (`code-reviewer` agents) and Summary.
  - Compared with Superpowers it has no TDD Iron Law, no persisted spec or plan files, no ledger, no worktree management and no per-task fresh-implementer and reviewer loop.
  - It uses TodoWrite for tracking and fans out exploration and review in parallel.

  Source: [claude-plugins-official/plugins/feature-dev/commands/feature-dev.md](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/feature-dev/commands/feature-dev.md)

### Inferences
- The stars-per-commit and fork ratios are unusual. Still, the commit-count match plus several independent trackers on a rising curve make "294.8k" the best available figure as of 2026-10-03. Report it as "about 295k per the GitHub UI, not API-verified".
- Superpowers' distinctive position is execution discipline (TDD, review loops, evidence) more than lifecycle breadth (BMAD) or durable stakeholder artifacts (Spec Kit).

### Gaps
- None of the critique or comparison articles could be read in full, because of proxy blocks. Their claims are snippet-level.
- The "787K installs" figure has only one snippet source and no primary confirmation.
- The full text of Simon Willison's post, including any caveats he raised, could not be read.

## 5. Fit with the harness: what can be reused as stage-agent content, what conflicts, and whether the license allows reuse

### Takeaway
The license is MIT, so content can be reused freely with attribution. Several skills map closely onto stage-agent prompts:
- brainstorming → idea interview / brief / spec;
- writing-plans → plan;
- SDD or executing-plans plus TDD → story implementation;
- verification-before-completion → evidence contract;
- requesting-code-review and the task-reviewer prompt → independent reviewer;
- finishing-a-development-branch → integration step.

The main conflicts:
- The skills assume one long-lived interactive chat. Approvals are in-chat replies, not durable external gates.
- Parallelism and review loops live inside the session, as subagents, not in the engine.
- The skills auto-trigger and chain into each other through a bootstrap that a stage-scoped job would need to suppress or replace.
- Artifact paths (`docs/superpowers/...`, `.superpowers/sdd/`) and plan formats differ from repo conventions, although user preferences override paths.

### Cited Findings
**License and reuse terms**
- MIT, Copyright (c) 2025 Jesse Vincent. — [LICENSE](https://github.com/obra/superpowers/blob/v6.4.2/LICENSE)
- Upstream does "not generally accept contributions of new skills" and rejects "project-specific configuration" and "fork-specific changes". Harness-specific adaptations would therefore live in a fork or overlay, not upstream. — [README](https://github.com/obra/superpowers/blob/v6.4.2/README.md); [RELEASE-NOTES v5.1.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)

**Override hooks a harness can use**
- "User instructions (CLAUDE.md, AGENTS.md, GEMINI.md, etc, direct requests) take precedence over skills". Example: if CLAUDE.md says "don't use TDD", the user wins. — [using-superpowers/SKILL.md](https://github.com/obra/superpowers/blob/v6.4.2/skills/using-superpowers/SKILL.md); [RELEASE-NOTES v5.0.0](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- Paths are configurable: specs ("User preferences for spec location override this default"), plans (same wording) and worktrees ("Explicit user preference always beats observed filesystem state"). — [brainstorming](https://github.com/obra/superpowers/blob/v6.4.2/skills/brainstorming/SKILL.md); [writing-plans](https://github.com/obra/superpowers/blob/v6.4.2/skills/writing-plans/SKILL.md); [using-git-worktrees](https://github.com/obra/superpowers/blob/v6.4.2/skills/using-git-worktrees/SKILL.md)
- Execution method can be preset: "If they have already explicitly supplied an execution method ... use the preserved method". — [writing-plans](https://github.com/obra/superpowers/blob/v6.4.2/skills/writing-plans/SKILL.md)
- Subagents skip the bootstrap through `<SUBAGENT-STOP>`. A stage job could reuse that convention to stop the 1% rule from pulling in unrelated skills. — [using-superpowers](https://github.com/obra/superpowers/blob/v6.4.2/skills/using-superpowers/SKILL.md)

**Points of conflict**
- In-session human gates:
  - brainstorming's HARD-GATE requires the human partner to approve the written spec and then review the plan and select its execution method. Approval is a chat reply ("Wait for the user's response").
  - finishing-a-development-branch waits for a 1/2/3 choice.
  - using-git-worktrees asks consent and asks about a failing baseline.

  Source: [brainstorming](https://github.com/obra/superpowers/blob/v6.4.2/skills/brainstorming/SKILL.md); [finishing-a-development-branch](https://github.com/obra/superpowers/blob/v6.4.2/skills/finishing-a-development-branch/SKILL.md); [using-git-worktrees](https://github.com/obra/superpowers/blob/v6.4.2/skills/using-git-worktrees/SKILL.md)
- Chaining: the Architectural path's terminal state is "invoke writing-plans", and plans name a "REQUIRED SUB-SKILL". The skills are built to flow into each other inside one session, which crosses the harness's stage boundaries. — [brainstorming](https://github.com/obra/superpowers/blob/v6.4.2/skills/brainstorming/SKILL.md); [writing-plans](https://github.com/obra/superpowers/blob/v6.4.2/skills/writing-plans/SKILL.md)
- In-session parallelism and loops: SDD runs implementers sequentially ("Never dispatch multiple implementation subagents in parallel"). It runs review and fix loops of up to 5 rounds as in-session subagents, and the controller makes "Rulings" without a human. dispatching-parallel-agents is in-session fan-out for independent debugging domains. — [SDD](https://github.com/obra/superpowers/blob/v6.4.2/skills/subagent-driven-development/SKILL.md); [dispatching-parallel-agents](https://github.com/obra/superpowers/blob/v6.4.2/skills/dispatching-parallel-agents/SKILL.md)
- Ephemeral state: the SDD workspace is git-ignored scratch, deleted at the end ("git history is the record now"), and destroyed by `git clean -fdx`. — [SDD](https://github.com/obra/superpowers/blob/v6.4.2/skills/subagent-driven-development/SKILL.md)
- Harness-coupled assumptions: Claude Code blocks writes under `.git/` (the reason for v6.0.3). Model naming is required on every dispatch. On Claude Code the SDD controller may run nested as a subagent. — [RELEASE-NOTES](https://github.com/obra/superpowers/blob/v6.4.2/RELEASE-NOTES.md)
- Opt-out telemetry: the brainstorming visual companion loads a logo from primeradiant.com that includes the version number. It can be disabled with `SUPERPOWERS_DISABLE_TELEMETRY`, `DISABLE_TELEMETRY` or `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`. — [README](https://github.com/obra/superpowers/blob/v6.4.2/README.md); [brainstorming/scripts/server.cjs](https://github.com/obra/superpowers/blob/v6.4.2/skills/brainstorming/scripts/server.cjs)

### Inferences
Inferences for Kruxia Flow stage design. These are not claims about Superpowers itself.

**Direct reuse candidates**
- *brainstorming* for the idea interview and brief/spec stages:
  - intent discovery, one question per message, 2–3 approaches, sectioned design, spec self-review checklist;
  - its spike/bounded/architectural classifier could pick the workflow variant.
- *writing-plans* for the plan stage. Global Constraints, Review Focus, per-task Files and Interfaces, and checkbox TDD steps are already machine-parseable enough to split into per-story Kruxia activities.
  - Its `task-brief` script, which extracts one task's text, is a ready model for "story brief" generation.
- *test-driven-development* plus *verification-before-completion* as the per-story implementation contract. The implementer report's RED/GREEN evidence block and `task-done`'s rule (ledger line only on exit 0) map onto an evidence schema of test command, exit code, output tail and commit range.
- *task-reviewer-prompt.md*, *re-review-prompt.md* and *requesting-code-review/code-reviewer.md* for the independent reviewer stage:
  - read-only reviewer;
  - diff handed over as a file (`review-package BASE HEAD`);
  - spec verdict plus quality verdict;
  - Critical/Important/Minor grading;
  - the "controller may not pre-judge findings" rule;
  - file:line citations.
- *receiving-code-review* for the fix stage that consumes reviewer findings.
- *systematic-debugging* as remediation content when a story's tests fail.
- *finishing-a-development-branch*: its verify-then-choose logic, with the menu replaced by a durable release gate.

**Adapting the gates**
- In-chat approvals should become Kruxia approval activities. In practice:
  - Strip or override the "wait for the user's response" steps in stage prompts.
  - Make each stage end at its artifact. Brainstorming should stop after the spec self-review and must not invoke writing-plans.
  - Have the engine present the artifact for external approval.
- The precedence rule (CLAUDE.md/AGENTS.md > skills) makes this override possible without forking, for example: "Approval gates are external; never wait for chat approval; stop after writing the artifact."

**Parallelism**
- Superpowers runs plan tasks sequentially inside one session.
- An engine running stories as separate activities can do engine-level fan-out only for stories whose Interfaces blocks show no dependency. The plan's Interfaces blocks are where `depends_on` can be derived.
- The "never parallel implementers" rule exists because they share one worktree. Engine fan-out would need one worktree or branch per activity, plus merge handling.

**Durability**
- Replace the git-ignored `.superpowers/sdd/` ledger with engine-recorded activity state, or persist the workspace as an artifact between activities.
- Keep "Rulings" as a structured output that the next human gate must surface. SDD's "Rulings I made" list is effectively a pending-review queue.

**Paths and bootstrap**
- Set spec and plan locations to the repo convention through CLAUDE.md, for example `docs/specs/...` and `docs/implementation/...`. Do not adopt `docs/superpowers/...`.
- Disable the auto-trigger bootstrap inside stage jobs, either by not installing the plugin hook or by vendoring selected SKILL.md files as stage prompts. Otherwise the 1% rule may launch brainstorming inside an implementation job.

**Licensing and versioning**
- MIT allows vendoring and modifying SKILL.md text and scripts. Keep the copyright notice.
- Pin to a tag, since upstream semantics change monthly.

### Gaps
- No source found describes anyone running Superpowers inside an external workflow engine or CI pipeline with external approval gates. Its fit with a durable harness is my own analysis, not documented prior art.
- I could not find whether Prime Radiant's commercial offering ("additional tooling, or managed spending") includes orchestration or durable execution. Only the README sentence is available. — [README](https://github.com/obra/superpowers/blob/v6.4.2/README.md)
