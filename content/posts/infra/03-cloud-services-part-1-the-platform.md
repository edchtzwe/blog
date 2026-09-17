+++
title = 'Cloud Services, Part 1: How the Platform Actually Sits'
date = '2026-09-17'
draft = false
tags = ['kubernetes', 'eks', 'gke', 'autoscaling', 'hpa', 'karpenter', 'alb', 'ingress', 'rds', 'cloudsql', 'iam', 'irsa', 'architecture', 'infrastructure']
categories = ["Engineering", "Infrastructure"]
series = ["Infrastructure"]
+++

Concepts only. No console walkthroughs, no HCL.

The premise: you can get an AI to emit Terraform, Crossplane manifests, and Kubernetes YAML in seconds. What it cannot do is tell you whether the architecture it just emitted is the one you wanted. That judgement requires you to hold the model of how the pieces sit. So this is the model.

Part 2 covers the edge (DNS), the governance boundary (who reconciles what), and why ECS exists.

---

## 1. The layer cake

There are three planes, and every managed service you will meet is one of them or a resale of one.

- **The control plane** decides. It holds desired state and runs loops to move reality toward it.
- **The data plane** carries traffic. It is what your users actually touch.
- **The glue** is identity and network boundaries. It is what lets the other two talk without letting anything else talk.

When someone says a platform is "opinionated," they mean they made the layer-split decision for you. When someone says a platform is "flexible," they mean you get to make it. Everything below is a variation on that trade.

---

## 2. EKS vs GKE, honestly

