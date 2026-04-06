---
name: kaito-k8s-rag
description: Search the Kubernetes source code using semantic retrieval powered by the KAITO RAG Engine. Use when users want to find code, understand implementations, explore repo structure, read source files, or search documentation in the Kubernetes repository.
---

# KAITO Kubernetes Codebase RAG

Search the Kubernetes source code using semantic retrieval powered by [KAITO RAG Engine](https://github.com/kaito-project/kaito).

## When to Use

Use this skill when the user asks about:
- How something works in the Kubernetes codebase (scheduler, kubelet, API server, controllers, etc.)
- Where a feature is implemented in k8s source
- Understanding k8s internals, data structures, interfaces, or control flow
- Finding code examples or patterns in upstream Kubernetes
- "How does k8s do X?", "Where is Y implemented?", "Show me the code for Z"

Do **not** use for: general Kubernetes usage questions (kubectl, YAML manifests, cluster operations) — only for questions about the **source code** itself.

## Configuration

| Variable | Description |
|---|---|
| `KAITO_RAG_HOST` | RAG engine endpoint (IP or hostname with optional port) |
| `KAITO_RAG_INDEX` | Index name to query |

Users should set these in their `TOOLS.md` or environment. The defaults point to the shared KAITO RAG instance with the full Kubernetes repository indexed.

## API Reference

This skill uses the KAITO RAG Engine `/retrieve` endpoint:

```
POST http://<KAITO_RAG_HOST>/retrieve
Content-Type: application/json

{
  "index_name": "<KAITO_RAG_INDEX>",
  "query": "<semantic search query>",
  "max_node_count": <1-10>
}
```

### Response Shape

```json
{
  "query": "...",
  "results": [
    {
      "doc_id": "...",
      "node_id": "...",
      "text": "<code snippet>",
      "score": 0.85,
      "dense_score": 0.82,
      "sparse_score": 42.7,
      "source": "both|dense_only|sparse_only",
      "metadata": {
        "file_path": "pkg/scheduler/framework/...",
        "repository": "https://github.com/kubernetes/kubernetes.git",
        "branch": "master",
        "language": "go",
        "commit": "abc123...",
        "timestamp": "2026-04-03T..."
      }
    }
  ],
  "count": 5
}
```

Key metadata fields:
- `file_path` — location in the kubernetes repo
- `score` — combined relevance (1.0 = best)
- `dense_score` / `sparse_score` — embedding vs keyword match scores
- `source` — whether match came from dense, sparse, or both signals

## How to Use

### Step 1: Formulate a Good Query

Convert the user's question into a semantic search query optimized for code retrieval. Tips:
- Be specific: "scheduler bind pod to node" > "scheduling"
- Include Go identifiers when known: "RunFilterPlugins scheduler framework"
- Ask about behavior: "how does kubelet evict pods when under memory pressure"
- Reference subsystems: "kube-apiserver admission webhook chain"

### Step 2: Call the Retrieve API

```bash
curl -s -X POST "http://${KAITO_RAG_HOST:-4.165.62.140}/retrieve" \
  -H 'Content-Type: application/json' \
  -d '{
    "index_name": "'"${KAITO_RAG_INDEX:-kubernetes-codebase}"'",
    "query": "<your query>",
    "max_node_count": 5
  }'
```

Use `max_node_count`:
- **3-5** for focused, specific lookups
- **10** (default) for general exploration — best cost/coverage tradeoff
- **15-20** when you need broad coverage or the topic spans multiple files

### Step 3: Interpret and Present Results

1. **Extract the code snippets** from `results[].text`
2. **Note file paths** from `results[].metadata.file_path` — these tell the user exactly where in the repo to look
3. **Sort by score** — results are pre-sorted, but note that `score: 1.0` with both dense+sparse signals are highest confidence
4. **Synthesize an answer** — don't just dump raw results. Explain what the code does, how the pieces fit together, and reference the file paths

### Presentation Format

When presenting results:
- Cite file paths as `pkg/scheduler/framework/plugins/volumebinding/binder.go` (relative to repo root)
- Link to GitHub when helpful: `https://github.com/kubernetes/kubernetes/blob/master/<file_path>`
- Show relevant code snippets in Go code blocks
- Explain the code in context of the user's question

### Step 4: Iterate if Needed

If the first query doesn't surface what you need:
- Rephrase with different terms or Go identifiers found in initial results
- Try narrower or broader queries
- Use metadata filters if available (not commonly needed)

## Example

**User:** "How does the Kubernetes scheduler handle pod preemption?"

**Query:** `"scheduler pod preemption victim selection"`

**Follow-up query if needed:** `"PostFilter preempt DefaultPreemption plugin"`

## Other Available Indexes

The RAG engine may have additional indexes. Check with:
```bash
curl -s http://${KAITO_RAG_HOST}/indexes
```

Known indexes: `kubernetes-codebase`, `kaito-docs`, `kaito-codebase`

## Troubleshooting

- **Connection refused**: Verify `KAITO_RAG_HOST` is reachable. The service runs on port 80 by default.
- **Empty results**: Try a broader query or check that the index exists via `GET /indexes`
- **Low scores**: Results with `score < 0.3` are weak matches — consider rephrasing