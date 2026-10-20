# OpsPilot — Phase List

Derived from the PDF's 17-week plan (ch.20), reordered around one rule: **build the deterministic spine and a stub-model test harness before the real agent**. Week ranges are planning assumptions for a team of ~3 engineers, not commitments. Each phase has exit criteria that must pass before the next begins.

Milestones: **M1** = full FreightResolve demo (end of Phase 6). **M2** = CivicResolve on the same engine (end of Phase 8). PDF approval scope is M1 + M2; Phases 9–12 are sequenced extensions.

| # | Phase | PDF weeks | Milestone |
|---|---|---|---|
| 0 | Bootstrap | 1 | |
| 1 | Contracts and schema | 1–2 | |
| 2 | Execution spine (stub model) | 2–3 | |
| 3 | Freight simulator | 4 | |
| 4 | Agent runtime | 3–5 | |
| 5 | Freight exceptions, approvals, concurrency | 6 | |
| 6 | Operator UI and demo studio | 6–7 | **M1** |
| 7 | Evaluation and security harness | 8 | |
| 8 | CivicResolve | 7–8 | **M2** |
| 9 | SupplyResolve | 9–10 | |
| 10 | PropertyResolve | 11–12 | |
| 11 | ClaimResolve | 13–14 | |
| 12 | Blast Radius Lite (full) and verifiers | 15 | |
| 13 | Release hardening and sales packaging | 16–17 | |

---

## Phase 0 — Bootstrap
**Goal:** a repo any engineer can clone and run.
- `git init`, `.gitignore`, `.editorconfig`, license/confidentiality header.
- Monorepo skeleton per spec §15; `docs/` with ADR template.
- Docker Compose: postgres, redis, api (hello), web (hello), simulator (hello).
- Makefile with the planned targets; Python (uv or poetry) and pnpm workspaces; pinned versions.
- CI: lint + test on push; pre-commit hooks.
- Move the blueprint PDF into `docs/`.

**Exit:** `make up` brings all containers healthy; `make test` runs a trivial passing test; CI green.

## Phase 1 — Contracts and schema
**Goal:** the canonical types and tables everything else depends on.
- Pydantic contracts: Case, BusinessEntity, Observation, ActionProposal, PolicyDecision, Approval, Execution, Verification, event envelope, tool descriptor, risk tier enum, state enum, verification status enum.
- OpenAPI → TypeScript type generation.
- Alembic migrations for all entities; `tenant_id` on every table; append-only event ledger (enforced with DB rule/trigger, not convention).
- Seeded dev tenant, users and roles; dev auth shim with role checks.
- Case CRUD endpoints (`POST/GET /v1/cases`).

**Exit:** migrations apply on an empty DB; ledger rejects UPDATE/DELETE; cross-tenant read returns nothing (test); contracts round-trip to TS.

## Phase 2 — Execution spine (stub model)
**Goal:** the safety machinery works with no LLM at all. This is the core of the product.
- Case state machine with legal-transition enforcement and `state_version` optimistic locking.
- Durable worker with interrupt/resume; a case survives process restart.
- Tool registry with typed I/O, scope, timeout, failure classes.
- Policy engine returning ALLOW/HOLD/DENY with reasons and rule version.
- Approval service: versioned, expiring, bound to `proposal_hash`.
- Executor: idempotency keys, compare-and-swap, re-validation before commit, timeout → reconcile-before-retry.
- Verifier framework: expected vs observed, status SUCCESS/PENDING/DISCREPANCY/UNKNOWN.
- `impact()` interface with a trivial implementation (Blast Radius Lite placeholder).
- `StubProvider` + a scripted fake domain ("toy pack") with one fake system, to exercise everything.
- Action ledger and OTel spans for every step.

**Exit (all as automated tests):** restart mid-case resumes; duplicate idempotency key does not double-write; stale version blocks the write; approval expires on state change; timeout triggers a read before any retry; unauthorized actor cannot approve; PENDING is never shown as DONE; no code path lets an agent output reach an adapter without a policy decision.

## Phase 3 — Freight simulator
**Goal:** a believable, stateful TMS with sharp edges.
- `services/simulator` with the freight module: shipments, trucks, carriers, rate contracts, vehicle alerts; fixtures sized per spec §12.1.
- Optimistic concurrency on assignment writes; carrier acknowledgment flow (async accept/reject).
- Fault injectors: carrier rejects, 500, timeout, stale version, duplicate request.
- Idempotent reset to a named fixture version; replay of an injection by name.
- `packages/connectors` Freight adapter implementing the 7-method connector interface; contract tests.

**Exit:** adapter contract tests pass against the simulator including every injected fault; reset restores byte-identical state.

## Phase 4 — Agent runtime
**Goal:** a real agent loop on top of the spine, behind the provider adapter.
- `LLMProvider` interface; `OpenAIProvider` (gpt-4o-mini, strict structured output); `StubProvider`; `ReplayProvider` (record/replay transcripts).
- LangGraph agent loop for INVESTIGATING / PLAN_READY / REPLANNING with action space: `call_tool`, `submit_plan`, `draft_message`, `escalate`. No execute action.
- Budgets: max tool calls, max replans, wall clock, cost cap → ESCALATED on breach.
- Prompts versioned under `services/agent-runtime/prompts/` (see prompts.md Part B).
- Untrusted-data wrapping for all tool outputs.
- Freight tools registered with tiers; agent can run a happy-path case end to end.

