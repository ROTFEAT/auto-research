# Auto-Research Skill Design Spec

> Multi-agent hypothesis-driven research skill for Claude Code

**Version:** v0.1 — 2026-03-18 — Initial design

## 1. Overview

`auto-research` is a skill that orchestrates three roles to systematically investigate complex questions:

- **Researcher (主session)**: Proposes hypotheses, analyzes evidence, coordinates the research loop
- **Executor (独立subagent)**: Verifies hypotheses through code experiments, web search, or data analysis
- **Checker (独立subagent)**: Guards convergence criteria and detects goal drift

Core principle: Executor and Checker are **fresh subagents each round** — zero context pollution. Only Researcher maintains cross-round memory.

> **Note:** This skill operationalizes the methodology from `program.md`. At runtime the skill follows `SKILL.md` only — `program.md` is not referenced.

## 2. Architecture

```
Main Session (Researcher)
  ├── dispatch → Executor subagent (new each round)
  │     └── Code / Web search / API calls → Evidence report
  ├── dispatch → Checker subagent (new each round)
  │     └── Original goal + convergence criteria + scoreboard → Judgment
  └── Local filesystem: research/<topic>/ (persistent state)
```

### 2.1 Role Definitions

| Role | Runtime | Responsibility | Input | Output |
|------|---------|----------------|-------|--------|
| Researcher | Main session | Propose hypotheses, analyze evidence, update scoreboard, decide next step | User question + evidence from each round | Hypothesis list, evidence plan, change proposals |
| Executor | Independent subagent (new per round) | Verify hypotheses per task instructions | Single-round task description + hypotheses + verification method | Evidence report (facts, sources, reliability grades) |
| Checker | Independent subagent (new per round) | Evaluate convergence, detect goal drift | Original goal + convergence criteria + current scoreboard + round changes + prior concerns | Judgment: CONVERGED / CONTINUE / DRIFT |

### 2.2 Isolation Principles

- Executor receives ONLY "what to do this round" — no knowledge of previous rounds
- Checker receives ONLY "original goal vs current state + accumulated open concerns" — unaffected by research process details
- Researcher holds global context, is the only role with cross-round memory
- Prompt templates enforce isolation by explicitly defining what each subagent receives

## 3. Research Loop

### Phase 0: Initialization

1. User states research question
2. Researcher parses question, generates 3-5 convergence criteria options
3. User selects/customizes convergence criteria
4. Researcher generates 3-7 initial hypotheses, presents to user for adjustment
5. Create workspace `research/<topic>/`, write `config.md`

### Phase 1-N: Research Rounds (max 10)

Each round:

1. **Researcher plans**: Select highest information-gain hypotheses to verify, define verification method (see Section 6.0 for decision protocol)
2. **Dispatch Executor**: Fresh subagent receives task description + target hypotheses + verification method. Executor can write code, search web, query APIs, analyze data. Returns evidence report.
3. **Handle Executor result** (see Section 3.1 for empty/unusable evidence handling)
4. **Researcher analyzes**: Update hypothesis scoreboard based on new evidence. Classify each hypothesis: Leading / Plausible / Weak / Refuted / Merged / Split. If all hypotheses are Refuted or Weak, generate 2-4 new hypotheses before proceeding (see Section 3.2).
5. **Dispatch Checker**: Fresh subagent receives original goal + convergence criteria + current scoreboard + round changes + prior open concerns. Returns judgment.
6. **Persist**: Save round output to `rounds/round-XX.md` (includes full scoreboard snapshot)
7. **Branch on judgment**:
   - `CONVERGED` → Phase Final
   - `CONTINUE` → Next round (back to step 1)
   - `DRIFT` → Rollback mechanism (see Section 4)

### Phase Final: Conclusion

Researcher generates final report (`conclusion.md`) containing:
- Final conclusion (one sentence)
- Confidence level (0-100) with reasoning
- Evidence chain (3-7 most critical pieces with sources)
- Rejected/weakened alternatives
- Uncertainties and knowledge gaps
- What would change the conclusion

### 3.1 Empty / Unusable Evidence Handling

When Executor returns empty or unusable results:

