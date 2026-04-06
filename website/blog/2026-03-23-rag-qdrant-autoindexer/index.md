---
title: "I Indexed the Entire Kubernetes Codebase into a RAG Pipeline and Here's What I Learned About Code Search at Scale"
date: 2026-03-23
description: "How semantic retrieval, local code search, and hybrid strategies compare when navigating 4.5 million lines of Go with real benchmarks."
authors:
- brandon-foley
tags: ["kaito", "ai", "rag", "ragengine", "autoindexer", "code-search"]
---

The Kubernetes codebase is 4.5 million lines of Go spread across 22,000+ files. When you're debugging a scheduler preemption issue at 2am during an oncall incident, you don't have time to `grep -r` your way through it. You need the right file, the right function, the right line, and you need it *now*.

So I built a system that lets me ask natural language questions like *"How does the scheduler handle pod preemption and victim selection?"* and get back the exact source files, function signatures, and call chains that answer it. Then I benchmarked three different retrieval strategies against 10 real questions spanning every major Kubernetes subsystem, and the results challenged some assumptions I had about leveraging RAG for coding.

This post walks through the architecture, the benchmark methodology, and the data. If you're building AI-powered code intelligence tools, these findings will save you weeks of experimentation.

## The Setup: KAITO RAGEngine + Qdrant + The Kubernetes Source

