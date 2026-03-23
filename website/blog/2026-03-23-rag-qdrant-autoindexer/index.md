---
title: "Build a Kubernetes Knowledge Base on AKS with KAITO RAGEngine, Qdrant, and AutoIndexer"
date: 2026-03-23
description: "Learn how to setup KAITO RAGEngine + AutoIndexer to add relevant context to your AI workloads."
authors:
- brandon-foley
tags: ["kaito", "rag", "qdrant", "ai", "open-source"]
---
# Build a Kubernetes Knowledge Base on AKS with KAITO RAGEngine, Qdrant, and AutoIndexer

## Overview

In this blog, we'll walk you through building an end-to-end Retrieval-Augmented Generation (RAG) knowledge base on AKS using three powerful open-source components: [KAITO RAGEngine](https://kaito-project.github.io/kaito/docs/rag), [Qdrant](https://qdrant.tech/) vector database, and [KAITO AutoIndexer](https://github.com/kaito-project/autoindexer). By the end, you'll have a system that automatically ingests the entire Kubernetes English documentation from GitHub, stores it as searchable vector embeddings in Qdrant, and exposes a `/retrieve` API you can use to power intelligent, context-aware AI applications.

## Introduction

Modern AI applications increasingly rely on Retrieval-Augmented Generation (RAG) to ground large language models (LLMs) in up to date data. Rather than depending solely on what a model memorized during training, RAG dynamically retrieves relevant documents and feeds them as context to the LLM, producing answers that are accurate, current, and grounded in your own content.

But building a production RAG pipeline involves orchestrating several moving parts:

