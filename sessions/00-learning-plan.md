# Session 00: The learning plan

**Date:** 2026-10-17
**Outcome:** a 24-week plan, a reading list, three projects, and a progress-tracking page.

[← Back to index](../index.md)

---

## What I asked

> Act as a Staff Software Engineering Leader at FAANG and coach me. I am a software engineering manager leading a team building an Agent Platform that runs on Kubernetes. I want to master Kubernetes applied to the cloud and Agentic workloads. Create a plan for me to learn, add important resources, books and papers. Also, come up with 3 project ideas I can build using Claude Code to learn.

Then: "yes, go ahead" (turn the plan into a shareable page with progress checkboxes).

**Progress tracker page:** [Kubernetes for Agent Platforms](https://claude.ai/artifact/QZFfZayRBw9SP6ia8gvzFq). It's private until shared from its Share menu. Each person's checkmarks save to their own Claude account; teammates need Contributor access for that.

---

## Coaching frame

You're a manager, so the goal isn't to become your team's best `kubectl` operator. You need three things:

1. **An accurate mental model** of Kubernetes, so you can tell when a design is fighting the system and when it's working with it.
2. **Fluency in what's different about agent workloads.** Most Kubernetes knowledge assumes stateless web services. Agents are long-running, stateful, bursty and expensive, and they run untrusted code. Your edge comes from understanding that gap.
3. **Judgment you can pass on.** That means design-review questions, failure stories and tradeoff calls your team learns from.

**Time budget:** about 5–7 hours a week. Spend roughly 40% reading, 40% hands-on and 20% writing or teaching.

**Habits that make it stick:**
- **Teach weekly.** Give a 15-minute "what I learned" segment at team sync. Teaching shows you where your understanding has gaps.
- **Read one KEP or design doc a week** from [`kubernetes/enhancements`](https://github.com/kubernetes/enhancements).
- **Run a monthly game day.** Break something on purpose in a sandbox cluster, then debug it without help.

---

## Phase 0: The mental model (weeks 1–4)

**The core idea:** Kubernetes is a database (etcd) plus an API server plus many independent control loops that reconcile desired state with actual state.

- API machinery: objects, `spec`/`status`, resourceVersion, watches, informers, optimistic concurrency
- The scheduler: filtering and scoring, affinity, taints and tolerations, topology spread, preemption, priority classes
- The pod lifecycle: init containers, native sidecars, probes, graceful termination, QoS classes, OOM behavior
- Networking: CNI, Services, kube-proxy vs eBPF (Cilium), DNS, NetworkPolicy, Gateway API
- Storage: PV/PVC, CSI, StatefulSets

**Hands-on:** [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way), then a local `kind` cluster for everything else.

**Milestone:** Whiteboard what happens between `kubectl apply -f deploy.yaml` and a running container, naming every component involved.

## Phase 1: Extending Kubernetes (weeks 5–8)

Your Agent Platform will almost certainly be built from CRDs and controllers, so this phase matters most for you.

- CRDs, controllers and operators; level-triggered vs edge-triggered reconciliation; idempotency
- Finalizers, owner references, garbage collection, status conditions
- Admission webhooks and ValidatingAdmissionPolicy (CEL)
- Kubebuilder and controller-runtime ([Kubebuilder Book](https://book.kubebuilder.io))
- API design: versioning, conversion webhooks, why `spec` and `status` are split

**Milestone:** Write a small controller and explain why a reconcile loop must tolerate running a thousand times on the same object.

## Phase 2: Production Kubernetes in the cloud (weeks 9–12)

- **Managed Kubernetes:** EKS, GKE and AKS differences; control-plane limits; upgrade strategy
- **Identity:** workload identity (IRSA or EKS Pod Identity, GKE Workload Identity)
- **Autoscaling:** HPA, VPA, KEDA, Cluster Autoscaler vs Karpenter, in-place pod resize
- **Multi-tenancy:** namespaces vs virtual clusters (vCluster) vs separate clusters, ResourceQuotas, LimitRanges
- **Delivery:** GitOps (Argo CD or Flux), progressive delivery (Argo Rollouts)
- **Cost:** bin-packing, spot instances, OpenCost

**Milestone:** Write a one-page decision record: "Our tenancy model for agent workloads and why."

## Phase 3: Agentic workloads (weeks 13–18)

| Agent property | Kubernetes concern | Tools and patterns to study |
|---|---|---|
| Runs LLM-generated or untrusted code | Isolation beyond containers | RuntimeClass, gVisor, Kata Containers, Firecracker microVMs, `kubernetes-sigs/agent-sandbox` |
| Long-running, stateful sessions | Pods are disposable; sessions aren't | Session affinity, checkpoint/restore, StatefulSets, durable execution (Temporal, Dapr Workflows) |
| Bursty, latency-sensitive startup | Cold starts hurt the user experience | Warm pools, image streaming, pre-pulled images, KEDA scale-from-zero |
| Calls tools and external APIs | Egress is the attack surface | Egress NetworkPolicy, Cilium FQDN policies, egress gateways, MCP gateways |
| GPU-backed inference | Scarce, expensive accelerators | DRA, Kueue, LeaderWorkerSet, JobSet, KServe, vLLM, llm-d, Gateway API Inference Extension |
| Multi-agent coordination | Service discovery, protocols | MCP, A2A, kagent, Dapr Agents |
| Cost depends on tokens, not CPU | Wrong scaling signal | Custom metrics (queue depth, in-flight tokens, KV-cache use) |

**Milestone:** Write a reference architecture doc for "one agent session, end to end": request, sandbox, tools, model call, state persistence, teardown and every trust boundary.

## Phase 4: Reliability, security and observability (weeks 19–22)

- **Observability:** OpenTelemetry plus the [GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/); trace agent → tool → model; token and cost metrics
- **SLOs for agents:** task success rate, time to first token, tool-call error rate, cost per task
- **Security:** Pod Security Standards, Kyverno or OPA Gatekeeper, Sigstore/cosign, SBOMs, External Secrets, short-lived tokens, limiting the damage a prompt injection can do
- **Failure modes:** eviction mid-tool-call, double-execution of non-idempotent tools, retry storms against model APIs

**Milestone:** Tabletop exercise: "An agent was prompt-injected and tried to exfiltrate data. Walk me through what stops it."

## Phase 5: Leadership synthesis (weeks 23–24)

- A **platform strategy doc** for the next 12–18 months: build vs buy, the abstractions you expose to agent developers, and what you won't build
- A **tech radar** for your team: adopt, trial, assess, hold
- Optional: **CKA**, then **CKS**

---

## Books

**Kubernetes core**
- *Kubernetes: Up and Running*, 3rd ed. (Burns, Beda, Hightower, Evenson). Start here.
- *Kubernetes Patterns*, 2nd ed. (Ibryam, Huß)
- *Programming Kubernetes* (Hausenblas, Schimanski). Essential for Phase 1.
- *Production Kubernetes* (Rosso, Lander, Brand, Harris). The closest fit to your role.
- *Kubernetes Best Practices*, 2nd ed. (Burns, Villalba, Strebel, Evenson)

**Security and systems**
- *Container Security* and *Learning eBPF* (Liz Rice)
- *Hacking Kubernetes* (Martin, Hausenblas)
- *Designing Distributed Systems*, 2nd ed. (Brendan Burns)
- *Designing Data-Intensive Applications* (Kleppmann)
- *Site Reliability Engineering* and *The SRE Workbook* ([free online](https://sre.google/books/))

**AI and leadership**
- *AI Engineering* (Chip Huyen)
- *Platform Engineering* (Camille Fournier, Ian Nowland)
- *The Staff Engineer's Path* (Tanya Reilly)

## Papers

**Where Kubernetes came from (read in this order)**
1. [Large-scale cluster management at Google with Borg](https://research.google/pubs/large-scale-cluster-management-at-google-with-borg/) (EuroSys 2015)
2. *Omega: flexible, scalable schedulers for large compute clusters* (EuroSys 2013)
3. [Borg, Omega, and Kubernetes](https://queue.acm.org/detail.cfm?id=2898444) (ACM Queue 2016). The most important one.
4. *Autopilot: workload autoscaling at Google* (EuroSys 2020)
5. *Mesos: A Platform for Fine-Grained Resource Sharing* (NSDI 2011)

**Isolation**
- [Firecracker: Lightweight Virtualization for Serverless Applications](https://www.usenix.org/conference/nsdi20/presentation/agache) (NSDI 2020)
- *The True Cost of Containing: A gVisor Case Study* (HotCloud 2019)

**AI infrastructure and agents**
- [Ray: A Distributed Framework for Emerging AI Applications](https://arxiv.org/abs/1712.05889) (OSDI 2018)
- [Efficient Memory Management for LLM Serving with PagedAttention](https://arxiv.org/abs/2309.06180) (vLLM, SOSP 2023)
- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
- *Dapper, a Large-Scale Distributed Systems Tracing Infrastructure* (Google, 2010)

**Posts, specs and feeds**
- Anthropic: [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents), and the engineering post on their multi-agent research system
- [Model Context Protocol spec](https://modelcontextprotocol.io)
- Design docs for [agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox), [Gateway API Inference Extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension), [Kueue](https://kueue.sigs.k8s.io) and [llm-d](https://llm-d.ai)
- KubeCon talks on the CNCF YouTube channel, [k8s.af](https://k8s.af), [iximiuz Labs](https://labs.iximiuz.com)

---

## Three projects to build with Claude Code

### Project 1: `AgentSession` operator, a sandboxed code-execution platform (Phases 1 and 3)

A CRD and controller where `kind: AgentSession` provisions an isolated pod for an agent to run code in.

- A warm pool of pre-started sandboxes; measure p50/p99 time-to-ready with and without it
- RuntimeClass switching between `runc` and gVisor; benchmark the overhead
- A deny-all egress NetworkPolicy per session, with an allowlist of domains
- TTL and idle reaping with finalizers; status conditions `Pending → Ready → Idle → Terminated`
- Stretch: snapshot the workspace to a PVC and resume on another node; compare with `kubernetes-sigs/agent-sandbox`

**What you learn:** reconciliation, CRD API design, isolation tradeoffs, the warm-pool vs cost tension.

### Project 2: Token-aware autoscaling and an inference gateway (Phases 2 and 3)

- A backend: vLLM with a small model (GPU), llama.cpp (CPU), or a mock server that simulates token latency
- KEDA scaling on queue depth or in-flight requests; k6 load tests showing CPU-based HPA scales on the wrong signal
- Gateway API Inference Extension for model-aware routing
- Kueue queuing batch eval jobs below interactive traffic, so you can watch preemption
- A Grafana dashboard: tokens/sec, time to first token, queue depth, cost per 1k requests

**What you learn:** autoscaling, scheduling priority, accelerator economics.

### Project 3: A resilient MCP tool gateway with policy and tracing (Phases 2 and 4)

- Per-tenant identity with service account tokens; per-tool authorization with OPA or Kyverno
- OpenTelemetry GenAI traces across agent → gateway → tool → model, in Jaeger or Tempo
- The agent workflow on Temporal; Chaos Mesh kills pods mid-tool-call and you verify non-idempotent tools never run twice
- A simulated prompt-injected agent, blocked by the policy layer

**What you learn:** zero-trust design for agents, observability, failure semantics.

### How to use Claude Code so you actually learn

- **Write the design doc yourself first,** then ask Claude Code to critique it before any code exists.
- **Put a teach mode in each project's `CLAUDE.md`:** "Explain each Kubernetes concept before you implement it. After each change, ask me one question to check my understanding."
- **Make it break things.** Ask for a subtle bug without being told where, then debug with `kubectl` alone.
- **Read every diff line by line.** If you can't explain a line, ask before accepting it.
- **Turn each project into a team asset** with a brown-bag session.

---

**Suggested start:** read *Borg, Omega, and Kubernetes*, start *Kubernetes the Hard Way*, and draft the `AgentSession` CRD on paper.

**Next:** [Session 01: Kubernetes is a set of control loops](01-week1-control-loops.md)
