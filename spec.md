# Prudlelabs OpsPilot — Engineering Specification

Source: `Prudlelabs_OpsPilot_Implementation_and_GoToMarket_Blueprint.pdf` (v1.0, 20 Oct 2026), distilled for engineering.
Items marked **[Decision]** are choices made beyond the PDF. Items marked **[TBD]** need an owner's answer.
Brand spelling everywhere (UI, docs, code, campaigns): **Prudlelabs OpsPilot**.

---

## 1. Purpose

OpsPilot is a reusable runtime for AI agents that resolve cross-system operational exceptions. A user or event opens a **case**. The agent investigates across approved business systems, proposes a plan, and proposes protected actions. Deterministic infrastructure decides whether each action may commit. After execution, the system verifies the result in the authoritative system of record.

One shared engine. Five **domain packs** (Civic, Supply, Freight, Claims, Property) configure it. Business systems are **stateful simulators** with fault injection. Everything is synthetic data.

### Non-goals (release 1)
- No autonomous legal, municipal, insurance-coverage or claims-settlement decisions.
- No replacement for ERP, TMS, municipal case management or insurance core systems.
- No proprietary tracing platform. Consume OpenTelemetry; write business-action evidence ourselves.
- No generic low-code multi-agent builder, marketplace or unbounded autonomy.
- No claim that simulated performance proves production ROI or regulatory compliance.

## 2. Core principles

1. **Agents propose; trusted code commits.** The LLM has no tool that writes to a business system. It can only emit an `ActionProposal`.
2. **Deterministic where it matters.** Arithmetic, payment/spend limits, permissions, SLA dates, duplicate checks and DB commits are code, never LLM output.
3. **Verified outcomes.** A green trace is not a business outcome. Always query authoritative state afterward. `PENDING`/`UNKNOWN` is never success.
4. **Safe escalation is a success.** Prefer a correct handoff to a fabricated completion or unsafe action.
5. **One agent per case.** Specialist services (search, optimization, policy, notifications) are tools, not extra agents.
6. **One engine.** No per-domain forks. Domain packs plug into shared event, action, approval and verifier contracts.
7. **Untrusted input stays data.** Emails, tool outputs and documents can never change policy or instructions.

## 3. What counts as agentic (acceptance lens)

| | Ordinary workflow | OpsPilot target |
|---|---|---|
| Control flow | Predefined steps | Agent selects investigative steps from tools in scope |
| Novel exception | Raise error | Reassess, re-plan, seek permission or escalate |
| Cross-system | Sequence known APIs | Choose systems, reconcile inconsistent answers |
| Writes | Triggered by conditions | Proposed by agent, validated by guardrails |
| Result | API response recorded | Independent business-state evidence verified |

A scenario that cannot show a *changed plan after a surprise* is a scripted demo and fails acceptance.

## 4. Architecture

### 4.1 Logical services

| Service | Responsibility | Implementation |
|---|---|---|
| Web console | Case queue, timeline, approvals, evidence, demo studio | Next.js, TypeScript, Tailwind |
| API | Case endpoints, tenant routing, auth, audit | FastAPI |
| Case orchestrator | Durable state transitions, interrupt/resume | Postgres state machine + background worker |
| Agent runtime | Agent loop inside investigate / plan / replan states | LangGraph (see 4.3) |
| Tool registry | Typed tools, scopes, versions, timeouts | Pydantic schemas + registry tables |
| Policy / impact | Rules, approval-required, related entities | Python rule layer (OPA/Cedar only if justified) |
| Executor | Trusted write adapters, idempotency, CAS | Python, per connector |
| Verification | Expected vs authoritative actual state | Per-domain verifiers |
| Simulators | Deterministic stateful mock business apps + fault injection | FastAPI + Postgres |
| Observability | Traces, metrics, action ledger, eval | OpenTelemetry + business ledger tables |

### 4.2 Deployment profile (now)
Docker Compose only: web, api, worker, postgres, redis, simulator, (otel collector optional). **[Decision]** Hosted demo and customer-pilot profiles are out of scope until Phase 10+.

### 4.3 State ownership **[Decision]**
Postgres is the single source of truth for case state. LangGraph runs only the agent loop *within* `INVESTIGATING`, `PLAN_READY` and `REPLANNING`. Its checkpoints hold agent-loop scratch state, not case state. Approval, execution and verification live outside LangGraph so they cannot be bypassed by a graph edge.

### 4.4 Simulators **[Decision]**
One `services/simulator` process with a router and schema per domain, not five microservices. Split later only if needed. Each domain module exposes: normal reads/writes, optimistic-concurrency writes, deterministic faults, fixture reset.

