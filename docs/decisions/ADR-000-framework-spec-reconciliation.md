# ADR-000 (Rev 2) — Reconciling the Delivery Spine with the TaskFlow Specification

| | |
|---|---|
| **Status** | Accepted · **supersedes Rev 1** |
| **Scope** | TaskFlow v1.0 (spec revised: Azure Pipelines · SonarQube Cloud · Sentry) |
| **Authoritative inputs** | `TaskFlow-Specification.md` (project) · Delivery Spine bundle (framework) |
| **What changed since Rev 1** | The spec was revised to adopt **Azure Pipelines** (D6), **SonarQube Cloud** (D8), and **Sentry** (D9, replacing App Insights). These were the three real conflicts in Rev 1 — now **resolved at the spec level**, in the framework's favour. Rev 1's contrary decisions (toward GitHub Actions, dual App-Insights+Sentry) are withdrawn. |

**Net result:** the spec and the Delivery Spine are now **aligned on every infrastructure conflict.** Only one genuine framework action remains — adding the Clean-Architecture rule (R4), which was never a *conflict* but a *gap*. Everything else is alignment or a configuration setting.

---

## Status of the Rev 1 decisions against the new spec

| # | Rev 1 conflict | New spec | Status now |
|---|---|---|---|
| **R1** | CI: framework = Azure Pipelines; spec = GitHub Actions | **D6: Azure Pipelines (Azure DevOps), source in GitHub** | ✅ **Resolved — aligned.** The framework's `azure-pipelines` skill is now the **active** CI layer (no longer inert). |
| **R2** | Packaging: Docker vs code-deploy | **D7: code-deploy, no Docker** (unchanged) | ⚙️ **Config, not a conflict.** `deploy-profile.packaging = code`. |
| **R3** | Observability: App Insights vs Sentry | **D9: Sentry replaces App Insights**; component #6 = Sentry | ✅ **Resolved — aligned and simpler.** Sentry only; it's also the spine's `incident.opened` / `/escape` producer. |
| **R4** | Clean Architecture mandated; framework has no rule | **§2.2 still mandates Clean Architecture** | 🔧 **Still open — the one real action.** Add the proactive rule (below). A framework *gap*, not a spec conflict. |
| **R5** | "SonarQube" vs SonarCloud | **D8: SonarQube Cloud (SaaS)** | ✅ **Resolved — aligned.** "SonarQube Cloud" is the SaaS (formerly SonarCloud); free for public repos. |

---

## R1 — CI: Azure Pipelines (now aligned)

**New spec.** D6: *"CI/CD on Azure Pipelines (Azure DevOps); source stays in GitHub … practising the Azure toolchain end-to-end."* §2.7 confirms build → tests → SonarQube Cloud gate (blocking) → Lighthouse, with an **OIDC workload-identity service connection** and **GitHub branch protection requiring the pipeline's checks**.

**Decision.** Use the framework's **`azure-pipelines` skill as-is** — this is precisely what it was built for (public GitHub repo + Azure Pipelines CI + deploy to Azure via OIDC). The earlier "one CI system" review concern is satisfied: **Azure Pipelines is the single CI.**

