You are a Kubernetes developer. Given the bug report below, produce a unified diff 
that fixes the issue. You have access to the KAITO RAG API AND a local clone of 
kubernetes/kubernetes at /home/vibe/kubernetes.

## MANDATORY CONSTRAINTS
1. You MUST start with KAITO RAG queries. Make at least 3 RAG queries BEFORE reading 
   any local files.
2. After RAG discovery, you may read local files to get exact line numbers, full 
   function bodies, and test infrastructure.
3. You MUST NOT search for existing pull requests, PR diffs, or solutions.
4. You MUST NOT use web_search or web_fetch. Only RAG API and local file reads.
5. Output a unified diff (patch format) that could be applied to kubernetes/kubernetes at HEAD.

## KAITO RAG API
POST http://<RAG_ENDPOINT>/retrieve
Content-Type: application/json
{"index_name":"kubernetes-codebase","query":"<your query>","max_node_count":10}
Response has results[].text (code), results[].metadata.file_path, results[].score.

## Local Kubernetes Repo
Path: /home/vibe/kubernetes
Use read tool or shell commands (grep, cat, sed) to read files. Do NOT modify any files.

## Bug Report
<issue text>

## Deliverables
1. Explain your understanding of the bug and your fix strategy
2. Produce a unified diff for the code fix
3. Produce a unified diff for test changes (update existing tests + add new tests)
4. Report token usage estimates