- **Zero evidence items**: Round is marked `[NO EVIDENCE]`. The round counts toward the max limit. Checker is still dispatched (it may detect that the verification approach is ineffective). Researcher must reformulate the verification method for the next round.
- **All evidence is D-grade**: Researcher proceeds but annotates the scoreboard that updates are based on low-reliability evidence only. Confidence scores must not increase based solely on D-grade evidence.
- **Off-topic evidence**: Researcher discards off-topic items, notes the issue, and re-examines whether the task description was clear enough. Counts as a partial round.

### 3.2 Hypothesis Exhaustion

When all hypotheses reach Refuted or Weak status:

1. Researcher generates 2-4 new hypotheses based on accumulated evidence and lessons learned
2. New hypotheses are displayed to the user for confirmation
3. The scoreboard is updated with new hypotheses (old Refuted ones remain for the record)
4. This counts as the planning phase for the round — no separate Executor dispatch needed if the new hypotheses are structurally different

### 3.3 Hypothesis Merge / Split

- **Merged**: When two hypotheses are essentially the same or one subsumes the other. The scoreboard records `H2 → merged into H1` with reasoning. Evidence from both carries forward to the merged hypothesis.
- **Split**: When a hypothesis is too broad to test. The scoreboard records `H3 → split into H3a, H3b` with reasoning. The original hypothesis is marked as Split (inactive).

## 4. DRIFT Rollback Mechanism

When Checker returns `DRIFT`:

1. Checker output includes: drift direction + drift cause + correction advice
2. Current round's `round-XX.md` is marked `[DRIFTED]` — preserved for audit but not used as basis
3. `scoreboard.md` rolls back to the snapshot stored in `round-(XX-1).md`
4. Evidence from drifted round is marked `[DRIFTED]` in `evidence-log.md` (not deleted)
5. Researcher re-plans with awareness of drift cause (the "lesson")
6. **DRIFT rounds count toward the max round limit** — prevents repeated drift from exhausting tokens

Why rollback over correction:
- Correction risks compounding errors — drifted reasoning subtly influences the "corrected" direction
- Rollback to last clean checkpoint ensures no contaminated reasoning carries forward
- Drift cause is preserved as a lesson, so the same drift is unlikely to recur

## 5. File Structure & Persistence

```
research/<topic>/
├── config.md              # Research config (question, convergence criteria, max rounds)
├── scoreboard.md          # Current hypothesis scoreboard (overwritten each round)
├── evidence-log.md        # Cumulative evidence log (append-only)
├── rounds/
│   ├── round-01.md        # Round 1 full output (includes scoreboard snapshot)
│   ├── round-02.md        # Round 2 full output
│   ├── round-03.md        # [DRIFTED] marked drift round
│   └── ...
└── conclusion.md          # Final conclusion report (generated on convergence)
```

### 5.1 File Behaviors

- `scoreboard.md`: Overwrite each round (always reflects latest state). On DRIFT, restore from previous round file's snapshot.
- `evidence-log.md`: Append-only. DRIFTED round evidence marked but not deleted. Evidence IDs are globally sequential (E1, E2, E3...) and never renumbered, even on DRIFT.
- `rounds/round-XX.md`: Immutable snapshots. Each round includes a full scoreboard snapshot for rollback support. Even DRIFTED rounds preserved for audit.
- `config.md`: Written once at initialization, read-only thereafter.

### 5.2 Round File Format

```markdown
# Round XX [STATUS: CONTINUE/CONVERGED/DRIFTED/NO_EVIDENCE]

## Researcher Task Assignment
- Verification goal: ...
- Target hypotheses: ...
- Verification method: code_experiment / web_search / data_analysis / mixed

## Executor Evidence Report
| ID | Claim | Source | Source Type | Reliability | Supports | Weakens |
...

## Researcher Hypothesis Update
- Changes: ...
- Reasoning: ...

## Scoreboard Snapshot
<!-- Full scoreboard at this point, used for DRIFT rollback -->
| Hypothesis | Status | Supporting Evidence | Weakening Evidence | Confidence (0-100) |
...

## Checker Judgment
- Judgment: CONTINUE/CONVERGED/DRIFT
- Reasoning: ...
- Open concerns: ...
```

