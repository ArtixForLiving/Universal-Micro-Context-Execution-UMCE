# Universal Micro-Context Execution (UMCE / MCMA)

A skill for AI coding agents that need to stay token-efficient when working on multi-file, long, or high-turn tasks within a single request.

## What is this?

UMCE is a context-efficiency protocol that prevents $\mathcal{O}(N^2)$ token snowballing within a single request. The more tool calls you make, the more history an agent must re-read every turn. UMCE reverses that trend by:

- Reading only the slices you need — never whole files
- Calling multiple tools in parallel in a single turn
- Editing surgically — matched-text replace, bottom-up
- Isolating state to scratchpad files instead of piling it in chat
- Circuit breaker after 3 consecutive identical failures, so you don't spiral forever

## When to use

| Use it | Don't use it |
|---|---|
| 5+ tool calls in one request | Single-file, single-line edit under 150 lines |
| 2+ files touched | Pure Q&A with no repository work |
| SDD, migrations, multi-file refactoring | Greenfield exploration where you don't yet know where the code lives |
| Build/test loops that repeat | Tasks that finish in under 5 turns |

> Calibration: measured A/B on an identical tree — one filter added to one 86-line file (7 turns): control 75k input tokens vs UMCE 103k input tokens (**+37%**, more expensive). Same experiment renaming a component across 5 files (10 turns): control 164k input tokens vs UMCE 153k input tokens (**–7%**, cheaper). It pays off on tasks that reuse context across many turns or many files; model estimates for larger loops are roughly **9–34%** savings at 15 turns with 600-line files, and **34–58%** at 20–40 turns with large files and verbose build loops. Not worth it on one-file one-edit jobs under ~5 tool calls.

## Installation

Clone this repository into your agent's skill directory:

```bash
# Claude Code
git clone <repo-url> ~/.claude/skills/universal-micro-context-execution

# Cross-runtime alias (Codex, Copilot CLI, Gemini CLI)
git clone <repo-url> ~/.agents/skills/universal-micro-context-execution
```

The skill is automatically detected through the `name` and `description` YAML frontmatter in `SKILL.md`.

## Repository structure

```
universal-micro-context-execution/
├── SKILL.md      # Full protocol (read by agents when relevant)
├── BENCHMARK.md  # Empirical calibration & RED-phase test scenarios
├── README.md     # This file (for humans)
├── LICENSE       # Usage license
└── .gitignore    # Files not to commit
```

## Key features

1. **Source-level ingestion filtering** — stop bloat before it enters context
2. **Parallel tool batching** — minimize inference turns
3. **Precise line targeting & anti-drift editing** — matched-text replace, bottom-up edits
4. **KV-cache safe state slimming** — telegraphic turns, scratchpad offloading
5. **Isolated sub-agent & SDD dispatch** — micro-payload without history
6. **Anti-spiral circuit breaker** — stop at strike 3

## License

See the `LICENSE` file in this repository.

## Contributing

Pull requests are welcome. If you find a loophole or a scenario where UMCE fails, open an issue with a RED-phase test scenario (see [BENCHMARK.md](BENCHMARK.md)).