### 4.5 Auth **[Decision]**
`tenant_id` on every table from day one. Real OIDC/RBAC deferred to pilot hardening (E12). Until then: seeded users with roles (`operator`, `approver`, `admin`, `sales_presenter`) via a dev-only auth shim that exercises the same role checks.

## 5. LLM layer

- Provider adapter: `LLMProvider.complete(messages, tools, response_schema, budget) -> LLMResult`.
- Default provider: **OpenAI `gpt-4o-mini`**. Model, timeout, max steps, max tokens and per-case cost budget are config.
- Always use strict structured output / JSON schema. Low temperature. No free-text control flow.
- `StubProvider`: scripted responses keyed by scenario step. Used for all spine tests and CI.
- `ReplayProvider`: records and replays real model transcripts for regression runs.
- Known risk: a small model is weakest at re-planning. The adapter must make a model swap a config change, and the eval suite must compare models on the same golden scenarios.

### Agent action space
The agent may emit exactly one of per step:
1. `call_tool` — R0 read tool in scope.
2. `submit_plan` — ranked candidate `ActionProposal`s with evidence refs, assumptions and alternatives.
3. `draft_message` — R1 draft; human preview required before send.
4. `escalate` — reason, evidence refs, what a human must decide.

There is no `execute` action. Execution is triggered only by the orchestrator after policy ALLOW or human approval.

Budgets: max tool calls per run, max replans per case, wall-clock timeout, model cost cap. Exceeding any → `ESCALATED` with reason.

## 6. Case lifecycle

| State | Meaning | Exit |
|---|---|---|
| NEW | Case created | Input validated, owner assigned |
| INVESTIGATING | Agent gathers/cross-checks facts | Enough evidence, or unresolved uncertainty |
| PLAN_READY | Candidate actions with cost/risk/evidence | Policy evaluation done |
| WAITING_APPROVAL | Protected action awaits human | Approve, reject or expire |
| EXECUTING | Trusted adapter performs action | Receipt returned or timeout classified |
| VERIFYING | Fresh authoritative state queried | success / pending / discrepancy / unknown |
| REPLANNING | New info invalidated plan | Replacement option or escalation |
| DONE / ESCALATED | Closed with proof or handoff | Immutable closeout record |

### Transitions **[Decision — inferred from the PDF]**
```
NEW → INVESTIGATING → PLAN_READY
PLAN_READY → (policy ALLOW) → EXECUTING
PLAN_READY → (policy HOLD)  → WAITING_APPROVAL
PLAN_READY → (policy DENY)  → REPLANNING (alternative) | ESCALATED
WAITING_APPROVAL → approve → EXECUTING
WAITING_APPROVAL → reject | expire | state-version-changed → REPLANNING
EXECUTING → receipt → VERIFYING
EXECUTING → timeout/unknown → VERIFYING (reconcile before any retry)
VERIFYING → success → DONE
VERIFYING → pending → VERIFYING (bounded rechecks) → ESCALATED on exhaustion
VERIFYING → discrepancy → REPLANNING | ESCALATED
VERIFYING → unknown → ESCALATED
any non-terminal → ESCALATED (budget exceeded, emergency flag, operator action)
```

### Resume, retries, concurrency
- Persist `case_id`, `thread_id`, `state_version`, `plan_version`, last authoritative observations.
- Every write proposal carries `idempotency_key`, `expected_version`, `purpose`, `expiry`, `actor`.
- After a timeout, query current state before retrying. Never assume a second call is safe.
- Re-validate authorization and preconditions immediately before commit.
- Use optimistic concurrency / compare-and-swap in the business system. A pre-check alone cannot prevent races.
- Approval is bound to a versioned proposal. If target state changes, the approval expires.
- No universal rollback. Model compensating operations and human-supported correction.

## 7. Data model

| Entity | Core fields |
|---|---|
| Tenant | tenant_id, region, roles, settings |
| DomainPack | pack_id, version, tool_scopes, rules, verifier |
| Case | case_id, tenant_id, domain, objective, priority, state, owner, state_version |
| BusinessEntity | entity_type, external_id, source_system, current_version |
| Observation | observed_at, source, fetched_by, data_hash, validity |
| ActionProposal | proposal_id, case_id, tool, arguments, preconditions, impact, proposal_hash |
| PolicyDecision | decision_id, rule_version, result (ALLOW/HOLD/DENY), reasons, expiry |
| Approval | approval_id, actor, proposal_version, status, timestamp |
| Execution | execution_id, idempotency_key, receipt, actual_state |
| Verification | expected_state, observed_state, evidence_refs, status |