### 5.3 Scoreboard Format

Confidence uses a 0-100 numeric scale for granularity. The final `conclusion.md` translates to High (70-100) / Medium (40-69) / Low (0-39) for human readability.

```markdown
| Hypothesis | Status | Supporting Evidence | Weakening Evidence | Confidence (0-100) |
|------------|--------|--------------------|--------------------|---------------------|
| H1: ...    | Leading | E1, E3            | E2                 | 75                  |
| H2: ...    | Weak    | E2                | E1, E3, E5         | 20                  |
| H3: ...    | → merged into H1 | — | — | — |
| H4: ...    | → split into H4a, H4b | — | — | — |
```

Valid statuses: `Leading` / `Plausible` / `Weak` / `Refuted` / `→ merged into HX` / `→ split into HXa, HXb`

### 5.4 Evidence Log Format

IDs are globally sequential and never change, even on DRIFT.

```markdown
| ID | Claim | Source | Reliability | Supports | Weakens | Round | Status |
|----|-------|--------|-------------|----------|---------|-------|--------|
| E1 | ...   | API test | A         | H1       | H2      | R1    | Valid  |
| E4 | ...   | Web search | B       | H3       | H1      | R3    | DRIFTED |
```

## 6. Prompt Templates

### 6.0 Researcher Decision Protocol

The Researcher is the main session following `SKILL.md`. Its decision-making is governed by these rules:

**Information-Gain Selection Heuristic:**
1. Prioritize hypotheses closest to the Leading/Plausible boundary (most likely to change ranking with new evidence)
2. Prioritize hypotheses that could be refuted with a single piece of evidence (high elimination value)
3. If multiple hypotheses are clustered at similar confidence, pick the one with the least evidence (most under-investigated)
4. Avoid re-testing hypotheses that already have 3+ pieces of consistent evidence

**Verification Method Selection:**
- `code_experiment`: When the hypothesis can be tested by writing and running code (API calls, scraping, computation, benchmarks)
- `web_search`: When the hypothesis requires finding existing information, reports, or external data
- `data_analysis`: When the hypothesis can be tested against existing local data or datasets
- `mixed`: When the verification requires a combination of the above

**Incorporating Checker Feedback:**
- `CONTINUE` with suggestions: Researcher MUST address the suggested focus area in the next round's planning
- `DRIFT` with correction advice: Researcher reads the drift cause, rolls back, and explicitly avoids the identified drift pattern
- Open concerns from Checker: Researcher tracks these in a cumulative list and passes to the next Checker

### 6.1 Executor Prompt Template

Located at: `auto-research/executor-prompt.md`

Placeholders:
- `{RESEARCH_QUESTION}`: Original research question
- `{VERIFICATION_GOAL}`: What to verify this round
- `{HYPOTHESES_TO_VERIFY}`: Specific hypotheses to test
- `{VERIFICATION_METHOD}`: code_experiment / web_search / data_analysis / mixed

Executor constraints:
- Only perform the described verification task, do not expand scope
- Distinguish facts from inferences, label explicitly
- Every evidence item must have source and reliability grade (A/B/C/D)
  - A: Original data, official documents, primary papers, traceable sources
  - B: High-quality summaries, professional media, industry research
  - C: Experiential articles, limited-evidence analysis
  - D: Rumors, unverified social media, opinionated content

Output format: Evidence table + verification process + unverified items + unexpected findings

### 6.2 Checker Prompt Template

Located at: `auto-research/checker-prompt.md`

Placeholders:
- `{ORIGINAL_QUESTION}`: Original research question (never changes)
- `{CONVERGENCE_CRITERIA}`: User-defined convergence criteria (never changes)
- `{CURRENT_SCOREBOARD}`: Current hypothesis scoreboard
- `{ROUND_CHANGES}`: Summary of what changed this round
- `{CURRENT_ROUND}` / `{MAX_ROUNDS}`: Progress tracking
- `{PRIOR_CHECKER_CONCERNS}`: Cumulative list of unresolved concerns from prior Checkers (maintained by Researcher)