**Consequences (already in the framework — now simply *used*):**
- Two pipelines from the skill: secretless `pr-validation.yml` + merge-only OIDC `deploy.yml`.
- §6.1 guardrails stay in their **Azure Pipelines** form (no secret in the PR pipeline; fork PRs can't read secrets; least-privilege OIDC connection) — matching spec §2.7/§2.10.
- The spine's gates (`pr-reviewer` at the tier model, eval gate on `.claude/**`, `trace-check`, ledger emit) run **inside the Azure pipeline**.
- `CLAUDE.md` already says Azure Pipelines — **no change needed** (spec and framework now agree).

---

## R2 — Packaging: code-deploy (config only)

**New spec.** D7 unchanged: code-deploy on the built-in runtime (F1 Free, no Docker).

**Decision.** `deploy-profile.packaging = code`; **two** deploy targets (Frontend App Service + API App Service, Linux). No container registry.

**Consequences.** Parameterise the skill's `deploy.yml` for **two App Service code-deploy targets** (the skeleton shows one — duplicate the deploy job per target, both via the OIDC connection). Update the `CLAUDE.md` stack line to drop "Docker"; the bundled `.dockerignore` is unused for now.

---

## R3 — Monitoring: Sentry only (now aligned)

**New spec.** D9: *"Sentry replaces Application Insights … one app-centric errors + performance tool."* Component #6 = Sentry; §2.10 covers DSN handling (backend DSN as app setting; frontend DSN public, ingestion-scoped).

**Decision.** **Sentry is the monitoring tool** (no App Insights). It doubles as the spine's production signal.

**Consequences.**
- SDKs: `Sentry.AspNetCore` (API), `@sentry/vue` (SPA). Backend DSN via app setting/Key Vault; SPA DSN public in runtime `config.json`.
- CI uploads **release + source maps** so traces de-minify.
- Wire one Sentry alert → `metrics-log.sh incident.opened` — this gives `change-failure` a **real producer** (it was *defined-not-collected*), and a Sentry-surfaced regression drives `/escape` → a permanent eval case.
- Rev 1's "two tools, distinct roles" reconciliation is **withdrawn** — there is only Sentry now.

---

## R4 — Clean Architecture: add the proactive rule (the one remaining action)

**New spec.** §2.2 still mandates Clean Architecture (Domain → Application → Infrastructure → Api); D1 reinforces independent tiers.

**Why still open.** This was never a spec-vs-framework *conflict* — the framework simply has **no dependency-direction rule** yet. The fix is a framework enhancement, authored **proactively** (you don't wait for a layering violation to escape).

**Decision / action.**
- Add rule **`design-patterns/inward-dependency-only`** to the `design-patterns` skill: Domain imports nothing outward; **no EF Core / `Microsoft.*` infrastructure types in Domain**; Application depends only on Domain abstractions.
- Provenance: **Author canon (Robert C. Martin)** — honestly *not* FAANG.
- Ship it with a **RED eval case** (a Domain class importing `Microsoft.EntityFrameworkCore` → `must_catch`) **and a clean control** (properly layered, must-not-flag) so it can't over-fire. Run RED → add rule → confirm GREEN.
- GC-eligible like any rule: if project references already make the violation impossible, re-check via `rule-firings`.

---

## R5 — Static analysis: SonarQube Cloud (now aligned)

**New spec.** D8: SonarQube Cloud (SaaS, free for public repos). §2.7 notes **CodeQL is GitHub-native and not part of the Azure Pipelines flow** — SonarQube Cloud covers static analysis; CodeQL is optional.

**Decision.** **SonarQube Cloud** as the blocking quality/security gate, run from the Azure pipeline via its **Azure DevOps extension** tasks, with the token in a **fork-protected variable group** (consistent with R1's no-secret-to-fork-PR guardrail). **CodeQL is omitted** from the required gate (optional, GitHub-native).

**Consequence.** The framework's Sonar references already cover "SonarQube/SonarCloud"; "SonarQube Cloud" is the same SaaS under its current name. No framework change.

---

## Where the spec and framework already align (no decision)

| Topic | Spec | Framework | Status |
|---|---|---|---|
| CI system | Azure Pipelines (D6) | Azure Pipelines (`azure-pipelines`) | **Aligned** |
| Deploy auth | OIDC workload-identity service connection (§2.7, §2.10) | OIDC, no stored secret | **Aligned** |
| Static analysis | SonarQube Cloud (D8) | SonarQube/SonarCloud | **Aligned** |
| Monitoring | Sentry (D9) | Sentry (Seer + escape signal) | **Aligned** |
| Real-time | Azure SignalR, Default mode (§2.4) | `signalr-patterns` requires the SignalR backplane | **Aligned** |
| Secrets | managed identity / Key Vault, none in config (§2.10) | no long-lived secrets in CI | **Aligned** |
| Public-repo posture | every PR diff effectively untrusted | §6.1 treats the diff as untrusted | **Aligned** |
| IaC | Bicep (§2.2) | (none) | **Additive** — adopt Bicep |
| Model usage | — | tier the model; never blanket-Opus | **Unchanged** |

---

## Nothing-unenforced check: spec NFRs (§1.6) → framework gate

| Spec NFR | Enforced by |
|---|---|
| Board ~200 cards < 1.5s; lists paginated | `sql-perf`, `efcore-conventions`, `performance-regression`, `rest-api-conventions` |
| Real-time propagation < 1s | `signalr-patterns` |
| JWT; **server-side authz on every mutation**; DTOs only at boundary; RFC 7807 | `rest-api-conventions`, `system-design`; authorization a **mandatory acceptance criterion on every mutating story** |
| WCAG 2.2 AA; loading/empty/error states; destructive confirms | `uiux-accessibility`, `vue-conventions`; Lighthouse hard gate in CI |
| Tests + migrations + quality/security gates | `tdd`, eval gate, SonarQube Cloud |
| Clean Architecture (§2.2) | `design-patterns/inward-dependency-only` (R4) |

---

## Actions that fall out of this record

1. **`azure-pipelines` skill is now active** — author `pr-validation.yml` + `deploy.yml` from it (no longer marked inert).
2. **`deploy-profile`:** `packaging = code`, **two** App Service targets (Frontend + API); update `CLAUDE.md` stack line to drop "Docker".
3. **Sentry:** add SDKs; wire the `incident.opened` producer + `/escape` trigger; remove any App Insights assumption.
4. **SonarQube Cloud:** Azure DevOps extension tasks, fork-protected token, blocking gate; CodeQL omitted (optional).
5. **R4:** add `design-patterns/inward-dependency-only` + RED case + clean control.
6. Keep this file as `docs/decisions/ADR-000-framework-spec-reconciliation.md`; **Rev 1 is superseded.**

*Superseded by: a future ADR only if a decision here is reversed — which must also amend the spec.*