Verification status values: `SUCCESS`, `PENDING`, `DISCREPANCY`, `UNKNOWN`.

### Example proposal
```json
{
  "case_id": "case_freight_2084",
  "proposal_id": "act_00031",
  "domain": "freight",
  "tool": "dispatch.assign_replacement_carrier",
  "resource": {"shipment_id": "SHP-2084", "carrier_id": "CAR-44"},
  "preconditions": {"shipment_version": 12, "carrier_available": true},
  "estimated_incremental_cost_cad": 420,
  "idempotency_key": "case_freight_2084:act_00031:v1",
  "approval_required": true,
  "expected_outcome": "carrier_assigned"
}
```

### Business action event envelope (versioned schema, all packs)
`event_id`, `occurred_at` (UTC), `tenant_id`, `case_id`, `trace_id`, `span_id`, `agent_run_id`, `action_id`, `system`, entity references, `event_type`, `status`, `idempotency_key`, `actor`, redacted `payload`, evidence reference. Full sensitive payloads stay out of telemetry by default.

Append-only ledger: events are never updated or deleted.

## 8. Connectors

Minimum interface (each adapter implements):
```
describe() -> capabilities, scopes, version
get(entity_type, external_id) -> record + version + observed_at
search(query, filters) -> records + provenance
propose(action) -> validated proposal + impact inputs
execute(approved_proposal, idempotency_key) -> receipt
verify(receipt, expected_state) -> observed_state + evidence
health() -> connectivity, latency, permissions
```

Modes: read-only API, stateful sandbox API (current), trusted write adapter, outbound webhook (HMAC, replay window, dedup), polling connector, customer-hosted bridge. Only the first three matter until pilot.

Onboarding checklist (per adapter): map IDs and system-of-record precedence; inventory read/write ops with least privilege; define normal/error/partial-commit/timeout/duplicate semantics; choose webhook vs polling; establish idempotency and preconditions (else a safe write wrapper); define authoritative proof of completion and its latency; contract-test against fixtures and fault injections.

## 9. HTTP API (core)

| Route | Purpose |
|---|---|
| `POST /v1/cases` | Create case; validate tenant, pack, objective |
| `GET /v1/cases/{id}` | State, evidence, timeline |
| `POST /v1/cases/{id}/run` | Start/resume workflow |
| `POST /v1/cases/{id}/events` | Insert domain events/signals |
| `GET /v1/cases/{id}/proposals` | Pending proposed actions |
| `POST /v1/approvals/{id}/decisions` | Approve/reject; version-checked |
| `POST /v1/cases/{id}/verify` | Trigger reconciliation |
| `POST /v1/demo/scenarios/{id}/reset` | Idempotent fixture reset (demo only) |
| `POST /v1/demo/scenarios/{id}/inject` | Inject approved failure (demo only) |

Demo routes are disabled outside the demo profile.

## 10. Safety, policy and Blast Radius Lite

### Risk tiers
| Tier | Examples | Minimum control |
|---|---|---|
| R0 Read | Search case, fetch inventory | Scoped identity, retrieval audit |
| R1 Draft | Customer message, plan | Human preview for outbound content |
| R2 Reversible write | Internal follow-up task | Rule check + idempotency |
| R3 External/costly write | Reassign carrier, issue PO | Fresh state, explicit limits, approval as configured |
| R4 High consequence | Payment, legal denial, emergency dispatch | Excluded from demo autonomy; human-owned |

### Enforcement
- Policy evaluates a normalized proposal with business context fetched and time-bounded.
- **ALLOW** → short-lived execution authorization bound to proposal hash, actor and state version.
- **HOLD** → approval task with evidence and expiry. No write occurs.
- **DENY** → auditable explanation. Agent may seek an alternative; it never bypasses.
- Adapters revalidate preconditions inside the authoritative transaction where supported.
- Emergencies escalate to the customer's established response procedures.
- **[Decision]** In the demo, every R3 write is HOLD. Cost thresholds are config values **[TBD: default threshold]**.

### Blast Radius Lite
Not a prediction engine. It traverses only explicitly modeled relationships: proposed action → directly affected entity → related commitments/open transactions. Output: linked entities, possible downstream changes, data freshness, missing relationships, confidence caveats. **[Decision]** The interface (`impact(proposal) -> ImpactPreview`) exists from Phase 2 with a trivial implementation; Freight gets "other deliveries on this vehicle" in Phase 5; full version Phase 11.

