# Reflection Brief

## Environment

- **Model(s):** System 1 used `claude-haiku-4-5-20251001`; System 2 used Anthropic model-authoritative token counting through `messages.count_tokens`.
- **OS/Python:** Linux/Vocareum environment; Python 3.13 runtime artifacts were present.
- **Approx API spend:** The latest System 1 all-fixture live run recorded approximately **$0.1078** (`runs/20261003_075406`), and a representative claim run recorded **$0.0322** (`runs/20261003_064204`). The evidence does not record a separate combined dollar total for all four systems.

## Part 1 — System 1: Stop-Reason-Driven Loop

### 1. Loop control

The representative trace in `evidence/system1_stop_reason_loop/trace_sample.jsonl` shows the sequence `tool_use → tool_use → tool_use → tool_use → tool_use → tool_use → end_turn` across 7 turns. The trace contains policy lookup, fact recording, clarification, classification/severity assessment, routing, and finally `end_turn`. In `claims_intake/loop.py`, continuation is controlled by the API `stop_reason`: `tool_use` continues the loop, `end_turn` returns, and unexpected stop reasons raise an error. This makes the model/API termination signal the control mechanism rather than a hard-coded iteration count.

### 2. Anti-pattern

`tests/test_antipatterns.py::test_no_integer_literal_iteration_cap_in_loop` rejects a `for range(<integer>)` or `while ... < <integer>` iteration cap in `loop.py`. Such a cap would make loop termination depend on an arbitrary fixed number of iterations instead of the model's `stop_reason`. The same test suite also checks that stop-reason logic is actually present in the loop. The full System 1 test suite recorded **29 passed**.

### 3. Tool design

Two terminal decision tools are `route_to_adjuster` and `escalate_to_human`: both end the workflow, but routing handles sufficiently confident classified claims while escalation handles unresolved or unsafe cases. `escalate_to_human` returns a structured summary containing fields such as policy ID, root cause, candidate claim types, case facts, recommended action, and confidence. Tool failures are represented by `_err(...)` as structured JSON with `is_error`, `error_category`, `is_retryable`, and `message` instead of a generic string or exception. This gives the model explicit information about whether a failure can be retried.

### 4. Numbers

The representative claim-03 run is `runs/20261003_064204`: it took **7 turns**, included **1 clarification**, used **26,064 input tokens and 1,224 output tokens**, and cost approximately **$0.0322**. The README estimates approximately **$0.05** for running all eight fixtures, so the single-claim run is naturally lower than that sample estimate. The latest all-eight live run, `runs/20261003_075406`, cost approximately **$0.1078**, showing that actual usage can vary substantially from a README estimate depending on the live model/tool trajectory.

## Part 1 — System 2: Long-Conversation Context Strategy

### 5. Reduction

The System 2 evidence in `evidence/system2_context_strategy/budget.json` reports a baseline of **38,708 tokens** and an assembled context of **16,852 tokens**, a **56.46% reduction**. The assembled context contains `case_facts` at 204 tokens, resolved refund at 412, resolved subscription at 465, and the active section at 15,789 tokens. The active section dominates because it is deliberately preserved verbatim rather than aggressively summarized. That preserves the exact ongoing conversation while compressing already-resolved material.

### 6. Summarize vs preserve

The assembly rule is to summarize resolved issues but preserve the active issue byte-for-byte, while placing case facts first and resolved issues before the active issue. The recorded section sizes were **204 tokens for case_facts, 412 for resolved_refund, 465 for resolved_subscription, and 15,789 for active** (`budget.json`). The resolved refund and subscription inputs were compressed from **12,334 to 399 tokens** and **11,475 to 452 tokens**, respectively. This makes the distinction concrete: resolved history is compressed, while the active working context remains intact.

### 7. Facts block

The evaluation evidence `evidence/system2_context_strategy/eval.jsonl` shows **Q1–Q6 all passing (6/6)** against the assembled context. The control evidence `eval_control.jsonl` shows Q1 unexpectedly passing but Q6 failing: when asked for the exact structured status token `in_progress`, the control model said that no structured status token was present. The compressed/evaluation context therefore preserved information needed for the expected answers, while the control context did not preserve the exact status representation. This demonstrates why the facts/context assembly contract matters rather than relying on a generic summary.

## Part 1 — System 3: Multi-Surface Monorepo Team

### 8. Path-scoped rules

The path-scoped rule in `.claude/rules/tests.md` has YAML frontmatter with `paths: "**/*.test.tsx"` and `paths: "**/*.test.ts"`. Its body states that it loads when editing any co-located test file anywhere in the tree and can apply on top of React or API rules. This is better than putting the same rule only in a directory `CLAUDE.md` because the testing convention is cross-cutting and should follow matching files regardless of which application directory contains them. The artifact is `/workspace/Configure Claude Code for a Multi-Surface Monorepo Team/04-plan-mode-and-explore-decision-doc/solution/.claude/rules/tests.md`.

### 9. Forked skill

The deploy-check skill frontmatter explicitly contains `context: fork` and an `allowed-tools` list limited to read-oriented operations such as `Read`, `Grep`, `Glob`, selected Git commands, and GitHub PR/check commands. The skill explains that verbose discovery output stays in the fork while only a structured pass/fail summary returns to the main session (`.claude/skills/deploy-check/SKILL.md`). Without the fork, file enumeration, diff inspection, and check traces would pollute the main conversation context. The skill therefore isolates task-specific, verbose validation while keeping the parent session focused.

### 10. Scope

