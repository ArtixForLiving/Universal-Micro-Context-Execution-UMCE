---
name: universal-micro-context-execution
description: Use when one request will span many tool calls, more than one file, a sub-agent dispatch, an SDD task, or a long build-test-fix loop; when context or cost grows faster than progress; when whole-file reads, verbose build logs, or pasted diffs flood the window
version: 2.1.0
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

**Calibration:** this protocol costs 2-3 extra calls up front. Measured A/B on an identical tree:

| Task | Control | Skill | Delta |
|---|---|---|---|
| One filter, one 86-line file, 7 turns | 75k input tokens | 103k input tokens | **+37%** (more expensive) |
| Rename component across 5 files, 10 turns | 164k input tokens | 153k input tokens | **–7%** (cheaper) |

It pays back on requests that reuse context across many turns or many files, not on one-file one-edit work. Model estimate for larger loops: 600-line files over 15 turns saves roughly **9–34%**; large files with verbose build loops over 20–40 turns saves roughly **34–58%**. Below ~5 tool calls, follow the phases loosely and skip any shell size check. See [BENCHMARK.md](BENCHMARK.md) for baseline data and RED-phase scenarios.

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

**2. Size before you read.** Choose the read mode from total lines without unnecessary shell overhead:

- **Native IDE tool mode:** If the editor's built-in file reader or outline tool already reports total lines in its response metadata, skip any shell line-count call and read the first 100 lines or the outline directly.
- **Shell fallback:** Otherwise, batch a line-count call in the same turn as the search burst:

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
| PowerShell 7+ | `<cmd> 2>&1 \| Tee-Object -Variable _out \| Out-Null; $_ec = $LASTEXITCODE; $_out \| Select-String -Pattern 'error\|FAILED\|panic\|warning' -Context 3,3 \| Select-Object -First 45; "exit=$_ec"` |
| bash / zsh | `<cmd> 2>&1 \| grep -E -C 3 "error\|FAILED\|panic\|warning" \| head -n 45; echo "cmd_exit=${PIPESTATUS[0]}${pipestatus[1]}"` |

Quiet flags by tool: `pytest -q --tb=line`, `cargo test -q`, `cargo check -q --message-format=short`, `tsc --noEmit --pretty false`, `npm test --silent`, `dotnet build -v q /clp:ErrorsOnly`, `git --no-pager`, `git diff --stat` before `git diff`, `--reporter=dot` or `--reporter=line` for JS runners.

**Empty output is ambiguous** — a clean run and a run that never started both print nothing through the filter. Capture the primary command's **true exit code** before or across the filter pipeline (`$_ec = $LASTEXITCODE` before filtering on PowerShell; `${PIPESTATUS[0]}${pipestatus[1]}` on bash/zsh, since trailing `exit=$?` only reads `head`/`grep`) so the filtered result is self-describing.

**Success path:** when the run is expected to pass, keep only the summary line (`| tail -n 5` on a bare `-q` run). Full verbose output is reserved for a failure you are actively reading, and even then through the filter.

**Never widen the log to "see everything."** Re-run with a narrower filter — one test file, one package, one grep — instead of relaxing the cap.

## Phase 3 — Edit Surgically

**Default edit primitive: matched-text replace.** Choose an `old_text` window of 5-15 lines that is unique in the file — a signature line plus two body lines is normally enough — and replace it. Matched-text edits survive unrelated churn and need no re-anchoring.

**Anchor `old_text` on structure, not formatting.** Trailing whitespace, tabs vs spaces, and reflowed comments break matches; pick lines whose content is semantically stable (signatures, error strings, imports).

**First-match-failure protocol (anti-blind retry):** If an `old_text` match fails once, never guess indentation or retry blindly. Immediately read a 25-line slice around the target line to verify exact indentation, or widen `old_text` to the nearest structural boundary before retrying.

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
| Running bare `npm test` / `cargo build` | 2,000 lines for one error | `-q`, `2>&1`, filter, `head -n 45`, true exit code |
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
- [ ] Tests or a build were re-run after the last edit, with filtered output and the primary command's true exit code
- [ ] The final diff matches the intended file list, nothing stray — `git --no-pager diff --stat` in a repo, `Compare-Object` against a pre-edit backup inside a non-repo workspace
- [ ] Scratchpad deleted, gitignored, or named in the answer
- [ ] The answer lists files and line ranges instead of pasting diffs
- [ ] If the circuit breaker fired, the root-cause report has three concrete bullets

See [BENCHMARK.md](BENCHMARK.md) for empirical A/B measurements and RED-phase test scenarios.
