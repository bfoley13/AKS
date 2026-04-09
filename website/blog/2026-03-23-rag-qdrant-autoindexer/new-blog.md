# I Gave an AI Agent 3 Kubernetes Issues and a RAG Index. Here's What Actually Happened.

*A field report on semantic retrieval, local code, and hybrid approaches to AI-assisted code generation in a 4M-line codebase.*

---

I've been using AI agents daily for some time now. Not as a novelty, but as part of my actual engineering workflow on Azure Kubernetes Service. Reviewing diffs, investigating incidents, writing KQL queries. The usual stuff.

But one question kept nagging me. If I hand an agent a bug report and just say "fix this," how close does it actually get? Not explaining the fix to me. Not summarizing the issue. Actually producing a diff I could submit as a pull request.

So I ran the experiment.

---

## The Setup

I took three open pull requests from the [`kubernetes` github repo](https://github.com/kubernetes/kubernetes). Real bugs, actively being fixed by real contributors. I extracted just the issue description (not the PR description, not the diff, nothing that would leak the answer) and gave each issue to three different agent configurations:

- **RAG Only**: Semantic search over an indexed copy of the entire Kubernetes codebase via [KAITO RAG Engine](https://github.com/kaito-project/kaito). No local files, no web access. The agent queries the index, gets ranked code snippets back, and works from those alone.
- **Hybrid (RAG + Local)**: Same RAG index, but also has a full local clone of kubernetes/kubernetes. The agent must start with RAG for discovery, then can read local files for precision — exact line numbers, full function bodies, test infrastructure.
- **Local Only**: Full clone, grep, find, cat. No RAG, no web. The agent explores the codebase the old-fashioned way.

Each agent ran in a completely isolated session. Same model (Claude Opus), same timeout (5 minutes), same output format. The only variable was how they could see code.

One critical constraint: the RAG and Hybrid agents were required to make RAG queries before producing any fix. Early experiments showed that without this mandate, hybrid agents would skip RAG entirely when the issue mentioned specific filenames — defeating the whole point of the comparison.

The three issues, in order of difficulty:

1. **#94198 — SubPath EEXIST Race**: Two containers in the same pod start simultaneously, both create the same subpath directory, one gets EEXIST and the pod dies. A concurrency bug in kubelet's volume mounting.

2. **#138175 — Image Pull Record Poisoning**: A preloaded image shares a content digest with a previously pulled image. The kubelet's credential mapping gets confused and blocks the preloaded image with ErrImageNeverPull. A logic bug in the image pull policy evaluation.

3. **#135541 — NUMA Topology Stall**: On NVIDIA GB200 systems with 34 NUMA nodes, the topology manager iterates 2^34 bitmask subsets and never finishes. A combinatorial explosion in device hint generation.

---

## The Numbers

| PR | Issue | RAG | Hybrid | Local |
|----|-------|-----|--------|-------|
| #134540 | SubPath EEXIST race | 12/20 (60%) | 13/20 (65%) | 12/20 (60%) |
| #138211 | Image pull poisoning | 17/20 (85%) | 18/20 (90%) | 17/20 (85%) |
| #138244 | NUMA topology stall | 16/20 (80%) | 17/20 (85%) | 14/20 (70%) |
| **Average** | | **75%** | **80%** | **62%** (est.) |

Scoring was out of 20 across five dimensions: correct files identified (0-4), fix location within file (0-4), fix mechanism / architectural correctness (0-4), test quality (0-4), and completeness across all affected files and edge cases (0-4).

---

## Token Economics

Every token is money. Here's what each session actually consumed:

| Session | Approach | PR | Retrieved Context | New Tokens | Output | Total |
|---------|----------|-----|-------------------|------------|--------|-------|
| pr134540-rag | RAG | #134540 | 262k | 120k | 3.6k | 382k |
| pr134540-hybrid | Hybrid | #134540 | 198k | 32k | 3.2k | 230k |
| pr134540-local | Local | #134540 | 130k | 24k | 2.8k | 154k |
| pr138211-rag | RAG | #138211 | 186k | 43k | 10k | 239k |
| pr138211-hybrid | Hybrid | #138211 | 817k | 41k | 7.6k | 866k |
| pr138211-local | Local | #138211 | 1,048k | 48k | 9.1k | 1,057k |
| pr138244-rag | RAG | #138244 | 119k | 45k | 3.1k | 164k |
| pr138244-hybrid | Hybrid | #138244 | 153k | 28k | 3.2k | 181k |
| pr138244-local | Local | #138244 | 144k | 27k | 3.0k | 171k |


A few things jump out.

RAG on #134540 consumed 382k tokens — nearly double Hybrid's 230k. Why? RAG sessions make more queries because each one returns snippets, not full files. The agent keeps asking follow-up questions to get more context. Hybrid asks RAG for the high-level picture, then reads the actual file locally in one shot. Less back-and-forth, fewer tokens.

Local is generally the cheapest. It reads files directly with no retrieval overhead. But cheapest doesn't mean best — Local scored lowest on average because it spends tokens exploring dead ends when the issue doesn't point directly at the right file.

The practical takeaway: Hybrid costs about 30% more than Local but scores 18 percentage points higher. RAG costs the most and lands in the middle. If you're optimizing for accuracy per dollar, Hybrid wins.

---

## What I Actually Learned

### The `%w` Gap: What AI Consistently Misses

The actual PR for #134540 was a two-layer fix. In `doSafeMakeDir`, change `%s` to `%w` in the error format string so the EEXIST error is wrapped rather than stringified. Then in the caller (`kubelet_pods.go`), add `!goerrors.Is(err, os.ErrExist)` to tolerate that specific wrapped error.

Zero out of three sessions got this. Every single one — RAG, Hybrid, Local — chose to swallow the EEXIST error at the source by adding `!os.IsExist(err)` guards around the mkdir call. Functionally similar, but architecturally wrong. The actual PR preserves the error and lets the caller decide what to tolerate.

This is a Go idiom thing. Error wrapping with `%w` for programmatic handling is a design choice that requires understanding the caller-callee contract. The agents see the immediate error and fix it locally. They don't think about who else might need to inspect that error and why.

Every approach scored 12-13/20 on this PR. The ceiling wasn't set by code access. It was set by the kind of reasoning the issue demanded.

### RAG-First Discovery Changes Fix Architecture

On #138211, leveraging the RAG for multiple queries before writing any code made a material difference. Multiple calls to the RAG pushed the agent to discover the policy evaluation layer — `RequireCredentialVerificationForImage`, `NeverVerifyPreloadedImages` — before jumping to a fix. That extra context led to the correct architectural choice: reset `imagePulledByKubelet` upstream rather than patching the credential lookup downstream.

RAG scored 17/20. It found the right fix location, the right mechanism, and wrote a focused test. The only thing it missed was updating two existing test expectations that the actual PR flipped from `true` to `false`.

Hybrid scored 18/20 — the highest single run across the entire experiment. After RAG pointed it at the right area, it read the full test file locally and discovered those existing test cases. It was the only session across all rounds that flipped existing test expectations rather than just adding new ones.

Local also scored 17/20 on this PR. When the issue says "look at MustAttemptImagePull in image_pull_manager.go," having the complete file is powerful. But it didn't find those test expectations to flip.

### When the Issue Is the Spec, Everyone Converges

#138244 was the most well-specified issue. It named the exact function, the exact file, the exact algorithmic problem (O(2^n) bitmask iteration), and the exact fix strategy (constrain to device-hosting NUMA nodes only). All three approaches scored within 3 points of each other. RAG got 16/20, Hybrid 17/20, Local 14/20.

Local's lower score came from test quality. Without RAG to surface test patterns and helper functions (`makeNUMADevice`, `NewResourceDeviceInstances`), the Local agent spent more time navigating the test infrastructure and produced a less complete test. Hybrid's RAG queries surfaced the test helpers in the first pass, so the local file reads were targeted and efficient.

The lesson: when issue quality is high enough, the retrieval method barely matters. The issue itself becomes the spec, and any competent agent can execute against it.

### Prompt Discipline Is Not Optional

Early in this experiment, the Hybrid agent prompt said "you may use the KAITO RAG API" and "you may also read local files." Both optional. The agent optimized for speed and skipped RAG entirely on 2 of 3 PRs, going straight to local files when the issue mentioned specific filenames. This made "Hybrid" results indistinguishable from "Local" results — a waste of a whole experimental arm.

Adding "you MUST start with RAG" and "at least 3 queries before reading local files" changed everything. Average Hybrid score jumped from 72% to 80%. RAG went from 65% to 75%. The mandatory exploration step forced broader context before the agent committed to a fix direction.

If you're building agent pipelines, the lesson is: don't make retrieval optional. If you have a RAG index, require the agent to use it. Agents will take shortcuts unless you explicitly prevent it.

---

## The Scorecard

Across 9 sessions on 3 real Kubernetes bugs:

**Hybrid wins.** 80% average, best single run (18/20), most consistent floor. The RAG-then-local workflow gives discovery breadth AND implementation precision.

**RAG is surprisingly strong.** 75% average with only code snippets from the RAG, no ability to read full files or explore directory structure. Semantic search over a well-indexed codebase gets you most of the way there.

**Local is high-variance.** 85% when the issue points at the right file, 60% when it doesn't. The agent drowns in 4 million lines of code without a compass.

**Issue quality dominates everything.** The difference between a vague bug report and a well-specified one mattered more than the difference between any two approaches. A clear issue with the right file and function named got 80%+ from everyone. A vague issue about a race condition capped everyone at 60-65%.

**AI can't do error-wrapping idioms.** The `%s` to `%w` change was 4 characters. It was the most important part of the PR. Zero sessions found it. This is the kind of insight that still requires a human who understands the codebase's error-handling contracts.

---

## What This Means for Your Codebase

If you're thinking about setting up RAG over your own codebase for agent-assisted development, here's what I'd actually recommend:

Index it. A semantic index over your codebase is worth the setup cost. KAITO made this straightforward, point the [KAITO AutoIndexer](https://github.com/kaito-project/autoindexer) at a repo, let it chunk and embed, and you have a retrieval endpoint. The RAG-only approach scored 75% with just an index and zero local file access.

Use hybrid if you can. If the agent has both RAG and a local clone, mandate RAG-first. The discovery phase matters. Don't let the agent skip it.

Write better issues. The single highest leverage thing you can do isn't improving your retrieval pipeline or fine-tuning your model. It's writing bug reports that name the file, the function, and the expected behavior. That turned a 60% fix into an 85% fix across every approach.

For an Agent that isnt running and testing the code, don't trust it with tests. Every session could produce new test cases. None could reliably update existing test expectations. If your PR needs to flip a `want: true` to `want: false` somewhere, you're still doing that yourself.

And don't trust it with Go error idioms. But you probably already knew that.

---

*The benchmark used Claude Opus via OpenClaw with KAITO RAG Engine indexing the kubernetes repository at HEAD. All sessions ran April 2026. North star was the actual PR diff from each open pull request. Code, prompts, and full session logs available on request.*