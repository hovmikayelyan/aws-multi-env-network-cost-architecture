# Multi-environment cloud network architecture & cost optimization (AWS)

A design proposal for a microservice platform (Kubernetes + VMs) running in dev, staging and prod, with managed
databases and messaging. It covers how to separate and connect environments, keep traffic private, balance load,
observe the network, and keep the bill under control.

## Contents
| File | Description |
|---|---|
| [`proposal.md`](proposal.md) | The full proposal: diagram, written design, cost levers, trade-offs and an AWS/Azure/GCP service mapping |
| [`aws_microservice_architecture_official_aws_icons.drawio`](aws_microservice_architecture_official_aws_icons.drawio) | Diagram with official AWS icons; open in [draw.io](https://app.diagrams.net) |
| [`architecture.mmd`](architecture.mmd) | Diagram source (Mermaid). Renders on GitHub; paste into [mermaid.live](https://mermaid.live) or draw.io to export PNG/SVG |

## Design at a glance
- **Environments:** one AWS account per environment plus Network, Shared Services, Log Archive and Audit accounts under AWS Organizations, with SCP guardrails.
- **Connectivity:** hub-and-spoke on Transit Gateway with separate prod / non-prod route tables and centralized, inspected egress.
- **Private access:** gateway and interface (PrivateLink) endpoints for AWS services, PrivateLink endpoint services for cross-account sharing, no public data tier.
- **Load balancing:** CloudFront + WAF -> public ALB (L7) -> EKS pods; internal ALB for service-to-service; internal NLB (L4) for VM services and PrivateLink.
- **Telemetry:** VPC and TGW Flow Logs to Athena, LB logs, Reachability Analyzer, Network Access Analyzer, Network Monitor, OpenTelemetry tracing.
- **Cost:** tagging and CUR 2.0 -> Athena/QuickSight, Budgets and anomaly detection, Savings Plans on the baseline, Spot, Graviton, off-hours shutdown of non-prod, endpoint-vs-NAT decisions, AZ-local traffic.

Cost figures in the proposal are approximate list prices; check the AWS Pricing Calculator for your region.
