---
name: universal-micro-context-execution
description: Use when one request will span many tool calls, more than one file, a sub-agent dispatch, an SDD task, or a long build-test-fix loop; when context or cost grows faster than progress; when whole-file reads, verbose build logs, or pasted diffs flood the window
version: 2.0.0
triggers:
  - complex_prompt
  - sdd_workflow
  - sub_agent_dispatch
  - multi_file_refactoring
  - long_debug_loop
---

# Universal Micro-Context Execution (UMCE / MCMA)

## Overview

Every tool result you ingest is billed again on every later turn of the same request. Naive "read whole file, run full build, repeat" produces O(N²) token growth inside a single request: turn 20 carries the whole history of turns 1-19. UMCE is the counter-protocol — ingest narrowly, batch aggressively, edit surgically, isolate state, and stop spirals early.

**Core principle:** the cheapest token is the one that never enters the transcript, and the cheapest turn is the one you never take.

This protocol trades two things for that: some up-front orientation work, and occasional extra tool calls to stay anchored. It is worth it only on multi-step work.

**Three non-negotiables.** Everything else here is a default you can exceed with a reason; these three are not.

1. **No blind edits.** Every hunk you change sits inside a window you actually read this request.
2. **No unverified "done."** Run the check before claiming it — the `verification-before-completion` skill owns the detail.
3. **No same-error fourth attempt.** Strike 3 stops and reports.

**Companions:** REQUIRED SUB-SKILL `verification-before-completion` before claiming done. `systematic-debugging` when a failure needs root-cause work; `context-engineering` for how context gets assembled before the request starts. UMCE governs only *how much* you read and how fast you loop.

**Calibration:** this protocol costs 2-3 extra calls up front. Measured on an 86-line file, one filter added across one file: without it 6 turns / 75k input tokens, with it 8 turns / 103k. It pays back on requests that reuse a file across many turns, not on one-file one-edit work. Below ~5 tool calls, follow the phases loosely and skip the size check.

## When to Use

- A request that will need 5+ tool calls, or touches 2+ files
- Spec-driven development, migrations, and refactors
- Dispatching sub-agents or delegate runs
- Build/test loops where the same failure can recur
- Symptom: you are at turn 15, the same file was read three times, and progress is unclear

**When NOT to use** — apply it loosely, not ceremonially:

- Single-file, single-edit task where the whole file is under 150 lines: read it, edit, done
- Pure Q&A with no repo work
- Greenfield exploration where you genuinely don't know where the code lives: do a wide **search burst** first (Phase 1 step 1 still applies), then narrow

## Quick Reference: Capacity Caps

| Surface | Default | Escalate to | Hard ceiling |
|---|---|---|---|
| File read per slice | 40-120 lines, anchored | full function body when it is the edit target | 250 lines |
| Slices per file per pass | 2 | 3 when editing two distant regions | 4 |
| Terminal output kept | 45 lines through the filter recipe | 80 lines for type errors | 120 lines |
| Scratchpad file | 40 lines | — | 60 lines |
| Parallel tools per turn | all independent calls | — | batching is the default |
| Same-error fix attempts | 2 | — | 3, then circuit breaker |

A cap is a default, not a religion: exceed it with a one-clause reason ("full 210-line module read: edit spans the file"). Exceeding silently is the failure, not exceeding.

## Phase 1 — Ingest Narrowly

Produce three things in order, one batched turn each where possible.

**1. Orient with a burst.** Fire all location searches at once: symbol names, error strings, config keys, route paths. Prefer the editor's native search/symbol/outline tool; fall back to a text search when none exists. Until you have an anchor, `grep` is cheaper than reading, and fixing the anchor is what makes a bounded read possible.

Where this doc writes `grep -n`, read `Select-String -Pattern` on PowerShell, `rg -n` when ripgrep is installed, and the editor's symbol tool when one is available.

**2. Size before you read.** One cheap call decides the read mode:

| Shell | Line count |
|---|---|
| PowerShell | `(Get-Content <file>).Count` |
| bash / zsh | `wc -l <file>` |

