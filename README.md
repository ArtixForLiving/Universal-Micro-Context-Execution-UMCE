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

> Calibration: on a small task (one file, 86 lines), UMCE is actually **more expensive** (+37% input tokens) because of up-front orientation cost. It pays off on tasks that reuse context across many turns or many files, not on one-file one-edit jobs.

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

Pull requests are welcome. If you find a loophole or a scenario where UMCE fails, open an issue with a RED-phase test scenario (see the Appendix in `SKILL.md`).