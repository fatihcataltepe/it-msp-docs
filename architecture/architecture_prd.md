# System Architecture PRD – Classifier-First Ticket Automation

## 1. Architecture Overview

This PRD describes a ticket automation system based on the **Classifier-First Pattern** where classification determines which tickets get automated and which stay manual.

### Core Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Ticket Created (HaloPSA)                                      │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────┐                                          │
│  │  CLASSIFIER      │                                          │
│  │  SERVICE         │──→ LLM (OpenAI)                          │
│  │                  │                                          │
│  │  Decision Gate   │                                          │
│  └────────┬─────────┘                                          │
│           │                                                    │
│      ┌────┴─────────────┐                                     │
│      │                  │                                     │
│  ┌───▼────┐        ┌────▼────┐                               │
│  │auto=YES│        │auto=NO  │                               │
│  │playbook│        │(manual) │                               │
│  └───┬────┘        └─────────┘                               │
│      │                                                       │
│      ▼                                                       │
│  ┌──────────────────┐                                       │
│  │ ORCHESTRATION    │                                       │
│  │ SERVICE          │                                       │
│  │ (Playbook Engine)│                                       │
│  └────────┬─────────┘                                       │
│           │                                                 │
│   ┌───────┼───────┬──────────┐                             │
│   │       │       │          │                             │
│   ▼       ▼       ▼          ▼                             │
│  [VERIFICATION]  [EXECUTOR]  [DELIVERY]                    │
│   (Azure AD)     (Azure AD)  (Twilio)                      │
│                                                            │
│  Results written back to HaloPSA                           │
│                                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Component Responsibilities

### 2.1 HaloPSA (Ticket System - External)

**Role:** Source of truth for all tickets

**Responsibilities:**
- Ingests tickets (email/portal)
- Sends `ticket.created` webhook
- Stores custom fields: `automation_status`, `playbook_id`, `automation_confidence`
- Displays ticket status and audit trail to L1 engineers
- Receives updates from our system (state, results, escalations)

**Integration:** Webhook → Classifier → back-update via API

---

### 2.2 Classifier Service (Internal)

**Role:** Decision Gate – determines if/how to automate

**Responsibilities:**
1. Receive ticket data from HaloPSA webhook
2. Call LLM (OpenAI) to classify intent and confidence
3. Fetch user context from Azure AD (manager, department, status)
4. Query Playbook Service to check if playbook exists
5. Apply blocker rules (disabled accounts, exclusions, etc.)
6. Return: `automated: true/false`, `playbook_id`, `context`

**Inputs:**
- ticket_id, subject, body, user_email (from HaloPSA)

**Outputs:**
- `automated` (BOOLEAN)
- `playbook_id` (if automated)
- `context` (enriched user data)
- `confidence` (0.0-1.0)
- `reasoning` (why yes/no)

**External Integrations:**
- OpenAI API (LLM classification)
- Azure AD Graph API (user enrichment)

---

### 2.3 Playbook Service (Internal)

**Role:** Playbook Registry – stores and serves workflow definitions

**Responsibilities:**
1. Store playbook definitions (YAML format)
2. Serve playbooks to Orchestration Service
3. Version management (v1.0, v1.1, etc.)
4. Enable/disable playbooks
5. Validate playbook syntax
6. Provide CRUD endpoints for playbook management

**Outputs:**
- Playbook definitions with step sequences, retry logic, timeouts

**No external integrations** (internal only)

---

### 2.4 Orchestration Service (Internal)

**Role:** Playbook Executor – runs automation workflows

**Responsibilities:**
1. Receive execution request: `ticket_id`, `playbook_id`, `context`
2. Load playbook definition from Playbook Service
3. Execute steps sequentially
4. Call appropriate microservice for each step
5. Handle timeouts and retry logic per playbook definition
6. Escalate on failure
7. Update HaloPSA with progress and results
8. Log complete audit trail

**Inputs:**
- ticket_id, playbook_id, context (from Classifier)