## 11. Evidence, observability, evaluation

Three evidence layers: **Execution** (OTel span + receipt), **Business state** (authoritative query), **Operational outcome** (acceptance/completion evidence).

Required measurements: resolution rate and safe-escalation rate against labeled set; tool/argument correctness; unsupported assertions; unauthorized write attempts; stale-version conflicts; duplicate attempts; verification coverage and mismatch detection; recovery success, replans, human interventions; duration, model/tool cost, latency, queue wait.

### Golden scenarios (every pack)
Happy path · Missing system data · Tool 500/timeout · Stale resource · Human rejects · Unexpected second actor · Prompt injection in a supplier/vendor email · Outcome not observed.

Seed ≥20 cases per domain: 8 straightforward, 8 exception-heavy, 4 adversarial, each with a deterministic expected end-state label. Re-run the suite on any prompt, tool, model or agent change.

## 12. Domain packs

Each pack provides: tools, fixtures, invariants, rules, verifier, injectors, scenario tests.

### 12.1 FreightResolve (first)
- Buyer: 3PL operations / dispatch director.
- Adapters: TMS, vehicle/telematics alerts, rate table, carrier availability, CRM, dispatch.
- Tools: `get_shipment`, `get_vehicle_alert`, `get_carriers`, `quote_alternative`, `reserve_carrier`, `update_dispatch`, `notify_customer`.
- Tiers **[Decision]**: reads and quotes R0; `notify_customer` R1; `update_dispatch` R2; `reserve_carrier` R3.
- Guardrails: carrier eligibility; cargo handling; carrier conflicts; approval above cost threshold.
- Verifier: carrier acknowledgment + updated TMS assignment + continuing delivery status.
- Fixtures: 50 shipments, 18 trucks, 20 carriers, 10 rate contracts.
- Surprise: chosen carrier rejects after the plan is drafted.
- KPIs: time-to-recovery plan; accepted replacements; incremental cost.
- Demo line: "A truck with three urgent deliveries breaks down."

### 12.2 CivicResolve (second)
- Buyer: 311 / public-works lead.
- Adapters: 311 CRM, asset registry, GIS jurisdiction lookup, crew dispatch, contractor queue, resident notification.
- Tools: `search_cases`, `get_asset_owner`, `get_work_orders`, `get_crew_capacity`, `propose_reassignment`, `update_case`, `notify_resident`.
- Guardrails: jurisdiction and asset-owner rules; human sign-off for priority changes; no automated legal/by-law decisions.
- Verifier: accepted assignment + latest work-order status; physical completion only after technician evidence.
- Fixtures: 70 complaints, 25 assets, 12 crews, 15 work orders, overlapping tickets.
- Surprise: asset is owned by a utility, not the municipality.
- KPIs: time-to-assignment; duplicate cases prevented; false closures.
- Demo line: "This streetlight complaint has not moved in three days."

### 12.3 – 12.5 Later packs (summary)
- **SupplyResolve**: ERP POs, inventory, production schedule, supplier portal. Surprise: preferred supplier rejects expedite. Verifier: transfer/supplier acceptance + updated ATP + schedule impact. Fixtures: 50 SKUs, 6 suppliers, 15 work orders, 3 warehouses, 100 open POs.
- **ClaimResolve**: claims system, policy records, repair-estimate portal. Never denial, fraud, coverage or payment. Surprise: new estimate contradicts inspection notes. Verifier: required docs present + discrepancies recorded + adjuster-ready (not settlement). Fixtures: 45 claims, 30 estimates, 14 repair facilities.
- **PropertyResolve**: property/tenant records, contractor schedules, work orders. Immediate human escalation for emergencies; do not infer physical repair. Surprise: contractor declines and adjacent unit reports a similar leak. Fixtures: 60 units, 15 contractors, 35 work orders, 8 emergencies.

## 13. Demo controller and operator UI

### Universal 8-minute script
0:00–0:45 problem and cost · 0:45–2:15 start case, investigate ≥3 systems · 2:15–3:30 plan with evidence and alternatives · 3:30–4:30 inject complication, observe re-plan · 4:30–5:45 policy hold, approval or safe escalation · 5:45–7:15 guarded write, query system of record · 7:15–8:00 result, audit evidence, KPI baseline.

### Demo studio must support
Start scenario; choose industry; guided or autonomous mode; inject exception; pause/resume; view tool execution; approve/reject; reset to known fixture; export case evidence. A presenter must run a full demo with no DB editing or developer help.