`Get-Item ... Length` gives bytes, which misjudges minified or dense files; count lines.

| File size | Read mode |
|---|---|
| ≤150 lines | whole file, one call |
| 151-800 lines | two 60-120 line slices around the anchors |
| >800 lines | anchors only; each slice ≤80 lines; read the signature region separately |

**3. Slice around anchors.** One read call per anchor, all anchors in the same turn. Include the anchor line plus enough neighbors to see the enclosing braces, decorators, or docblock — 20 lines above, 40 below is a sane default shape.

**Exclude these directories in every list or search call:** `node_modules`, `dist`, `build`, `out`, `.next`, `.nuxt`, `target`, `bin`, `obj`, `vendor`, `.git`, `__pycache__`, `.venv`, `coverage`, `.turbo`, `.cache`.

## Phase 2 — Batch Turns

Every sequential turn resends the accumulated window, so collapse turns wherever the calls are independent.

**Before each turn, list every call you know you need, then send them in one response:**

- 3 symbol searches + 2 read slices + 1 config read → one turn, six calls
- The target function slice AND the type/exception/enum definitions it references → one turn
- Independent edits in different files → one turn
- A failing test file AND its implementation → one turn

**Split only when a call needs another call's output** (e.g. resolve an ID before using it). "I will learn the path from the first result" is the one legitimate reason to go sequential — say so in one clause.

**Dependency pre-fetch** means: resolve *now* what the edit will surely need — signatures, callers of the function, the test that covers it.

## Cross-Phase: Terminal Output Discipline

Build, test, and log output is the worst offender: thousands of lines for six useful ones, and each turn re-bills them.

**Recipe per run:** quietest flag the tool offers → `2>&1` → `-C 3` context → keyword filter → `head -n 45`.

| Shell | Keep this much |
|---|---|
| PowerShell 7+ | `<cmd> 2>&1 \| Select-String -Pattern 'error\|FAILED\|panic\|warning' -Context 3,3 \| Select-Object -First 45` |
| bash / zsh | `<cmd> 2>&1 \| grep -E -C 3 "error\|FAILED\|panic\|warning" \| head -n 45` |

Quiet flags by tool: `pytest -q --tb=line`, `cargo test -q`, `npm test --silent`, `dotnet build -v q`, `git --no-pager`, `git diff --stat` before `git diff`, `--reporter=dot` or `--reporter=line` for JS runners.

**Empty output is ambiguous** — a clean run and a run that never started both print nothing through the filter. Append the exit code to the same command (`; "exit=$LASTEXITCODE"` on PowerShell, `; echo "exit=$?"` on bash) so the filtered result is self-describing.

**Success path:** when the run is expected to pass, keep only the summary line (`| tail -n 5` on a bare `-q` run). Full verbose output is reserved for a failure you are actively reading, and even then through the filter.

**Never widen the log to "see everything."** Re-run with a narrower filter — one test file, one package, one grep — instead of relaxing the cap.

## Phase 3 — Edit Surgically

**Default edit primitive: matched-text replace.** Choose an `old_text` window of 5-15 lines that is unique in the file — a signature line plus two body lines is normally enough — and replace it. Matched-text edits survive unrelated churn and need no re-anchoring.

**Anchor `old_text` on structure, not formatting.** Trailing whitespace, tabs vs spaces, and reflowed comments break matches; pick lines whose content is semantically stable (signatures, error strings, imports).

**Line-based patch fallback.** When the tool patches by line range:

1. Order edits **bottom-up** (highest line number first) so earlier insertions do not shift later targets.
2. Re-anchor after every structural edit: one search for the next target's unique string before touching it.
3. Keep a deletion and an insertion on the same range in separate calls.

**Whole-file rewrite is correct when** creating a new file, the file is under 150 lines, or 60%+ of its lines change. Otherwise patching wins.

## Phase 4 — Isolate State

Keep the transcript for decisions and results. Move bulk state to disk and compress your own voice.

**Telegraphic turns.** While tools are still running, one or two sentences per turn. Shape:

```
Anchors: handleRetry, RetryPolicy. Reading both plus the retry test.
```