The retrieval backend is [KAITO RAGEngine](https://github.com/kaito-project/kaito), a Kubernetes-native RAG pipeline that runs on AKS. It uses [Qdrant](https://qdrant.tech/) as the vector store and [BAAI/bge-small-en-v1.5](https://huggingface.co/BAAI/bge-small-en-v1.5) as the embedding model. The full Kubernetes source tree was chunked and indexed into a `kubernetes-codebase` index.

The API is simple:

```bash
curl -X POST http://$RAG_HOST/retrieve \
  -H 'Content-Type: application/json' \
  -d '{
    "index_name": "kubernetes-codebase",
    "query": "scheduler pod preemption victim selection",
    "max_node_count": 10
  }'
```

You get back ranked chunks with relevance scores, file paths, and metadata. Latency is typically 90-150ms.

But the interesting question isn't whether RAG *works*, it's whether it works *well enough* for code, where precision matters more than any other domain.

## The Benchmark: 10 Questions Across Every Major Kubernetes Subsystem

I designed 10 questions that a Kubernetes engineer would actually ask during development or incident response. Each targets a different subsystem and requires understanding multiple files, call chains, and data structures:

| # | Question | Target Subsystem |
|---|---------|:---:|
| 1 | How does the scheduler handle pod preemption and victim selection? | `scheduler/framework/plugins/defaultpreemption` |
| 2 | How does the scheduler evaluate topology spread constraints during filtering? | `scheduler/framework/plugins/podtopologyspread` |
| 3 | How does the API server chain admission webhooks? | `apiserver/pkg/admission/plugin/webhook` |
| 4 | How does the API server implement watch caching and event fan-out? | `apiserver/pkg/storage/cacher` |
| 5 | How does kubelet evict pods under memory pressure? | `kubelet/eviction` |
| 6 | How does kubelet probe containers for liveness/readiness? | `kubelet/prober` |
| 7 | How does the Deployment controller implement rolling updates? | `controller/deployment` |
| 8 | How does the garbage collector handle cascading deletion? | `controller/garbagecollector` |
| 9 | How does kube-proxy in iptables mode program ClusterIP rules? | `proxy/iptables` |
| 10 | How does the RBAC authorizer evaluate request permissions? | `auth/authorizer/rbac` |

Each question requires knowledge of struct definitions, interface contracts, function call chains across multiple files, and sometimes cross-subsystem interactions.

I then tested three retrieval strategies against all 10:

1. **RAG-Only** — Pure semantic retrieval via KAITO RAGEngine
2. **Hybrid** — RAG for semantic discovery + local `ripgrep` for structural code navigation
3. **Local-Only** — Pure keyword search with `ripgrep` across the cloned repo

Each strategy was implemented as an [OpenClaw](https://github.com/openclaw/openclaw) skill, a reusable, declarative instruction file that teaches an AI agent *how* to use a tool. The RAG-only skill tells the agent how to call the `/retrieve` API and interpret results. The hybrid skill layers on local search instructions: use RAG file paths to scope `ripgrep`, extract function bodies with `sed`, and discover related files the embedding model missed. No custom code, just structured guidance that any OpenClaw instance can pick up and execute.

This matters because the skills are portable. Drop the SKILL.md into any OpenClaw workspace with access to a KAITO RAGEngine endpoint, and the agent immediately knows how to search the indexed codebase. The hybrid skill just needs a local clone and `ripgrep` installed. The agent handles the orchestration, calling the API, parsing results, running grep, merging context, all based on the skill's instructions.

Metrics tracked per query: retrieval latency, context tokens consumed, unique source files returned, and a qualitative **completeness score**, what percentage of key implementation aspects each strategy surfaced.

## Finding #1: The Chunk Count Sweet Spot Is Not What You'd Expect

Before comparing strategies, I needed to tune `max_node_count`, or how many chunks the RAG returns per query. I tested n=5, n=10, and n=20.

| Metric | n=5 | n=10 | n=20 |
|--------|:---:|:---:|:---:|
| Avg Latency | 107ms | 111ms | 117ms |
| Avg Context Tokens | 1,640 | 3,372 | 6,194 |
| Avg Unique Files | 3.7 | 6.6 | 11.5 |
| Avg Top Score | 0.662 | 0.700 | 0.753 |

The latency difference is negligible, only 10ms between n=5 and n=20. What changes dramatically is file coverage and token cost.

**n=10 is the sweet spot.** It delivers 80% of n=20's file coverage (6.6 vs 11.5 unique files) at 54% of the token cost (3,372 vs 6,194 tokens). The diminishing returns from n=10 to n=20 are steep. You pay an extra 2,822 tokens to get 4.9 more files, most of which are CHANGELOGs, proto definitions, and OpenAPI specs. For the test questions, this turned out to be noise.

The real insight: **n=5 misses entire subsystems.** For Question 9 (iptables proxy), n=5 returned only test files and documentation. The actual `proxier.go` implementation didn't appear until n=10. If you're building a code intelligence tool on top of RAG, defaulting to n=5 can produce answers that look correct but are missing key implementation aspects.

## Finding #2: RAG Is Fast but Structurally Blind

Here's the headline result. Three strategies, same 10 questions:

| Metric | RAG-Only | Hybrid | Local-Only |
|--------|:---:|:---:|:---:|
| **Avg Latency** | 111ms | 4,591ms | 27,820ms |
| **Avg Context Tokens** | 3,372 | 7,930 | 9,184 |
| **Avg Unique Files** | 6.6 | 10.0 | 10.2 |
| **Completeness Score** | **37%** | **65%** | **100%** |

Read that completeness number again. **RAG-only surfaces 37% of the key implementation aspects.** That means for nearly two-thirds of the structural code knowledge you need interface definitions, call chains, sibling implementations, config types, but the RAG returns nothing.

This isn't a KAITO-specific problem. It's a fundamental limitation of embedding-based retrieval for code. Embeddings capture *semantic similarity* (what the code is about), but they don't capture *structural relationships* (who calls what, which interface does this implement, what other files handle the same pattern differently).

Here's a concrete example. For Question 9, "How does kube-proxy in iptables mode program ClusterIP rules?", here's what each strategy found:

| Aspect | RAG-Only | Hybrid | Local-Only |
|--------|:---:|:---:|:---:|
| `iptables/proxier.go` implementation | ❌ | ✅ | ✅ |
| `syncProxyRules` function | ❌ | ✅ | ✅ |
| KUBE-SVC / KUBE-SEP chain structure | ❌ | ❌ | ✅ |
| Cross-implementation comparison (nftables, IPVS, winkernel) | ❌ | ✅ | ✅ |
| Proxier struct definition | ❌ | ❌ | ✅ |

RAG returned *zero* useful aspects for this question. It found test files, a README about nftables, and some tangentially related code. The actual implementation, the thing you'd need to understand to debug a kube-proxy issue, was completely absent.

This pattern repeats across questions. RAG consistently misses:

- **Interface definitions** — Found the type but not the interface it implements
- **Entry points** — Found the implementation but not where it's called from
- **Sibling implementations** — Found iptables but not nftables/IPVS/winkernel
- **Configuration types** — Found the runtime code but not the config structs that control its behavior

## Finding #3: Hybrid Search Is the Practical Winner

The hybrid strategy of using RAG for semantic discovery then `ripgrep` for structural navigation, hits the practical sweet spot:

```
RAG says: "The scheduler preemption logic is in defaultpreemption/default_preemption.go"
Local search says: "That function is called from schedule_one.go:288, and there's a PodGroup 
variant in podgrouppreemption.go, and the PostFilter interface is defined in framework/interface.go:265"
```

RAG provides the *"where to look"* signal. Local search provides the *"what's actually there"* precision. Together, they produce answers that include:

- Exact file paths and line numbers
- Complete function signatures
- Caller → callee relationships
- Parallel implementations across subsystems
- Interface contracts that define the abstraction boundaries

The cost is latency: 4.6 seconds average vs 111ms for RAG-only. That's because `ripgrep` across 4.5 million lines of Go takes 3-5 seconds per query, even on a fast machine. But for a code investigation tool (as opposed to a real-time chatbot), 4.6 seconds is perfectly acceptable.

**The key architectural insight:** The hybrid approach uses RAG file paths to *scope* the local search. Instead of grepping the entire repo blind (27.8 seconds), it greps within the 6-10 files RAG identified as relevant, then does a targeted broader search for specific identifiers. This cuts local search time by 6x compared to the local-only strategy.

## Finding #4: Local-Only Is Exhaustive but Impractical

Local-only search achieved 100% completeness on every question, it never misses, because it's doing exhaustive keyword matching. But the tradeoffs are severe:

- **251x slower than RAG** (27.8 seconds vs 111ms)
- **Requires domain expertise** to write good grep patterns. The queries that worked used identifiers like `syncProxyRules`, `runAttemptToDeleteWorker`, `selectVictimsOnNode`, you need to already know the codebase to ask the right questions.
- **No relevance ranking** — returns everything that matches, including generated code, test helpers, and apply-configuration boilerplate. You spend more time filtering noise.
- **Requires a local clone** — the Kubernetes repo is ~1.3GB. Not always practical.

Local search has its place, but it's not viable as an interactive code intelligence backend.

## Finding #5: Token Economics Change the Calculus

In production, you're paying for tokens, both the context tokens you send to the LLM and the output tokens it generates. Here's how the strategies compare:

| Strategy | Avg Tokens/Query | Cost per 10 Queries (@ $3/M input) | Annual Cost (100 queries/day) |
|----------|:---:|:---:|:---:|
| RAG-Only (n=5) | 1,640 | $0.05 | $18 |
| RAG-Only (n=10) | 3,372 | $0.10 | $37 |
| RAG-Only (n=20) | 6,194 | $0.19 | $68 |
| Hybrid | 7,930 | $0.24 | $87 |
| Local-Only | 9,184 | $0.28 | $100 |

The token differences might look small per query, but they compound. An engineering team running 100 queries a day would spend $37/year with RAG n=10 vs $87/year with hybrid, and the hybrid answers are nearly twice as complete.

More importantly, the *quality* of tokens matters. RAG's 3,372 tokens are all semantically relevant snippets selected by the embedding model. Local-only's 9,184 tokens include generated code, swagger docs, and test wrappers that dilute the LLM's attention. Hybrid's 7,930 tokens combine the best of both, RAG's curated snippets plus local search's precise function bodies.

## The Architecture for Production

Based on these findings, here's the architecture I'd recommend for a production code intelligence system:

```
┌──────────────────────────────────────────────────────┐
│                  Query Router                        │
│                                                      │
│  1. Send query to RAG (n=10)              ~111ms     │
│  2. Check top score                                  │
│     ├── Score ≥ 0.8 → Return RAG-only     ~111ms     │
│     └── Score < 0.8 → Augment with local  ~4.5s      │
│         a. Extract file paths from RAG               │
│         b. ripgrep key identifiers in those files    │
│         c. ripgrep across pkg/ for missed files      │
│         d. Merge and deduplicate context             │
│  3. Send combined context + question to LLM          │
└──────────────────────────────────────────────────────┘
```

The score-based routing is important. When RAG returns a top score ≥ 0.8, the semantic match is strong enough that local search adds little value. In our benchmark, ~30% of queries had top scores ≥ 0.8. This fast-path optimization cuts average latency from 4.6s to ~3.3s while preserving completeness on the harder questions.

## What This Means for AI-Powered Code Tools

If you're building code intelligence tools, whether it's a codebase Q&A bot, an AI-powered code reviewer, or an IDE plugin, here are the lessons:

**1. RAG alone isn't enough for code.** Embedding similarity finds the right neighborhood, but code understanding requires structural navigation such as call chains, interface hierarchies, sibling implementations. Plan for a hybrid architecture from day one.

**2. Chunk count tuning matters more than model choice.** Going from n=5 to n=10 improved our file coverage by 78% with zero latency penalty. That's a bigger impact than most embedding model upgrades.

**3. Use RAG to scope, not to answer.** The highest-value use of RAG in code search is identifying *which files are relevant*. Once you know `default_preemption.go` is the right file, a simple `grep` gives you the function body, the callers, and the interface it implements, things embedding models can't surface.

**4. Measure completeness, not just relevance.** Standard RAG benchmarks measure whether the top-k results contain the answer. For code, the question is whether you found *all the structural pieces* needed to understand the system. Our completeness score, aspects found vs total aspects, revealed gaps that standard metrics would miss.

**5. The economics favor hybrid.** At $87/year for 100 queries/day, hybrid search costs less than a single engineer-hour saved per month. If it prevents even one extra hour of code archaeology per month, it's ROI-positive.

## Try It Yourself

The full stack runs on AKS using three open-source components:

- [**KAITO RAGEngine**](https://github.com/kaito-project/kaito) — The retrieval pipeline with Qdrant vector storage
- [**KAITO AutoIndexer**](https://github.com/kaito-project/autoindexer) — Automated ingestion from Git repos
- [**Qdrant**](https://qdrant.tech/) — Production-grade vector database with hybrid search

Index any codebase (not just Kubernetes), tune `max_node_count`, and bolt on local search for the queries where semantic retrieval falls short. The deployment walkthrough is in our [companion post on setting up KAITO RAGEngine with Qdrant and AutoIndexer on AKS](https://github.com/kaito-project/kaito-cookbook/tree/master/examples/qdrant-rag-autoindexer).

---

## Appendix: Raw Benchmark Data

### Per-Question Metrics (all three strategies)

| # | Topic | RAG Latency | Hybrid Latency | Local Latency | RAG Tokens | Hybrid Tokens | Local Tokens | RAG Files | Hybrid Files | Local Files | Completeness: RAG / Hybrid / Local |
|---|-------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 1 | Scheduler Preemption | 133ms | 4,729ms | 43,277ms | 2,986 | 8,632 | 11,783 | 5 | 9 | 13 | 33% / 67% / 100% |
| 2 | Topology Spread | 109ms | 3,619ms | 16,984ms | 3,858 | 9,875 | 12,672 | 7 | 10 | 13 | 60% / 60% / 100% |
| 3 | Admission Webhooks | 93ms | 3,002ms | 22,763ms | 2,703 | 15,777 | 10,159 | 7 | 13 | 12 | 40% / 80% / 100% |
| 4 | Watch Cache | 130ms | 5,342ms | 24,102ms | 3,332 | 5,577 | 5,505 | 4 | 4 | 6 | 40% / 40% / 100% |
| 5 | Kubelet Eviction | 98ms | 4,463ms | 32,045ms | 2,991 | 8,610 | 8,048 | 4 | 11 | 9 | 20% / 80% / 100% |
| 6 | Kubelet Probes | 108ms | 5,761ms | 31,848ms | 3,207 | 5,360 | 6,886 | 8 | 11 | 7 | 40% / 80% / 100% |
| 7 | Deployment Rolling Update | 92ms | 5,478ms | 29,453ms | 4,986 | 8,102 | 11,730 | 9 | 10 | 13 | 60% / 80% / 100% |
| 8 | GC Cascading Delete | 88ms | 5,517ms | 24,954ms | 2,380 | 3,769 | 8,193 | 6 | 7 | 7 | 40% / 60% / 100% |
| 9 | kube-proxy iptables | 131ms | 3,573ms | 29,724ms | 2,897 | 6,851 | 8,606 | 6 | 10 | 7 | 0% / 60% / 100% |
| 10 | RBAC Authorizer | 127ms | 4,421ms | 23,052ms | 4,380 | 6,749 | 8,258 | 10 | 15 | 15 | 40% / 40% / 100% |

### Chunk Count Tuning (RAG-Only)

| Metric | n=5 | n=10 | n=20 |
|--------|:---:|:---:|:---:|
| Avg Latency | 107ms | 111ms | 117ms |
| Avg Tokens/Query | 1,640 | 3,372 | 6,194 |
| Avg Unique Files | 3.7 | 6.6 | 11.5 |
| Avg Top Score | 0.662 | 0.700 | 0.753 |
| Token Cost vs n=5 | 1.0x | 2.1x | 3.8x |
| File Coverage vs n=5 | 1.0x | 1.8x | 3.1x |

### Infrastructure

- **RAG Engine**: KAITO RAGEngine on AKS, GPU-based (`Standard_NV36ads_A10_v5`)
- **Vector Store**: Qdrant v1.13.4 with persistent storage
- **Embedding Model**: BAAI/bge-small-en-v1.5
- **Index**: `kubernetes-codebase` — full Kubernetes source
- **Local Search**: ripgrep 14.x on Linux (x64)
- **LLM**: Claude Opus 4.6 (via GitHub Copilot) for answer synthesis