Checker checklist:
1. Convergence check: Does current state satisfy user-defined convergence criteria?
2. Goal drift check: Are current hypotheses still answering the original question?
3. Evidence sufficiency: Does the leading hypothesis have support from multiple independent sources?
4. Round efficiency: Are remaining rounds sufficient to complete research?
5. Prior concerns: Have previously raised concerns been addressed?

Output format: Judgment (CONVERGED/CONTINUE/DRIFT) + reasoning per check item + open concerns list + next-step recommendations

## 7. User Interaction Points

### 7.1 Initialization (Interactive)

1. User inputs research question
2. Researcher generates 3-5 convergence criteria options → user selects (multi-select + custom)
3. Researcher generates 3-7 initial hypotheses → user can add/remove/adjust

### 7.2 Per-Round Summary (Informational)

After each round, display to user:
- Round finding summary (1-3 sentences)
- Current scoreboard
- Checker judgment
- If CONTINUE: "Continuing to next round..." (user can interrupt to stop)

### 7.3 Critical Notifications

- DRIFT detected: Notify user of drift direction and cause, explain rollback
- Max rounds reached without convergence: Generate `conclusion.md` with mandatory `Confidence: Low — forced stop at max rounds` annotation. Checker is NOT dispatched on forced stop (the stop is procedural, not evidential).
- Hypothesis exhaustion: Notify user that new hypotheses are needed, present for approval

### 7.4 Automation Level & User Stop Mechanism

The research loop runs automatically between user intervention points. User does NOT need to approve each round — only notified at checkpoints.

**How to stop:** The user sends a message during the per-round summary phase (between rounds). The Researcher checks for user input after displaying each round summary. If the user signals stop:
1. The current in-progress round (if any) completes normally
2. No new round is started
3. Researcher generates `conclusion.md` from current scoreboard state with a note: `Research stopped by user at round X`

## 8. Skill File Structure

```
auto-research/
├── SKILL.md                  # Main skill definition (entry, flow, rules)
├── executor-prompt.md        # Executor subagent prompt template
├── checker-prompt.md         # Checker subagent prompt template
└── templates/
    ├── config-template.md    # config.md template
    ├── round-template.md     # round-XX.md template
    ├── scoreboard-template.md # scoreboard.md template
    └── conclusion-template.md # conclusion.md template
```

## 9. Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|----------------|------------------|
| Passing session history to Executor | Context pollution, Executor may be influenced by prior conclusions | Only pass current round's task description |
| Passing research process to Checker | Checker loses objectivity | Only pass original goal + current state + open concerns |
| Skipping Checker when "nothing changed" | Even stable rounds need convergence check | Always dispatch Checker every round |
| Correcting DRIFT instead of rolling back | Drifted reasoning contaminates corrections | Rollback to last clean state |
| Letting Executor expand verification scope | Scope creep wastes rounds | Researcher pre-defines exact verification task |
| Continuing after max rounds | Infinite token consumption | Stop, present best current conclusion |
| Single hypothesis from start | Confirmation bias | Always maintain 3-7 competing hypotheses |
| Increasing confidence on D-grade evidence | Unreliable sources inflate certainty | D-grade evidence cannot raise confidence scores |
| Ignoring prior Checker concerns | Quality issues compound across rounds | Pass cumulative concerns to each Checker |

## 10. Convergence Criteria Examples

To help users define good convergence criteria, Researcher should generate options like:

- "At least N hypotheses refuted with evidence grade B or above"
- "Leading hypothesis supported by 3+ independent sources"
- "Leading hypothesis confidence reaches 70+ with no competing hypothesis above 40"
- "Able to answer the original question with high confidence"
- "Key unknowns identified and documented, even if not all resolved"
- Domain-specific criteria based on the question type

## 11. Context Window Management

As rounds progress, the Researcher's main session context grows. To manage this:

- After round 5, Researcher should summarize earlier rounds into a compact form, keeping only: scoreboard, evidence log, key lessons, and open concerns
- Subagent isolation naturally prevents context bloat in Executor and Checker (fresh each round)
- The file-based persistence (`rounds/`, `scoreboard.md`, `evidence-log.md`) serves as the source of truth — the Researcher can re-read files if context is compressed
