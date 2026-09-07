# SAHIIX Stack v2.1 — Architecture

> **Version:** 2.1  
> **Date:** 2026-09-07  
> **Owner:** sahiixx (AI Systems Architect · AIOS)  
> **Status:** Canonical reference — supersedes ad-hoc per-repo diagrams where they conflict

---

## 0. One-sentence summary

**SAHIIX Stack v2.1** is a multi-layer autonomous AI operating system for Dubai real-estate revenue operations and personal/enterprise agentic workloads, built on a shared agentic harness, OPA dispatch, edge proxy governance, and event-driven specialist agents.

---

## 1. Layer cake

```
┌─────────────────────────────────────────────────────────────────┐
│  L7  EXPERIENCE                                                  │
│      sahiixx-os · portfolio · Jarvis · CLI · agno (App Builder)  │
├─────────────────────────────────────────────────────────────────┤
│  L6  VERTICAL PRODUCTS                                           │
│      Sovereign Revenue OS · NEXUS · Lead Machine · friday-os     │
├─────────────────────────────────────────────────────────────────┤
│  L5  AGENT RUNTIME (OPA)                                         │
│      sahiixx-agency · TaskRouter · MessageBus · ApprovalManager  │
│      QualificationAgent · GeoMatchAgent · Discovery · RealEstate │
├─────────────────────────────────────────────────────────────────┤
│  L4  AGENTIC HARNESS                                             │
│      agentic-harness · contracts · patterns · model routing      │
│      (Prompt Chaining · ReAct · Orchestrator-Workers · …)        │
├─────────────────────────────────────────────────────────────────┤
│  L3  EDGE & GOVERNANCE                                           │
│      sahiix-proxy (JWT · tiered rate-limit · governance KV ·     │
│                   WATI webhook · Termux)                         │
├─────────────────────────────────────────────────────────────────┤
│  L2  INTEGRATION & TOOLING                                       │
│      n8n / Hermes MCP · Azure Foundry · freellmpool · Ollama     │
│      WhatsApp / Telegram · CRM · scrapers                        │
├─────────────────────────────────────────────────────────────────┤
│  L1  DATA & MEMORY                                               │
│      Neon Postgres · Redis · Qdrant · Titans memory · GraphSight │
├─────────────────────────────────────────────────────────────────┤
│  L0  INFRA                                                       │
│      Cloudflare · Azure · Termux edge · Docker / GHCR · Vercel   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Runtime topology (v2.1)

| Component | Repo | Port / URL | Role |
|-----------|------|------------|------|
| **OPA API** | `sahiixx-agency` | `:8082` (local) / `:8080` (Docker) | Dispatch, adapters, metrics |
| **Dashboard** | `sahiixx-agency/dashboard` | `:3000` | Graph + ops UI |
| **MCP server** | `sahiixx-agency` | `:8081` (SSE) | Tool plane |
| **Edge Proxy** | `sahiix-proxy` | Termux / edge | Auth, rate, governance, WATI |
| **OS shell** | `sahiixx-os` | Cloudflare Pages | Command center |
| **Portfolio** | `sahiix-portfolio` | sahiix-portfolio.pages.dev | Public face + pilots |
| **Agno** | `agno` (**private**) | Vite `:8080` · Vercel | App Builder workspace (Grok Build) |
| **E2E harness** | `sahiixx-e2e` | Playwright | System + AI + Lead Machine tests |
| **Systems panel** | `systems-panel` | Astro static | Status of all modules |

### Model routing (shared contract)

| Purpose | Deployment | Endpoint |
|---------|------------|----------|
| Default | `gpt-5.6-sol` | `/openai/v1/chat/completions` |
| Deep reasoning | `claude-opus-5` | `/openai/v1/responses` **only** |
| Legacy | `gpt-5` | `/openai/v1/chat/completions` |
| Embeddings | `text-embedding-3-small` | `/openai/v1/embeddings` |

Base: Azure AI Foundry · auth via `$AZURE_FOUNDRY_API_KEY` (never commit).

---

## 2.1 Agno — App Builder Workspace (private)

| Field | Value |
|-------|--------|
| **Repo** | [`sahiixx/agno`](https://github.com/sahiixx/agno) (private) |
| **Stack** | React 19 · TanStack Start/Router/Query · Vite 8 · Tailwind 4 · Better Auth · Kysely/PG · Zod · Zustand |
| **Deploy** | Vercel (`.vercel/`) |
| **Preview contract** | `0.0.0.0:8080` via `npm run dev` / `startup.sh` |
| **Role in stack** | L7 product surface — rapid app scaffolding under the Grok Build / agent sandbox contract (`AGENTS.md`) |
| **Created** | 2026-09-03 · active through 2026-09-06 |

Agno is the private **App Builder Workspace**: agents (and you) ship playable/demo-quality apps into a sandboxed preview. Auth and DB are opt-in per app; platform injects secrets on deploy. It sits alongside `sahiixx-os` as an experience layer, not a replacement for OPA or the revenue vertical.

---

## 3. Lead Machine (v2.1 core vertical)

```
WhatsApp / Web / Telegram
        │
        ▼
  LeadCaptureAgent  ──►  lead.created
        │
        ▼
  QualificationAgent ──►  lead.qualified   (score, segment, decision)
        │
        ▼
  GeoMatchAgent      ──►  lead.matched     (area / inventory)
        │
        ▼
  SchedulingAgent    ──►  lead.scheduled   (Tier-2 gated)
        │
        ▼
  ReportingAgent     ──►  funnel metrics