The System 3 hierarchy tests distinguish project-level configuration from user-level configuration. The project-level configuration is stored in the repository, such as `.claude/commands/`, `.claude/rules/`, and `.claude/skills/`, so the whole team can receive the same behavior through version control. The user-level scope is `~/.claude/` and is personal rather than shared through repository version control; the tests require a concrete user-scope example. This distinction is enforced by `tests/test_us01_claude_md_hierarchy.py`, and the System 3 test suite recorded **35 passed**.

## Part 1 — System 4: Multi-Shift Quality Monitoring

### 11. Push work down

The warm database contains historical defect records, but the orchestration pipeline asks the database for only defects newer than its checkpoint. The indexed query in `shift_monitor/warm.py` is `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, with indexes including `idx_defects_ts` and `idx_defects_shift_ts`. The recorded shift run in `evidence/system4_orchestration/shift_run_output.txt` used a `since` checkpoint and reported `new=0`, while the recorded shift summary contained the relevant shift findings. Because filtering and limiting happen in the SQL/warm tier through `gather_new_defects()` → `defects_since()`, the model does not need to receive the full warm history.

### 12. Crash recovery

`03-crash-recovery/solution/shift_monitor/recovery.py` defines `STALE_RESUME_THRESHOLD_MINUTES = 30`. An incomplete state resumes when its latest step is no more than 30 minutes old; a state with no steps, a completed state, or an incomplete state older than 30 minutes starts fresh. A fresh run can inject the findings already captured in the manifest as a summary, which can be more reliable than resuming stale intermediate context. This avoids carrying potentially obsolete or partially corrupted working context into a new shift cycle.

### 13. Small state

The evidence file `04-fork-scratchpad/solution/evidence/system4_orchestration/hot_state_size.txt` records `data/hot_state.json` at **643 bytes**, well below the 5 KB target. Keeping hot state this small limits the amount of state that must be carried into context and makes crash recovery and hypothesis forking inexpensive. It also encourages the system to store compact operational state rather than treating the model context as a database. The measured 643-byte artifact is the authoritative evidence for this question.

## Part 2 — Synthesis

### 14. Three layers

The **Model** layer provides reasoning and tool selection, as seen in System 1's stop-reason/tool-use trace and System 3's task-specific skill. The **Harness** layer provides deterministic enforcement and context management, including `loop.py` stop-reason handling, System 2's assembly contract, path-scoped rules, and the forked deploy-check skill. The **Orchestration** layer manages state and workflow across time, including System 4's SQL pre-filtering, crash recovery decision, and isolated hypothesis scratchpads.

### 15. Deterministic vs prompt-guided

A deterministic behavior is System 2's assembly contract: the active section is preserved byte-for-byte and the required headers/order are enforced by code and tests. A prompt-guided behavior is System 1's instruction to classify exactly once, assess severity once, clarify genuinely missing information, and then route or escalate. The deterministic contract can be verified directly by tests, while the prompt-guided behavior still depends on the model following the workflow and producing appropriate tool calls. This separation is important because prompts guide behavior but code should enforce critical invariants.

### 16. Context has two faces

System 2 manages context **within a long conversation** by reducing 38,708 baseline tokens to 16,852 while preserving the active section and compressing resolved history (`budget.json`). System 4 manages context **across sessions/time** by keeping a 643-byte hot state and deciding whether to resume or start fresh based on a 30-minute threshold. One controls token pressure inside a conversation, while the other controls how much stale operational state is carried across a crash or shift boundary. Both treat context as a bounded engineering resource rather than an unlimited transcript.

### 17. Reliability not visible in one run

System 2's test suite is evidence of a behavior that a single successful model run would not establish: the assembly contract and anti-pattern checks are repeatedly verified, and the full suite recorded **30 passed** in `evidence/system2_context_strategy/pytest_S2.log`. A single run could appear correct even if the implementation silently changed ordering, dropped the active section, or violated the required budget. Tests expose those regressions without depending on one particular model response. This is a key difference between demonstrated behavior and enforced reliability.

### 18. Blast radius

System 4's hypothesis forking has a deliberately bounded blast radius: `fork.py` creates a separate fork directory and copies `hot_state.json` into it, while findings are kept in isolated scratchpads before being merged. This means an exploratory hypothesis cannot directly contaminate the main scratchpad during investigation. The practical containment mechanism is the orchestration boundary itself: the fork is isolated, the merge is explicit, and exploratory findings do not become main state until they are deliberately merged. This limits failures to the fork during investigation.

## Part 3 — Honest Assessment

### 19. What broke

The first System 1 live all-eight run did not achieve the README's expected **7 routed, 1 escalated** outcome; the latest recorded run `runs/20261003_075406` produced **3 routed, 5 incomplete, and 0 escalated**, at approximately **$0.1078**. A representative claim-03 run did successfully complete with 7 turns and route after one clarification, so the underlying loop and tools can work on individual cases, but the full fixture run was not reliable enough to claim the expected result. The honest lesson is that the implementation and tests were structurally correct while the live model behavior still had incomplete outcomes that require further investigation before claiming full acceptance.

### 20. What I would change architecturally

The most important architectural change I would make is to move more critical workflow invariants from prompt guidance into deterministic orchestration checks. System 1 demonstrates why: the code correctly interprets `stop_reason`, but the live eight-case run still produced incomplete outcomes even though the unit tests passed. I would keep the model responsible for reasoning and tool selection while making the harness explicitly detect incomplete terminal workflows, preserve the last useful state, and apply a controlled recovery or escalation path. This keeps model flexibility while reducing the operational risk of silently incomplete workflows.