| RAG Component | Purpose | Example |
| --- | --- | --- |
| **Vector Store** | Stores text as vector embeddings — numerical representations of meaning — enabling semantic search that goes beyond keyword matching | [Qdrant](https://qdrant.tech/), [FAISS](https://faiss.ai/) |
| **Embedding Model** | Converts text into dense vectors that capture semantic meaning, so similar concepts produce similar vectors even when the words differ | [BAAI/bge-small-en-v1.5](https://huggingface.co/BAAI/bge-small-en-v1.5) |
| **Retriever + LLM** | The retriever finds the most relevant documents from the vector store; the LLM uses those documents as context to generate accurate, grounded responses | [LlamaIndex](https://docs.llamaindex.ai/) |
| **Data Ingestion** | Automates pulling documents from source repositories, processing them, and indexing them into the vector store — keeping your knowledge base fresh | [KAITO AutoIndexer](https://github.com/kaito-project/autoindexer) |

Setting all of this up manually is complex and prone to error. That's where the [Kubernetes AI Toolchain Operator](https://kaito-project.github.io/kaito/docs/) (KAITO) comes in.

KAITO is a [CNCF Sandbox project](https://www.cncf.io/projects/kaito/) that makes it easy to deploy, serve, and scale AI workloads on Kubernetes without becoming a DevOps expert. Its **RAGEngine** Custom Resource Definition (CRD) gives you an end-to-end RAG pipeline out of the box, and its companion **AutoIndexer** operator automates the entire data ingestion lifecycle. Together, they let you focus on building AI applications while Kubernetes handles the infrastructure.

In this blog, we'll use a practical example: **indexing the official Kubernetes English documentation** from the [kubernetes/website](https://github.com/kubernetes/website) GitHub repository, storing the embeddings in a production-grade Qdrant vector database, and querying the knowledge base using the RAGEngine `/retrieve` API.

> **Note:** KAITO RAGEngine, AutoIndexer, and Qdrant are deployed as open-source solutions in this example and are not currently managed services on AKS.

## What Are the Key Components?

Before we start deploying, let's understand what each component brings to the table.

### KAITO RAGEngine

[KAITO RAGEngine](https://kaito-project.github.io/kaito/docs/rag) is a Kubernetes CRD that provisions a complete RAG pipeline with a single YAML manifest. Under the hood, it:

- Configures a **vector store** (FAISS by default, or Qdrant as we'll use here) for storing document embeddings
- Runs a **local embedding model** (default: [BAAI/bge-small-en-v1.5](https://huggingface.co/BAAI/bge-small-en-v1.5)) to convert documents into vectors
- Integrates with [LlamaIndex](https://github.com/run-llama/llama_index) for LLM-based document retrieval
- Exposes REST APIs for indexing documents and retrieving relevant context

The key RAGEngine APIs include:
- **`POST /index`** — Index documents into a named index
- **`GET /indexes/{index_name}/documents`** — List indexed documents
- **`POST /retrieve`** — Retrieve relevant document chunks using hybrid search (dense + sparse vectors)

The RAGEngine can run on **general-purpose CPU compute** (e.g., Azure `Standard_D8_v3`) for development and lighter workloads, but for production scenarios with large-scale document ingestion or low-latency embedding requirements, you may want to leverage **GPU-accelerated nodes** (e.g., `Standard_NV36ads_A10_v5`) to significantly speed up the embedding model inference.

### Qdrant Vector Database

[Qdrant](https://qdrant.tech/) is a high-performance, open-source vector database purpose-built for similarity search and AI applications. While KAITO RAGEngine supports FAISS as a default in-memory vector store, Qdrant brings several production advantages:

- **Persistent storage** — Your vector embeddings survive pod restarts and cluster upgrades
- **Hybrid search** — Combines dense (semantic) and sparse (keyword) vector search for more accurate retrieval
- **Scalability** — Handles millions of vectors with efficient indexing and filtering
- **Rich metadata filtering** — Filter search results by metadata fields (source, file path, timestamp, etc.)
- **Production-ready** — Built-in health checks, REST and gRPC APIs, and battle-tested in production workloads

By pointing KAITO RAGEngine at an external Qdrant instance, you get a durable, high-performance vector store that persists independently from the RAG service itself.

### KAITO AutoIndexer

[KAITO AutoIndexer](https://github.com/kaito-project/autoindexer) is a Kubernetes operator that automates document ingestion into KAITO RAGEngine. Instead of manually curling documents into the `/index` API, AutoIndexer handles the complete lifecycle:

- **Automated data ingestion** from multiple source types: Git repositories, static URLs, and databases
- **Scheduled indexing** with cron-based scheduling (e.g., re-index every 6 hours)
- **Incremental indexing** — Only processes new or changed content based on commit history
- **Drift detection and remediation** — Detects when documents in the index drift from the source and can auto-remediate
- **Multi-format support** — Handles text, Markdown, code files, and PDFs
- **Batch processing** — Efficiently indexes large repositories in configurable batches

The AutoIndexer follows the standard Kubernetes CRD/controller pattern. You define an `AutoIndexer` custom resource specifying your data source, target RAGEngine, and index name — the controller does the rest.

## Architecture Overview

Here's how the components work together:

```
┌──────────────────────────────────────────────────────────────────────┐
│                          AKS Cluster                                 │
│                                                                      │
│  ┌──────────────────┐    Indexes docs    ┌────────────────────────┐  │
│  │  AutoIndexer     │ ──────────────────▶│  KAITO RAGEngine       │  │
│  │  Controller      │   POST /index      │                        │  │
│  │                  │                    │  • Embedding Model     │  │
│  │  Watches:        │                    │    (bge-small-en-v1.5) │  │
│  │  • Git repos     │                    │  • LlamaIndex          │  │
│  │  • Static URLs   │                    │  • REST API Server     │  │
│  │  • Databases     │                    │                        │  │
│  └──────────────────┘                    └──────────┬─────────────┘  │
│                                                     │                │
│         ┌───────────────────────────────────────────┘                │
│         │ Stores/retrieves embeddings                                │
│         ▼                                                            │
│  ┌──────────────────┐                                                │
│  │  Qdrant          │                                                │
│  │  Vector Database │                                                │
│  │                  │                                                │
│  │  • Hybrid search │                                                │
│  │  • Persistent    │                                                │
│  │    storage (PVC) │                                                │
│  └──────────────────┘                                                │
│                                                                      │
│         ┌─────────────────────────────────────────────┐              │
│         │              User / Application             │              │
│         │                                             │              │
│         │  curl POST /retrieve                        │              │
│         │    → Returns relevant document chunks       │              │
│         └─────────────────────────────────────────────┘              │
└──────────────────────────────────────────────────────────────────────┘
```

The flow is:
1. **AutoIndexer** clones the Kubernetes docs from GitHub, processes the Markdown files, and sends them to RAGEngine's `/index` API in batches
2. **RAGEngine** converts documents into vector embeddings using the local embedding model and stores them in **Qdrant**
3. **Users and applications** query the `/retrieve` endpoint to get relevant document chunks for LLM-generated answers grounded in the Kubernetes documentation

## Let's Get Started: Deploying the Full Stack on AKS

### Prerequisites

- An Azure subscription
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) installed
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installed
- [Helm](https://helm.sh/docs/intro/install/) v3 installed

### Step 1: Create an AKS Cluster

Create a resource group and an AKS cluster with node auto-provisioning enabled:

```bash
# Create a resource group
az group create --name myRAGResourceGroup --location eastus

# Create an AKS cluster
az aks create \
    --resource-group myRAGResourceGroup \
    --name myRAGCluster \
    --enable-addons monitoring \
    --generate-ssh-keys \
    --node-provisioning-mode Auto

# Get credentials
az aks get-credentials --resource-group myRAGResourceGroup --name myRAGCluster
```

### Step 2: Create Dedicated Node Pools

We'll create separate node pools for Qdrant and the RAGEngine to isolate workloads:

**Qdrant Node Pool:**

```bash
az aks nodepool add \
    --resource-group myRAGResourceGroup \
    --cluster-name myRAGCluster \
    --name qdrantpool \
    -s Standard_D16s_v5 \
    -c 1 \
    --labels workload=qdrant
```

**RAGEngine Node Pool (CPU-based):**

For most RAG workloads, CPU nodes are sufficient and cost-effective:

```bash
az aks nodepool add \
    --resource-group myRAGResourceGroup \
    --cluster-name myRAGCluster \
    --name ragcpupool \
    -s Standard_D8_v3 \
    -c 1 \
    --labels workload=ragengine
```

> **Tip:** For high-performance workloads requiring GPU-accelerated embedding, you can use a GPU SKU like `Standard_NV36ads_A10_v5` instead. The RAGEngine `instanceType` in your YAML would need to match accordingly.

### Step 3: Deploy Qdrant Vector Database

Clone the KAITO cookbook repository which contains the manifests:

```bash
git clone https://github.com/kaito-project/kaito-cookbook.git
cd kaito-cookbook/examples/qdrant-rag-autoindexer
```

Deploy Qdrant with persistent storage:

**1. Create a PersistentVolumeClaim for Qdrant storage:**

```yaml
# qdrant-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: qdrant-storage
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
  storageClassName: default
```

```bash
kubectl apply -f qdrant-pvc.yaml
```

**2. Deploy the Qdrant Deployment:**

```yaml
# qdrant.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: qdrant
  name: qdrant
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: qdrant
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: qdrant
    spec:
      containers:
      - image: qdrant/qdrant:v1.13.4
        imagePullPolicy: IfNotPresent
        livenessProbe:
          httpGet:
            path: /livez
            port: 6333
          initialDelaySeconds: 10
          periodSeconds: 30
        name: qdrant
        ports:
        - containerPort: 6333
          name: http
        - containerPort: 6334
          name: grpc
        readinessProbe:
          httpGet:
            path: /readyz
            port: 6333
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          limits:
            cpu: "14"
            memory: 56Gi
          requests:
            cpu: "8"
            memory: 32Gi
        volumeMounts:
        - mountPath: /qdrant/storage
          name: qdrant-data
      nodeSelector:
        workload: qdrant
      volumes:
      - name: qdrant-data
        persistentVolumeClaim:
          claimName: qdrant-storage
```

```bash
kubectl apply -f qdrant.yaml
```

**3. Create the Qdrant Service:**

```yaml
# qdrant-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: qdrant
  namespace: default
spec:
  ports:
  - name: http
    port: 6333
    protocol: TCP
    targetPort: 6333
  - name: grpc
    port: 6334
    protocol: TCP
    targetPort: 6334
  selector:
    app: qdrant
  type: ClusterIP
```

```bash
kubectl apply -f qdrant-service.yaml
```

Verify Qdrant is running:

```bash
kubectl get pods -l app=qdrant
```

### Step 4: Install KAITO Components

Install the KAITO Workspace operator and RAGEngine controller via Helm:

**KAITO Workspace:**

```bash
helm repo add kaito https://kaito-project.github.io/kaito/charts/kaito
helm repo update

helm upgrade --install kaito-workspace kaito/workspace \
  --namespace kaito-workspace \
  --create-namespace \
  --wait \
  --take-ownership
```

**KAITO RAGEngine:**

```bash
helm upgrade --install kaito-ragengine oci://ghcr.io/kaito-project/charts/ragengine \
  --version 0.9.3-qdrant.2 \
  --namespace kaito-ragengine \
  --create-namespace \
  --take-ownership
```

Verify both installations:

```bash
helm list -n kaito-workspace
helm list -n kaito-ragengine
```

### Step 5: Deploy the RAGEngine Custom Resource

Now deploy the RAGEngine, pointing it at Qdrant as the vector store:

```yaml
# ragengine.yaml
apiVersion: kaito.sh/v1beta1
kind: RAGEngine
metadata:
  name: ragengine
  namespace: default
spec:
  compute:
    count: 1
    instanceType: "Standard_D8_v3"
    labelSelector:
      matchLabels:
        workload: ragengine
  embedding:
    local:
      modelID: "BAAI/bge-small-en-v1.5"
  inferenceService:
    contextWindowSize: 4096
  storage:
    vectorDB:
      engine: qdrant
      url: http://qdrant.default.svc.cluster.local:6333
```

```bash
kubectl apply -f ragengine.yaml
```

Let's break down the key fields:

| Field | Purpose |
| --- | --- |
| `spec.compute` | Specifies the node SKU and label selector for scheduling the RAGEngine pod on our pre-created `ragcpupool` |
| `spec.embedding.local.modelID` | The embedding model used to convert documents into vectors. `BAAI/bge-small-en-v1.5` is a lightweight, high-quality model |
| `spec.inferenceService.contextWindowSize` | The context window size for the LLM — controls how much retrieved text can be sent as context |
| `spec.storage.vectorDB.engine` | Tells RAGEngine to use **Qdrant** instead of the default in-memory FAISS store |
| `spec.storage.vectorDB.url` | The in-cluster URL of our Qdrant service |

Verify the RAGEngine is ready:

```bash
kubectl get ragengines
kubectl describe ragengine ragengine
```

Wait for the RAGEngine pod to be running and the status conditions to show ready.

### Step 6: Install and Configure AutoIndexer

Install the AutoIndexer operator:

```bash
helm upgrade --install kaito-autoindexer oci://ghcr.io/kaito-project/charts/autoindexer \
  --version 0.0.0-dev.2 \
  --namespace kaito-autoindexer \
  --create-namespace \
  --take-ownership
```

Now, create an AutoIndexer custom resource that pulls the Kubernetes English documentation from the [kubernetes/website](https://github.com/kubernetes/website) GitHub repository:

```yaml
# k8s-docs-autoindexer.yaml
apiVersion: autoindexer.kaito.sh/v1alpha1
kind: AutoIndexer
metadata:
  name: k8s-docs-autoindexer
spec:
  dataSource:
    type: Git
    git:
      repository: https://github.com/kubernetes/website.git
      branch: main
      paths:
        - "content/en/docs/**/*.md"
      excludePaths:
        - "*.png"
        - "*.jpg"
        - "*.svg"
  indexName: k8s-docs
  ragEngine: ragengine
  driftRemediationPolicy:
    strategy: Manual
```

```bash
kubectl apply -f k8s-docs-autoindexer.yaml
```

Let's break down what this AutoIndexer does:

| Field | Purpose |
| --- | --- |
| `spec.dataSource.type: Git` | Tells AutoIndexer to clone a Git repository as the data source |
| `spec.dataSource.git.repository` | The Kubernetes website repo containing all official documentation |
| `spec.dataSource.git.paths` | Targets only Markdown files under `content/en/docs/` — the English documentation |
| `spec.dataSource.git.excludePaths` | Skips image files that can't be meaningfully embedded |
| `spec.indexName` | The name of the index in RAGEngine/Qdrant where documents will be stored |
| `spec.ragEngine` | References the RAGEngine CR we created in Step 5 |
| `spec.driftRemediationPolicy.strategy` | Set to `Manual` — if the source drifts from the index, we'll decide when to re-index |

Verify the AutoIndexer is running:

```bash
# Check the AutoIndexer custom resource status
kubectl get autoindexers

# Watch the indexing job logs
kubectl logs -l autoindexer.kaito.sh/name=k8s-docs-autoindexer --follow
```

You should see log output showing the indexing progress:

```
"body": "Initialized git data source handler for repository: https://github.com/kubernetes/website.git"
"body": "AutoIndexer initialized for index 'k8s-docs' with data source type 'Git'"
"body": "Starting document indexing process"
"body": "Cloning repository from https://github.com/kubernetes/website.git"
"body": "Found 850 files in repository for indexing"
"body": "Indexing batch of 10 documents (10/850 files processed)"
"body": "HTTP Request: POST http://ragengine.default.svc.cluster.local/index \"HTTP/1.1 200 OK\""
...
"body": "Indexing completed successfully"
"body": "AutoIndexer job completed successfully"
```

The AutoIndexer will clone the repository, discover all matching Markdown files, process them into chunks, and send them in batches to the RAGEngine `/index` API — which in turn generates embeddings and stores them in Qdrant.

## Leveraging the RAGEngine: Querying Your Knowledge Base

Once the AutoIndexer has completed indexing, the exciting part begins, querying your Kubernetes documentation knowledge base. To make the RAGEngine accessible outside the cluster, we'll expose it via a Kubernetes `LoadBalancer` Service that provisions a public IP.

### Expose the RAGEngine with a Public IP

Create a `LoadBalancer` Service that fronts the RAGEngine:

```yaml
# ragengine-public-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: ragengine-public
  namespace: default
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 5000
  selector:
    app: ragengine
```

```bash
kubectl apply -f ragengine-public-service.yaml
```

Wait for Azure to provision a public IP and retrieve it:

```bash
kubectl get svc ragengine-public --watch
```

Once the `EXTERNAL-IP` column transitions from `<pending>` to an IP address, note it down:

```
NAME                TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
ragengine-public    LoadBalancer   10.0.45.123   20.XX.XX.XX     80:31234/TCP   2m
```

Store it in an environment variable for convenience:

```bash
export RAGENGINE_IP=$(kubectl get svc ragengine-public -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "RAGEngine endpoint: http://$RAGENGINE_IP"
```

> **⚠️ Security Note:** This creates a publicly accessible endpoint with no authentication. For production workloads, consider adding TLS termination, or restrict access with a [Network Security Group (NSG)](https://learn.microsoft.com/en-us/azure/aks/concepts-security#azure-network-security-groups) to limit traffic to known IP ranges. You can also use `loadBalancerSourceRanges` in the Service spec to allowlist specific CIDRs:
> ```yaml
> spec:
>   loadBalancerSourceRanges:
>   - "203.0.113.0/24"  # Your office/VPN CIDR
> ```

### Using the `/retrieve` API

The `/retrieve` endpoint is the core of the RAGEngine's search capability. It performs **hybrid search** — combining dense (semantic) and sparse (keyword) vector matching — to return the most relevant document chunks for your query.

**Basic Retrieve Request:**

```bash
curl -X POST http://$RAGENGINE_IP/retrieve \
     -H "Content-Type: application/json" \
     -d '{
       "index_name": "k8s-docs",
       "query": "How do I configure a liveness probe in Kubernetes?",
       "max_node_count": 5
     }'
```

**Example Response:**

```json
{
  "query": "How do I configure a liveness probe in Kubernetes?",
  "results": [
    {
      "doc_id": "a1b2c3d4...",
      "node_id": "e5f6a7b8...",
      "text": "## Define a liveness command\n\nMany applications running for long periods of time eventually transition to broken states, and cannot recover except by being restarted. Kubernetes provides liveness probes to detect and remedy such situations...",
      "score": 0.87,
      "dense_score": 0.92,
      "sparse_score": 0.78,
      "source": "hybrid",
      "metadata": {
        "autoindexer": "default_k8s-docs-autoindexer",
        "source_type": "git",
        "repository": "https://github.com/kubernetes/website.git",
        "branch": "main",
        "file_path": "content/en/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes.md",
        "change_type": "full",
        "timestamp": "2026-03-23T10:15:30.000Z"
      }
    },
    ...
  ],
  "count": 5
}
```

Notice the rich metadata in each result — you can see the exact file path in the Kubernetes docs repo, the Git commit, and the relevance scores from both dense and sparse search. The hybrid approach means you get results that match both the semantic meaning of your question *and* keyword relevance.

**Retrieve Request Parameters:**

| Parameter | Description |
| --- | --- |
| `index_name` | The name of the index to search (matches the AutoIndexer's `indexName`) |
| `query` | The natural language question or search query |
| `max_node_count` | Maximum number of document chunks to return |

### Programmatic Access with the Python Client

For application developers, KAITO provides the [`kaito-rag-engine-client`](https://pypi.org/project/kaito-rag-engine-client/) Python library for programmatic access. Since the RAGEngine is now exposed via a public IP, you can connect directly from any machine — no `kubectl port-forward` required:

```bash
pip install kaito-rag-engine-client
```

```python
from kaito_rag_engine_client import Client
from kaito_rag_engine_client.models import RetrieveRequest
from kaito_rag_engine_client.api.index import retrieve_index

# Point the client at the RAGEngine's public IP
RAGENGINE_URL = "http://20.XX.XX.XX"  # Replace with your EXTERNAL-IP

client = Client(base_url=RAGENGINE_URL)

# Retrieve relevant Kubernetes docs
retrieve_resp = retrieve_index.sync(
    client=client,
    body=RetrieveRequest.from_dict({
        "index_name": "k8s-docs",
        "query": "How do I set resource limits on a container?",
        "max_node_count": 5,
    })
)

# Use the results in your application
for result in retrieve_resp.results:
    print(f"Score: {result.score}")
    print(f"Source: {result.metadata.get('file_path', 'unknown')}")
    print(f"Text: {result.text[:200]}...")
    print("---")
```

Because the endpoint is publicly reachable, this same code works from local development machines, CI/CD pipelines, serverless functions, or any other service — making it straightforward to integrate RAGEngine context retrieval into chatbots, developer tools, documentation search engines, or any application that benefits from grounded AI responses.

## What We've Built

Let's recap the full system:

1. **Qdrant** runs as a persistent, high-performance vector database on a dedicated AKS node pool, storing all document embeddings with full durability
2. **KAITO RAGEngine** provides the embedding pipeline and REST API layer — converting documents to vectors, storing them in Qdrant, and serving hybrid search queries
3. **KAITO AutoIndexer** continuously monitors the Kubernetes documentation GitHub repo, automatically ingesting new and updated content into the RAGEngine

The result is a **self-updating Kubernetes documentation knowledge base** that anyone in your organization can query using natural language. No manual document processing, no custom ingestion scripts, no vector database tuning — just declarative Kubernetes resources.

## Next Steps

Now that you have a working RAG knowledge base on AKS, here's how to extend it:

- **Add more data sources** — Create additional AutoIndexer resources to index your internal documentation, runbooks, or code repositories alongside the Kubernetes docs
- **Configure scheduled re-indexing** — Add a `schedule` field to your AutoIndexer (e.g., `"0 */6 * * *"` for every 6 hours) to keep your knowledge base fresh automatically
- **Enable drift detection** — Switch your `driftRemediationPolicy.strategy` to `Auto` to automatically reconcile when the source drifts from the index

## Resources

- [KAITO RAGEngine Documentation](https://kaito-project.github.io/kaito/docs/rag)
- [KAITO AutoIndexer Repository](https://github.com/kaito-project/autoindexer)
- [KAITO Project](https://github.com/kaito-project/kaito)
- [Qdrant Documentation](https://qdrant.tech/documentation/)
- [`kaito-rag-engine-client` Python Package](https://pypi.org/project/kaito-rag-engine-client/)