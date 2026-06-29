# TaskFlow — Specification

Authoritative spec for the project, in two sections: **1. Requirements** (what we're building) and **2. Infrastructure** (how it's hosted, shipped, and scaled). Specification only — implementation lives in the repo and the companion plan docs. Full user stories: `TaskFlow-SRS.md`.

| | |
|---|---|
| Product | TaskFlow — real-time collaborative project management (Trello-style) |
| Stack | .NET 10 API · Vue 3 SPA · SQL Server · Azure SignalR |
| Hosting | Azure, free tier, **ephemeral** (spin up → demo → destroy) |
| Status | Spec v1.0 |

---

# 1. Requirements

## 1.1 Overview
Teams organise work into **boards → lists → cards**, move cards by drag-and-drop, and see each other's changes in real time. The product demonstrates production-grade full-stack engineering: clean architecture, real-time collaboration, role-based access, and a credible delivery pipeline.

## 1.2 Goals & success criteria
- **Real-time collaboration** — another user sees a change within ~1s, no refresh.
- **Fast first use** — create a board and first card in under a minute.
- **Clear, server-enforced authorization** — per-board roles.
- **Demo-ready quality** — deployable on demand, with loading/empty/error states and a guest path.

## 1.3 Scope (v1.0)
**In:** authentication; per-board roles; boards/lists/cards CRUD with ordering; drag-and-drop; labels, assignees, due dates, priorities; comments; real-time sync; per-board activity feed; "My Tasks" across boards; search/filter.

**Out (future):** multiple workspaces/orgs; file attachments; email/push notifications; checklists/custom fields/automation; native mobile apps; third-party integrations.

## 1.4 Roles (per board)
| Role | Capabilities |
|---|---|
| **Owner** | Everything, plus manage members and delete the board |
| **Editor** | Create/edit/move/delete lists & cards, comment, manage labels |
| **Viewer** | Read-only |

Roles are **per board** (a user may be Owner of one and Viewer of another) and enforced **server-side** on every mutation.

## 1.5 Feature areas
Authentication & accounts · Boards · Lists · Cards (with drag-and-drop positioning) · Labels & filtering · Comments · Real-time collaboration · Activity feed · "My Tasks". Each with acceptance criteria in `TaskFlow-SRS.md`.

## 1.6 Non-functional requirements
- **Performance:** a board (~200 cards) loads in < ~1.5s; real-time propagation < ~1s; all list endpoints paginated.
- **Security:** JWT auth; authorization enforced server-side; DTOs only at the API boundary; validated input; RFC 7807 error responses.
- **Usability & accessibility:** loading/empty/error states everywhere; responsive; destructive actions confirmed; **WCAG 2.2 AA**.
- **Reliability & quality:** structured logging, global error handling, EF migrations, unit + integration tests, automated quality + security gates in CI.

---

# 2. Infrastructure

## 2.1 Principles
- **Free / near-zero cost now; scale-ready later** — every ceiling is a SKU/config change, not a rewrite.
- **Each tier is an independent, separately scalable compute service.**
- **Stateless services** — horizontal scale-out needs no session affinity.
- **Ephemeral by default** — provision for a demo, capture proof, then destroy.

## 2.2 Tech stack
- **Backend:** .NET 10 / ASP.NET Core Web API, Clean Architecture (Domain → Application → Infrastructure → Api), EF Core, SQL Server.
- **Real-time:** Azure SignalR Service (Default mode).
- **Frontend:** Vue 3 + Vite + Pinia + TypeScript (SPA).
- **IaC:** Bicep. **CI/CD:** GitHub Actions.

## 2.3 Key infrastructure decisions
| # | Decision | Rationale |
|---|---|---|
| D1 | **Frontend and backend are separate compute services**, each on its own hosting plan (App Service, Linux) | Lets each tier be scaled, replicated, and load-balanced independently later (system-design practice) |
| D2 | **Ephemeral lifecycle** — one disposable environment per demo | $0 standing cost; "proof on demand" |
| D3 | **Managed identity everywhere; no secrets in config** | Security; nothing to leak or rotate manually |
| D4 | **Runtime `config.json`** for the SPA (URLs not baked at build) | One frontend build runs against any environment/instance |
| D5 | **Azure SignalR offloads client sockets** | API stays stateless → clean horizontal scale-out |
| D6 | **CI/CD on GitHub Actions** (not Azure-hosted) | Free for public repos; Azure is the deploy target only |
| D7 | **Hosting compute is code-deploy on built-in runtime** (not Docker) on free tier | F1 Free can't run custom containers; containers (Container Apps) are a future option |

## 2.4 Components
| # | Component | Role |
|---|---|---|
| 1 | **Frontend Web App** (App Service, Linux) | Serves the Vue 3 SPA; own plan; independently scalable |
| 2 | **Backend API Web App** (App Service, Linux) | .NET 10 Web API; own plan; stateless; independently scalable |
| 3 | **Azure SignalR Service** | Real-time backplane (clients connect here, not to the API) |
| 4 | **Azure SQL Database** (serverless) | Relational store; future: read replica / failover group |
| 5 | **Key Vault** | Secrets (JWT key), via managed identity |
| 6 | **Application Insights + Log Analytics** | Telemetry / monitoring |
| 7 | *(future)* **Front Door / Application Gateway** | L7 load balancer + WAF, single entry point |
| 8 | *(future)* **Azure Cache for Redis** | Hot-read cache behind the `ICacheService` seam |

