---
title: "I Gave an AI Agent Kubernetes Bugs and a RAG Index. Here's What Actually Happened."
date: 2026-03-23
description: "A field report on semantic retrieval, local code, and hybrid approaches to AI-assisted code generation in a 4M-line codebase."
authors:
- brandon-foley
tags: ["kaito", "ai", "rag", "ragengine", "autoindexer", "code-search"]
---

I've been using AI coding agents daily for some time now, not as a novelty, but as part of my actual engineering workflow. There's a lot of surface-level optimism about what these agents can do, especially with techniques like retrieval-augmented generation (RAG) that promise to make large codebases "searchable" to models. You might believe we're close to a world where you paste a bug report, say "fix this," and get back a merge-ready pull request.

But how well does that hold up in practice? Not on toy examples or self-contained scripts, but against real bugs in a codebase with millions of lines of code, where the fix spans multiple files and the root cause isn't obvious from any single snippet. I wanted to find out — so I ran a series of structured experiments, giving agents real bug reports from a large-scale project (Kubernetes repo) and measuring how close they got to correct, complete fixes without any guidance or hints.

Going in, my assumption was straightforward: performance would come down to how well the system retrieves relevant code. Better search — whether through RAG or filesystem traversal — should be the dominant factor. A model that finds the right files should produce the right fix.

What I found was more nuanced, and at times, surprising. The bottleneck wasn't where I expected. The patterns that emerged — around retrieval, multi-file reasoning, scope inference, and architectural understanding — challenged some of my initial beliefs and revealed failure modes I hadn't anticipated.

In the sections that follow, I'll walk through the experimental setup, the specific results, and the lessons I took away. If you're integrating agents into a real engineering workflow, or building tools that support them, I think these findings will change how you think about what actually matters.

## The Setup

