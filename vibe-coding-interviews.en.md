# **Vibe Coding / Agentic Flow Interview Question Bank**

***Note: There is no canonical "correct answer" in this question bank. The attached "reference answers" are meant to spark discussion, not to constrain it.***

***This document has comment access enabled. If you find anything inaccurate or wrong, criticism and suggestions are welcome.***

Target audience:

- Developers who use agentic CLI tools such as Claude Code and Codex on a daily basis.

Credit:

- Sihao Liu \<[sihao@cs.ucla.edu](mailto:sihao@cs.ucla.edu)\>, [https://github.com/SihaoLiu](https://github.com/SihaoLiu)
- Bangyan Wang \<wangbangyan@gmail.com\>, [https://github.com/AcrossForest](https://github.com/AcrossForest)

References:

- [https://code.claude.com/docs/llms.txt](https://code.claude.com/docs/llms.txt)
- [https://github.com/openai/codex/blob/main/docs/getting-started.md](https://github.com/openai/codex/blob/main/docs/getting-started.md)
- Public documentation from Anthropic / OpenAI.

---

## **Table of Contents**

I. Background Knowledge

II. Detail Walkthrough

III. Workflow

IV. System Design (Open-Ended)

V. Concepts & Philosophy (Open-Ended)

VI. The Hidden Lore

---

## **I. Background Knowledge**

### **Q1. What is a /command? How does it differ from a skill? When should you use a command (now deprecated) versus a skill?**

**Reference answer:**

* **/command (Slash Command):** The original extension mechanism in Claude Code. Users drop `.md` files into `.claude/commands/` and trigger them in a conversation by typing `/command-name`. Essentially "a prompt snippet that the user explicitly invokes." It is now officially marked as deprecated in Claude Code, superseded by skills.

* **Skill:** The new-generation extension mechanism, located at `.claude/skills/<name>/SKILL.md` (or in `~/.claude/skills/`, or inside a plugin). Key differences:

  1. **Can be invoked autonomously by the model** — A skill declares a `description` in its YAML frontmatter, and Claude decides whether to enable it semantically, instead of relying solely on the user typing `/`.

  2. **Supports auxiliary files** — A skill directory can carry scripts, reference docs, and sub-templates.

  3. **Lifecycle-managed** — Skills are re-injected during `/compact` (first 5K tokens, with a 25K global budget).

  4. **Fine-grained control** — Frontmatter can specify `allowed-tools`, `disable-model-invocation`, `paths` (path scoping), `effort`, and `model`.

* **When to use what:**

  * **Prefer skill**: For almost every new scenario, especially when you want the model to activate at the right moment automatically, or when you need bundled scripts/example files.

  * **Cases where command still makes sense (rare)**: Fully deterministic, one-line prompt wrappers that must be explicitly triggered by a human (e.g., legacy team assets). Not recommended for new projects.

---

### **Q2. What is an agent? When should we use an agent versus a skill?**

**Reference answer:**

* **Agent (Sub-agent):** A "miniature Claude" with its own system prompt, its own context window, and its own tool allowlist. In Claude Code, it is defined under `.claude/agents/<name>.md` and dispatched from the main conversation via the Agent tool. When the sub-agent finishes, only a "summary" is returned to the main conversation; intermediate steps do not pollute the parent context.

* **Skill:** A set of instructions/procedures for the model to read. It **does not open a new context**; instead, it guides what Claude does next in the current conversation.

**Core distinction:**

| Dimension | Skill | Agent |
| :---- | :---- | :---- |
| Context | Shares the main context | Independent context |
| Invocation cost | Low (just injected text) | High (one full inference round) |
| Suited for | "How-to" guidance, standards, checklists | Heavy tool calls, sub-tasks that may produce a lot of noise |
| Output | Changes the behavior of subsequent main conversation | Returns a summary |

* **Decision principles:**

  * Task **produces a lot of tool output** (reading dozens of files, running tests, scraping logs) → use **agent** to avoid polluting the main context.

  * Task is **teaching Claude a procedure** (e.g., "run lint before commit") → use **skill**.

  * Task is **read-only exploration/search** → prefer built-in agents like Explore.

  * Task requires **multi-step reasoning** whose thought process should stay in the main conversation → don't open an agent; let the main conversation handle it.

---

### **Q3. What is a sub-agent? Can sub-agents communicate with each other?**

**Reference answer:**

* **Sub-agent:** See Q2. Key traits: "independent context + independent tool set + only returns a summary."

* **Can they talk to each other?**

  * **No, not directly, by default.** Claude Code's sub-agent communication model is **star-shaped**: every sub-agent talks only to the main conversation, which acts as a hub and relays information between them.

  * The experimental **Agent Teams** feature (requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) introduces a `SendMessage` tool that allows peer agents to talk directly via IDs; but that is a different model (see Q4).

* **Why this design?** Forcing information back to the main conversation keeps state traceable and auditable, and prevents uncontrollable loops of dialogue between agents.

---

### **Q4. What is the biggest difference between an Agent Team and a sub-agent?**

**Reference answer:**

* **Sub-agent:** "Hire a temp to do one task and report back." One-shot, star-shaped, report-style. Main conversation dispatches → sub-agent executes → returns summary → sub-agent is destroyed.

* **Agent Team:** "Assemble a long-running team whose members can call out to each other." Persistent, mutually reachable, parallel.

  * Each teammate has a persistent, independent context.

  * They exchange messages directly via `SendMessage(to=<agent_id>, message=...)`.

  * Suited for "multiple long-running roles working in parallel" (e.g., a frontend engineer + a backend engineer + a reviewer all working simultaneously).

* **Biggest differences:**

  * **Topology**: Sub-agent is a "parent → child" tree; Agent Team is a graph with peer-to-peer communication.

  * **Lifecycle**: Sub-agent is fire-and-forget; teammates are long-lived.

  * **State sharing**: A sub-agent only returns a final summary; teammates can converse back and forth and maintain shared context.

* **Trade-off**: Agent Teams are closer to "multi-agent collaboration" research, but are harder to debug and prone to context blowup. Most everyday tasks are well-served by sub-agents.

---

### **Q5. What is MCP? How is it different from an API?**

**Reference answer:**

* **MCP (Model Context Protocol):** An open protocol led by Anthropic for exposing "external tools/resources/prompts" to large models in a **standardized format**. An MCP server process can declare the tools, resources, and prompts it offers, and clients (Claude Code, Claude Desktop, Codex, etc.) discover and invoke them via a unified protocol. **Differences from "calling an API directly":**

| Dimension | Raw API | MCP |
| :---- | :---- | :---- |
| Description | Natural language + docs; every LLM/client implements its own adapter | Standard schema; declared once, usable by every client |
| Tool discovery | Must be told to the model up front | Client lists tools automatically on connect |
| Auth | Each API has its own scheme | Standard OAuth 2.0, supports dynamic client registration |
| Transport | Varies per HTTP API | Three standard transports: stdio / HTTP / SSE |
| Reuse | Every LLM platform has to redo the work | One MCP server, usable by every MCP-aware client |

* **Analogy:** APIs are "USB cables" — each vendor's shape is different. MCP is "USB-C" — one unified interface for all LLM clients.

* **The real distinction:** MCP doesn't just carry data; it **self-describes capability** — tool signatures, parameter schemas, mutability, confirmation requirements — so the model can reason about "when to call, how to call."

---

### **Q6. Can a sub-agent spawn its own sub-agents?**

**Reference answer:**

* **No.** The official Claude Code documentation explicitly states: sub-agents cannot launch sub-agents.

* **Why:**

  1. **Prevent infinite recursion** — Once nesting is allowed, agent-tree depth becomes unbounded, and debugging and cost control turn into nightmares.

  2. **Keep information aggregated** — All summaries must flow back to the main conversation so that the user and the main agent have a complete, auditable trace.

  3. **Avoid exponential context blowup** — Every nested call is its own context; nesting can explode latency and bills.

* **What if you really need "nested delegation"?**

  * Use **skills** inside the sub-agent to organize internal steps (skills don't open a new context).

  * Have the main conversation dispatch sub-agents **serially** or **in parallel** (flattened).

  * Use the Agent Team model (but that is peer-to-peer, not nested).

---

## **II. Detail Walkthrough**

### **Q7. What are CLAUDE.md / AGENTS.md? What is the load order and priority?**

**Reference answer:**

* **CLAUDE.md (Claude Code) / AGENTS.md (Codex):** Project- or user-level "persistent prompt injections." They are automatically loaded into the system prompt at the start of every session, used to declare "rules I want the agent to always remember."

* **Load layers (Claude Code):**

  1. **Managed policy** — Enterprise-controlled, deployed by IT, cannot be overridden.

  2. **User** — `~/.claude/CLAUDE.md`, personal global rules.

  3. **Project** — `./CLAUDE.md` at the repo root, project-level rules.

  4. **Local** — `CLAUDE.local.md`, personal per-project private overrides (typically gitignored).

* **Priority:** Later-loaded layers override earlier ones (project overrides user; managed cannot be overridden).

* **Recommendations:**

  * Keep each file < 200 lines; if too long, split it out via `@path/to/file.md` references.

  * Don't write "project background introductions"; write "hard rules the agent must obey."

* **Caveats:**

  * The key to maintaining a good CLAUDE.md is that it should **not** record information the agent will naturally pick up while exploring the project. It should record information of the kind that is "second nature to a project veteran, but takes a newcomer a long time to discover."

  * **Avoiding rot in CLAUDE.md is the first principle.** A CLAUDE.md full of rotten, out-of-date information causes far more harm than a sparse one — when in doubt, leave it out.

  * It should be a "quick-reference guide and pitfall-avoidance guide," not a "project introduction."

---

### **Q8. What are hooks? Name a few hook scenarios you commonly use.**

**Reference answer:**

* **Hook:** A **deterministic callback** registered against a Claude Code/Codex lifecycle event (e.g., PreToolUse, PostToolUse, SessionStart, Stop), executed by the harness rather than the LLM, so **it cannot be forgotten or bypassed by the model**.

* **Types (Claude Code):** command (shell), http (POST endpoint), prompt (single-turn LLM evaluation), agent (multi-turn LLM), mcp_tool.

* **Common scenarios:**

  1. PostToolUse on `Edit|Write` → auto-run prettier/eslint/gofmt.

  2. PreToolUse on `Edit(/.env*)` → block (exit 2) to prevent accidental key edits.

  3. Stop → send a desktop notification telling me Claude is idle.

  4. SessionStart → print current git status and env vars so the agent knows the situation upfront.

  5. PreToolUse on `Bash(rm -rf *)` → ask mode, force user confirmation.

---

### **Q9. What permission modes exist? What scenarios suit each?**

**Reference answer:**

| Mode | Behavior | Suited for |
| :---- | :---- | :---- |
| default | Ask the first time each tool is used | An unfamiliar new project |
| acceptEdits | Auto-allow file edits and fs commands | A familiar project, accelerating iteration |
| plan | Read-only mode, no writes allowed | Code review or design |
| auto | A background classifier decides safety | A "semi-automatic" mode where you trust the classifier |
| dontAsk | Anything not on the allowlist is rejected | Strict allowlist-only production scenarios |
| bypassPermissions | Allow everything (still keeps fuses like `rm -rf /`) | Inside a sandbox / container / dev container |

---

### **Q10. During /compact, what context is preserved and what is lost?**

**Reference answer:**

* **Preserved (re-injected):**

  * System prompt

  * Repo-root CLAUDE.md + path-unrestricted rules

  * Auto memory (re-read from disk)

  * Conversation summary (the summary that Claude generates)

  * The top-N high-priority skill descriptions (within the 25K token budget)

* **Lost:**

  * Detailed historical tool-call output (e.g., file contents read, bash output)

  * Path-scoped nested `CLAUDE.md` / `.claude/rules/*.md` (only re-triggered when a matching file is read again)

  * Full MCP tool schemas (only the names are kept; the schema is re-fetched on next use)

* **Practical advice:** Don't put "things that must be remembered long-term" in the chat history; either write them into CLAUDE.md, or persist them explicitly via auto memory.

---

### **Q11. What is the relationship between a plugin and a skill?**

**Reference answer:**

* **Skill:** A single capability unit (one SKILL.md + auxiliary resources).

* **Plugin:** A **packaged, distributable** extension that can bundle multiple skills, and additionally:

  * sub-agent definitions

  * hooks

  * MCP servers

  * LSP servers

  * binary executables

  * default settings

* Metadata is described in `.claude-plugin/plugin.json`, and after install, skills inside a plugin are invoked with a **namespace**: `/<plugin-name>:<skill-name>`, to avoid collisions.

* In short: **skills are parts; a plugin is an assembled vehicle.**

---

### **Q12. What are ToolSearch / deferred tools? Why design it this way?**

**Reference answer:**

* **Observation:** At startup, the **full JSON schemas of MCP tools and some built-in tools are not loaded immediately**; only their names are revealed to the model. When the model needs a specific tool, it pulls the schema in via ToolSearch.

* **Why:**

  1. **Save tokens** — A large MCP server can have dozens or hundreds of tools, whose schemas can easily add up to tens of thousands of tokens.

  2. **Reduce attention noise** — Unrelated tools hanging in the prompt long-term interfere with model decisions.

  3. **Lazy loading** — Detailed signatures only enter context when actually needed.

* **Cost:** Using a tool for the first time costs one extra ToolSearch round-trip; but compared with the token savings, this is a great trade.

---

## **III. Workflow**

### **Q13. You've been handed a "medium-complexity new feature" task. How do you decompose it using agentic tools?**

**Reference answer (one typical flow):**

1. **Enter plan mode** (Shift+Tab or your binding) and explore the code in read-only mode.

2. Use the Explore sub-agent to search relevant files in parallel (find entry points, data flow, tests).

3. Use the Plan sub-agent or the main conversation itself to produce a **TDD-style design**:

   * Write acceptance criteria / test cases first

   * Then design the minimal implementation

   * Then write a verification strategy

4. Use ask-codex or ask-gemini for a second opinion (cross-model review) to patch blind spots.

5. Use AskUserQuestion to let the user decide on critical design forks.

6. ExitPlanMode → switch to acceptEdits mode and start implementation.

7. Before writing code, write (or update) tests first, then let the agent implement and run tests live.

8. After completion, self-review with `/review` or `superpowers:requesting-code-review`.

9. Before submitting, run through the `verification-before-completion` skill.

---

### **Q14. When should you spawn a sub-agent, and when shouldn't you?**

**Reference answer:**

* **Should spawn:**

  * Need to read 10+ files for investigation (Explore).

  * Need lots of grep / find / log analysis whose output is noisy.

  * Need an independent code review or security review.

  * Need to run multiple independent sub-tasks in parallel (dispatching-parallel-agents).

* **Should NOT spawn:**

  * Task only needs 1-3 tool calls → the main conversation is faster.

  * Task requires the main conversation to retain **its reasoning** for later decisions → opening an agent loses context.

  * Task needs **multiple interactive confirmations** → sub-agents can't ask the user.

  * Task is "read a file at a known path" → just Read, don't bother with an agent.

---

### **Q15. How do you avoid polluting the main conversation context?**

**Reference answer:**

1. **Heavy reads / investigation → use an Explore sub-agent**, return only a summary.

2. **Long-output commands → narrow with head/grep**, or write to a file and let the agent selectively read it.

3. **Wrap repetitive tasks into skills** so you don't keep typing rules into the chat.

4. **Run /compact at milestones** to compress.

5. **Don't make the agent re-read the same file** — file state is tracked by the harness; after editing you don't need to re-read to verify.

6. **Put "things to remember long-term" into CLAUDE.md / auto memory**, not into one-off chat messages.

7. **Load MCP tools on demand** — don't connect every MCP server at once.

---

### **Q16. There's a bug in the code. How do you debug it using agentic tools?**

**Reference answer:**

1. Enter systematic-debugging mode via the `superpowers:systematic-debugging` skill.

2. **Don't rush to change code** — reproduce first; write the smallest case / test that reliably triggers the bug.

3. Let the test **fail first** to confirm your understanding of the bug.

4. Use Explore to find every file involved in the bug's execution path.

5. Hypothesize → verify → fix (not patch-by-guess).

6. After fixing, run all related tests + run `verification-before-completion`.

7. For tricky bugs, use `codex:rescue` to have Codex perform an independent diagnosis and cross-check the reasoning.

---

### **Q17. In a multi-developer repo, how do you make agentic workflows "team-ready"?**

**Reference answer:**

* **Shared layer:**

  * Repo-root CLAUDE.md / AGENTS.md — hard rules shared by the whole team (code style, PR process, prohibitions).

  * `.claude/skills/`, `.claude/agents/`, `.claude/commands/` all checked into git.

  * `.claude/settings.json` — shared permission allowlist, hooks.

* **Personal layer:**

  * `CLAUDE.local.md` (gitignored) — personal preferences.

  * `.claude/settings.local.json` — personal permission overrides.

* **Distribution:**

  * Package the team's shared capabilities as a **plugin**, host it on an internal marketplace, and new members onboard with a single `/plugin install`.

* **Audit:**

  * Use PreToolUse hooks to log audits on sensitive operations.

  * The PR template should ask for the key decision points made during agent collaboration.

---

## **IV. System Design (Open-Ended)**

The following questions have no canonical answer; the point is to surface the candidate's trade-off framework and risk awareness.

### **Q18. Design an "agentic low-code platform." What design principles would you prioritize?**

**Discussion directions:**

1. **Who is the source of truth?** Is the agent-generated code the source, or is the low-code DSL? Bidirectional sync between the two is an engineering nightmare; you must pick one.

2. **Reversibility and version control** — After an agent changes 50 components in one click, you must be able to diff, revert, and review.

3. **Sandboxing and permission tiers** — A platform user's agent must not touch the database directly; it must go through a permission gateway (analogous to permission mode).

4. **Context supply** — How does business knowledge enter the agent? Do you require users to write CLAUDE.md-style guides, or do you auto-distill from existing data?

5. **Observability of failure** — Failures from the agent's third-party API/MCP calls must be replayable, not just "the agent said it failed."

6. **Don't replace the user's thinking** — A well-designed agentic platform reduces **typing**, not **decisions**. Critical forks must require explicit confirmation.

7. **Composable** — Platform capabilities should be composable like skills/plugins, not a pile of magic buttons.

8. **Cost is visible** — The token / API / latency of every agent operation must be exposed up front, or users lose control.

---

### **Q19. Design an internal "MCP gateway" for an enterprise. How would you handle permissions, auditing, and rate limiting?**

**Discussion directions:**

* **Permissions:** Who can list which tools? A read-only role should not see the schema of `delete_*` tools, reducing the surface for accidental calls.

* **Auditing:** Log every tool call with user / agent_id / arguments / response size / latency.

* **Rate limiting:** QPS / concurrency of MCP tool calls per user, to keep runaway agents in check.

* **Data masking:** Tool responses should pass through a middle layer to scrub PII / secrets before returning to the agent.

* **Degradation strategy:** When the backing service is down, the MCP gateway should return a structured error rather than time out, so the agent knows "this path is closed."

* **Versioning:** What happens when a tool schema changes? Add version numbers; old clients still see the old schema.

* **Isolation:** MCP server processes from different tenants must be isolated to prevent cross-tenant data exposure.

---

### **Q20. If you were designing the multi-agent collaboration model of Claude Code / Codex, would you pick "star (parent-child)" or "mesh (peer-to-peer)"? Why?**

**Discussion directions:**

* **Star pros:** Easy to debug, controllable context, clear responsibility. Cons: the main conversation becomes a bottleneck; parallelism is limited.

* **Mesh pros:** Closer to a real team, higher parallelism. Cons: prone to "agent-to-agent loops," "context blowup," "unclear responsibility," and "debugging hell."

* **Realistic answer:** Star is enough for most tasks; only the rare, multi-role complex task benefits from mesh, and even then it needs guardrails like **message budgets**, **max total threads**, and an **observable conversation graph**.

* **Further reflection:** Is the ROI of multi-agent collaboration actually higher than "one stronger single agent"? As model capabilities keep improving, multi-agent orchestration may turn out to be a short-term workaround.

---

### **Q21. Design an "AI security review agent." What pitfalls should you avoid?**

**Discussion directions:**

* **Don't just look at the diff** — Security problems often surface at unchanged callers. You must be able to expand into surrounding context.

* **Don't blindly trust passing tests** — The places tests don't cover (deserialization, command injection) are where issues usually live.

* **Don't let the agent run attack commands itself** — The review phase must be read-only; otherwise the review agent becomes an attack surface.

* **Cross-check** — Have a second model (e.g., Codex) review independently to reduce single-model blind spots.

* **Don't chase "zero false positives"** — A security review should prefer false positives over false negatives, but tier them (P0 / P1 / P2) so humans can focus.

* **Explainable** — The agent must give a reasoning chain for "why is this a vulnerability," not just drop a line saying "this is unsafe."

---

### **Q22. Design a "long-term memory" mechanism (similar to Claude Code's auto memory). How do you decide what to remember and what to discard?**

**Discussion directions:**

* **Should remember:** Preferences the user has repeatedly corrected (code style, naming), project-level hard constraints, easily-forgotten trivia (port numbers, secret locations).

* **Should NOT remember:** One-off temporary conversations, content containing sensitive data, outdated facts (a bug long since fixed).

* **How to update?** Merge on write (avoid duplicates), periodic GC (down-weight or delete stale entries), users can audit / edit the memory file with one click.

* **How to avoid pollution?** Cap the memory size (e.g., 25KB); above the cap, evict based on "usage frequency + user marking."

* **Shared across projects vs. project-isolated?** Preferences are shared across projects; project knowledge is not — otherwise project A's secrets get misused in project B.

---

## **V. Concepts & Philosophy (Open-Ended)**

### **Q23. Why should we "keep the context simple"? Isn't a prompt stuffed with information better?**

**Discussion directions:**

* **Attention is a finite resource** — An LLM's attention mechanism does not weigh relevant content uniformly; excessive irrelevant information **dilutes** the weight of the key instructions (the "lost in the middle" phenomenon).

* **Token cost** — Every inference is billed/computed over the full context; long contexts increase both latency and bill.

* **Debuggability** — When things go wrong, a short context makes it easy to identify which instruction conflicted; a long context is almost impossible to post-mortem.

* **Evolvability** — The simpler the context, the easier it is to iterate / replace / reorder; a complex context is spaghetti.

* **Philosophical level:** "Simple" doesn't mean "less information"; it means "high signal-to-noise ratio." Let the agent know **what it needs to know**, not **everything that might be relevant**.

---

### **Q24. Should an agent be "proactive" or "reactive"? When should it ask the user, and when should it decide on its own?**

**Discussion directions:**

* **Reversibility of the action** — Reversible (editing a local file): decide. Irreversible (push, delete, send email): ask.

* **Blast radius** — Affects only the local machine: decide. Affects a shared system (CI, PR, Slack): ask.

* **Uncertainty** — When the agent isn't sure, it should ask, not gamble.

* **User availability** — User online: ask more freely. Background / offline tasks (cron): bias toward conservative self-decision.

* **Design philosophy:** A truly good agent neither "asks nothing" nor "asks everything"; it **asks the right questions** — ask at critical forks, stop pestering on trivia.

---

### **Q25. Is "Vibe Coding" hype or a paradigm shift?**

**Discussion directions (multiple views encouraged):**

* **Hype camp:** It's just smarter autocomplete; agents still make basic mistakes in complex projects; maintenance costs are underestimated.

* **Paradigm camp:** The "primary unit" of programming is rising from "line/function" to "intent/constraint"; the engineer's role shifts from "typist" to "architect + reviewer"; the bottleneck on software delivery speed is breaking.

* **Middle ground:** It is a paradigm shift, but not a 1:1 replacement — it amplifies the capability of high-skill engineers (because they can produce good specs and good reviews), and equally amplifies the destructive power of low-skill engineers (because they can also produce garbage quickly).

* **Key follow-up questions:**

  * If the agent writes code, where does knowledge accumulate? In the repo (CLAUDE.md) or in individual heads?

  * Will "code review" become more important than "writing code"?

  * How will education, entry-level barriers, and junior positions change?

---

### **Q26. An agent has repeatedly failed to fix a bug. Should you let it keep trying, or stop and take over?**

**Discussion directions:**

* **Signal recognition:** The agent is "looping through variants of the same wrong solution" → its context is already polluted, and continuing is mostly futile.

* **Sunk cost:** How many tokens have already been burned doesn't matter; what matters is "can another X tokens solve it." If the probability/cost is unfavorable, stop.

* **Meta-issue:** When an agent is stuck, it usually means **information is missing**, not that **reasoning failed** — at that point a human should add information (paste logs, paste docs, paste the failing case), not let the agent keep guessing.

* **Healthy habit:** Set yourself a ceiling (e.g., "if the agent fails to fix it three times, I take over") to avoid unconscious over-reliance.

* **Philosophy:** An agent is an amplifier, not a substitute. When what it amplifies is a **dead end**, the deeper you go, the worse.

---

### **Q27. Why is "test-driven development (TDD)" even more important in an agentic workflow than in a traditional one?**

**Discussion directions:**

* **Agents are prone to writing wrong code confidently** — They fabricate APIs, fields, return values; tests are the only objective referee.

* **Tests are the contract you use to communicate with the agent** — You don't need to describe "how to do it" in detail; you just say "after you're done, this test must pass."

* **TDD gives the agent a feedback loop** — Without a failing test, the agent can't tell whether it's wrong.

* **TDD forces vague requirements into executable specs** — which is exactly the input agents need most.

* **Risk:** Conversely, if the agent writes both the tests and the implementation, it may "make the test pass for the sake of passing." Critical tests (acceptance-level, security-level) should be written by humans or independently reviewed.

---

### **Q28. "The code agents write is hard to read." Is this a tool problem or a usage problem?**

**Discussion directions:**

* **Tool side:** Models do have tendencies toward "over-abstraction / over-defensive / over-commented" code; constraints in the system prompt (Claude Code's default prompt, for example, says "don't add meaningless comments") can help.

* **Usage side:** The user provided no constraints / no review / accepted the first try → garbage code piles up. This is a **usage problem**, not a tool problem.

* **Root solutions:**

  1. Write hard readability rules into CLAUDE.md (function length, naming, comment conventions).

  2. Make "simplify / refactor / remove redundancy" a separate step, instead of trusting the first try.

  3. Externalize review standards into skills (such as `simplify`, `code-reviewer`) to force agents to self-audit.

* **Deeper take:** "I can't read what the agent wrote" is usually because the reader didn't take time to understand it. If you don't read it at all, you've outsourced production code to an intern who can't be held accountable.

---

### **Q29. "I let the agent run all night and I can't tell what it did" — is this the agent's problem to solve, or the user's?**

**Discussion directions:**

* **Observability is the tool provider's responsibility** — The agent must emit an auditable sequence of operations (file diffs, command log, decision points).

* **Comprehension is the user's responsibility** — No matter how good the tool is, no human can audit 1000 tool calls line by line overnight; you must **agree on boundaries in advance** (e.g., permission mode = plan, or restrict it to specific sub-tasks).

* **Design philosophy:** The longer the autonomous task, the more it needs **strong constraints + phased checkpoints**, not "let it run, then audit."

* **Extreme case:** A fully unsupervised long-running agent has a risk profile similar to "an unsupervised junior intern + sudo" — basically unacceptable in production.

---

### **Q30. Thought question: will agentic tools eventually eliminate the "programmer" profession?**

**Discussion directions (no right answer):**

* **Won't disappear, will be **reshaped**:** The typing-code part gets outsourced, but **understanding requirements / decomposing systems / judging correctness / bearing responsibility** will always need humans.

* **One kind of "programmer" will disappear:** The mid-to-lower roles defined entirely as "type out the spec implementation" are significantly at risk.

* **New roles will appear:**

  * "Agent architect" — designs agent collaboration graphs, skill libraries, context strategies.

  * "AI reviewer" — a senior engineer who can review agent output at high throughput.

  * "Context engineer" — structures domain knowledge into a form agents can digest.

* **Historical analogy:** Compilers didn't eliminate assembly programmers, but they did dramatically reduce the population of "hand-written x86" coders. The level of abstraction at which programmers work will rise again, rather than disappear.

* **Personal-stance question:** In 5 years, what fraction of your daily work do you think will still be "you typing"? 30%? 5%?

---

## **VI. The Hidden Lore**

### **Q31. How do you design an "interruptible" interactive application using the "non-interactive" modes of Claude Code / Codex?**

**A31:** Directly interrupt the non-interactive claude/codex session, record the session id, and resume with a new prompt (chat with non-interactive session = interrupt and resume with new prompt). Some solutions (and failure paths) you can probe with the candidate:

- Consider tmux as the underlying driver — interrupting means sending messages straight into the tmux pane. What are the upsides? Downsides?
- Consider terminal-level "interception"; briefly discuss implementation approaches.
- Consider customizing the interruption mechanism via open-source apps like OpenCode.
- Consider building your own `tool_call` tool.
- What are the pros and cons of each approach above?

---

### **Q32. What is the overall core flow inside Superpowers? What is the essence of Brainstorming? When do you use Inline Execution vs. Sub-agent Driven Execution?**

**A32:** The core flow of Superpowers is: Brainstorm → Writing Plan → Inline / Subagent-Driven Execution (with TDD). Superpowers later added a series of features around merging features, isolating worktrees, and a standard process for closing a PR. But the core remains Spec-Plan-Execution.

The essence of Brainstorming is a **"forced anchor"** in Q&A form. Related skills like "grill-me" share the same essence: through multi-turn dialogue with the user, **force the user to think clearly about "what I actually want to do,"** thereby converting the LLM's execution path from an "explore + implement" path into a pure implementation process.

The difference between Inline and Subagent execution: an Inline-executing agent is the same agent that planned and discussed the spec, so when context allows, it can achieve better results (better alignment). Subagent execution farms each step out to a sub-agent with a clean context — the upside is more context room; the downside is that each new agent must spend time and tokens orienting itself after receiving the main agent's task, which can introduce execution drift and missed context.

A candidate hits the bar for this question if they can discuss, from their own practice, the difference and observed effects between Inline and Subagent execution, and clearly articulate **where their personal "boundary"** between the two lies.

**My two cents:** I actually think `writing-plans` is not a great design, because the spec's inherent "ambiguity" is already sufficient — in the current environment (2026/05) — to guide most models toward the target design. Inserting a "writing-plan" layer between the spec and concrete execution causes two negative consequences: 1. The plan's positioning is unclear — is it "design guidance" or "concrete implementation"? 2. Agents tend to write large amounts of implementation code inside the plan, and any drift in that "concrete code guidance" gets amplified in the next stage by Subagent-Driven Execution, which leads to two bad outcomes: a. The executing agent has no single source of ground truth (spec vs. plan) to correct against; b. The plan's drift amplifies the "intelligence jaggedness" between documents in the system, causing the spec in subsequent stages to drift even further. For this reason, after Brainstorming produces a spec, I personally skip the writing-plan stage entirely.

---

### **Q33. Would you use Claude Code's `/init`? Why?**

**A33:** `/init`'s design intent is to provide onboarding material — namely [CLAUDE.md](http://CLAUDE.md) — for the user at the start of a project. The intent is good: it lets short-lived Claude Code sessions quickly understand the project's context. But [CLAUDE.md](http://CLAUDE.md) holds an extremely high priority in the Claude Code ecosystem, and it serves two purposes at once: **project memory (factual info)** and **project rules**.

This dual positioning means [CLAUDE.md](http://CLAUDE.md) appears in every system prompt of the conversation, which makes it extremely hard to violate (see Q7 for the detailed priority rules) — and consequently the content of CLAUDE.md is very stubborn.

But as the project evolves, the "reality" of the project will gradually drift from the "memory/rules" in [CLAUDE.md](http://CLAUDE.md). Due to the document's high priority, it is in fact very hard to "automatically" delete/modify it, leading to so-called "rot": the [CLAUDE.md](http://CLAUDE.md) produced by `/init` becomes outdated quickly. There are mechanisms intended to keep this document fresh automatically — see Auto-memory / Auto-dream.

**My two cents:** I think the mechanism of "automatically managing project memory" is, in fact, asking the model to perform an action that contradicts its own working mechanism — a contradictory action where the model must simultaneously:

Forward: act on facts and follow the rules inside [CLAUDE.md](http://CLAUDE.md).

Reverse: modify the facts and rules inside CLAUDE.md.

This essentially asks the model to perform a "non-aligned" action. Given CLAUDE.md's system-level priority, I lean toward not using `/init` and maintaining project memory by hand.

---

### **Q34. What are an Agent's stdin, stderr, and stdout?**

**A34:** Every ordinary program has three Unix file descriptors: stdin, stdout, stderr. If we treat an Agent as a kind of "program," then its stdin is naturally the prompt, and its stdout is the transcript of the entire execution trace. Under this standard-program model, what is the Agent's "error"? An Agent's "standard error stderr" is essentially the **record of human interventions** that occur after the initial prompt has started running. This is also why, when designing a low-code agent platform, you must support capturing the execution trace *after* the agent has been interrupted (see Q31).

**My two cents:** If the candidate recognizes the importance of capturing human-intervention traces as a way to optimize the agent's execution end-to-end, that's an excellent insight. But you should get them to give a concrete example of "what they would do with the intervention trace after capturing it."

---

### **Q35. What are the core design dimensions of context management in Coding CLIs?**

**A35:** This is an open-ended design-classification question — it asks the candidate to identify dimensions by examining existing Coding CLI designs, not to recite a product's feature list.

The differences between Coding CLIs (Claude Code, Codex, Cursor, etc.) in how they manage context look bewildering at first glance, but they can be understood along a few orthogonal dimensions. One reference classification splits the design space into **three dimensions**:

1. **Context transfer model** (see Q36): How does "what to do" and "the result of doing it" flow between execution units?
   - Input fidelity: Lossless / Lossy
   - Output disposition: Drop / Preserve / Archive

2. **Injection management model** (see Q37): How does knowledge text (project rules, capability descriptions, etc.) enter the context, and who manages it?
   - State management: Stateful / Stateless
   - Trigger conditions: User-driven / Harness-driven / Model-driven

3. **Packaging and distribution** (see Q38): Where do capabilities come from, and how are they discovered?
   - Convention-based / API-based

This design space is not exclusive to any one product — different Coding CLIs pick different positions on the same map; specific products are points on the map. The particular choice of dimensions is not unique; the three above are just one reference. What matters is the ability to organize thinking systematically, not to enumerate features one by one.

---

### **Q36. When dispatching a task to an AI, you need a way to "tell it what to do" and a way to receive "how it went." How many design options exist for this? Try to design a classification scheme where Claude Code's sub-agent, inline execution, and resume/SendMessage each find a place.**

**A36:** One reference classification is to split the problem into two independent sub-questions — **how does the input get there** and **what happens to the output**.

**Input fidelity (how does the sub-task acquire information):**

- **Lossless**: The sub-task inherits a complete context snapshot of the main session. Zero information loss, best alignment quality, but token cost doubles.
- **Lossy**: The main session compresses the task into a brief description. Saves tokens, but loses hard-to-articulate context like "directions already ruled out" and "the user's implicit preferences."

**Output disposition (what happens to the context the sub-task accumulated):**

- **Drop**: The sub-task's context is destroyed; only a summary is returned. The main session stays clean.
- **Preserve**: The sub-task executes directly in the main context (inline); all intermediate output stays in place. Maximum information, but accumulated noise.
- **Archive**: The context is removed from the current window but persisted via an addressable handle, ready to be re-awakened. The defining trait is being **addressable** — simply saving logs does not count as Archive.

Crossing the two yields a 2×3 matrix:

| | Drop | Preserve | Archive |
| :---- | :---- | :---- | :---- |
| **Lossless** | No native support (simulate via fork + take-only-result) | Inline execution, `@skill` | `--fork-session`, `/branch` |
| **Lossy** | Sub-agent | `/compact` | `--resume`, Agent Team `SendMessage` |

**So why does Claude Code have sub-agent, fork, and resume all at once?** Because they sit at different coordinates in the matrix and each solves a different problem:

- **sub-agent (Lossy-Drop)**: The standard "dispatch and report back" model. Clean isolation + token economy, but both input and output have information loss.
- **fork (Lossless-Archive)**: For exploring multiple directions in parallel at a decision fork; also the natural way to implement a "Lossless-Drop sub-task" (inherit full context + isolated noise).
- **resume (Lossy-Archive)**: When a sub-task has finished 90% of the work but needs a final adjustment based on feedback, restoring an existing session is more efficient than dispatching a new sub-agent — the old session retains complete understanding of the task.

Starting from the most conservative Lossy-Drop as the base model, there are also **three orthogonal extension directions**:

- **Nesting**: Allow sub-tasks to dispatch their own sub-tasks. Solves the task-scale problem. Costs: uncontrollable tree depth, debugging difficulty, exponential cost growth. Claude Code explicitly forbids it (max depth = 1; see Q6).
- **Resumable**: Don't destroy a sub-task after completion; allow it to be re-awakened and continue. Analogous to a coroutine's `yield` / `send`. Claude Code supports `--resume` at the main-session level, but sub-agent-level resumability is not yet implemented — a feature with strong community demand.
- **Fork**: Copy the full context to an independent session at some point in time. Analogous to Unix `fork()`.

Each added extension dimension adds one order of system complexity.

**My two cents:** Lossy losses are **bidirectional** — the input side loses "context" and the output side loses "process." The most severe losses are usually in the parts that are hardest to articulate: "directions already ruled out," "the user's implicit preferences," "the chain of reasoning behind a judgment."

In practice, when you dispatch a complex task to a sub-agent and it comes back having gone off-track, it's often not that the sub-agent is dumb — it's that your task description couldn't carry "the nuances from the previous 30 minutes of discussion." This is an essential limitation of the Lossy-Drop model, not something prompt engineering can fully solve.

A practical diagnostic: the latest Claude Code lets you inspect a sub-agent's full context. Go look at the first prompt the sub-agent received — imagine you're a new hire reading only that text. Do you have everything you need to complete the task? Usually you'll find several key pieces of information have been worn away — the sub-agent is often starting work from an ambiguous brief.

---

### **Q37. You have a set of knowledge or style-guide documents you want the AI to follow or be aware of. There are many ways to get that text loaded into the context — from manual copy-paste to having the model read it automatically. How many distinct mechanisms can you think of across this full design spectrum? What are the trade-offs of each? Try to design a classification scheme that covers them.**

**A37:** Take a code style guide as an example — you wrote one for your team and want the AI to always follow it. One reference classification splits the problem into two independent sub-dimensions.

**Sub-dimension 1: State management (Stateful vs Stateless)**

Imagine two common problems:

- **Duplicate injection (1→2)**: You type `/check-style` (a custom slash command — effectively the same as typing the full prompt template into the conversation), and the system injects the style guide. A little later you type it again — a **stateless** system would inject another copy, wasting tokens.
- **Loss and recovery (0→1)**: The style guide is already in context, but compaction compresses it into a summary ("we talked about code style"), and the details are gone. From this moment on, the AI no longer knows your specific style requirements.

Both problems have the same root cause: **the model has no reliable self-awareness of "what is currently in my context."**

The **Stateful** solution: the harness maintains an external registry of "what should be there." It deduplicates on repeated triggers and auto-recovers after compaction. **Stateless** re-injects the full text on every trigger, with no deduplication and no recovery.

In Claude Code, Command (deprecated) is stateless; Skill is stateful — which is exactly the core reason Skill replaced Command: it solved compact-safety and deduplication (see Q1).

**Sub-dimension 2: Trigger conditions (who decides to load?)**

- **User-driven**: The user explicitly invokes (`/command`, manual `@skill`). Most predictable, but the user must remember.
- **Harness-driven**: Programmatically triggered by the Coding CLI upon specific software events. The key trait is **deterministic and reproducible** — the same event always triggers the same injection, with no model judgment involved.
- **Model-driven**: The model decides autonomously based on semantics (Skill body lazy-load, MCP ToolSearch). Most flexible, but can misjudge and can forget.

Harness-driven triggers come in many forms, for example:

- **Session startup** (Always-on): Root CLAUDE.md, Auto Memory — loaded unconditionally at startup.
- **Path access** (Access-triggered): path-scoped rules, nested CLAUDE.md — auto-injected when the model reads a file under a matching path.
- **Turn boundary**: Each turn gets runtime metadata injected — current date, working-set summary (files being edited, recently mentioned paths).
- **Post-tool-execution**: After a file-editing tool runs, the harness queries the LSP server and injects diagnostics into context so the model sees compile errors/warnings before its next reasoning step.

Model-driven usually needs to work together with Harness-driven — the model must first know that a piece of knowledge exists before it can decide to load it when needed. Take Skills as an example: the Coding CLI always uses Harness-driven loading to inject all Skill metadata (name + description) at startup; the model sees this index and only actively loads the full body when it judges the current task requires it. This is **tiered loading** — metadata is guaranteed by the harness so "the model knows the capability exists"; the body is pulled in on demand by the model to save tokens (see Q12).

**Cross matrix:**

| | User-driven | Harness-driven | Model-driven |
| :---- | :---- | :---- | :---- |
| **Stateful** | Manual `@skill` | Root CLAUDE.md, Auto Memory, path-scoped rules, turn metadata, LSP diagnostics | Skill body, MCP ToolSearch |
| **Stateless** | `/command` (deprecated) | Counter-intuitive, shouldn't appear | Hand-rolled loading ("please read file X") |

**Only the bottom-left of the Stateless row (User-driven + Stateless) has existed in practice** — the rest of the Stateless combinations are either counter-intuitive or not viable. Claude Code's extension-mechanism evolution moves from the bottom of the table toward the top.

**Hand-rolled loading (degenerate mode)**: Writing "after startup, please read file X, and recursively read other files X references" in CLAUDE.md. Fully relies on the model's instruction-following with no harness support — works the first time, but after compaction it either re-reads (wasting tokens) or the model thinks "I've read it" and skips (loses information). It hands state-management responsibility to an executor that has no persistent state.

Back to the code style guide example, different scenarios call for different mechanisms:

- Project-level hard rules → root CLAUDE.md (Stateful + Always-on)
- Path-specific rules (e.g., security rules under `src/auth/*`) → path-scoped rule (Stateful + Access-triggered)
- Multi-project reusable check procedures → Skill (Stateful + tiered loading)

---

### **Q38. Skill and MCP are both "capability extensions." What's the fundamental difference in how they are distributed?**

**A38:** Skill and MCP both solve "how to expose additional capability to the AI," but their distribution mechanisms are completely different — they correspond to the two values of the independent dimension of "how capabilities are packaged and discovered."

**Convention-based**: Capabilities are declared via file-system structure. The system scans agreed-upon directory paths; specific directory names and file names tell it how to handle them. The path itself carries semantics.

```
.claude/skills/code-review/
├── SKILL.md         ← seeing SKILL.md tells the system this is a skill
├── checklist.md     ← auxiliary resource, referenced by relative path
├── examples/
└── scripts/
```

Distribution: git commit → git push → team members git pull, effective immediately.

- **Pros**: Transparent (`ls` shows everything), natural support for auxiliary resources (scripts and templates in the same directory), natural git versioning, zero runtime dependency.
- **Cons**: Traditionally requires cross-tool consensus — your `.claude/skills/` may not be recognized by other tools (though this barrier is shrinking: Cursor and DeepSeek-TUI, for example, now recognize `.claude/` directory structures). The capability set is static; it can't be dynamically added or removed at runtime.

Claude Code's Skill, Sub-agent, Rule, and Plugin are all convention-based.

**API-based**: Capabilities sit behind an independent process and self-describe via a standard protocol. The client connects and sends "list your capabilities"; the server returns structured capability descriptions. The client doesn't need to know how the server is internally organized.

Distribution: publish one MCP server; every MCP-aware client can use it.

- **Pros**: Strong encapsulation (implementation language and internal structure are opaque to the client), cross-platform (one service usable by every client), dynamic capability set (the service can return different tool lists based on state).
- **Cons**: Opaque (users cannot see inside the server; when things go wrong, you can't just `ls` to inspect), runtime dependency (needs to launch and manage an extra process), no natural place for auxiliary resources.

API-based also brings an incidental feature: **lifecycle management**. Since capability providers are independent processes, connect/disconnect events are natural lifecycle markers. For example, an LSP MCP server can cache results and progress from the previous round of code analysis, avoiding redundant recomputation on every call; a database MCP server needs to maintain a persistent network connection. These scenarios inherently require a long-lived process — something convention-based static files cannot replace.


##

## **Appendix: Recommended Learning Path**

1. **Beginner:** Read https://code.claude.com/docs/llms.txt; get familiar with `/skill`, `/agent`, `/plugin`, `/compact`, `/memory`.

2. **Intermediate:** Pick three things you do often (commit messages, PR summaries, running tests) and turn each into a skill.

3. **Advanced:** Write a plugin for your team that bundles skills + agents + hooks + an MCP server.

4. **Expert:** Study complex PreToolUse hook decisioning, the experimental Agent Teams feature, and checkpoint design for long-running (overnight) tasks.

---