The managed control plane on both clouds is: `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, and `etcd`. You never SSH into them, you never patch them, and there is nothing underneath for you to log into — the control plane does not run on your nodes. That is the thing you are paying for — and it costs the same headline number on both clouds, roughly $0.10 per cluster-hour (~$73/month). GKE softens it with a free allowance for one zonal cluster per billing account, which is why "EKS is the expensive one" is only half the story.

What you still own is **the nodes**. Three models, increasing order of delegation:

- **EKS Managed Node Groups** — AWS owns the Auto Scaling Group and the AMI lifecycle. You pick instance types. Someone still has to reason about bin-packing.
- **Karpenter** — you declare node *requirements* (architecture, capacity type, zones, taints), and Karpenter provisions instances directly from the EC2 fleet API. It replaces the cluster-autoscaler-plus-ASG pattern entirely, packs better and scales up in seconds. It also has a larger blast radius, because it makes fleet decisions without a human-designed group boundary.
- **GKE Autopilot** — Google owns the nodes. You are billed per pod, and you give up DaemonSets and privileged workloads. GKE Standard with Node Auto-Provisioning is the Karpenter analogue.

### Access: what you can and can't reach

This trips people up coming from plain EC2, and it is worth being blunt about, because it is the clearest line between "a cloud VM you own" and "a managed service".

**The control plane: no access, full stop.** On EKS it runs in an AWS-owned account; on GKE, a Google-owned project. There is no node to SSH into, no filesystem to inspect, and no keypair that would change that. The only doors are the Kubernetes API and the cloud APIs, both authenticated as you. If your instinct is "just get on the box and look", that instinct has nowhere to land here — deliberately. EKS and GKE do not gate control-plane access behind a permission you could be granted. They simply do not expose it.

**A node: possible, and an antipattern.** Nodes are EC2 instances (EKS) or Compute Engine VMs (GKE Standard) in your own account, so mechanically you can attach a keypair and SSH in. Don't. Adding a keypair to reach your nodes is an antipattern, and not only because a standing credential sitting outside your IAM model is a bad idea. It is worse than that: that key gives shell access to a box holding every pod's filesystem, network identity, and the node's own instance credentials — a lateral-movement path straight past your identity boundary. It also tempts you into treating nodes as pets, hand-fixing what the platform is supposed to replace. A node that needs you to log in is a node you should be replacing. Keep this strictly for emergencies, and treat every use as an incident rather than a tool.

Notice what this actually is: an EC2 or Compute Engine affordance, not a Kubernetes one. The platform never offered you node access. The VM underneath just happens not to stop you.

**A pod or container: also no.** There is no SSH daemon and no port to reach it on. A container is a process with namespaces, not a small VM, and treating it as one is how you end up with a container that only works because of something you did by hand.

**The shell you actually use — `kubectl exec`.** `kubectl exec -it <pod> -- sh` is the direct analogue of `docker exec -it <container> sh`. Same idea, same caveat: it is a debugging tool, not a management channel. Nothing you change inside a pod survives a restart, and every fix you make in there should come back out as a manifest change. If you are exec-ing in as a routine, that is a signal about your observability, not a workflow.

The thing to internalise: **"managed" never means you have delegated the design.** You still choose subnet topology, node families, taints, disruption budgets, and upgrade cadence. The cloud takes the patching, not the decisions.

---

## 3. Autoscaling: two loops and a blind HPA

This is the section people get wrong in production, so it gets the most space.

The chain is:

1. **metrics-server** scrapes kubelet and exposes the resource metrics API.
2. **HPA** reads a metric and rewrites `spec.replicas` on a Deployment.
3. The new pods that don't fit anywhere sit in `Pending`.
4. **Cluster Autoscaler** (or Karpenter) sees Pending pods with matching node requirements and adds capacity.

Two loops, two clocks, two failure modes. **The HPA does not know whether there is room.** It will cheerfully scale a Deployment to 40 replicas on a three-node cluster, report a healthy autoscaler, and leave 31 pods Pending. Then the node loop runs on a completely different timescale: the autoscaler polls every few seconds, but instance launch plus kubelet registration plus image pull is two to five minutes. The classic incident is not a scaling failure — it is a p99 latency spike that lives entirely inside the window between the pod loop reacting and the node loop catching up.

### The metrics gap — where these two clouds genuinely differ

Out of the box, CPU and memory are all the HPA can see. To scale on anything else you need a **custom or external metrics** path, and this is where the platforms diverge hard.

**EKS:** the Kubernetes Metrics Server is *not deployed by default*. AWS's own documentation says so plainly. You install it yourself. So a fresh EKS cluster cannot run even a CPU-based HPA until you ship a component. Then, if you want to scale on load rather than CPU — ALB request-count-per-target, queue depth, a 70%-of-target saturation number — that is an external metric, and you install a Prometheus Adapter or KEDA via Helm, wire it into the custom metrics API, and own it forever.

**GKE:** metrics-server is already there. The custom and external metrics path ships with the platform via Managed Service for Prometheus and the Cloud Monitoring adapter, so an HPA on a custom metric is configuration rather than installation.

Same Kubernetes API. Enormously different amounts of glue. **That glue is the actual product difference between the two clouds** — not the pricing page, and not the feature list.

---

## 4. Ingress on EKS

On AWS, `Ingress` is not real until the **AWS Load Balancer Controller** is running in the cluster. That controller is what turns a Kubernetes object into a load balancer:

- An `Ingress` object becomes an **ALB** — Layer 7, with listeners and target groups. The controller registers pod IPs into target groups directly (IP mode), so traffic lands on pods without an instance hop.
- A `Service` of `type: LoadBalancer` becomes an **NLB** — Layer 4.
- The ALB lives in **public subnets across at least two AZs**. The pods live in private subnets. That split is the whole point.
- ALB health checks are **not** Kubernetes readiness probes. A pod can be `Ready` in Kubernetes and still draining in the target group. This mismatch produces some of the most confusing "why is 5% of traffic 502-ing" incidents you will ever debug.
- The controller needs **IRSA** — an IAM role bound to its service account. Without it, it cannot call the ELB API, and the Ingress silently never materialises.

GKE's equivalent path is GKE Ingress or the Gateway API driving Google Cloud Load Balancing. The structural difference is that Google hands you a **global anycast VIP** rather than a per-region load balancer. That single design choice causes everything in Part 2's DNS section.

---

## 5. The data layer, and identity as the real connective tissue

RDS and CloudSQL are not complicated to place: DB subnet groups in isolated subnets, no route to the internet, `publicly_accessible = false`. Connections are understood through **security group referencing** on AWS — app SG to DB SG on 5432 — never CIDR ranges, because CIDR ranges rot the moment someone adds a subnet. GCP does the same job with firewall rules and network tags.

Credentials come from Secrets Manager or Secret Manager and are projected in via the CSI driver or the External Secrets Operator. They are never baked into a ConfigMap. This is not paranoia; it is the difference between a rotated secret and a breach.

But here is the part worth stating explicitly, because it reframes the entire diagram: **your API and your database are not connected by the network path.** The route exists, but the route is not the edge. The edge is the IAM binding plus the security group rule. Everything else is routing.

Both clouds solve this with the same concept wearing different names — **IRSA / EKS Pod Identity** (AWS) and **Workload Identity** (GCP). A Kubernetes ServiceAccount is mapped to a cloud identity, and the pod receives short-lived credentials with no static keys anywhere. Different services, identical concept. This is the clearest example in the whole stack of why learning the concept beats memorising the product.

---

## Where this leaves you

Everything above is a chain you assemble yourself: control plane, nodes, autoscaler, metrics adapters, CNI, ingress controller, certificates, IAM wiring. It is a lot of surface, and every piece of it is a thing that can drift.

[Part 2](https://edchtzwe.github.io/blog/posts/infra/04-cloud-services-part-2-wiring-and-governance/) covers the edge (why AWS drags you into Route 53 and GCP doesn't), the governance boundary (Terraform's clock versus Crossplane's), and the honest case for why someone would throw all of the above away and use ECS instead.

If you want the tool-boundary version of this — which of these resources belongs in Terraform and which belongs in Crossplane — that was [the previous post](https://edchtzwe.github.io/blog/posts/infra/02-terraform-crossplane-k8s-mental-model/).
