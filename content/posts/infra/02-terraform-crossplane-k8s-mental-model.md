+++
title = 'Terraform x Crossplane x Kubernetes: The Mental Model for Modern Cloud Infrastructure'
date = '2026-09-13'
draft = false
tags = ['terraform', 'crossplane', 'kubernetes', 'aws', 'gcp', 'infrastructure', 'architecture']
categories = ["Engineering", "Infrastructure"]
series = ["Infrastructure"]
+++

Anyone can ask an AI model to vomit out a chunk of Terraform HCL or a 200-line Kubernetes Custom Resource YAML. Syntax has never been cheaper.

But here is the catch: **no amount of AI can design your infrastructure for you if you don’t understand how the moving parts fit together.** And if you don’t understand the underlying mental model, you won’t even realize when AI produces a fragile, drifted disaster until your production database gets wiped or an IAM role silently opens a back door.

Let’s skip the textbook definitions and walk through the ELI5 mental model of combining **Terraform**, **Crossplane**, and **Kubernetes** across AWS and GCP.

---

## The Baseline System: `vid_2_vec`

To keep this grounded, let's look at a concrete architecture based on our [`vid_2_vec`](https://github.com/edchtzwe/vid_2_vec/blob/master/infra/README.md) service:

- **1 API Gateway / Web Layer:** Go (Echo framework) serving REST endpoints.
- **4 Background Workers:** Asynchronous Go services processing video chunks and vector embeddings.
- **1 In-Memory Broker/Cache:** Managed Redis (AWS ElastiCache / GCP Memorystore).
- **1 Relational Database:** Managed PostgreSQL with pgvector (AWS RDS / GCP CloudSQL).
- **Compute:** Kubernetes (AWS EKS or GCP GKE).

When building infrastructure for this workload, you quickly discover that cloud resources fall into two completely distinct categories. Treating them the same way is where engineering teams get burned.

---

## Category 1: The Bedrock (Where Accidental Changes are Fireable Offenses)

Think of your foundation:
- VPCs, CIDR blocks, subnets, route tables, NAT gateways.
- KMS master encryption keys.
- Baseline IAM root permission boundaries.
- The Kubernetes cluster control plane itself (EKS / GKE).

These are foundational components. You provision them once, tune them rarely, and almost never want automated agents or developers constantly mutating them on the fly. 

If someone accidentally deletes a subnet or modifies a root KMS policy, production goes down and it's an all-hands-on-deck catastrophe.

### The Tool: Terraform

Terraform is built for this world. It is **pipeline-driven and point-in-time**:
1. You write your infrastructure as code in Git.
2. A pull request is reviewed by senior engineers.
3. Your CI/CD pipeline runs `terraform plan`, gets human sign-off, and runs `terraform apply`.
4. The Terraform process finishes and shuts down.

Because these bedrock resources change once a quarter, you *can* wait for the next git commit and CI/CD pipeline trigger to apply changes. A point-in-time tool is the right safety guardrail here.

---

## Category 2: The Living Ecosystem (High-Risk, Fast-Moving, Sensitive to Drift)

Now look at the resources tied closely to the application lifecycle:
- IAM role policy attachments (e.g. giving our Go workers access to specific S3 buckets or KMS keys via IRSA / Workload Identity).
- RDS / CloudSQL security group bindings and sensitive parameter groups.
- Application-specific queues, pub/sub topics, or object storage buckets.

In the real world, this is where things go wrong:
- An engineer jumps into the AWS/GCP console during a 2 AM incident to monkeypatch an IAM permission or security group rule.
- A manual configuration tweak causes silent **configuration drift**.
- The next time Terraform runs weeks later, it tries to destroy or revert things unexpectedly — or worse, the security hole stays open for months because nobody triggered a pipeline.

### The Tool: Crossplane

This is where **Crossplane** shines. 

Crossplane runs directly **inside your Kubernetes cluster** as a set of custom controllers. It turns your Kubernetes API into a universal cloud control plane.

Instead of running only when a CI pipeline is triggered, Crossplane runs an **active, continuous reconciliation loop**:
1. You declare an RDS instance or an IAM Role Policy attachment as a Kubernetes Custom Resource (CRD).
2. Crossplane constantly monitors the real cloud resource in AWS/GCP (every few seconds/minutes).
3. If someone goes into the AWS console at 2 AM and monkeypatches the IAM role or alters a database parameter, Crossplane spots the drift immediately and reconciles it back to the declared source of truth.

---

## The Mental Model: The Coffee Grinder vs. The Grind Setting

Let’s use an analogy every engineer who runs on caffeine will understand: **a commercial coffee grinder.**