### Screens
Industry launcher · Case inbox · Case workspace · Plan + impact · Execution trail · Outcome verification · Demo studio · Ops & evaluation.

### Usability standards
Plain operational language ("carrier accepted", not "200 OK"). Pending/unknown displayed separately from failed and completed. Approvals show action, affected entity, cost, risk, expiry and evidence. Important facts link to source record + timestamp. Keyboard accessible, readable contrast, screen-reader semantics, clear error recovery.

## 14. Security and privacy (minimum)
- Tenant isolation on every case, tool, entity and evidence record.
- Least-privilege connectors; separate read and write credentials.
- No internet, shell or file-system tools for agents.
- Prompt-injection defenses on documents, search results and tool outputs.
- PII redaction in telemetry; encrypted storage/transport; region and retention settings.
- Exportable audit evidence; runbooks.
- Deferred to pilot hardening: OIDC/SSO, managed vault, key rotation, backups, retention enforcement.
- The synthetic demo does not establish PIPEDA, AODA, municipal procurement or customer security compliance.

## 15. Repository layout
```
opspilot/
  apps/web/                 Next.js operator + sales UI
  services/api/             FastAPI tenant and case endpoints
  services/agent-runtime/   LangGraph workers + checkpoints
  services/policy/          rules + approvals + impact
  services/simulator/       stateful mock business apps
  packages/contracts/       Pydantic + generated TS types
  packages/connectors/      adapters, auth, retries
  domain-packs/{civic,supply,freight,claims,property}/
  tests/scenarios/          happy, fault, concurrency, red-team
  infra/docker/             compose, migrations, seed data
  docs/                     ADRs, runbooks, demo scripts
```
Engineering defaults: pin dependencies; version prompts and tools; Alembic migrations; feature flags per domain pack; no untracked in-process background work; CI gates for unit, integration, scenario and adversarial tests; demo data stores never shared with future customer systems.

## 16. Epics and acceptance

| Epic | Pri | Acceptance |
|---|---|---|
| E01 Tenant, user, identity | P0 | Tenant-isolated API, role checks, audit events |
| E02 Case lifecycle | P0 | Persist/resume, versions, owner, timeline |
| E03 Tool registry + contracts | P0 | Typed I/O, scopes, failure classes |
| E04 Agent planning | P0 | Evidence-backed plan, bounded tool selection |
| E05 Approval and guards | P0 | No protected write without valid approval/version |
| E06 Connector simulator | P0 | Stateful reads/writes, deterministic seeds |
| E07 Outcome verifier | P0 | Independent state fetch; mismatch flagged |
| E08 Demo injector/reset | P0 | Named faults replay reliably |
| E09 Industry packs | P1 | Each pack passes scenario + verifier tests |
| E10 Impact preview | P1 | Correct linked entities + freshness caveats |
| E11 Evaluation/telemetry | P0 | Trace, business event, labeled golden tests |
| E12 Pilot hardening | P2 | Secrets, SSO, retention, security review, backups |

### Test matrix
Unit (authz rules, state transitions, extraction schema, amount/time arithmetic) · Contract (tool schemas, ID mapping, error parsing, tenant boundaries) · Integration (adapters vs simulator with injected faults) · E2E (create case, view plan, reject/approve, verify, export evidence) · Concurrency (two cases, same carrier/stock; one valid allocation) · Security (prompt injection, forged approvals, replayed webhook, cross-tenant access, tampered version) · Regression (full golden suite on any prompt/model/tool change).

### Go / no-go gates (targets, not observed results)
- Technical: ≥90% of labeled safe cases reach correct outcome or safe escalation.
- Safety: **zero** unauthorized critical writes in adversarial and concurrency tests.
- Maintainability: shared engine; no copy-pasted domain fork.
- Buyer/commercial gates apply to pilot conversion, not engineering.

## 17. First milestone
One full FreightResolve case: detect breakdown → investigate shipment and carrier systems → propose alternatives → inject carrier rejection → manager approval → valid reassignment → verify acceptance in simulated TMS. Then CivicResolve on the same engine with zero duplicated orchestration.

## 18. Open questions
- **[TBD]** Default approval cost threshold for Freight.
- **[TBD]** Who is the Canadian domain reviewer for each pack (required before public sales).
- **[TBD]** Per-case model budget for `gpt-4o-mini` (cost cap, max steps).
- **[TBD]** Whether the sales presenter UI and the operator UI share one app or split by role.
- **[TBD]** Job queue vs durable workflow engine (Celery/RQ/Arq vs Temporal). Default assumption: Redis-backed worker, revisit if resume logic grows complex.
