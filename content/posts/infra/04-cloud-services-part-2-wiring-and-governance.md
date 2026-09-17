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

### AWS: the load balancer gives you a name, not an address

An ALB does not have a static IP. It is a managed, elastic load balancer, so AWS hands you back a DNS name — `name-id.elb.region.amazonaws.com` — whose underlying addresses change over time. AWS's own guidance is that you must never point an A record at a load balancer's IPs.

Now the apex problem. `example.com` cannot be a CNAME: DNS forbids a CNAME at the zone apex, because the apex has to carry the `SOA` and `NS` records. So a CNAME to the ALB is legal at `www.example.com` and illegal at `example.com`.

AWS's answer is the **alias record** — a Route 53 *proprietary* extension to DNS. It is a record type that resolves to an AWS resource rather than to an IP, and it is legal at the apex. It exists only inside Route 53.

So: if you want your bare domain served by an ALB, the alias record is the mechanism, and Route 53 is the only place it exists. Then ACM compounds it — a public certificate for the ALB needs DNS validation, which means CNAME records in the zone, which the Route 53 console will just create for you. Two proprietary conveniences, and you are now structurally committed.

Note the tell: AWS's own blog on solving apex challenges with third-party DNS points out that **NLB provides static IPs for use with A records**, while ALB does not. The exception proves the rule.

### GCP: the load balancer gives you an address

A Google Cloud external load balancer is fronted by a **reserved static IP**. You allocate the address, you point an A record at it. No alias type, no proprietary extension, no provider lock-in — any DNS server on the planet can serve that A record. GKE Ingress and Gateway API land you on the same global anycast VIP: a stable address, so a plain A record works at the apex.

**So the answer to "why does AWS have Route 53 but GCP doesn't" is:** GCP does have the equivalent — Cloud DNS, which is a perfectly good product and cheaper per zone. What GCP doesn't have is a *reason to force you into it*. Its load balancers expose stable addresses, so DNS stays a boring A record and your choice of DNS provider never becomes an architectural decision. AWS's load balancers expose names, so you need the one record type that only Route 53 offers.

It is not that one cloud is better at DNS. It is that **the kind of address a cloud hands you determines how much DNS you are forced to care about.**

### CloudFlare: the escape hatch

CNAME flattening at the apex solves precisely the problem AWS created. CloudFlare will serve an apex record that behaves like an alias, for both clouds, on the free tier for most use cases.

Which is why the most common production shape looks nothing like the tutorials: registrar wherever, CloudFlare doing DNS, Route 53 optional or absent entirely.

### GoDaddy: the control test

GoDaddy's DNS has no apex flattening. Run the same domain through both clouds:

- **Against GCP's static IP** — an A record at the apex. Works fine. GoDaddy is completely adequate.
- **Against an ALB** — no alias type, no flattening, and CNAMEs are illegal at the apex. You can serve `www` and you cannot serve the bare domain. GoDaddy cannot do it, through no fault of its own.

Same DNS product, same user, two different outcomes — purely because the two clouds hand over different kinds of address. That is the entire lesson in one test: **the DNS provider is never the constraint. The address type is.**

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
