# Kubernetes for Agent Platforms: learning log

A 24-week coaching plan for mastering Kubernetes for cloud and agentic workloads.

**Progress tracker:** [Kubernetes for Agent Platforms](https://claude.ai/artifact/QZFfZayRBw9SP6ia8gvzFq)

## Sessions

| # | Date | Session | Phase | Status |
|---|---|---|---|---|
| 00 | 2026-10-17 | [The learning plan](sessions/00-learning-plan.md) | Planning | Done |
| 01 | 2026-10-17 to 18 | [Kubernetes is a set of control loops](sessions/01-week1-control-loops.md) | Phase 0 | Homework reviewed; model answers recorded |
| 02 | 2026-10-19 to 21 | [From containers to a cluster you build by hand](sessions/02-week2-containers-and-hard-way.md) | Phase 0 | Labs done; model answers recorded |
| 03 | 2026-10-21 to 23 | [The cluster's trust model](sessions/03-week3-trust-model.md) | Phase 0 | Labs 04–06 done; Hard Way stopped by decision |
| 04 | 2026-10-23 | [Your own APIs: designing AgentSession](sessions/04-phase1-your-own-apis.md) | Phase 1 | API tested; six design decisions agreed |

## Phase 0: closed (2026-10-23)

Covered: control loops and reconciliation, containers from scratch, and *Kubernetes the Hard Way* through the trust model (certificates, kubeconfigs, Secrets encryption). Stopped before starting the control plane, by decision: the remaining labs are mostly mechanics, and the key lesson, the identity model, was already learned. See the [decision record](sessions/03-week3-trust-model.md#decision-stop-kubernetes-the-hard-way-here-2026-10-23).

## How each session file is organized

- **Before you start: key terms** opens every session: each new term explained in plain language for someone with no context. Read it first.
- **Resources** closes every session: books (with specific chapters), papers, articles and docs, and YouTube videos. Video links were checked against YouTube; chapters are cited by title.

## How sessions work from Phase 1 on

- **Questions ask what you'd decide, ask or block,** not how things work.
- **Depth goes where the Agent Platform's risk is:** sandboxing, identity, multi-tenancy, GPU scheduling. Everything else stays shallow.
- **Claude runs the mechanical steps;** you make the calls and review the results.
- **Learning happens through Project 1,** the `AgentSession` operator, built with Claude Code across the phase.

## Phase 1 plan

**Goal:** understand how platforms are built *on* Kubernetes, well enough to review your team's designs and spot weak ones.

| # | Session | What you do | What you'll be able to judge |
|---|---|---|---|
| 04 | **Your own APIs: CRDs and the operator pattern** | Design the `AgentSession` API on paper (spec, status, conditions). Install a CRD in kind and see that the API server stores your objects even though nothing acts on them. | Whether a proposed API is a good contract: what belongs in spec vs. status, and what users should never set |
| 05 | **Controllers: making the API do something** | Claude Code scaffolds the operator with Kubebuilder; you review the reconcile loop. Break it on purpose: duplicate events, restarts, deleted pods. | Whether a controller is safe: idempotent, level-triggered, cleans up after itself (finalizers, owner references) |
| 06 | **Guardrails: admission and policy** | Enforce "every agent sandbox runs on gVisor and can't reach the network" with a ValidatingAdmissionPolicy. Compare with webhooks. | Where to put platform rules, and the cost of each option when it fails |
| 07 | **Running operators in production** | API versioning and upgrades, operator failure modes, observability, RBAC for the operator itself. | **Phase 1 milestone:** a design review of an operator, with the questions you'd ask your team |

**Setup needed for Session 05:** Go and Kubebuilder (not installed yet). Sessions 04 and 06 use the kind cluster you already have.

## Assets

- [Lab 1 cluster diagram, animated (SVG)](assets/lab1-cluster-animated.svg)
- [Lab 1 cluster diagram, static (SVG)](assets/lab1-cluster.svg)
- [Lab 1 cluster diagram, interactive (open in a browser)](assets/lab1-cluster-interactive.html)
- [Session 02 setup walkthrough, animated (SVG)](assets/session02-setup-animated.svg)