```

**Status (honest):**

| Agent | Status |
|-------|--------|
| QualificationAgent | **Live** in OPA (`adapters/qualification_agent.py`) |
| GeoMatchAgent | **Code** on branch + registration docs |
| Event topics | Contracted (`lead.*`) — subscribers partial |
| LeadCapture / Scheduling / Reporting | Spec + E2E contracts; code TBD |
| NEXUS WhatsApp intake | Live (separate path) |

**E2E:** `sahiixx-e2e` ships pure scorer + schema tests + optional live OPA endpoint (PR #1).

---

## 4. Agentic harness patterns (mandatory)

Every production agent path must pick the simplest pattern that works:

1. Prompt Chaining  
2. Routing  
3. Parallelization  
4. Orchestrator–Workers  
5. Evaluator–Optimizer  
6. ReAct  
7. Reflection  

Escalation rule: start simple → add Reflection only on verification failure → Planning only when dependencies appear → Multi-Agent only when a single context window is insufficient.

Reliability envelope (all layers):
- Bounded loops (max iterations + wall-clock)
- Tool sandboxing
- Guardrails at input / mid-loop / output
- Context compression
- Self-verification before commit

---

## 5. Edge Proxy (v0.2.1)

`sahiix-proxy` is the governance choke-point for Termux / edge deployments:

- JWT auth (tiered: free / standard / pro / enterprise)
- Rate limiting with KV store
- Governance rules (user / path / multi-condition AND)
- WATI webhook admin commands
- CORS + `X-Proxy-Version`
- Strong Jest E2E suite (auth, rate, governance, webhook, admin, CORS, errors)

---

## 6. What changed in v2.1

| Area | v2.0 (approx) | v2.1 |
|------|---------------|------|
| Qualification | Spec only | **Live adapter** + pure E2E oracle |
| GeoMatch | — | Agent code + registration path |
| E2E | Ad-hoc Playwright | Dedicated `lead-machine` project + smoke CI |
| Edge | Scaffold | Production-grade proxy + Termux scripts |
| App Builder | — | **agno** private workspace documented |
| Architecture docs | Fragmented | This canonical stack document |
| Model routing | Per-repo | Shared AGENTS.md contract across repos |
| Event fabric | MessageBus exists | Explicit `lead.*` topic contracts |

---

## 7. Repo map (priority)

### Core (must stay green)
- `sahiixx-agency` — OPA runtime
- `sahiix-proxy` — edge governance
- `agentic-harness` + `agentic-harness-integration`
- `sahiixx-e2e` — system tests
- `sovereign-revenue-os` (private) — revenue vertical
- `agno` (**private**) — App Builder workspace
- `sahiixx-os` — shell

### Supporting
- `sahiix-portfolio`, `sahiix-os-docs`, `systems-panel`
- `friday-os`, `sovereign-swarm-v2`, `sahiixx-bus`
- Memory / graph: `sahiixx-titans-memory`, `sahiixx-graph-sight`

### Experimental / reference
- ADK samples, OpenAI agent suites, n8n forks, moltbot experiments

---

## 8. Build order (next 30 days)

1. **Merge** sahiixx-e2e PR #1 (Lead Machine contracts + CI smoke)
2. **Merge** Stack v2.1 + profile + agency GeoMatch PRs
3. **Register** GeoMatchAgent in `_SPECIALIZED_ADAPTERS` + routing rule
4. **Wire** NEXUS WhatsApp → `lead.created` event on MessageBus
5. **Expose** `/api/opa/lead/qualify` so E2E live test un-skips
6. **Harden** proxy → OPA path (proxy as front door for agent calls)
7. Keep **agno** aligned with shared AGENTS.md model-routing contract where agents build apps

---

## 9. Non-goals (v2.1)

- Replacing n8n for every integration (it remains the “hands”)
- Auto-executing Tier-2 financial actions without ApprovalManager
- Unifying all 200+ repos into one monorepo
- Claiming agents are live when only the schema exists
- Making agno public (stays private App Builder)

---

## 10. Related documents

- `SAHIIXX_OS_ARCHITECTURE.md` — earlier OpenClaw + n8n agent team view
- `E2E-WIRING-PACK.md` (portfolio) — honest gap analysis
- `AGENTS.md` (per repo) — model routing contract
- `agno/AGENTS.md` — App Builder sandbox contract
- `INTEGRATION_CONTRACT.md` — cross-service contracts

---

**SAHIIX Stack v2.1 — Autonomous AIOS for revenue + agents + app building.**
