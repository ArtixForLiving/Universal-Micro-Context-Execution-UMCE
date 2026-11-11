# UMCE Benchmarks & RED-Phase Test Scenarios

Empirical calibration data, verification scenarios, and release-gate checks for [SKILL.md](SKILL.md). Extracted from the core skill file in `v2.1.0` to save ~450 baseline tokens per turn.

## Baseline Calibration (A/B Measurements)

Both arms ran from identical directory trees and produced equivalent diffs with green builds.

### 1. Small Single-File Task (86-line file, 1 filter, build verification)

| Arm | Turns | Iterations | Input tokens | Read & execution pattern |
|---|---|---|---|---|
| Control | 5 | 6 | 74,952 | One 30-line slice, no size check |
| Skill (`v2.0.0`) | 5 | 8 | 103,180 | 3 parallel searches, 2 slices, whole-file read, `RISK` line |

**Findings (`v2.0.0`):**
- **What held:** The skill arm batched searches, reported files by line range, and appended an exit status.
- **Overhead cost:** `+37%` input tokens and 2 extra iterations for a small single-file edit. The shell line-count check was the single largest overhead on small tasks.
- **Dry-run gaps found:**
  1. Non-git workspaces lack `git diff --stat` (addressed with a `Compare-Object` pre-edit backup fallback).
  2. Empty filtered output is ambiguous without the command's exit code.
  3. Trailing `echo "exit=$?"` reports the exit code of `head`/`grep` instead of the primary command (`v2.1.0` fixes this with `${PIPESTATUS[0]}${pipestatus[1]}` on bash/zsh and `$_ec = $LASTEXITCODE` before filtering on PowerShell).

### 2. Multi-File & Long-Loop Benchmarks

| Task | Control | Skill (`v2.0.0`) | Delta | Source |
|---|---|---|---|---|
| One filter, 86-line file, 7 turns | 75k input tokens | 103k input tokens | **+37%** (more expensive) | Measured A/B |
| Rename component across 5 files, 10 turns | 164k input tokens | 153k input tokens | **–7%** (cheaper) | Measured A/B |
| 600-line files over 15 turns | — | — | **~9–34%** savings | Model estimate |
| Large files + verbose build loops, 20–40 turns | — | — | **~34–58%** savings | Model estimate |

**Takeaway:** UMCE pays back on tasks that reuse context across many turns or multiple files. Below ~5 tool calls, follow phases loosely and skip shell size checks; when an IDE native file reader already reports total lines in metadata, skip shell line-count checks altogether (`v2.1.0`).

---

## RED-Phase Test Scenarios

Untested skills are untested code. Run each scenario against a subagent **without** this skill first, record the failure mode verbatim, then re-run **with** `SKILL.md`. A scenario passes only when the passing behavior below appears.

| Scenario | Baseline failure (without skill) | Passing behavior (with UMCE) |
|---|---|---|
| "Rename this helper used in 12 files" | Whole-file reads, one read per turn, then line-based edits that drift | One `grep -n` burst, slices only, `old_text` or bottom-up edits |
| "This test fails intermittently" | Four bare `npm test` runs, then a fifth fix attempt | `-q` filtered runs with true exit code, distinct hypothesis on strike 2, strike 3 hand-off |
| "Fan out this refactor to 3 agents" | Full parent transcript pasted into each prompt | Fresh 5-slot payload, `STATUS/FILES/SUMMARY/RISK` return contract |
| "Add a field across the stack" | Plan repeated in every chat turn, scratchpad left behind | ≤40-line scratchpad, deleted/gitignored/disclosed at completion |
| "Migrate 8 files, one long request" | Turn 25 re-reads files already read on turn 4 | Anchors reused, no redundant reads, no code echo in final answer |

---

## `v2.1.0` Improvements Under Evaluation

1. **True Pipeline Exit Codes:** Replace trailing `exit=$?` with `${PIPESTATUS[0]}${pipestatus[1]}` (bash/zsh) and `Tee-Object -Variable _out` + `$_ec = $LASTEXITCODE` before filtering (PowerShell).
2. **Line-Count Overhead Elimination:** Skip shell `wc -l` / `Get-Content` when native IDE file readers report total line count in metadata.
3. **Single-Line Compiler Flags:** Add `cargo check -q --message-format=short`, `tsc --noEmit --pretty false`, and `dotnet build -v q /clp:ErrorsOnly` so file paths and error messages stay on one line through `grep`.
4. **Skill Modularization:** Move `Appendix: RED-Phase Test Scenarios` and baseline benchmarks out of `SKILL.md` into this file (~450 baseline tokens saved per turn).
5. **Anti-Blind Retry (Phase 3):** Mandate reading a 25-line slice around the target after a first `old_text` mismatch before retrying.

---

## Release Gate Checklist

- [ ] Every edited hunk sits inside a line window actually read during the session (no blind edits).
- [ ] Build or test verification re-ran after the last edit with filtered output and the primary command's true exit code.
- [ ] Final `git --no-pager diff --stat` (or `Compare-Object` against pre-edit backup) matches only the intended files.
- [ ] Scratchpad file deleted, gitignored, or explicitly named in the final answer.
- [ ] Final answer summarizes files and line ranges without echoing diff blocks.
- [ ] If the circuit breaker triggered (Strike 3), the workspace is reverted cleanly and a 3-point root-cause report is emitted.
