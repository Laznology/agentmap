# agentmap — token-savings benchmark results

**Headline: 70.3% fewer tokens** to perform three common "understand the
codebase" tasks when an agent queries agentmap instead of reading raw files with
`cat` / `grep` / `find`. Measured on the public
[vercel/ai-chatbot](https://github.com/vercel/ai-chatbot) repo, reproducible
with one command. Every number below is captured tool output from this run — no
hand-tuned figures.

## Captured run (`bench.mjs`)

```
agentmap token-savings benchmark
repo:  github.com/vercel/ai-chatbot  (cloned to /tmp/am-sample)
sha:   2becdb4
env:   node v26.3.0, 154 mapped files
est:   tokens = chars / 4

Scenario                                  Baseline tok  agentmap tok  Saved %
------------------------------------------------------------------------------
A. Understand file deps (lib/utils.ts)            583           517    11.3%
B. Find symbol (ChatMessage)                     1950            20      99%
C. Repo overview (tree + cat 3 hub files)        3065          1127    63.2%
------------------------------------------------------------------------------
TOTAL                                            5598          1664    70.3%

HEADLINE: 70.3% fewer tokens (5598 -> 1664) across 3 scenarios.
```

## What each scenario compares

The benchmark contrasts the bytes an agent must pull into context for a task,
via the naive shell approach vs the equivalent single agentmap query.

| # | Task | Baseline (naive agent) | agentmap query |
|---|------|------------------------|----------------|
| A | Understand a file's dependencies | `cat lib/utils.ts` + `grep -rln <basename> .` (read the file + find who imports it) | `repomap.mjs --any lib/utils.ts` (exports + imports + dependents, no source body) |
| B | Find where a symbol lives | `grep -rn ChatMessage .` (every line that mentions it) | `repomap.mjs --find ChatMessage` (definition site(s) only) |
| C | Get a repo overview to start | `find . -name '*.ts*'` (file tree) + `cat` top-3 hub files | `repomap.mjs --map` (token-budgeted ranked symbol digest) |

Targets are **auto-derived from the repo's own map** (top hub file, top-ranked
exported symbol, top-3 hub files for the overview), so the same script runs on
any ts-morph-mappable repo. On vercel/ai-chatbot the auto-picked targets were:
hub file `lib/utils.ts`, symbol `ChatMessage`.

## Per-scenario baseline vs tool commands

```
[A] baseline: cat lib/utils.ts + grep -rln <basename> .
     agentmap: repomap.mjs --any lib/utils.ts

[B] baseline: grep -rn ChatMessage .
     agentmap: repomap.mjs --find ChatMessage

[C] baseline: find . -name '*.ts*' + cat 3 hub files
     agentmap: repomap.mjs --map
```

## Reproduce it

```bash
# Clone the public target repo
git clone https://github.com/vercel/ai-chatbot /tmp/am-sample

# Run the benchmark (Node >= 18, tested on v26.3.0)
node /path/to/agentmap/benchmark/bench.mjs /tmp/am-sample
```

Run it against any other repo by passing a different path (defaults to cwd):

```bash
node benchmark/bench.mjs /path/to/repo
```

The script is **zero-dependency** (only `node:child_process` / `node:path`). A
machine-readable `@@JSON@@{...}` footer is appended for CI/scripting.

## Environment (as captured)

- **node** v26.3.0 ; **ts-morph** 28.0.0 (agentmap's single dependency)
- **repo** `github.com/vercel/ai-chatbot` @ sha `2becdb4`
- **mapped files** 154 (TS/TSX/JS/MJS/CJS that agentmap's ts-morph pass sees)

## Honest caveats — read before quoting the number

1. **Token estimate is `chars / 4`.** A rough heuristic (the same one agentmap
   uses internally), not a real BPE tokenizer count. It applies to **both** sides
   of every comparison, so the *ratio* / saved-% is far more robust than the
   absolute token figures. The raw char counts (in the `@@JSON@@` footer) are the
   ground truth; treat token columns as ±10%.

2. **Single repo, single commit.** Results are from one public Next.js app
   (154 files) at one point in time. Savings scale with repo size for scenarios B
   and C (more files to grep / list) and with fan-in for scenario A (more
   importers to find). A tiny repo shows smaller savings; a large monorepo,
   larger. Re-run on your own repo before generalizing.

3. **The grep baseline is deliberately *fair*, not worst-case.** It prunes
   `node_modules` / `.next` / `.git` (`--exclude-dir`) to mirror a competent
   agent. A naive `grep -rn <symbol> .` *without* those excludes hits minified
   bundles and produces far more baseline tokens — a dishonest near-100%
   "saving." The 70.3% is the prune-the-junk number.

4. **Scenario equivalence is approximate.** agentmap returns *structured*
   answers (definition sites, dependents, ranked digest); the shell baselines
   return *raw* lines/source the agent must still parse. The comparison is "bytes
   into context for the same question," which favors agentmap partly because it
   has already done the parsing — that is the value prop, not a like-for-like
   algorithmic comparison.

5. **What's NOT measured here.** Answer *quality* / completeness, wall-clock
   agent time, and end-to-end "did the agent finish the task faster." This
   benchmark measures context-token volume only.

## Machine-readable footer

```
@@JSON@@{"repo":"/tmp/am-sample","node":"v26.3.0","fileCount":154,"sha":"2becdb4","totalBaseTok":5598,"totalToolTok":1664,"savedPct":70.3,"rows":[{"name":"A. Understand file deps (lib/utils.ts)","baselineCmd":"cat lib/utils.ts + grep -rln <basename> .","toolCmd":"repomap.mjs --any lib/utils.ts","base":583,"tool":517,"baseChars":2332,"toolChars":2068},{"name":"B. Find symbol (ChatMessage)","baselineCmd":"grep -rn ChatMessage .","toolCmd":"repomap.mjs --find ChatMessage","base":1950,"tool":20,"baseChars":7800,"toolChars":80},{"name":"C. Repo overview (tree + cat 3 hub files)","baselineCmd":"find . -name '*.ts*' + cat 3 hub files","toolCmd":"repomap.mjs --map","base":3065,"tool":1127,"baseChars":12260,"toolChars":4507}]}
```

## Files

- benchmark script: `benchmark/bench.mjs`
- this file: `benchmark/RESULTS.md`
- tool under test: `repomap.mjs`