**The final answer carries the detail** — summary, files touched, verification evidence, open questions. No code echo: name the file and line range, quote only a line whose meaning changed.

**Scratchpad for long work** (5+ files, or a multi-phase plan). One file, ≤40 lines, invariants and checklist only:

```markdown
# .umce-state
GOAL: retry policy on the payment client
DONE: policy type, 2 call sites
TODO: worker loop, admin toggle, tests
INVARIANT: no new deps; keep RetryPolicy constructor signature
ANCHOR: src/payments/worker.ts:88
```

**Close the scratchpad loop before you finish** — do one of these:

- delete the file, or
- keep it and name it in the final answer.

For repeat work keep a gitignored `scratch/` directory in the repo. When the tree must stay pristine, put the scratchpad in `%TEMP%` (Windows) or `/tmp` instead. Never leave an untracked scratchpad in a clean-tree repo without saying so.

## Phase 5 — Dispatch Micro-Payloads

A sub-agent has its own fresh window; passing it *your* history multiplies cost by the number of children and buys it nothing. Build the payload from scratch, five slots, in order:

1. **Task contract** — one-sentence objective, acceptance criteria, exhaustive list of files it may touch.
2. **Invariants** — 5-10 bullets, all non-negotiable, each checkable ("keep exported `RetryPolicy` shape", "no dependency additions").
3. **Signature skeleton** — only the types/signatures the agent must obey, bodies stripped to `{ ... }`.
4. **Target slice** — absolute file path(s), including the current function body.
5. **Return contract** — exact output shape:

```
STATUS: DONE | BLOCKED
FILES: <path:line range> (comma separated)
SUMMARY: <max 5 lines>
RISK: <one line, or NONE>
```

Same rule for SDD task units: a spec task is a contract, not a link to the whole spec. Pass the one requirement it implements, the acceptance check, and the invariants.

Dispatch in parallel only when tasks are independent. Shared files or shared build output go sequential — see `dispatching-parallel-agents`.

## Phase 6 — Circuit Breaker

**Two strikes, then a ledger.** After each failed fix, restate the failure in one line: what you changed, what the error still says. The second strike charges a *different* hypothesis, not a variation of the first (strike 1 argues the value away in the caller, strike 2 changes the type itself).

**On strike 3, stop iterating and hand off:**

1. Show what is now dirty: `git --no-pager diff --stat` in a repo, `Compare-Object` against a pre-edit copy otherwise.
2. Revert broken exploratory edits so the tree matches its pre-request state.
3. Emit a root-cause report: attempted, why it failed, options ahead. Three bullets, then stop and let the user choose.

**Spiral signature:** the same error line recurs across three turns, each attempt grows wider (touching more files), and no new information enters. Widen before strike 3 only when the error message newly implicates another layer; otherwise the retry narrows.

## Common Mistakes

| Mistake | What it costs | Fix |
|---|---|---|
| Reading the whole file "to be safe" | 600 lines billed on every later turn | `grep -n` the symbol, read 60 lines around it |
| Grepping inside `node_modules` | Thousands of duplicate hits | Pass the exclusion list, or scope to `src/` |
| One tool call per turn on a fan-out search | Each turn re-sends the whole window | Send all independent calls in one response |
| Passing the parent transcript to a sub-agent | Cost multiplied by child count, focus diluted | Build the five-slot payload fresh |
| Running bare `npm test` / `cargo build` | 2,000 lines for one error | `-q`, `2>&1`, filter, `head -n 45` |
| Editing top-down by line number | Later line numbers shift mid-edit | `old_text` replace, or bottom-up patches |
| Rewriting a file to change four lines | Diff noise plus a fresh review | Matched-text replace |
| Retrying the same fix a fourth and fifth time | 15 turns for one root cause | Strike 3: `diff --stat`, revert, report |
| Leaving the scratchpad untracked at the end | Dirty tree, surprise on commit | Delete it, or gitignore `scratch/` |
| Running the bash filter recipe inside PowerShell | Broken pipeline, re-run cycles | Use the shell row that matches the interpreter |

