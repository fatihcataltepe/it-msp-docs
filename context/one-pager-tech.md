# High-level technical buckets

Below is a concise decomposition of the system into primary functional buckets. For each bucket I state: responsibilities, inputs/outputs, and the **precise role LLMs should play** (major/moderate/minor/none), plus short notes on constraints and safety.

---

# 1. Ingestion & Normalization

**Responsibilities:** collect tickets, PSAs, runbooks, docs, inventories, agent telemetry, and external events; normalize into canonical formats.
**Inputs:** email/ticket payloads, PSA webhooks, RMM events, docs (PDF/HTML/Markdown), IdP metadata.
**Outputs:** normalized ticket object, client profile updates, document index.
**LLM role:** **Minor** — light text cleaning, summarization of long docs, metadata extraction from semi-structured text.
**Notes:** prefer deterministic parsers for structured fields; use LLMs only for noisy text extraction.

---

# 2. Context & Knowledge Layer (Client Profiles + KB)

**Responsibilities:** maintain per-client profiles (tools, policies, permissions), canonical KB, embeddings for retrieval, versioned runbooks, asset inventory.
**Inputs:** normalized docs, inventory syncs, approved automations, past run history.
**Outputs:** RAG-ready context for downstream components, policy constraints.
**LLM role:** **Major (via RAG)** — retrieve-and-rank relevant docs, summarize policies into machine-readable constraints, map ambiguous tickets to client-specific policy rules.
**Notes:** store dense vectors (vector DB) + deterministic metadata; always include provenance for retrieved facts.

---

# 3. Triage & Classification

**Responsibilities:** classify intents, priority, SLA mapping, route tickets, detect similarity to existing automations.
**Inputs:** normalized ticket, client profile, recent ticket history.
**Outputs:** canonical intent, candidate automation IDs, routing decisions.
**LLM role:** **Major** — intent classification, similarity scoring, contextual intent normalization.
**Notes:** combine LLM confidence with deterministic classifiers; treat LLM output as probabilistic and add thresholds for auto-actions.

---

# 4. Natural-Language Interaction / Engineer Assistant

**Responsibilities:** conversational interface for engineers to query and instruct the system in NL; present suggested automations and receive approvals.
**Inputs:** engineer NL, suggested automations, run logs.
**Outputs:** structured action plan / instruction sequence, approval signals, clarifying questions.
**LLM role:** **Major** — translate NL into stepwise plans, surface required checks, produce human-readable rationale.
**Notes:** use function-calling / typed outputs (JSON plan) rather than free text; require explicit confirmations for high-impact commands.

---

# 5. Automation Authoring (Draft Generation)

**Responsibilities:** convert resolved workflows into reusable automations (scripts/workflows/playbooks), generate pre/post-checks, rollback steps, tests.
**Inputs:** captured action logs, step traces, context from KB, policy constraints.
**Outputs:** automation draft (templated code/workflow), test cases, metadata (inputs, permissions needed).
**LLM role:** **Major** — step extraction, code/workflow generation, test scaffolding, commentary and rationale.
**Notes:** output must follow strict templates; run static analysis and sandboxed unit tests before human review.

---

# 6. Execution Plane (Connectors & Runners)

**Responsibilities:** run automations against client environments via connectors (APIs, agents), manage secrets, enforce RBAC, execute rollback.
**Inputs:** approved automation artifacts, runtime inputs, client credentials.
**Outputs:** execution logs, results, success/failure signals.
**LLM role:** **None/Minor** — not for execution. LLMs may produce human-readable summaries of execution results but must NOT execute actions directly.
**Notes:** deterministic, auditable execution engine with retry/circuit-breaker; explicit separation between planner (LLM) and executor.

---

# 7. Orchestration & Policy Engine