**Outputs:**
- Step execution results
- Terminal state (RESOLVED or ESCALATED)
- Updates to HaloPSA

**External Integrations:**
- HaloPSA API (status updates)

---

### 2.5 Verification Service (Internal Microservice)

**Role:** Identity verification – MFA and manager approval

**Sub-modules:**
1. **MFA Verification**
   - Trigger MFA push via Azure AD
   - Poll for approval
   - Timeout handling (fallback to manager approval)

2. **Manager Approval**
   - Send approval request via Teams
   - Fallback to SMS (Twilio) after 5 minutes
   - Handle approval/denial webhooks

**External Integrations:**
- Azure AD (MFA push, user identity)
- Microsoft Teams (adaptive cards)
- Twilio (SMS fallback)

---

### 2.6 Executor Service (Internal Microservice)

**Role:** Execute remediation – password resets, provisioning, etc.

**Sub-modules:**
1. **Password Reset Executor**
   - Generate Temporary Access Pass (TAP)
   - Apply to Azure AD
   - Return TAP for delivery

2. **(Future) Access Provisioner**
   - Grant application access
   - Manage RBAC roles

3. **(Future) Device Diagnostics**
   - Trigger RMM scripts
   - Collect logs

**External Integrations:**
- Azure AD (password reset, TAP creation)

---

### 2.7 Delivery Service (Internal Microservice)

**Role:** Credential delivery – send TAP/password to user

**Responsibilities:**
1. Send credential via SMS or email
2. Retry on failure
3. Log delivery status

**External Integrations:**
- Twilio (SMS)

---

### 2.8 Database Layer (Internal)

**Stores:**
- **tickets** – ticket records with automation status
- **classification_results** – every classification decision
- **playbook_executions** – playbook run records
- **playbook_step_executions** – individual step results
- **playbooks** – playbook definitions (YAML)

---

### 2.9 Azure AD (External)

**Role:** Identity provider and action executor

**Used by:**
- Classifier (user enrichment: manager, department, enabled status)
- Verification (MFA push, approval status)
- Executor (password reset, TAP creation)

**Integrations:**
- Microsoft Graph API (read user, trigger MFA, reset password)

---

### 2.10 OpenAI (External)

**Role:** LLM for ticket classification

**Used by:**
- Classifier (classify ticket category, sub-category, confidence)

**Integration:**
- OpenAI API (GPT-4)

---

### 2.11 Microsoft Teams (External)

**Role:** Manager approval notifications

**Used by:**
- Verification Service (send adaptive card to manager)
- Receives approval/denial via webhook

**Integration:**
- Teams API (send message, receive card actions)

---

### 2.12 Twilio (External)

**Role:** SMS delivery and SMS-based approvals

**Used by:**
- Verification Service (SMS fallback for manager approval)
- Delivery Service (SMS delivery of credentials)

**Integration:**
- Twilio REST API (send SMS, receive SMS responses)

---

## 3. Data Flow

### 3.1 Happy Path: Password Reset Automation

```
1. USER
   └─→ Emails support: "I can't log in"

2. HALOPSA
   ├─→ Creates ticket HLP-12345
   └─→ POST /webhooks/ticket_created

3. CLASSIFIER SERVICE
   ├─→ Receives webhook
   ├─→ Calls OpenAI: classify subject + body
   ├─→ Result: PASSWORD_RESET (confidence 0.95)
   ├─→ Calls Azure AD: fetch user manager, department, status
   ├─→ Checks Playbook Service: playbook exists? YES
   ├─→ Checks blockers: user enabled? YES
   └─→ Returns: automated=true, playbook_id=password_reset, context={...}

4. ORCHESTRATION SERVICE
   ├─→ Receives: ticket_id, playbook_id, context
   ├─→ Loads playbook: password_reset.yml v1.2
   └─→ Executes steps:

       STEP 1: verify_mfa
       ├─→ Calls Verification Service
       ├─→ Calls Azure AD: send MFA push
       ├─→ Result: USER APPROVED in 2 minutes
       └─→ Next step: reset_password

       STEP 2: reset_password
       ├─→ Calls Executor Service
       ├─→ Calls Azure AD: create TAP
       ├─→ Result: TAP created, valid 15 min
       └─→ Next step: deliver_credentials

       STEP 3: deliver_credentials
       ├─→ Calls Delivery Service
       ├─→ Calls Twilio: send SMS "Your TAP: XXXXX"
       ├─→ Result: SMS delivered
       └─→ Next step: resolved

5. HALOPSA
   ├─→ Orchestration updates ticket
   ├─→ Status: RESOLVED
   ├─→ Audit log: [Classify → Verify → Reset → Deliver → Resolved]
   └─→ Total time: 10 minutes

6. USER
   └─→ Receives SMS with TAP, logs in
```