- **Terraform is buying, wiring, and bolting the espresso machine and commercial grinder to the counter.** You buy the Mazzer grinder once. You plumb the water line. You wire 240V power. You don’t swap out your commercial grinder every morning, and accidentally ripping it off the counter shuts down the whole cafe.
- **Crossplane is the automated dial-lock and continuous calibration check.** The grind size setting (fine, coarse, dose timing) is touchy. During a frantic morning rush, an intern or tired barista tweaks the dial to fix a single sour shot and walks away. If left uncalibrated, every subsequent cup is undrinkable swill and customers riot. 

If you rely on **Terraform** to check the grind dial, the setting only gets verified once a month when someone explicitly runs a store maintenance checklist. You’ll serve thousands of bitter cups before anyone notices.

With **Crossplane**, an automated digital sensor checks the dial every 30 seconds. The second someone bumps the grind setting out of spec, the motor snaps the dial right back to standard.

| Dimension | Terraform | Crossplane |
| :--- | :--- | :--- |
| **Coffee Analogy** | **Buying and bolting down the grinder** (Permanent, heavy, rarely changed) | **Continuous grind-dial calibration** (Sensitive, prone to monkeypatches, needs instant correction) |
| **Execution Model** | Point-in-time (runs in CI/CD on trigger) | Continuous control loop (runs 24/7 in K8s) |
| **Reconciliation** | Reconciles only when someone runs `apply` | Reconciles actively and continuously |
| **Best Used For** | VPCs, Subnets, Routing, EKS/GKE Clusters, KMS | App IAM roles, S3 buckets, RDS configs, Redis |
| **Failure Mode** | Drift goes unnoticed until next pipeline run | Cluster failure impacts cloud resource management |

---

## How It Works in Practice (AWS & GCP)

Here is how the architecture cleanly separates for `vid_2_vec`:

```text
+-------------------------------------------------------------------+
| 1. TERRAFORM (CI/CD Pipeline Apply)                               |
|    - VPC, Subnets, Internet Gateways, NAT Gateways                |
|    - EKS / GKE Cluster Control Plane & Node Pools                 |
|    - Root KMS Keys & Base Networking Policies                     |
+---------------------------------+---------------------------------+
                                  |
                                  v
+---------------------------------+---------------------------------+
| 2. KUBERNETES CLUSTER (EKS / GKE)                                 |
|                                                                   |
|   +--------------------------+   +----------------------------+   |
|   | App Workload             |   | Crossplane Control Loop    |   |
|   | - Go Echo API (1 Pod)    |   | - provider-aws / gcp       |   |
|   | - Video Workers (4 Pods) |   | - Reconciles live state    |   |
|   +------------+-------------+   +--------------+-------------+   |
+----------------|--------------------------------|-----------------+
                 |                                |
                 | (Uses)                         | (Continuously Manages)
                 v                                v
+-------------------------------------------------------------------+
| 3. CLOUD MANAGED SERVICES (AWS / GCP)                             |
|    - RDS Postgres / CloudSQL (with pgvector)                      |
|    - Redis (ElastiCache / Memorystore)                            |
|    - App-specific IAM Policies & Object Storage Buckets           |
+-------------------------------------------------------------------+
```

1. **Bootstrap Phase (Terraform):**
   Terraform provisions the network foundation, security baselines, and the Kubernetes cluster. Once the cluster is up, Terraform steps back.
2. **Operational Phase (Crossplane):**
   Inside the cluster, Crossplane providers for AWS/GCP are installed. When `vid_2_vec` needs a database or an IAM role for video ingestion, it is declared alongside the application manifests. Crossplane provisions the RDS/CloudSQL database and continuously guards against policy drift.

---

## Why Understanding This Matters (The Anti-AI Defense)

AI code assistants are great at generating syntax:
- Ask AI for a Terraform VPC module, and it will output valid HCL.
- Ask AI for a Crossplane Managed Resource YAML, and it will give you valid schema.

What AI **cannot** do is tell you where the boundary of responsibility belongs. If you let an automated continuous controller manage your foundational VPC routing, a bad sync could take down your entire network. If you rely on point-in-time Terraform for sensitive IAM role bindings, a rogue console edit could sit undetected for months.

When you master the architectural mental model, you stop viewing tools as competing factions. You use Terraform to lay the concrete foundation, and Crossplane inside Kubernetes to govern the living, evolving system.

---

That boundary is only meaningful on top of the platform it governs. [Cloud Services, Part 1](https://edchtzwe.github.io/blog/posts/infra/03-cloud-services-part-1-the-platform/) covers how the pieces actually sit — control plane, nodes, the autoscaling loops, ingress, and the data layer — and [Part 2](https://edchtzwe.github.io/blog/posts/infra/04-cloud-services-part-2-wiring-and-governance/) covers DNS, the reconciliation boundary, and why ECS exists.
