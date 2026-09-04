# AI/ML Platform Architecture Reference

## Full-spectrum decision map (train → serve → operate)

| Stage | AWS | Azure | GCP |
|-------|-----|-------|-----|
| Data prep | S3 + Glue + EMR | Blob ADLS + Synapse/Databricks | GCS + Dataflow/Dataproc |
| Feature store | SageMaker Feature Store | Databricks Feature Store / Fabric | Vertex AI Feature Store |
| Model training | SageMaker | Azure ML / Databricks | Vertex AI (custom + AutoML) |
| Managed LLM APIs | Bedrock | Azure OpenAI / Foundry | Vertex AI (Gemini) |
| Fine-tuning | SageMaker + Bedrock custom models | Azure OpenAI fine-tuning / Foundry | Vertex AI tuning |
| Model serving | SageMaker endpoints | Azure ML endpoints / Foundry | Vertex AI endpoints |
| GPU infra | EC2 P/G (or EKS + Karpenter) | NC-series VMs / AKS | GKE + A3/H100 or Compute Engine |
| Vector store | OpenSearch / pgvector (RDS) | AI Search / Cosmos vector | Vertex AI Vector Search / AlloyDB |
| Orchestration | Step Functions / MWAA | ADF / Azure ML pipelines | Vertex Pipelines / Composer |
| MLOps registry | SageMaker Registry | Azure ML Registry | Vertex Model Registry |
| Observability | CloudWatch + SageMaker Monitor | Azure ML Monitor + App Insights | Vertex Model Monitoring |
| Guardrails/evals | Bedrock Guardrails | Azure AI Content Safety | Vertex Responsible AI |

## Architecture patterns

### Pattern 1 — RAG application (the most common build)

```text
Ingest:  docs → object store → (chunk) → embedding model → vector store
Query:   user → app → embedding → vector search (top-k) → prompt assembly
         → LLM (Bedrock/Azure OpenAI/Vertex) → grounded answer → user
Cache:   Redis/Memorystore for hot queries; CDN for static
Guardrails: Bedrock Guardrails / Azure AI Content Safety / Vertex Responsible AI
Eval:    golden-set eval every model/prompt change (promptfoo / RAGAS / vendor evals)
```

Terraform-first: everything except model training jobs is IaC (stores, endpoints, queues, monitors).

### Pattern 2 — MLOps pipeline (training at scale)

```text
Trigger (code/data drift) → pipeline orchestrator
  → data validation (Great Expectations/TFDV)
  → train (GPU job on EKS/ML compute/Vertex)
  → evaluate (metrics vs champion; fail below threshold)
  → register model (registry) → approval → staged rollout (canary/shadow)
  → monitor (drift + quality) → retrain loop
```

### Pattern 3 — Agentic AI serving

```text
Agent runtime (Bedrock Agents / Azure AI Agent / Vertex AI Agent Builder)
+ tool layer (Lambda/Functions/Cloud Run) + memory (DynamoDB/Cosmos/Firestore)
+ gateway (API GW/APIM/Apigee) with rate limits + token budgets
```

## Cost control (AI-specific — this is where bills explode)

```text
□ Token budgets + rate limits at the gateway
□ Cache embeddings and LLM responses (Redis) — biggest single saving
□ Rightsize models (Haiku/Flash/4o-mini for classification; frontier only when needed)
□ Provisioned throughput only for steady production load
□ GPU: spot/preemptible for training; committed use for inference fleets
□ Log/telemetry cost caps — AI apps generate huge traces
```

## Security (AI-specific)

```text
□ Prompt injection: treat model output as untrusted; never execute model output directly
□ PII: redact/anonymize before training or inference logging
□ Model access: Bedrock endpoints/IAM, Azure OpenAI RBAC + content filtering,
  Vertex IAM + VPC Service Controls
□ Data exfil: keep training data + vector stores inside the cloud boundary
□ Eval + guardrail before production, and continuously after
```

## MUST

- Eval harness with a golden set before ANY production LLM deployment
- Version every model + prompt; registry entries per deployment
- Monitor drift and answer quality, not just uptime
- Design for model swap (abstraction layer) — frontier models churn fast

## MUST NOT

- Don't put raw secrets/PII into prompts or training data
- Don't trust model output for authz decisions
- Don't provision GPUs 24/7 for a once-a-night batch job
- Don't deploy without content safety + rate limits