## 2.5 Topology (logical)
```
                  (future) Front Door / App Gateway — WAF · routing · LB
                                 │
            ┌────────────────────┴────────────────────┐
            ▼                                          ▼
 Frontend Web App (App Service)            Backend API Web App (App Service)
 [own plan · scalable]                     [own plan · scalable · stateless]
            │ serves SPA                              │
   browser ─┴──────────── REST / WebSocket ──────────┤
                                                      ├── Azure SQL  (future: + replica)
                                                      ├── Azure SignalR Service
                                                      ├── Key Vault (managed identity)
                                                      └── App Insights / Log Analytics
```
The browser loads the SPA from the Frontend Web App, then calls the API (REST) and connects to SignalR. No server-to-server dependency between the two web apps — they scale independently.

## 2.6 Hosting profiles — Now vs Later
| Concern | Now (free / ephemeral) | Later (paid, spun up for practice) |
|---|---|---|
| Frontend / Backend plans | Free F1, 1 instance each | B1/S1/P-v3 with **autoscale**, multi-instance |
| Load balancing | none (single instance) | App Service built-in cross-instance LB; **Front Door / App Gateway** in front |
| Database | SQL serverless (free offer, auto-pause) | + **read replica / failover group** |
| Real-time | SignalR Free | SignalR Standard (more units, autoscale) |
| Caching | none | **Redis** behind `ICacheService` |
| Cost | ~$0 | a few $/hour while up — destroy after |

> **F1 Free cannot scale out** (1 instance, no Always On, no slots). Genuine scaling/replication/LB practice requires Basic+ tiers, spun up ephemerally then torn down.

## 2.7 CI/CD
- **CI** (GitHub Actions, every push/PR): build → tests (coverage) → **SonarCloud quality gate (blocking)** → **CodeQL** → frontend lint/build + **Lighthouse** (accessibility a hard gate). Branch protection requires green checks before merge.
- **CD:** build-once artifacts deployed to the target environment; DB migrated via **expand-contract** (parallel change), with the runner's IP opened on the SQL firewall just-in-time. Auth to Azure via **OIDC** (no stored cloud secrets).
- **Cost:** $0 on a **public repo** (Actions unlimited; Sonar + CodeQL free for public).

## 2.8 Environments & lifecycle
- **Default — ephemeral demo:** a single disposable environment in one resource group; **provision → demo/screenshot → destroy** (one resource-group delete). Persisted across teardowns: the OIDC app registration and subscription-scoped role assignment.
- **Optional — staging + production:** two isolated stacks (own resource groups, per-env parameter files, CI → staging → approval → production). Available but not run continuously under the ephemeral strategy.

## 2.9 Networking & configuration
- **Frontend → API:** browser-side calls; **CORS configured in API code** (allow the frontend origin) until a gateway fronts both under one hostname.
- **Runtime config:** the SPA reads API + SignalR URLs from `config.json` fetched at boot — same build targets any environment/instance.
- **SignalR:** browser connects directly to the service (negotiated via the API), so API instances never hold sockets.
- **Health probes:** `/health` (liveness, DB-free) and `/health/ready` (readiness, checks SQL) — ready for LB/orchestrator use.

## 2.10 Security
- Managed identity for SQL, SignalR, and Key Vault — **no keys/passwords in config**.
- **Entra-only SQL authentication.**
- JWT signing key in **Key Vault**, referenced by the API's managed identity.
- CORS owned in code; frontend `config.json` holds only **public** URLs.

## 2.11 Scaling & resilience hooks (reserved for later)
Designed-in so each is additive: per-service **horizontal scale-out**; **App Service autoscale**; **load-balancer layer** (built-in + Front Door/App Gateway); **database replication** (read replicas, failover group); **SignalR units**; **Redis caching** behind the existing interface; **async eventing** behind the event-publisher seam; split **health probes**.

## 2.12 Cost posture
- **Now ≈ $0:** free-tier components + ephemeral lifetime; worst case a few cents of Key Vault operations per spin-up.
- **Scaling features cost money** (Basic+ App Service, Front Door/App Gateway, Redis, SQL replicas) — incurred only during deliberate, time-boxed practice, then destroyed.

## 2.13 Constraints & non-goals
**Constraints (accepted):** frontend on its own App Service forgoes a managed static host's free CDN/auto-SSL (recoverable via Front Door); F1 Free is single-instance with cold starts; ephemeral URLs change each spin-up; Sonar/CodeQL free only on public repos.

**Non-goals (now):** implementing scaling, replication, load balancers, caching, async workers, custom containers, or a persistent/custom-domain deployment — the topology and seams only need to make these additive later.

---

*Companion docs: `TaskFlow-SRS.md` (full requirements) · `architecture-scalability.md` (scale ladder & seams) · `infrastructure-spec.md` (infra detail) · `plan-ephemeral-demo.md` (deploy/destroy) · `plan-staging-production.md` (two-env option) · `devops-setup.md` (setup steps).*