**Exit:** with `StubProvider`, deterministic happy path to DONE with verification; with the live model (manual run), a happy-path Freight case completes; an out-of-scope or malformed tool call is rejected and logged without crashing the case.

## Phase 5 — Freight exceptions, approvals, concurrency
**Goal:** the behaviors that prove it is agentic *and* safe.
- Carrier-rejects-after-plan: agent re-plans to a new eligible carrier.
- Approval flow for R3 `reserve_carrier`; reject path evaluates alternatives; expiry path.
- Concurrency: two cases competing for the same carrier — one valid allocation persists.
- Freight "other deliveries on this vehicle" impact preview.
- Timeout/500 recovery via reconcile-first.
- Outcome-not-observed → PENDING/UNKNOWN, not success.

**Exit:** all Freight golden scenarios (spec §11) pass with the stub provider; at least the happy path and carrier-rejection pass with the live model.

## Phase 6 — Operator UI and demo studio (M1)
**Goal:** a sales presenter can run the Freight demo alone.
- Screens: industry launcher, case inbox, case workspace, plan + impact, execution trail, outcome verification, demo studio.
- Approve/reject with action, entity, cost, risk, expiry and evidence visible.
- Demo studio: start scenario, guided/autonomous mode, inject exception, pause/resume, reset, export evidence.
- Accessibility pass: keyboard, contrast, semantics.
- Playwright E2E for the full Freight flow.

**Exit (M1):** a non-developer runs the 8-minute Freight script from a reset state twice in a row with identical outcomes, with no DB or CLI access.

## Phase 7 — Evaluation and security harness
**Goal:** regression protection before adding more packs.
- Labeled scenario runner producing resolution rate, safe-escalation rate, unauthorized-write attempts, verification coverage, replans, cost, latency.
- ≥20 Freight cases (8 / 8 / 4) with expected end-state labels.
- Security tests: prompt injection in carrier/vendor email, forged approval, replayed webhook, cross-tenant access, tampered state version.
- Model comparison run: `gpt-4o-mini` vs a larger model on the same suite; report to decide defaults.
- CI gate: golden suite on any prompt/tool/model change.

**Exit:** zero unauthorized writes across adversarial and concurrency tests; metrics report generated; CI gate enforced.

## Phase 8 — CivicResolve (M2)
**Goal:** prove the engine is reusable.
- Civic simulator module, fixtures (spec §12.2), adapters, tools, rules, verifier, injectors.
- Jurisdiction surprise: utility-owned asset.
- 20 labeled Civic cases; scenario and security tests.
- **No changes to the orchestrator, policy core, approval or executor** other than bug fixes. If the engine needs a change, record an ADR.

**Exit (M2):** Civic 8-minute demo runs from reset; Civic golden suite passes; diff review shows no domain logic leaked into the core.

## Phase 9 — SupplyResolve
Simulator (ERP, inventory, POs, supplier portal), tools, spend/substitute-approval rules, ATP verifier, supplier-decline injector, 20 labeled cases, demo. Exit: pack passes its scenario, verifier and adversarial suites with no core changes beyond ADR-recorded fixes.

## Phase 10 — PropertyResolve
Simulator, tools, emergency-flag classifier with deterministic human escalation (never LLM-only), contractor-decline + adjacent-unit injector, verifier that does not infer physical repair, 20 labeled cases, demo. Exit: emergency scenarios always escalate regardless of model output.

## Phase 11 — ClaimResolve
Simulator, tools, estimate comparison, hard guardrails (no denial, fraud, coverage or payment actions exist in the tool set), adjuster-ready verifier, estimate-conflict injector, 20 labeled cases, demo. Exit: static check proves no tool maps to R4 actions; scenario suite passes.

## Phase 12 — Blast Radius Lite (full) and verifiers
Per-domain relationship models, freshness and missing-relationship caveats, confidence notes displayed in the approval UI, verifier hardening across packs. Exit: impact previews list correct linked entities in fixtures; limitations are shown in the UI.

## Phase 13 — Release hardening and sales packaging
- Pilot hardening (E12 scoping): OIDC, vault, retention, backups, runbooks — only as scoped for pilots.
- Five resettable demos, runbooks, demo scripts, recorded 3-minute videos.
- Content kit per PDF ch.16 (one-pagers, security factsheet, ROI worksheet, pilot questionnaire, SOW template) — owned by product/sales with engineering review.
- Domain-reviewer sign-off per pack before any public sales use.

**Exit:** release gates in spec §16 pass; five demos reproducible from reset by someone who did not build them.

---

## Phase-gate checklist (apply to every phase)
- [ ] Exit criteria pass as automated tests where possible
- [ ] `make lint` and `make test` green; no skipped safety tests
- [ ] Hard rules in CLAUDE.md still hold
- [ ] ADR written for any deviation from spec.md
- [ ] spec.md, CLAUDE.md and this file updated to match reality