---

### 3.2 Fallback Path: MFA Timeout → Manager Approval

```
STEP 1: verify_mfa
├─→ Calls Azure AD: send MFA push
├─→ Waits 300 seconds (5 minutes)
├─→ NO USER RESPONSE
└─→ Result: TIMEOUT

FALLBACK TO STEP 2: verify_manager
├─→ Calls Verification Service
├─→ Calls Teams: send approval card to jane.smith@company.com
├─→ Waits 300 seconds
├─→ Manager approves at 4 minutes
└─→ Result: APPROVED

CONTINUE STEP 3: reset_password
├─→ Calls Executor Service
├─→ Calls Azure AD: create TAP
└─→ Continue normally...
```

---

### 3.3 Non-Automation Path: No Playbook Defined

```
1. USER
   └─→ Emails: "My laptop screen is broken"

2. CLASSIFIER SERVICE
   ├─→ Receives: hardware_issue ticket
   ├─→ Calls OpenAI: classify
   ├─→ Result: HARDWARE_ISSUE (category: INCIDENT)
   ├─→ Checks Playbook Service: playbook exists? NO
   └─→ Returns: automated=false, reason="No playbook defined"

3. HALOPSA
   ├─→ Updates ticket
   ├─→ automation_status: "manual"
   └─→ Ticket stays in L1 queue (untouched by us)

4. L1 ENGINEER
   └─→ Handles manually
```

---

## 4. External Tools (Max 5 Selected)

| Tool | Purpose | Used By | Integration Type |
|------|---------|---------|------------------|
| **Azure AD** | User identity, MFA, password reset | Classifier, Verification, Executor | Graph API REST |
| **OpenAI** | Ticket classification (LLM) | Classifier | REST API |
| **Twilio** | SMS delivery + SMS approvals | Verification, Delivery | REST API |
| **Microsoft Teams** | Manager approval notifications | Verification | Teams API REST |
| **HaloPSA** | Ticket system (source of truth) | Classifier, Orchestration | REST API + Webhooks |

---

## 5. Playbook Format (YAML Example)

```yaml
# playbooks/password_reset.yml
name: password_reset
version: "1.2"
enabled: true
description: "Automated password reset with MFA or manager approval"

category: REQUEST
sub_category: PASSWORD_RESET

steps:
  
  - id: verify_mfa
    name: "Verify User via MFA"
    service: verification.mfa
    timeout: 300
    on_success: reset_password
    on_failure: verify_manager
    
  - id: verify_manager
    name: "Request Manager Approval"
    service: verification.manager
    timeout: 600
    on_success: reset_password
    on_failure: escalate_manager_timeout
    
  - id: reset_password
    name: "Reset Password"
    service: executor.reset_password
    timeout: 60
    retry:
      max_attempts: 3
      backoff: exponential
    on_success: deliver_credentials
    on_failure: escalate_reset_failed
    
  - id: deliver_credentials
    name: "Deliver TAP"
    service: delivery.sms
    timeout: 60
    retry:
      max_attempts: 3
    on_success: resolved
    on_failure: escalate_delivery_failed
    
  - id: resolved
    type: terminal
    status: RESOLVED
    
  - id: escalate_manager_timeout
    type: escalation
    reason: "Manager approval timeout"
```

---

## 6. System Diagram Components (for Figma)

