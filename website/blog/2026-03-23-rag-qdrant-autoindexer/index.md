---
title: "I Gave an AI Agent Kubernetes Bugs and a RAG Index. Here's What Actually Happened."
date: 2026-03-23
description: "A field report on semantic retrieval, local code, and hybrid approaches to AI-assisted code generation in a 4M-line codebase."
authors:
- brandon-foley
tags: ["kaito", "ai", "rag", "ragengine", "autoindexer", "code-search"]
---


I've been using AI agents daily for some time now, and not as a novelty, but as part of my actual engineering workflow. One question that kept nagging me was, If I hand an agent a bug report and just say "fix this," how close does it actually get? Not explaining the fix to me. Not summarizing the issue. Actually producing a diff I could submit as a pull request.

So I ran the experiment.


## The Setup

I took open pull requests from the [`kubernetes` github repo](https://github.com/kubernetes/kubernetes). Real bugs, actively being fixed by real contributors. I extracted just the issue description (not the PR description, not the diff, nothing that would leak the answer) and gave each issue to three different agent configurations:

- **RAG Only**: Semantic search over an indexed copy of the entire Kubernetes codebase via [KAITO RAG Engine](https://github.com/kaito-project/kaito). No local files, no web access. The agent queries the index, gets ranked code snippets back, and works from those alone.
- **Hybrid (RAG + Local)**: Same RAG index, but also has a full local clone of kubernetes/kubernetes. The agent must start with RAG for discovery, then can read local files for precision — exact line numbers, full function bodies, test infrastructure.
- **Local Only**: Full clone, grep, find, cat. No RAG, no web. The agent explores the codebase the old-fashioned way.

Each agent ran in a completely isolated session. Same model (Claude Opus), same timeout (5 minutes), same output format. The only variable was how they could see code.

One critical constraint: the RAG and Hybrid agents were required to make RAG queries before producing any fix. Early experiments showed that without this mandate, hybrid agents would skip RAG entirely when the issue mentioned specific filenames — defeating the whole point of the comparison.



## The Hypothesis (What I Expected vs. What Actually Matters)

Going in, I assumed the main difference would be how well each approach finds the right code.

What actually mattered more was something else entirely. Agents are good at fixing what they can see locally, but bad at discovering what else needs to change.

Retrieval changes how they search.  RAG surfaces snippets, local tools traverse the repo, but it doesn’t change how they reason about system boundaries, call chains, or architectural intent.

That distinction shows up everywhere in the results: multi-file bugs, test updates, error handling patterns, and integration points.


## The Issues

| # | PR | Bug | SIG | Size | Files |
|---|------|-----|-----|------|-------|
| 1 | #134540 | SubPath volume mount race condition (EEXIST) | sig/storage | XS | 3 |
| 2 | #138211 | Image pull record poisoning (preloaded images) | sig/node | M | 5 |
| 3 | #138244 | NUMA topology stall on GB200 systems (O(2^n)) | sig/node | L | 2 |
| 4 | #136013 | OOM score panic on zero memory capacity | sig/node | S | 2 |
| 5 | #138269 | Scheduler SchedulingGates missing Pod/Update event | sig/scheduling | S | 2 |
| 6 | #138158 | ServiceCIDR deletion race in allocator | sig/network | M | 3 |
| 7 | #135970 | Missing initializer in StatefulSet slice builder | sig/apps | M | 4 |
| 8 | #138000 | Windows kube-proxy stale remote endpoint cleanup | sig/network | XL | 5 |
| 9 | #138191 | Container status sort by attempt (clock rollback) | sig/node | M | 5 |

They span kubelet, scheduler, networking, storage, and apps. From a one-line guard clause to a 900-line cross-file refactor.


## The Scoreboard

Each result was graded across five dimensions (0–4 each):

- Files: Did the diff touch the correct files?
- Location: Was the fix applied in the correct function/layer?
- Mechanism: Was the approach architecturally correct or just functionally similar?
- Tests: Were existing tests updated and/or meaningful new tests added?
- Completeness: Were all affected files, call sites, and edge cases covered?

Scores reflect alignment with the actual PR diff, not just whether the fix “works.”

| PR | Issue | RAG | Hybrid | Local |
|----|-------|-----|--------|-------|
| #134540 | SubPath EEXIST race | 7/20 (33%) | 8/20 (40%) | 1/20 (7%) |
| #138211 | Image pull poisoning | 8/20 (40%) | 14/20 (70%) | 9/20 (45%) |
| #138244 | NUMA topology stall | 9/20 (45%) | 10/20 (50%) | 10/20 (50%) |
| #136013 | OOM score zero-memory | 15/20 (75%) | 17/20 (85%) | 20/20 (100%) |
| #138269 | SchedulingGates Pod/Update | 18/20 (90%) | 18/20 (90%) | 19/20 (95%) |
| #138158 | ServiceCIDR deletion race | 16/20 (80%) | 17/20 (85%) | 18/20 (90%) |
| #135970 | StatefulSet slice builder | 16/20 (80%) | 18/20 (90%) | 17/20 (85%) |
| #138000 | Windows stale endpoints | 11/20 (55%) | 11/20 (55%) | 8/20 (40%) |
| #138191 | Container status sort | 9/20 (45%) | 15/20 (75%) | 15/20 (75%) |
| **Average** | | **12.1/20 (61%)** | **14.2/20 (71%)** | **13.0/20 (65%)** |


## Timing

Every session had a 5-minute ceiling. Here's how long each actually took:

| PR | RAG | Hybrid | Local |
|----|-----|--------|-------|
| #134540 | 55s | 3m00s | 5m00s |
| #138211 | 2m30s | 3m30s | 2m30s |
| #138244 | 41s | 44s | 2m00s |
| #136013 | 31s | 55s | 41s |
| #138269 | 44s | 1m12s | 1m23s |
| #138158 | 1m41s | 1m48s | 52s |
| #135970 | 1m23s | 2m43s | 1m32s |
| #138000 | 1m28s | 4m03s | 4m57s |
| #138191 | 1m34s | 3m54s | 2m52s |
| **Average** | **1m16s** | **2m25s** | **2m24s** |

RAG is consistently the fastest. It doesn't read files, navigate directories, or wait for grep results. It asks the index, gets snippets, and generates. Average wall-clock: **1 minute 16 seconds**.

Hybrid is the slowest on average at **2 minutes 25 seconds**. The mandatory RAG-first phase adds overhead before local exploration even begins. On complex PRs like #138000 and #138191, Hybrid pushed past 3-4 minutes.

Local matches Hybrid's average but for different reasons. Hybrid spends time on RAG queries followed by targeted file reads. Local burns time on broad exploration: grepping, finding, iterating through directories.

Two sessions hit the 5-minute wall: Local on #134540 (spent too long cloning) and Local on #138000 (too much code to explore in the XL PR).


## Token Economics

Token usage isn’t just about how much context the agent reads—it’s dominated by how many times it calls the model.

Claude’s API is stateless, so every call replays the full conversation. That makes call count the primary cost multiplier.

Three patterns stood out:

- Hybrid is the most expensive, not because it reads more, but because it makes the most calls. The RAG to local loop creates repeated round-trips, and each one replays a growing context window.
- RAG and Local cost about the same, for opposite reasons. RAG pulls more new context (via retrieval), while Local makes more exploratory calls. The totals converge.
- Fewer calls beats fewer tokens.  Sessions that stayed under ~5 calls were consistently cheaper, regardless of approach.

Here’s the full breakdown:

| PR | Approach | Calls | New Tokens | Output | Cached (replay) | Total (all) |
|----|----------|-------|-----------|--------|----------------|-------------|
| #134540 | RAG | 3 | 46K | 1K | 89K | 136K |
| #134540 | Hybrid | 12 | 27K | 3K | 262K | 292K |
| #134540 | Local | 12 | 25K | 3K | 249K | 277K |
| #135970 | RAG | 5 | 45K | 4K | 183K | 232K |
| #135970 | Hybrid | 12 | 33K | 6K | 446K | 485K |
| #135970 | Local | 9 | 27K | 4K | 284K | 315K |
| #136013 | RAG | 2 | 30K | 2K | 21K | 53K |
| #136013 | Hybrid | 5 | 28K | 2K | 98K | 128K |
| #136013 | Local | 3 | 24K | 1K | 44K | 69K |
| #138000 | RAG | 3 | 15K | 3K | 60K | 78K |
| #138000 | Hybrid | 20 | 44K | 8K | 690K | 742K |
| #138000 | Local | 14 | 36K | 10K | 380K | 426K |
| #138158 | RAG | 6 | 34K | 3K | 183K | 220K |
| #138158 | Hybrid | 7 | 32K | 3K | 223K | 258K |
| #138158 | Local | 4 | 26K | 2K | 115K | 143K |
| #138191 | RAG | 5 | 40K | 3K | 170K | 213K |
| #138191 | Hybrid | 4 | 13K | 2K | 86K | 100K |
| #138191 | Local | 3 | 13K | 2K | 57K | 72K |
| #138211 | RAG | 6 | 87K | 3K | 330K | 420K |
| #138211 | Hybrid | 4 | 18K | 2K | 93K | 113K |
| #138211 | Local | 3 | 17K | 3K | 62K | 82K |
| #138244 | RAG | 4 | 63K | 1K | 159K | 223K |
| #138244 | Hybrid | 2 | 7K | 1K | 25K | 33K |
| #138244 | Local | 3 | 8K | 2K | 52K | 62K |
| #138269 | RAG | 3 | 32K | 2K | 74K | 108K |
| #138269 | Hybrid | 5 | 37K | 4K | 179K | 220K |
| #138269 | Local | 7 | 29K | 5K | 218K | 252K |

**Averages:**

| Approach | Calls | New | Output | Cached | Total |
|----------|-------|-----|--------|--------|-------|
| **RAG** | 4 | 44K | 2K | 141K | 187K |
| **Hybrid** | 8 | 27K | 3K | 234K | 264K |
| **Local** | 6 | 23K | 4K | 162K | 189K |


The worst case was #138000 (Windows kube-proxy): Hybrid made 20 calls and accumulated 690K cached tokens, driving total usage to 742K. The issue wasn’t the size of the code it was the number of back-and-forth steps.

Across all runs, call count was the biggest driver of both cost and latency.

## What I Actually Learned

### Nobody gets the subtle stuff

The PR for #134540 was a two-layer fix. In `doSafeMakeDir`, change `%s` to `%w` in the error format string so the EEXIST error is wrapped rather than stringified. Then in the caller (`kubelet_pods.go`), add `!goerrors.Is(err, os.ErrExist)` to tolerate that specific wrapped error.

0 out of 27 sessions got this. Every approach chose to swallow the EEXIST error at the source. Functionally similar, architecturally wrong. The actual PR preserves the error and lets the caller decide what to tolerate.

This is a Go idiom thing. Error wrapping with `%w` for programmatic handling is a design choice that requires understanding the caller-callee contract. The agents see the immediate error and fix it locally. They don't think about who else might need to inspect that error.

Every agent fixed the symptom. None preserved the contract.

### The “partial fix blindness” problem

PR #134540 was also a trap for Local and Hybrid. Part of the fix was already merged at HEAD. `!os.IsExist(err)` was already in the subpath files. Both local approaches saw this and concluded "already fixed," stopping investigation. They missed that the PR adds a second defense layer at the caller level and changes the error wrapping verb.

This is a real hazard for code-aware agents. if the codebase already has a partial fix, the agent sees it and moves on. It doesn't ask "is there more to do?"

### Mandatory RAG changes fix architecture

On #138211, forcing 3 RAG queries before writing any code made a material difference. The mandatory exploration pushed the agent to discover the policy evaluation layer before jumping to a fix. That extra context led to better architectural choices.

On #138191, the impact was even starker. RAG-only scored 9/20 because it only fixed `containerByCreatedThenID` and completely missed the second sort type `containerStatusByCreated`. Without local file access to grep for all sort implementations, it found half the problem. Both Hybrid and Local found both sorts and scored 15/20.

### Well-specified issues flatten the playing field

PRs #136013 (OOM score guard) and #138269 (SchedulingGates event) were the most precisely described issues. They named the exact function, the exact file, and the exact expected behavior. Scores clustered tight:

#136013: RAG 75%, Hybrid 85%, Local 100%.
#138269: RAG 90%, Hybrid 90%, Local 95%.

Local won both. When the issue says exactly where to look, grep is all you need. RAG adds overhead without adding signal. When issue quality is high enough, the retrieval method barely matters. The issue itself becomes the spec.

### Large PRs expose a scope discovery gap

PR #138000 (Windows kube-proxy, 913 lines, 5 files) was the biggest non-refactor PR. All three approaches correctly identified and fixed the core bug, an inverted local/remote priority filter in `getAllEndpointsByNetwork`. But none discovered that `proxier.go` also needed changes: deferred cleanup integration, a `deleteTerminatedEndpoints` refactor, and a `NETWORK_TYPE_L2BRIDGE` constant.

Agents focus on the root cause file and miss integration changes. The 671 lines of `proxier_test.go` integration tests were unreachable by all approaches. This is a fundamental limit: agents fix the bug they can see, not the surrounding code that needs to evolve with it.

### Agents prefer new fields over reusing existing ones

On #138191 (container status sorting), the PR used the existing `RestartCount` field on the Status struct for the sort comparator. All three agents added a new `Attempt` field instead. Functionally correct, architecturally heavier. Nobody discovered that `RestartCount` already serves as the equivalent of `Attempt` for container status.

I saw this across multiple PRs: agents prefer adding new things to reusing existing abstractions. They don't have the institutional knowledge to know "we already have a field for that."

### Prompt discipline is not optional

Early in this experiment, the Hybrid agent prompt said "you may use the KAITO RAG API" and "you may also read local files." Both optional. The agent optimized for speed and skipped RAG entirely on 2 of 3 PRs, going straight to local files when the issue mentioned specific filenames.

Adding "you MUST start with RAG" changed everything. The mandatory exploration step forced broader context before the agent committed to a fix direction.

If you're building agent pipelines, don't make retrieval optional. If you have a RAG index, require the agent to use it. Agents will take shortcuts unless you explicitly prevent them.


## Where Things Break

After 27 sessions across 9 PRs, some clear patterns emerge.

### Hybrid wins overall, but not by a landslide

71% average vs 65% for Local and 61% for RAG. Hybrid's edge is consistency. It never scored below 40% and hit 90% twice. It combines RAG's discovery breadth with Local's implementation precision.

### Local wins on well-specified bugs

When the issue names the file and function, Local scored highest on 4 of 9 PRs. No retrieval overhead, no API round-trips, just read the code and fix it. The more precise the issue, the less RAG matters.

### RAG is fastest and cheapest, but falls apart on multi-file bugs

Average 1m16s, often under a minute. But RAG can only see what the index returns. On #138191, it found one sort type but missed the second. On #138000, it found the bug file but missed the integration file. If the fix spans files that don't share keywords, RAG loses.

### Issue quality matters more than retrieval method

The score spread between approaches on a well-specified bug like #138269 was 5 points. On a vague one like #134540, it was 33 points. Writing a better bug report is worth more than switching from RAG to Hybrid.

### Agents can't update existing tests

Every session could produce new test cases. None could reliably update existing test expectations. If your PR needs to flip a `want: true` to `want: false` somewhere, you're still doing that yourself. Hybrid on #138211 was the only session across all 27 that flipped existing test expectations.

### There's a complexity ceiling around 5 files

Below 5 files changed, at least one approach scored 85% or higher on every PR. Above 5 files, the best score on any approach was 75%. Multi-file integration changes (updating call sites, renaming types across consumers, threading new parameters) are beyond single shot code generation regardless of retrieval method.

## The Bottom Line

Across all approaches, the biggest limitation wasn’t generating code, it was figuring out the full scope of what needed to change.

Retrieval changes how agents find code. It doesn’t fix their tendency to reason locally and miss system-level implications.

| | RAG | Hybrid | Local |
|--|-----|--------|-------|
| **Avg Score** | 61% | 71% | 65% |
| **Avg Time** | 1m16s | 2m25s | 2m24s |
| **Avg New Tokens** | 44K | 27K | 23K |
| **Best For** | Speed, well-scoped bugs | Consistency, multi-file bugs | Precise issues with exact file and function |
| **Worst At** | Multi-file scope discovery | Cost on complex PRs | Vague issues, large codebases |

If you’re setting up AI-assisted development on a large codebase:

**Index your codebase.**
A semantic index is worth it. Even RAG-only reached 61% with no local file access. Retrieval gives agents a fast, high-signal starting point.

**Use hybrid, but enforce the workflow.**
Combining retrieval with local files is the most consistent approach, but only if you require the agent to use both. If retrieval is optional, it will be skipped.

**Write issues like specs.**
The highest-leverage improvement wasn’t tooling, it was issue quality. When the bug report named the file, function, and expected behavior, all approaches converged toward correct fixes.

**Optimize for fewer steps, not smaller context.**
Cost and latency scaled with the number of model calls, not just token volume. Reducing back-and-forth interactions is the biggest lever.

**Don’t rely on agents for test updates.**
They can generate new tests, but rarely modify existing expectations correctly.

**Expect local reasoning, not system reasoning.**
Agents reliably fix the code in front of them, but miss adjacent changes, integration points, and existing abstractions. Plan for human review on anything that spans multiple files.

We’re not bottlenecked on code generation anymore. We’re bottlenecked on problem framing and scope discovery.

Until agents can reliably answer “what else needs to change?”, they’ll remain strong assistants for local fixes, but weak contributors to system-level changes.

---

*The benchmark used Claude Opus via [OpenClaw](https://github.com/openclaw/openclaw) with [KAITO RAG Engine](https://github.com/kaito-project/kaito) indexing the kubernetes/kubernetes repository at HEAD. All sessions ran in isolated subagents with no shared state, April 2026. Ground truth was the actual PR diff from each open pull request.*

---

## Apendix

### Grading Criteria

#### Scoring Rubric (/20, five dimensions × 4pts each)

##### 1. Correct Files Identified (4pts)

| Score | Criteria |
|-------|----------|
| 4 | All files in the ground truth diff are touched |
| 3 | Most files touched, or all correct files identified but one missed in output |
| 2 | Core file(s) correct but missed secondary files (tests, callers, integration points) |
| 1 | Only partially overlaps with PR files |
| 0 | Completely wrong files |

**What counts:** The agent's diff must touch the file, not just mention it. Mentioning a file in analysis without producing changes gets +1 partial credit at most.

##### 2. Fix Location Within File (4pts)

| Score | Criteria |
|-------|----------|
| 4 | Same function, same insertion/modification point as PR |
| 3 | Same function, slightly different position (e.g., guard placed 5 lines earlier/later) |
| 2 | Right file but wrong function, or right function but wrong layer (e.g., fixing at callee vs caller) |
| 1 | Right package but wrong file or completely wrong function |
| 0 | Wrong package entirely |

**Key distinction:** "Wrong layer" is a 2. Fixing the bug at the error source when PR fixes it at the caller (or vice versa) means the agent understood the symptom but not the design intent.

##### 3. Fix Mechanism / Architectural Correctness (4pts)

| Score | Criteria |
|-------|----------|
| 4 | Same approach as PR — same data flow, same abstractions, same error handling pattern |
| 3 | Functionally correct fix but different architecture (e.g., inline vs deferred, swallowing vs wrapping errors, new field vs reusing existing field) |
| 2 | Partially correct — fixes the immediate symptom but introduces a different tradeoff or misses a subtle requirement |
| 1 | Addresses the right problem area but fix is wrong or incomplete enough to not work |
| 0 | Wrong fix entirely |

**This is the hardest dimension to score.** The key question: "Would a reviewer accept this as equivalent to the PR, or would they request a redesign?" A 3 means "works but reviewer would suggest a better approach." A 4 means "reviewer would approve as-is."

##### 4. Test Quality (4pts)

| Score | Criteria |
|-------|----------|
| 4 | Updates existing test expectations correctly AND adds meaningful new test cases covering the fix |
| 3 | Good new tests but misses updating existing expectations, OR updates existing but weak new tests |
| 2 | Tests exist but are shallow — e.g., standalone `t.Run` instead of using existing table structure, or only happy path |
| 1 | Minimal or broken tests — wrong assertions, won't compile, or test the wrong thing |
| 0 | No tests produced |

**Important nuance:** I don't penalize for missing tests that cover code the agent didn't change. If the agent didn't touch `proxier.go`, I don't dock test points for missing `proxier_test.go` — that gets captured in Completeness instead. Tests are scored relative to the scope of the agent's fix.

**Existing test updates matter a lot.** Flipping `want: true` → `want: false` on existing cases was something almost no session did. That's the difference between a 3 and a 4.

##### 5. Completeness (4pts)

| Score | Criteria |
|-------|----------|
| 4 | All affected files, all edge cases, all call site updates, all mechanical changes |
| 3 | Core fix complete, missed one secondary concern (cosmetic renames, one call site, one edge case) |
| 2 | Core fix present but significant parts of the PR are missing (whole files untouched, integration layer missing) |
| 1 | Only the most obvious change, most of the PR scope is missing |
| 0 | Barely started |

**This is the catch-all.** Things that land here:
- Missing call site updates (e.g., every caller of a function whose signature changed)
- Missing mechanical/cosmetic changes (variable renames, constant additions)
- Missing integration-layer changes (proxier.go changes when only hns.go was fixed)
- Missing edge case handling
- Scope discovery failures (agent fixes root cause file but misses that 3 other files need updating)

##### Cross-cutting Rules

1. **No answer-leaking credit.** If a session found the actual PR diff via GitHub API, the entire run is disqualified and rerun with constraints.

2. **Functional equivalence counts.** If the agent's fix is architecturally different but would pass the same test suite and handle the same edge cases, that's a 3 on mechanism, not a 2.

3. **Cosmetic differences don't matter.** Variable names (`sanitizedImage` vs `sanitizedForPolicyCheck`), comment verbosity, log message wording — all ignored.

4. **Compilation matters.** If the diff clearly wouldn't compile (wrong function signatures, missing imports), that's a -1 on whichever dimension it affects.

5. **Scoring is against PR, not against perfection.** A creative fix that's arguably *better* than the PR still gets scored against what the PR actually did. This keeps scoring objective.

##### What I Learned About This Rubric

The biggest scoring gap across all PRs was consistently **Completeness**. Agents nail the core fix (dimensions 1-3 tend to score 3+) but miss integration changes, call site updates, and test infrastructure. The second-weakest dimension was **Test Quality** — specifically updating existing tests rather than just adding new ones.

The rubric slightly favors well-scoped PRs (XS/S) where there's less to miss on Completeness, which is why Local scored 20/20 on `#136013` (2-file, 12-line change) but only 8-16/20 on XL PRs.