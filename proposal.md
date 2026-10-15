# Cloud Architecture Proposal – multi-environment microservice platform (AWS)

**Scope.** Microservices on Kubernetes (EKS) and on VMs (EC2), in dev / staging / prod; internal service-to-service traffic,
external users, and managed services (RDS/DynamoDB, SQS/SNS). Chosen cloud: **AWS** (GCP/Azure equivalents in the last table).


## 1. Environment separation and interconnection
- **One AWS account per environment** (dev, staging, prod) plus dedicated *Network*, *Shared Services*, *Log Archive* and *Audit*
  accounts, organised into OUs under **AWS Organizations / Control Tower**. Account boundary = blast-radius and IAM boundary;
  **SCPs** enforce guardrails (deny leaving the org, deny public S3/RDS, restrict regions, require tags).
- **Non-overlapping CIDRs** (10.10/16 dev, 10.20/16 staging, 10.30/16 prod, 10.0/16 shared) so everything can be routed.
- **Hub-and-spoke with Transit Gateway** in the Network account, shared to the org with **AWS RAM**. Spokes attach their VPC.
  Separate TGW route tables: *prod* and *nonprod* are isolated (no route dev↔prod); both can reach *shared* (CI, registry, observability)
  and the **egress VPC** (NAT + Network Firewall) for centralized, inspected internet egress and DNS via Route 53 Resolver.
- Each VPC: 3 AZs, public subnets (ALB only), private app subnets (EKS nodes, EC2), isolated data subnets (RDS, no route out).

## 2. Private connectivity
- **Gateway endpoints** (S3, DynamoDB – free) and **interface endpoints / PrivateLink** (ECR api+dkr, SQS, SNS, Secrets Manager,
  CloudWatch Logs, STS, KMS) so managed-service traffic never touches the internet or NAT. Endpoint policies restrict to our org (`aws:PrincipalOrgID`).
- Databases and brokers have **no public endpoints**; security groups reference SG IDs (not CIDRs); EKS API endpoint private + restricted.
- Services shared across accounts/VPCs without full network peering are published as **PrivateLink endpoint services** behind an NLB
  (consumer sees only that service, overlapping CIDRs are not a problem).
- Pods reach AWS APIs via **IRSA / Pod Identity**; no static credentials. Egress to SaaS only through the central Network Firewall (domain allow-list).

## 3. Traffic routing and load balancing
| Path | Component | Why |
|---|---|---|
| Internet → apps | **CloudFront + WAF + Shield** → **public ALB (L7)** across 3 AZs | TLS termination, host/path routing, WAF, caching of static content |
| ALB → pods | **AWS Load Balancer Controller**, `target-type: ip`, readiness gates | Skips kube-proxy hop, graceful draining on rollout |
| Service ↔ service (K8s) | ClusterIP + CoreDNS; cross-cluster/VM calls via **internal ALB** (L7, gRPC/HTTP2 aware) | Path routing and per-service health checks |
| VM services / TCP / PrivateLink | **Internal NLB (L4)** | Static IPs, low latency, required for PrivateLink service |
| DNS | Route 53 public + private zones, **health-checked weighted/failover records** | Regional DR and blue/green cutover |

HA: every tier spans 3 AZs (cross-zone LB, EKS node groups per AZ, RDS Multi-AZ, one NAT per AZ in prod), PodDisruptionBudgets
and topology-spread constraints; deregistration delay + readiness gates for zero-downtime deploys.

## 4. Network telemetry and monitoring
- **VPC Flow Logs** (all VPCs) and **TGW Flow Logs** → central S3 in Log Archive, queried with **Athena** (top talkers, rejected flows, cross-AZ bytes).
  Prod: all traffic, 1-min; dev/staging: sampled/`REJECT`-only to limit cost.
- **ALB/NLB access logs** → S3; **CloudWatch** metrics (TargetResponseTime, 5xx, UnHealthyHostCount, TGW bytes/packet drops) with alarms.
- **Reachability Analyzer** and **Network Access Analyzer** in CI/for incidents to prove/disprove "A can reach B" and detect unintended paths (e.g. dev→prod).
- **CloudWatch Network Monitor / Internet Monitor** for synthetic latency & packet loss on key paths; **Route 53 health checks** for edge availability.
- **Traces and service metrics**: ADOT/OpenTelemetry collector → X-Ray + Prometheus/Grafana in Shared Services; Container Insights for EKS; VPC CNI metrics (IP exhaustion).
- **Deep debugging**: Traffic Mirroring to a packet-capture appliance, enabled on demand only.
- **Runbook triage order**: Route 53/CloudFront → ALB metrics → target health → SG/NACL/route (Reachability Analyzer) → flow logs `REJECT` → pod/app traces.
- Security signals: GuardDuty (flow-log/DNS anomalies) → Security Hub in the Audit account.

