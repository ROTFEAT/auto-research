# Auto-Research Skill

Multi-agent hypothesis-driven research skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

Three isolated roles collaborate to systematically investigate complex questions:

| Role | Runtime | Job |
|------|---------|-----|
| **Researcher** | Main session | Proposes hypotheses, analyzes evidence, coordinates loop |
| **Executor** | Fresh subagent each round | Verifies hypotheses via code / web search / data analysis |
| **Checker** | Fresh subagent each round | Guards convergence criteria, detects goal drift |

Executor and Checker are **context-isolated** — spawned fresh every round with zero knowledge of previous rounds. Only Researcher maintains cross-round memory.

## Installation

### Option 1: Skills CLI (Recommended)

```bash
npx skills add ROTFEAT/auto-research --all
```

This installs the skill to all detected agents (Claude Code, Cursor, Codex, etc.).

For global installation:

```bash
npx skills add ROTFEAT/auto-research --all -g
```

### Option 2: Manual Installation

Clone and symlink into your Claude Code skills directory:

```bash
git clone https://github.com/ROTFEAT/auto-research.git
mkdir -p ~/.claude/skills
ln -s "$(pwd)/auto-research/skills/auto-research" ~/.claude/skills/auto-research
```

Or copy directly:

```bash
git clone https://github.com/ROTFEAT/auto-research.git
cp -r auto-research/skills/auto-research ~/.claude/skills/
```

### Verify Installation

In Claude Code, the skill should appear when you run `/skills` or when Claude detects a research-type question.

## Usage

In a Claude Code session, say something like:

```
研究一下 Polymarket 的做市商策略是什么
```

```
Research whether WebSocket or SSE is better for real-time dashboards
```

```
调查 Rust 在嵌入式领域的采用率趋势
```

The skill automatically activates when it detects a complex research question.

## How It Works

```
Phase 0: Initialization
  ┌─ User states research question
  ├─ AI generates convergence criteria → User selects
  ├─ AI generates 3-7 initial hypotheses → User adjusts
  └─ Creates research/<topic>/ workspace

Phase 1-N: Research Loop (max 10 rounds)
  ┌─────────────────────────────────────────┐
  │ 1. Researcher plans verification task   │
  │ 2. → Dispatch Executor (fresh subagent) │
  │ 3. Researcher analyzes evidence         │
  │ 4. → Dispatch Checker (fresh subagent)  │
  │ 5. Save round to file                  │
  │ 6. CONVERGED → done                    │
  │    CONTINUE  → next round              │
  │    DRIFT     → rollback & retry        │
  └─────────────────────────────────────────┘

Phase Final: Conclusion
  └─ Generate conclusion.md with evidence chain
```

### DRIFT Rollback

When the Checker detects goal drift, the system **rolls back** (not corrects) to the last clean state. The drifted round is preserved for audit but not used as basis. This prevents compounding errors from contaminated reasoning.

### File Output

All research output is persisted to local files:

```
research/<topic>/
├── config.md              # Question, convergence criteria, max rounds
├── scoreboard.md          # Current hypothesis scoreboard
├── evidence-log.md        # Cumulative evidence (append-only)
├── rounds/
│   ├── round-01.md        # Full round snapshot (immutable)
│   ├── round-02.md
│   └── ...
└── conclusion.md          # Final report
```

## Skill Structure

```
skills/auto-research/
├── SKILL.md                  # Main skill definition
├── executor-prompt.md        # Executor subagent prompt template
├── checker-prompt.md         # Checker subagent prompt template
└── templates/
    ├── config-template.md
    ├── round-template.md
    ├── scoreboard-template.md
    └── conclusion-template.md
```

## Key Design Decisions

- **Subagent isolation**: Executor and Checker never see session history — prevents confirmation bias and context pollution
- **Rollback over correction**: DRIFT triggers a rollback to the last clean checkpoint, not an in-place fix
- **Numeric confidence (0-100)**: Granular tracking of hypothesis confidence across rounds
- **Hypothesis evolution**: Supports Merge (combining redundant hypotheses) and Split (decomposing broad ones)
- **Evidence grading (A-D)**: Every evidence item is graded by source reliability; D-grade cannot raise confidence

## Design Spec

Full design specification: [`docs/superpowers/specs/2026-03-18-auto-research-design.md`](docs/superpowers/specs/2026-03-18-auto-research-design.md)

## License

MIT