**Responsibilities:** decide auto-run vs require approval, enforce client policy and compliance, schedule canaries, manage multi-step orchestrations across systems.
**Inputs:** policy rules, automation metadata, triage outputs, runtime state.
**Outputs:** go/no-go decisions, execution plans, audit triggers.
**LLM role:** **Minor** — suggest policy exceptions or recommend policy refinement; final enforcement must be rule-based.
**Notes:** policy engine must be authoritative and deterministic; LLM recommendations treated as suggestions.

---

# 8. Observability, Logging & Auditability

**Responsibilities:** store execution logs, change history, approvals, provenance; surface dashboards and alerts.
**Inputs:** execution logs, automation versions, approvals, ticket history.
**Outputs:** searchable logs, compliance reports, audit trails.
**LLM role:** **Minor** — log summarization, automated postmortems, human-readable RCA drafts.
**Notes:** preserve raw logs for forensic work; LLM outputs must cite log offsets and IDs.

---

# 9. Feedback, Similarity Discovery & Candidate Recommendation

**Responsibilities:** detect recurring patterns across tenants, recommend new automations, prioritize candidate backlog.
**Inputs:** ticket clusters, automation success/failure metrics, embeddings.
**Outputs:** ranked automation candidates, prioritization signals, training data for improvements.
**LLM role:** **Major** — cluster interpretation, cross-tenant pattern matching, draft candidate automations and rationale.
**Notes:** always present confidence and proposed human verification plan.

---

# 10. Security, Governance & Compliance

**Responsibilities:** secrets handling, least-privilege enforcement, approval gating for sensitive actions, compliance retention policies.
**Inputs:** automation permission requirements, client compliance profiles, secrets vault.
**Outputs:** enforced RBAC, audit-ready reports, alerts on risky automations.
**LLM role:** **None** for enforcement; **Minor** for recommending safer permission scoping or policy text refinements.
**Notes:** never grant LLMs access to raw secrets; all privileged operations must go through audited connectors.

---

# 11. Platform Infrastructure & Model Ops

**Responsibilities:** host models, manage vector DB, scaling, telemetry, CI/CD for automations and connectors, model versioning and evaluation.
**Inputs:** training data, usage telemetry, model performance metrics.
**Outputs:** model updates, embedding refreshes, runtime endpoints.
**LLM role:** **N/A** (infrastructure supports LLMs).
**Notes:** prefer hybrid hosting (in-cloud hosted models for non-sensitive workloads, private model infra for sensitive tenant data).

---

# Quick mapping: Bucket → LLM role (summary)

* Ingestion & Normalization: **Minor**
* Context & Knowledge: **Major** (RAG)
* Triage & Classification: **Major**
* NL Assistant: **Major**
* Automation Authoring: **Major**
* Execution Plane: **None**
* Orchestration & Policy Engine: **Minor**
* Observability & Audit: **Minor**
* Feedback/Similarity Discovery: **Major**
* Security/Governance: **None/Minor**
* Platform Infra / Model Ops: **N/A**

---

# Allowed LLM responsibilities (rules-of-engagement)

* **Allowed:** intent classification, step extraction, RAG-based retrieval and summarization, plan generation (typed JSON), automated test scaffolds, failure diagnosis suggestions, candidate ranking.
* **Disallowed:** direct execution of actions, accessing secrets, making irreversible changes without explicit human-approved deterministic execution steps, unsandboxed code execution.
* **Constraint patterns:** function-calling with typed outputs, templates for generated code, deterministic validators, sandboxed test harness, approval gates for high-impact automations.

---

# Risk mitigations & design patterns

* **Planner/Executor separation:** LLMs propose and justify; deterministic runners execute.
* **Typed interfaces:** LLMs return JSON plans conforming to strict schema; validators reject non-conforming outputs.
* **RAG with provenance:** every retrieval includes source IDs and confidence.
* **Canary & test runs:** new automations run in sandbox/limited scope before full auto-run.
* **Human-in-loop thresholds:** auto-run only when confidence + historical success exceed configurable thresholds.
* **Audit-first design:** immutable logs, automation versioning, signer identities.