### Swimlanes (Left to Right)
1. **HaloPSA** (external)
2. **Classifier Service** (internal)
3. **Playbook Service** (internal)
4. **Orchestration Service** (internal)
5. **Microservices** (Verification, Executor, Delivery)
6. **External Tools** (Azure AD, OpenAI, Teams, Twilio)
7. **Database** (internal)

### Key Boxes
- Classifier Service (decision logic)
- Playbook Service (workflow registry)
- Orchestration Service (executor)
- Verification Service (MFA + Manager)
- Executor Service (password reset)
- Delivery Service (SMS)
- Azure AD (identity)
- OpenAI (classification)
- Twilio (SMS)
- Microsoft Teams (approvals)
- HaloPSA (tickets)

### Arrows
1. HaloPSA → Classifier (webhook: ticket.created)
2. Classifier → OpenAI (LLM call)
3. Classifier → Azure AD (user enrichment)
4. Classifier → Playbook Service (check playbook exists)
5. Classifier → HaloPSA (update automation_status)
6. Classifier → Orchestration (execute_playbook)
7. Orchestration → Playbook Service (load playbook)
8. Orchestration → Verification (execute step)
9. Orchestration → Executor (execute step)
10. Orchestration → Delivery (execute step)
11. Verification → Azure AD (MFA push)
12. Verification → Teams (send approval card)
13. Verification → Twilio (SMS fallback)
14. Executor → Azure AD (password reset)
15. Delivery → Twilio (send SMS)
16. All services → Database (logging)
17. Orchestration → HaloPSA (status updates)

---

## 7. API Endpoints Summary

### Classifier Service
- `POST /api/v1/classify_ticket` – Classify and decide automation

### Playbook Service
- `GET /api/v1/playbooks/{playbook_id}` – Get playbook definition
- `POST /api/v1/playbooks` – Create new playbook (admin only)
- `PUT /api/v1/playbooks/{playbook_id}/activate` – Activate version

### Orchestration Service
- `POST /api/v1/orchestration/execute_playbook` – Start playbook execution
- `GET /api/v1/orchestration/execution/{ticket_id}` – Get execution status

### Verification Service
- `POST /api/v1/verification/mfa` – Trigger MFA
- `POST /api/v1/verification/manager_approval` – Request manager approval

### Executor Service
- `POST /api/v1/executor/reset_password` – Reset password

### Delivery Service
- `POST /api/v1/delivery/sms` – Send SMS

### Webhooks (Inbound)
- `POST /webhooks/halopsa/ticket_created` – From HaloPSA
- `POST /webhooks/teams/approval_action` – From Teams (approval/denial)
- `POST /webhooks/twilio/sms_response` – From Twilio (SMS responses)

---

## 8. Database Tables

| Table | Purpose |
|-------|---------|
| `tickets` | Ticket records with automation status |
| `classification_results` | Every classification decision |
| `playbooks` | Playbook definitions (YAML) |
| `playbook_executions` | Playbook run records |
| `playbook_step_executions` | Individual step execution results |

---

## 9. Key Design Principles

1. **Classifier-First:** Classification determines automation, not the other way around
2. **Safe Non-Automation:** Unautomatable tickets never touched by our system
3. **Playbook-Driven:** All automation flows defined in YAML, no hardcoded logic
4. **Modular Services:** Each service independent, scalable, replaceable
5. **HaloPSA Source of Truth:** All ticket data lives in HaloPSA
6. **Audit Trail:** Every decision and step logged
7. **Graceful Fallbacks:** MFA fails → manager approval; manager approval fails → escalate

---

## 10. Deployment Model

- **Classifier Service:** Lambda or containerized, called synchronously from webhook
- **Playbook Service:** REST API, stateless, reads from database
- **Orchestration Service:** Long-running processes (can be Lambda with SQS or Kubernetes)
- **Microservices:** Lambda or containerized, called by orchestration
- **Database:** PostgreSQL with read replicas
- **Caching:** Redis for idempotency keys (24h TTL)