## 5. Cost optimization and spend monitoring
**Visibility (FinOps "inform")**
- Mandatory cost-allocation tags (`env`, `team`, `service`, `cost-center`) enforced by SCP / tag policies; account-per-environment already gives a clean split.
- **Cost and Usage Report 2.0** (Data Exports) → S3 → **Athena/QuickSight** dashboards (the AWS analogue of BigQuery billing export on GCP); EKS cost by namespace via split cost allocation / OpenCost.
- **AWS Budgets** per account & team with 50/80/100 % alerts, **Cost Anomaly Detection** to Slack, monthly unit-cost review (cost per 1k requests).

**Optimization (FinOps "optimize")**
| Lever | Applied to | Trade-off |
|---|---|---|
| **Compute Savings Plan** (1-yr, no upfront) sized to the *baseline* (~60–70 % of steady prod usage) | EKS nodes, EC2, Fargate | Up to ~30-40 % off; commit only to the floor, leave peaks on-demand; less flexible than on-demand |
| **Spot** via Karpenter (diverse instance types) | dev, staging, stateless prod workers | ~60–70 % cheaper; interruptions → needs PDBs/retries, not for stateful/DB |
| **Graviton (arm64)** | EKS, RDS, Lambda | ~20 % cheaper/perf-per-$; multi-arch images needed |
| Right-sizing (Compute Optimizer, VPA recommendations), bin-packing | all | Needs headroom policy to avoid throttling |
| **Schedule off-hours shutdown** of dev/staging | dev, staging | Saves ~65 % of those envs; slower morning start |
| **Gateway endpoints for S3/DynamoDB**; interface endpoints only where traffic > endpoint hourly cost | NAT data-processing ($0.045/GB) avoided | Interface endpoints cost ~$0.01/hr/AZ + $0.01/GB – worth it only above a few GB/day |
| **Centralized egress** (one NAT/firewall) vs per-VPC NAT | prod keeps NAT per AZ for resilience; dev/staging share/ single NAT | Central egress saves NAT hourly fees but adds TGW processing ($0.02/GB) and a shared failure point – decision depends on egress volume |
| Keep traffic **AZ-local** (topology-aware routing, zonal LB affinity) | K8s east-west | Cross-AZ ≈ $0.01/GB each way; reduces cost but can unbalance load |
| Flow-log sampling/retention, S3 lifecycle (IA → Glacier), log-level discipline | observability | Less forensic detail in non-prod |
| CloudFront caching, compression | edge | Cuts ALB/origin data transfer and compute |

Prices above are approximate us-east-1 list prices; verify with the AWS Pricing Calculator for the target region before committing.
**Governance loop:** monthly review of Savings Plan utilization/coverage (target ≥ 95 % / 70–80 %), idle resources (unattached EIPs/EBS, idle NAT/LBs), anomaly findings.

## Design trade-offs (performance ↔ complexity ↔ cost)
- **TGW hub-and-spoke vs VPC peering:** peering is cheaper and lower latency but is a mesh (N² links, non-transitive); TGW costs per attachment + per GB but scales and centralizes policy. Chosen for 4+ VPCs and auditability.
- **Account-per-env vs shared account:** more setup/overhead, but strongest isolation and clean cost attribution.
- **ALB vs NLB:** ALB for L7 features; NLB for L4/PrivateLink/static IP. Using both costs two LB fixed fees but each does what it is best at.
- **PrivateLink everywhere vs NAT:** better security/no egress charge but endpoint per-AZ fees; apply selectively by traffic volume.

## Multi-cloud mapping (for reference)
| AWS | Azure | GCP |
|---|---|---|
| Organizations + accounts | Management groups + subscriptions | Organization + folders/projects |
| Transit Gateway | Virtual WAN / hub VNet + peering | Shared VPC / NCC hub |
| PrivateLink | Private Link / Private Endpoint | Private Service Connect |
| ALB / NLB | Application Gateway / Load Balancer (+Front Door) | External/Internal Application LB / Network LB |
| VPC Flow Logs, Reachability Analyzer | NSG flow logs, Network Watcher | VPC Flow Logs, Connectivity Tests, Network Intelligence Center |
| CUR → Athena | Cost Management exports | Billing export → BigQuery |
| Savings Plans | Reservations / Savings Plans | Committed Use Discounts |