## Rationalizations

| Excuse | Reality |
|---|---|
| "The whole file is context" | Context you already have is billed at every later turn. Six lines are usually enough to edit. |
| "Batching might confuse the results" | Results come back labeled per call. Confusion is the *sequential* path — you re-read stale numbers. |
| "Better to keep verbose logs in case I need them" | You can always re-run with a wider filter; you cannot un-bill lines already in the transcript. |
| "Reading everything is cheaper than missing something" | Missing something costs one extra turn; reading everything costs every turn. |
| "This fix obviously works, no need to re-verify" | Strike counting exists because the third same-error retry is 15 turns deep before anyone notices. |
| "A sub-agent should know the discussion" | Discussion is decisions. Decisions fit in the invariants slot; the rest is drag. |
| "Surgical edits are slower" | One 5-line `old_text` beats a 400-line rewrite plus a re-read to verify it. |
| "Telegraphic turns look lazy" | Detail belongs in the answer. Mid-loop prose is re-sent every turn for no reader. |

## Red Flags

Stop and re-read this skill when you catch any of these:

- Third read of the same file in one request
- More than one search call for the same symbol
- A tool call per turn stretching past three consecutive turns
- A second and third fix attempt aimed at the same error line
- `old_text` replace failing twice on the same region
- Terminal result longer than ~60 lines in the transcript
- Scratchpad still lying around after the work is finished
- A payload to a sub-agent that opens with "here is the conversation so far"

## Verification

Before the final answer:

- [ ] Every edited hunk was inside a window I had actually read — no blind edits
- [ ] Tests or a build were re-run after the last edit, filtered output kept
- [ ] The final diff matches the intended file list, nothing stray — `git --no-pager diff --stat` in a repo, a compare against a pre-edit backup inside a non-repo workspace
- [ ] Scratchpad deleted, gitignored, or named in the answer
- [ ] The answer lists files and line ranges instead of pasting diffs
- [ ] If the circuit breaker fired, the root-cause report has three concrete bullets

## Appendix: RED-Phase Test Scenarios

Untested skills are untested code. Run each scenario against a subagent **without** this skill first, record the failure mode verbatim, then re-run with it. A scenario only passes when the behavior shape below appears.

| Scenario | Baseline failure to record | Passing behavior |
|---|---|---|
| "Rename this helper used in 12 files" | Whole-file reads, one Read per turn, then line-based edits that drift | One `grep -n` burst, slices only, bottom-up or matched-text edits |
| "This test fails intermittently" | Four bare `npm test` runs, then a fifth fix attempt | `-q` filtered runs, distinct hypotheses, strike 3 hand-off |
| "Fan out this refactor to 3 agents" | Full transcript pasted into each prompt | Five-slot payload, `STATUS/FILES/SUMMARY/RISK` return contract |
| "Add a field across the stack" | Scratchpad or plan repeated in every chat turn, left behind at the end | ≤40-line scratchpad, deleted or disclosed at the end |
| "Migrate 8 files, one long request" | Turn 25 re-reads files from turn 4 | Anchors reused, no redundant reads, no code echo in the final answer |

### Baseline measured on a small task

Both arms ran the same task (add one filter, one 86-line file, verify with a build) from an identical tree. Both produced the same correct diff and a green build.

| Arm | Turns | Iterations | Input tokens | Read shape |
|---|---|---|---|---|
| Control | 5 | 6 | 74,952 | one 30-line slice, no size check |
| Skill | 5 | 8 | 103,180 | 3 parallel searches, 2 slices, whole-file read, `RISK` line |

Holds: the skill arm batched searches, disclosed files by line range, and appended the exit code. Costs: +37% input tokens and 2 extra iterations for a 4-hunk change in one file — small-task evidence that the calibration paragraph above is needed. The size-check step was the biggest overhead; skipping it below ~5 tool calls is the fix.

Also found by the dry run: this workspace is not a git repo, so `git diff --stat` needs a non-repo fallback (added to Phase 6 and Verification), and empty filtered output needs an appended exit code to be readable (added to Terminal Output Discipline).