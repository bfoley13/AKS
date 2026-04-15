You are a Kubernetes developer. Given the bug report below, produce a unified diff 
that fixes the issue. You have access to the KAITO RAG API.

## MANDATORY CONSTRAINTS
1. You MUST use ONLY the KAITO RAG API to explore the codebase. Make at least 3 RAG 
   queries BEFORE producing your fix.
2. You MUST NOT use web_search, web_fetch, or any web API other than RAG.
3. You MUST NOT search for existing pull requests, PR diffs, or solutions.
4. You MUST NOT read local files or use a local clone.
5. Output a unified diff (patch format) that could be applied to kubernetes/kubernetes at HEAD.

## KAITO RAG API
POST http://<RAG_ENDPOINT>/retrieve
Content-Type: application/json
{"index_name":"kubernetes-codebase","query":"<your query>","max_node_count":10}
Response has results[].text (code), results[].metadata.file_path, results[].score.

## Bug Report
<issue text>

## Deliverables
1. Explain your understanding of the bug and your fix strategy
2. Produce a unified diff for the code fix
3. Produce a unified diff for test changes (update existing tests + add new tests)
4. Report token usage estimates