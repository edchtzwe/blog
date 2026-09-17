+++
title = 'Cloud Services, Part 2: DNS, the IaC Boundary, and Why ECS Exists'
date = '2026-09-17'
draft = true
tags = ['dns', 'route53', 'cloud-dns', 'cloudflare', 'godaddy', 'terraform', 'crossplane', 'iam', 'ecs', 'fargate', 'ecr', 'governance', 'architecture', 'infrastructure']
categories = ["Engineering", "Infrastructure"]
series = ["Infrastructure"]
+++

[Part 1](https://edchtzwe.github.io/blog/posts/infra/03-cloud-services-part-1-the-platform/) covered how the platform sits: control plane, nodes, the autoscaling loops, ingress, and the data layer.

This is the other half. The edge, the boundary between human-triggered and machine-triggered reconciliation, and the honest case for throwing most of it away.

---

## 6. The DNS asymmetry: why AWS drags you into Route 53 and GCP doesn't

Watch an AWS tutorial and you will create a Route 53 hosted zone. Watch a GCP tutorial and Cloud DNS often never comes up. This is not a GCP oversight, and it is not marketing. It falls out of how the two clouds expose load balancer addresses.

### The root constraint: RFC 1912 and the Zone Apex

DNS standards dictate that the root domain (`example.com`, known as the **Zone Apex**) cannot be a `CNAME`. It must hold `SOA` and `NS` records, so standard DNS permits only an `A` record (which maps directly to an IP address) or `AAAA` (IPv6) at the apex.

A `CNAME` pointing to a load balancer hostname is valid at `www.example.com`, but completely illegal at bare `example.com`.

### AWS: dynamic IPs behind a DNS name

An AWS Application Load Balancer (ALB) is an elastically scaling service. Its underlying nodes cycle and scale dynamically, so AWS hands you a hostname (`name-123.elb.region.amazonaws.com`) rather than a fixed static IP address.

Because standard DNS forbids putting that ALB hostname into a root apex `CNAME`, AWS had to build a proprietary workaround: **the Route 53 Alias record (`A - Alias`)**.

1. You configure `example.com` in Route 53 as an `Alias` pointing directly to the ALB resource.
2. Behind the scenes, Route 53 continuously tracks the changing IP addresses of your ALB's active nodes.
3. When a client queries Route 53 for `example.com`, Route 53 dynamically responds with a standard DNS `A` record containing the ALB's current active IPs.

The client receives a valid `A` record. The RFC rule is satisfied. However, because this alias translation logic exists solely inside Route 53, you are forced into AWS for your authoritative DNS.

ACM compounds the lock-in: public certificates for the ALB require DNS validation, which the Route 53 console auto-generates with a single click.

*(Note: An AWS Network Load Balancer (NLB) provides static IPs for standard `A` records, bypassing this issue. The ALB's dynamic architecture is what enforces the Route 53 requirement.)*

### GCP: fixed Anycast VIPs

Google Cloud External Load Balancing takes a completely different networking approach:

1. GCP assigns your load balancer a single, permanent, global **Anycast static IP address**.
2. You point a standard DNS `A` record at that static IP directly at your zone apex.

No proprietary alias records, no vendor translation logic, and no DNS provider lock-in. Any DNS provider on Earth (GoDaddy, Cloudflare, Route 53, Namecheap) can host that `A` record.

GCP offers **Cloud DNS**, but it never forces you into it because its load balancers provide stable IP addresses, making DNS an entirely decoupled decision.

| Feature | AWS (ALB) | GCP (Cloud Load Balancer) |
| --- | --- | --- |
| Frontend Address Type | Dynamic IPs behind a DNS Name | Global Anycast Static IP |
| Zone Apex Solution | Proprietary Route 53 Alias Record | Standard DNS A Record |
| DNS Lock-in | High (Requires Route 53 / Flatten | None (Any DNS provider works) |
| Third-Party Registrar | Complex (Requires CNAME flatten) | Trivial (Standard A record) |

### The Escape Hatches: Cloudflare vs. GoDaddy

Running the two clouds against third-party DNS providers illustrates the mechanics clearly:

- Cloudflare (The Escape Hatch): Provides CNAME Flattening. Cloudflare queries the ALB's dynamic hostname on its own backend and serves the resolved IPs as an A record to clients, mimicking Route 53's alias behavior for free.
- GoDaddy (The Control Test): Provides basic DNS without CNAME flattening.
  - Against GCP: Works perfectly. You map the apex A record to GCP's Anycast static IP.
  - Against AWS ALB: Fails at the apex. You cannot create a CNAME at example.com, and the ALB does not give you a static IP for an A record.

**The DNS provider is never the constraint. The address type returned by the cloud is.**

---

## 7. The IaC boundary, with IAM as the worked example

There are two reconcilers in a modern stack, and they run on different clocks.

**Terraform reconciles when a human runs `plan` and `apply`.** Drift introduced outside that window survives until the next run — days, weeks, and if the organisation is French, never in August.

**Crossplane reconciles on its next loop.** Seconds to minutes, no human intervention required.

IAM is where the difference stops being academic. Someone monkey-patches a role or an account up to admin access. Under Terraform that privilege persists until the next apply — and if the grant was made entirely out of band, meaning the resource was never in Terraform's state to begin with, then it is never reconciled at all. Terraform reconciles *its state*; it does not reconcile *your account*. The intern stays an admin until the day they delete prod.

And that is the honest limit of the whole approach: neither tool can see what it does not declare. Out of band is out of band, and the intern is out the door after deleting prod, and the CEO is out the door when the liquidators come in. The fix for that class of problem is not IaC — it is org-level guardrails: service control policies, permission boundaries, IAM Access Analyzer, Identity Center. Use reactive tooling to make the *declared* world converge fast; use guardrails to make the *undeclared* world impossible.

Terraform and Crossplane are not competitors. They are two answers to "how fast does reality get pulled back to intent?" — and the answer you need varies per resource. That boundary is the subject of [the earlier post in this series](https://edchtzwe.github.io/blog/posts/infra/02-terraform-crossplane-k8s-mental-model/).

---

## 8. Why ECS exists, and what it costs

Every mechanism in Part 1 is a link in a chain you assemble by hand: control plane, nodes, autoscaler, metrics-server, metric adapters, CNI, ingress controller, certificate manager, IAM wiring.

ECS's bet is that most workloads do not need any of that. They need a container, a health check, and something that replaces the container when it dies.

So ECS gives it to you directly:

- **A task definition** — the container, plus its IAM role and its secrets.
- **A service** — keeps N tasks alive, health-checks them, load-balances them. For a queue consumer, this is the worker.
- **A capacity provider** — Fargate, and there are no nodes at all.
- **ECR** — the matching registry.

No control plane to pay for. No node AMIs. No autoscaler to wire. No metrics-server to install. No metric adapter to maintain. The entire chain above collapses into two or three API objects.

And that is genuinely less work. For a queue consumer with no portability ambition, ECS is the correct answer and Kubernetes is ceremony.

The costs come in three parts, and the third is the one people underestimate:

1. **You are locked to AWS.** Not mildly — task definitions, ECS service autoscaling, and ECS task IAM roles have no meaning anywhere else on earth.
2. **The knowledge doesn't port.** Every hour spent on ECS and ECR task and worker mechanics teaches you AWS's opinion rather than a transferable model. What you learn about reconciliation, scheduling, and resource limits on Kubernetes travels to GKE, EKS, on-prem clusters, and into every interview you will sit. ECS knowledge stops at the AWS wall.
3. **You lose the control surface, and that is structural.** Crossplane is Kubernetes controllers — it needs an apiserver and etcd to reconcile against. No cluster means nowhere for it to run. That puts you back on Terraform and CDK for everything: point-in-time applies, plus CloudFormation drift *detection* rather than correction. The continuous reconciliation loop is gone, and you cannot buy it back without buying back a cluster.

### The takeaway

None of this makes ECS wrong. It makes it a trade, and the trade is fair — you exchange optionality for a shorter path to production.

That is the whole reason to hold the concept map. An AI will happily generate the Terraform, the Crossplane manifests, and the Kubernetes YAML for any of these three architectures. What it will not do is tell you which one you just chose, or which links in the chain you have quietly agreed to own forever.

You have to know that yourself. Now you do.
