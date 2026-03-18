---
name: auto-research
description: >
  Multi-agent hypothesis-driven research. Researcher proposes,
  Executor verifies, Checker guards convergence. Use for any
  complex question requiring systematic evidence collection.
---

# Auto-Research: Multi-Agent Hypothesis-Driven Research

Systematically investigate complex questions using three isolated roles: Researcher (you), Executor (subagent), and Checker (subagent).

## When to Use

- Complex questions that cannot be answered with a single search or code run
- Questions where multiple explanations are plausible and need evidence to distinguish
- Research that requires iterating between hypothesis and verification
- Any question where you need to avoid confirmation bias

## The Three Roles

| Role | Who | Isolation |
|------|-----|-----------|
| **Researcher** | You (main session) | Full context across rounds |
| **Executor** | Fresh subagent per round | Sees ONLY current round's task |
| **Checker** | Fresh subagent per round | Sees ONLY original goal vs current state |

<HARD-GATE>
NEVER pass session history to Executor or Checker. NEVER skip Checker dispatch. NEVER continue past max rounds. These rules are non-negotiable.
</HARD-GATE>

## Process

### Phase 0: Initialization

1. **Parse the research question.** Reframe as a verifiable question if needed.

2. **Generate 3-5 convergence criteria options** using AskUserQuestion (multi-select). Examples:
   - "Leading hypothesis confidence reaches 70+ with no competitor above 40"
   - "At least N hypotheses refuted with evidence grade B+"
   - "Leading hypothesis supported by 3+ independent sources"
   - Generate domain-specific criteria based on the question type

3. **Generate 3-7 initial hypotheses.** Present to user for adjustment. Always include:
   - Conventional explanation
   - Counter-intuitive explanation
   - Simplest baseline
   - Measurement bias / null hypothesis

4. **Create workspace:**
   - Create directory `research/<topic-slug>/rounds/`
   - Write `config.md` from `templates/config-template.md`
   - Initialize `scoreboard.md` from `templates/scoreboard-template.md`
   - Initialize empty `evidence-log.md`

### Phase 1-N: Research Rounds (max rounds from config, default 10)

**For each round:**

#### Step 1: Researcher Plans

Select hypotheses to verify using the information-gain heuristic:
1. Prioritize hypotheses closest to the Leading/Plausible boundary
2. Prioritize hypotheses that could be refuted with a single piece of evidence
3. If tied, pick the least-investigated hypothesis
4. Avoid re-testing hypotheses with 3+ consistent evidence pieces

Choose verification method:
- `code_experiment`: Test by writing/running code (API calls, scraping, computation)
- `web_search`: Find existing information, reports, external data
- `data_analysis`: Analyze existing local data or datasets
- `mixed`: Combination of the above

#### Step 2: Dispatch Executor

Read `executor-prompt.md`, fill placeholders, dispatch as Agent subagent (`subagent_type: "general-purpose"`).

**Executor gets ONLY:**
- Research question
- This round's verification goal
- Target hypotheses
- Verification method

**Executor NEVER gets:** session history, previous rounds, scoreboard, evidence log.

#### Step 3: Handle Executor Result

- **Normal evidence:** Proceed to Step 4.
- **Zero evidence items:** Mark round as `[NO EVIDENCE]`. Still dispatch Checker. Reformulate verification method next round.
- **All D-grade evidence:** Proceed but annotate scoreboard. D-grade evidence CANNOT raise confidence scores.
- **Off-topic evidence:** Discard off-topic items, note the issue, re-examine task clarity.

#### Step 4: Researcher Analyzes

Update the hypothesis scoreboard:
- Classify each: `Leading` / `Plausible` / `Weak` / `Refuted`
- **Merge** if two hypotheses are essentially the same: `H2 → merged into H1`
- **Split** if a hypothesis is too broad to test: `H3 → split into H3a, H3b`
- Update confidence scores (0-100)
- Append new evidence to `evidence-log.md` with globally sequential IDs

**If all hypotheses are Refuted/Weak:** Generate 2-4 new hypotheses, present to user for confirmation before proceeding.

#### Step 5: Dispatch Checker

Read `checker-prompt.md`, fill placeholders, dispatch as Agent subagent (`subagent_type: "general-purpose"`).

**Checker gets:**
- Original question (from config.md, never changes)
- Convergence criteria (from config.md, never changes)
- Current scoreboard
- This round's changes summary
- Prior Checker open concerns (cumulative list you maintain)
- Current round / max rounds

#### Step 6: Persist

Save round to `rounds/round-XX.md` using `templates/round-template.md`. The round file MUST include a **full scoreboard snapshot** (used for DRIFT rollback). Update `scoreboard.md`.

#### Step 7: Branch

- **CONVERGED** → Phase Final
- **CONTINUE** → Display round summary to user, proceed to next round
- **DRIFT** → Execute rollback (see DRIFT Rollback below)

### DRIFT Rollback

1. Mark current `round-XX.md` as `[DRIFTED]`
2. Read the scoreboard snapshot from `round-(XX-1).md`
3. Overwrite `scoreboard.md` with that snapshot
4. Mark this round's evidence as `[DRIFTED]` in `evidence-log.md`
5. Notify user: drift direction, cause, and what you'll do differently
6. Re-plan with the drift lesson. **DRIFT rounds count toward max limit.**

### Phase Final: Conclusion

Generate `conclusion.md` from `templates/conclusion-template.md`:
- Final conclusion (one sentence)
- Confidence level (0-100) + High/Medium/Low label
- Evidence chain (3-7 critical pieces with sources)
- Rejected alternatives
- Uncertainties and knowledge gaps
- What would change the conclusion

### Forced Stop (Max Rounds Reached)

If max rounds reached without convergence:
- Generate `conclusion.md` with `Confidence: Low — forced stop at max rounds`
- Do NOT dispatch Checker (stop is procedural)
- Present best current conclusion + unresolved uncertainties to user

### User Stop

User can send a message between rounds to stop:
- Current in-progress round completes
- No new round starts
- Generate `conclusion.md` with note: `Research stopped by user at round X`

## Per-Round User Display

After each round, show:
```
### Round X Summary
[1-3 sentence finding summary]

| Hypothesis | Status | Confidence |
|------------|--------|------------|
| H1: ...    | Leading | 75        |
| ...        | ...     | ...       |

Checker: [CONTINUE/CONVERGED/DRIFT] — [brief reasoning]
```

## Context Window Management

After round 5, summarize earlier rounds into compact form. Keep only:
- Current scoreboard
- Evidence log
- Key lessons and open concerns
- File-based persistence is the source of truth — re-read files if needed

## Anti-Patterns — DO NOT

| DO NOT | DO INSTEAD |
|--------|------------|
| Pass session history to Executor | Only pass current round's task |
| Pass research process to Checker | Only pass original goal + current state + open concerns |
| Skip Checker when "nothing changed" | Always dispatch Checker every round |
| Correct DRIFT instead of rolling back | Rollback to last clean state |
| Let Executor expand scope | Pre-define exact verification task |
| Continue after max rounds | Stop, present best conclusion |
| Start with single hypothesis | Always 3-7 competing hypotheses |
| Raise confidence on D-grade evidence | D-grade cannot raise scores |
| Ignore prior Checker concerns | Pass cumulative concerns to each Checker |
