# FinOps / Cost Reference

## Universal levers (all providers)

```text
1. Commitments   — steady-state savings: Savings Plans/RI (AWS), Reservations/
                   Savings Plans (Azure), CUDs (GCP). Buy after 2+ weeks of stable usage data.
2. Spot/Preemptible — stateless, fault-tolerant, batch: 60-90% off
3. Right-sizing   — Compute Optimizer / Azure Advisor / GCP active assist; act quarterly
4. Storage tiers  — S3 Intelligent-Tiering / Blob lifecycle / GCS lifecycle → archive
5. Scale-to-zero  — Lambda / Functions / Cloud Run for spiky workloads
6. Auto-shutdown  — dev/test schedules (Instance Scheduler / Azure Automation / Cloud Scheduler)
7. Orphan cleanup — unattached disks, IPs, NICs, LBs, snapshots, old AMIs/images
8. Tagging + budgets — cost allocation tags mandatory; budget alerts at 50/80/100%
9. Data transfer  — keep traffic in-region; watch cross-AZ/zone and egress fees
10. Network        — NAT/data-processing fees: VPC endpoints (AWS), Private Endpoints (Azure),
                    Private Google Access (GCP)
```

## Provider-specific quick hits

### AWS
```text
□ Compute Savings Plans (flexible) > EC2 RIs for mixed estate
□ S3 Intelligent-Tiering for unpredictable access
□ Lambda: arm64 + right memory (cost scales with memory×time)
□ EBS gp3 > gp2 (cheaper + better); delete old snapshots
□ CloudFront + Regional Data Transfer discounts
□ AWS Cost Explorer + Cost Anomaly Detection
```

### Azure
```text
□ Azure Hybrid Benefit (Windows Server / SQL on Azure)
□ Reserved instances for SQL/VM steady state (1y/3y)
□ Blob: Hot→Cool→Archive lifecycle; delete orphaned managed disks
□ Dev/Test subscriptions with auto-shutdown
□ Azure Advisor cost tab + Cost Management budgets
```

### GCP
```text
□ CUDs on Compute Engine, Cloud SQL, BigQuery slots
□ Sustained-use discounts (automatic, no action)
□ BigQuery: partition + cluster tables, careful with on-demand scans
□ Cloud Storage lifecycle → Archive; delete old snapshots
□ FinOps Hub + budget alerts per project
```

## Cost estimate format (deliverable)

```text
| Component | Service | SKU/Tier | Est. monthly | Notes |
|---|---|---|---|---|
| Compute   | ...     | ...      | $X           | ...   |
| Database  | ...     | ...      | $X           | ...   |
| Network   | egress/transfer estimate      | $X           | ...   |
| Observability | logs/metrics/APM estimates | $X           | ...   |
| TOTAL     |         |          | $X           | ±20%  |

+ Assumptions: traffic volume, storage growth, redundancy level
+ Optimization plan: commitments timeline, spot candidates, tiering
+ Cross-provider TCO table when comparing AWS vs Azure vs GCP
```

## MUST

- Tag everything before cost discussions start
- Budgets + alerts on day 1, not after the first surprise bill
- Re-check commitments quarterly against usage (commitments are not set-and-forget)
- Present cost with assumptions stated (traffic, growth, redundancy) — never a bare number

## MUST NOT

- Don't quote prices from memory — fetch current pricing (provider pricing pages/APIs) or state the estimate is directional
- Don't recommend 3-year commitments for a workload that's weeks old
- Don't ignore egress/transfer — it's the most commonly underestimated line