I took open pull requests from the [`kubernetes` github repo](https://github.com/kubernetes/kubernetes).  Real bugs, actively being fixed by real contributors.  I extracted just the issue description (not the PR description, not the diff, nothing that would leak the solution) and gave each issue to three different agent configurations:

- **RAG Only**: Hybrid retrieval over an indexed copy of the Kubernetes codebase via [KAITO RAG Engine (Qdrant)](https://kaito-project.github.io/kaito/docs/rag), combining BM25 for keyword matching with embedding based semantic search. KATIO also provides an auto-indexing controller which is perfect for indexing huge git repos with the capability of incremental indexing.  Results are merged and ranked before being returned as context snippets.  No local files or web access; the agent only sees retrieved chunks.
- **Hybrid (RAG + Local)**: Same RAG index, but also has a full local clone of kubernetes repository.  The agent must start with RAG for discovery, then can read local files for precision.
- **Local Only**: Full clone, grep, find, cat.  No RAG, no web.  The agent explores the codebase through direct filesystem traversal.

Each agent ran in a completely isolated session.  Same model (Claude Opus 4.6), same timeout (5 minutes), same output format.  The only variable was how they could see code.

One critical constraint: the RAG and Hybrid agents were prompted to make RAG queries before producing any fix.  Early experiments showed that without this mandate, hybrid agents would skip RAG entirely when the issue mentioned specific filenames, defeating the whole point of the comparison.

## The Test Cases (Benchmark)

I used a set of real, in-flight bug fixes from the Kubernetes repository as a benchmark.  Each test case is an open pull request where the agent is given only the issue description and asked to produce a patch.  They span kubelet, scheduler, networking, storage, and apps.  From a one-line guard clause to a 900-line multi-file refactor.

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

Each generated diff was compared against the corresponding pull request diff and graded across five dimensions (0–4 each):

- Files: Did the diff touch the correct files?
- Location: Was the fix applied in the correct function/layer?
- Mechanism: Did the fix preserve correct system level invariants and layering, not just patch symptoms?
- Tests: Were existing tests updated or meaningful new tests added?
- Completeness: Did the patch propagate changes across all required call sites and dependent paths?

This evaluation uses the PR diff as the reference implementation.  That is not a perfect trut.  PRs reflect iterative discussion and may not represent the only or final correct solution.  Here, it serves as a proxy for ‘what human contributors converged on under review constraints.’

## The Scoreboard

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

Performance varied more by task type than by approach.  Multi-file and integration heavy bugs showed clear separation, while single-file fixes converged across all methods.  On average, Hybrid performed best (71%), but the gap is small relative to per task variance.

The gap isn’t massive, but it’s consistent.  Hybrid never collapsed on any single test case, while RAG struggled more on multi-file bugs and Local was more variable depending on how well specified the issue was.

At the same time, no approach consistently reached close to perfect scores on complex changes, reinforcing that the primary limitation isn’t generating code, it’s identifying the full scope of what needs to change.

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

RAG is consistently the fastest. It doesn't read files, navigate directories, or wait for grep results.  It asks the index, gets snippets, and generates.  Average wall-clock: **1 minute 16 seconds**.  This suggests retrieval reduces exploration overhead but also reduces structural exploration depth.

Hybrid is the slowest on average at **2 minutes 25 seconds**.  The mandatory RAG-first phase adds overhead before local exploration even begins.  On complex PRs like #138000 and #138191, Hybrid pushed past 3-4 minutes.  The tool switching loop dominates latency more than actual file access.

Local matches Hybrid's average but for different reasons.  Hybrid spends time on RAG queries followed by targeted file reads.  Local burns time on broad exploration through grepping, finding, and iterating directories.  Exploration cost is highly sensitive to how well the issue text anchors the search.

## Token Economics

Token usage isn’t just about how much context the agent reads—it’s dominated by how many times it calls the model.

Claude’s API is stateless, so every call replays the full conversation.  That makes call count the primary cost multiplier.

A few terms used below:

- Calls: Number of model invocations in a session
- New Tokens: Tokens newly introduced (retrieved code, file contents, prompts)
- Output: Tokens generated by the model
- Cached (replay): Tokens resent on each call due to conversation replay
- Total: Sum of all tokens processed across the session

**Averages across all runs:**

| Approach | Calls | New Tokens | Output | Cached (replay) | Total |
|----------|-------|-----|--------|--------|-------|
| **RAG** | 4 | 44K | 2K | 141K | 187K |
| **Hybrid** | 8 | 27K | 3K | 234K | 264K |
| **Local** | 6 | 23K | 4K | 162K | 189K |

Three patterns stand out:

- Hybrid is the most expensive, not because it reads more, but because it makes the most calls.  The RAG to local loop creates repeated round-trips, and each one replays a growing context window.
- RAG and Local end up with similar total cost, but for opposite reasons.  RAG pulls in more new context via retrieval, while Local makes more exploratory calls.  The totals converge.
- Fewer calls beats fewer tokens.  Sessions that stayed under ~5 calls were consistently cheaper, regardless of approach.

Across all runs, call count was the biggest driver of both cost and latency.

## What I Actually Learned

### Agent fix locally, not systemically

Agents are good at fixing the problem directly in front of them, but struggle to reason about the broader system.

PR #134540 is a good example. The correct fix required preserving an error (`%w`) so the caller could handle it, and then updating the caller to tolerate that specific case.  Every agent instead swallowed the error at the source. Functionally similar, but architecturally wrong.

This shows up consistently: agents fix symptoms in isolation, but miss the contracts between components.

### Scope discovery is the real bottleneck

The biggest failure mode wasn’t incorrect fixes, it was incomplete ones.

Agents routinely fixed the “main” bug but missed adjacent changes:

- On #138191, RAG-only fixed one sort implementation but missed a second.
- On #138000, all approaches fixed the core bug but missed required changes in `proxier.go` and related integration logic.
- On #134540, Local and Hybrid saw a partial fix already in the codebase and stopped early, missing the second required change.

The common pattern that emerged was agents don’t ask “what else needs to change?”  They stop once the immediate issue appears resolved.  This is most visible in multi-file changes, where agents fix the local bug but fail to propagate required updates across integration points.

### Retrieval changes discovery, not reasoning

RAG meaningfully affects how agents find code, but not how they reason about it.

Forcing RAG usage improved results in some cases. On #138211, mandatory retrieval pushed the agent to discover the policy evaluation layer before implementing a fix, leading to a better architectural choice.

But the limitation remains, once the relevant code is found, the agent still reasons locally.  Retrieval helps with navigation, not with understanding system wide implications.

### Agents prefer adding over reusing

When given a choice, agents tend to introduce new abstractions rather than reuse existing ones.

On #138191, the correct fix used the existing `RestartCount` field.  All agents instead introduced a new `Attempt` field. Functionally correct, but architecturally heavier.

This pattern showed up repeatedly.  Agents don’t reliably recognize when the system already contains the concept they need. They solve the problem in isolation rather than integrating with existing design.  This also shows up in tests.  Agents tend to add new tests rather than update existing ones, even when behavior changes require modifying assertions.

### Issue quality dominates everything

Well-specified issues flatten the differences between approaches.

On PRs like #136013 and #138269, the issue descriptions named the exact file, function, and expected behavior.  All approaches converged to high scores, with Local slightly ahead simply due to speed and direct access.

When the issue effectively acts as a spec, retrieval method matters far less.  When the issue is vague, performance diverges significantly.

## The Bottom Line

These results come from large codebases (millions of lines), where correctness depends on system wide understanding rather than local edits.

They are not about raw model capability, but about how agent workflows behave under scale.

### Indexing helps, but doesn’t solve scope

Semantic search (RAG) improves discovery in large repositories, but does not ensure coverage of all affected components.

### Workflow matters more than model choice

Hybrid setups perform best only when retrieval is enforced. Otherwise, agents default to local reasoning and miss broader context.

### Scope discovery is the core bottleneck

Agents reliably fix visible issues, but fail to consistently identify all dependent changes across the system.

This is the dominant failure mode in multi-file and integration heavy changes.

### Issue quality is the strongest lever

Well defined bug reports reduce performance variance more than tooling or retrieval strategy.

### Skills can help, but they don’t remove the system problem

This experiment did not use any explicit coding “skills” or curated agent playbooks.  Only prompts and tool access.

In principle, structured skills (such as repo exploration strategies or architectural summarization) could improve performance by guiding agents toward better system level reasoning.

However, in large, evolving codebases, these skills would need to be continuously maintained and updated to remain aligned with the actual repository structure.  That makes them less like a one-time improvement and more like an additional system to operate and maintain.

As a result, while skills may improve outcomes, they do not remove the core bottleneck observed in this study: scope discovery and system level reasoning under scale.

---

*The benchmark used Claude Opus via [OpenClaw](https://github.com/openclaw/openclaw) with [KAITO RAG Engine](https://github.com/kaito-project/kaito) indexing the kubernetes/kubernetes repository at HEAD. All sessions ran in isolated subagents with no shared state, April 2026. Ground truth was the actual PR diff from each open pull request.*

---

## Appendix

### Grading Criteria

#### Scoring Rubric (/20, five dimensions × 4pts each)

##### 1. Correct Files Identified (4pts)

| Score | Criteria |
|-------|----------|
| 4 | All files in the PR diff are touched |
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
