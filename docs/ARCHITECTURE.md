# Custodian — Architecture Diagrams

Visual guide to how Custodian works. Every diagram below renders directly on
GitHub (Mermaid). The original hand-drawn board is `Custodian.excalidraw` in
the repo root — open it at [excalidraw.com](https://excalidraw.com) (menu → Open).

1. [The big picture](#1-the-big-picture)
2. [The agent pipeline](#2-the-agent-pipeline-one-invoice-start-to-finish)
3. [The six governance layers](#3-the-six-governance-layers)
4. [What happens before money moves](#4-what-happens-before-money-moves)
5. [Sandboxed OCR for invoice images](#5-sandboxed-ocr-for-invoice-images)
6. [Approval rules (Cedar + Dogwood)](#6-approval-rules-cedar--dogwood)
7. [Kill switch and tamper-proof audit log](#7-kill-switch-and-tamper-proof-audit-log)

---

## 1. The big picture

A human submits an invoice in the console. Four AI agents process it inside
the backend. Every consequential action is checked by the policy service,
and everything is written to an audit log that can prove what happened.

```mermaid
flowchart LR
    user(["👤 AP clerk / Controller / CFO"])

    subgraph app["Application"]
        console["custodian-console<br/>Next.js dashboard"]
        backend["custodian-backend<br/>FastAPI + LangGraph<br/>4 AI agents"]
        ledger["custodian-ledger<br/>double-entry ledger<br/>+ vendor records"]
        runner["custodian-sandbox-runner<br/>launches OCR sandbox"]
    end

    subgraph guard["Guardrails"]
        policy["policy-service<br/>Cedar + Dogwood<br/>/authorize"]
        cp["custodian-control-plane<br/>kill switch + heartbeats"]
        audit["custodian-audit-log<br/>SHA-256 hash chain"]
    end

    subgraph ai["AI gateway"]
        litellm["LiteLLM proxy<br/>per-agent keys + budgets"]
        openai["OpenAI<br/>reasoning route"]
        groq["Groq<br/>routine + guardrail routes"]
    end

    keycloak["Keycloak<br/>human login"]
    minio[("MinIO<br/>Object Lock")]
    pg[("Postgres<br/>checkpoints + ledger")]

    user --> console
    console -- login --> keycloak
    console -- submit / approve --> backend
    backend -- every LLM call --> litellm
    litellm --> openai
    litellm --> groq
    backend -- "may I do this?" --> policy
    policy -- checks --> cp
    backend -- image invoices --> runner
    backend -- pay --> ledger
    backend -- log every step --> audit
    policy -- log every decision --> audit
    audit --> minio
    backend --> pg
    ledger --> pg
```

---

## 2. The agent pipeline (one invoice, start to finish)

The four agents are nodes in one LangGraph graph, checkpointed in Postgres.
A human-review pause is a real `interrupt()` — the run waits durably until
someone approves or rejects in the console.

```mermaid
flowchart TD
    start(["Invoice submitted<br/>pasted text or image"]) --> ex

    ex["🔍 Extraction agent<br/>guardrail scan → PII redaction → LLM extracts fields"]
    ex -->|"unsafe content / failed"| endFail(["⛔ Stopped"])
    ex -->|extracted| risk

    risk["📊 Risk-scoring agent<br/>fraud / anomaly score vs vendor history"]
    risk -->|"high risk → flag"| human
    risk -->|routine| appr

    appr["✅ Approval agent<br/>LLM decision + critic check<br/>+ Cedar /authorize"]
    appr -->|rejected| endRej(["❌ Rejected"])
    appr -->|"needs a human<br/>(over $5,000, new vendor, denied)"| human
    appr -->|approved| pay

    human{{"🧑‍💼 Human review<br/>paused in approval queue"}}
    human -->|approve| pay
    human -->|reject| endRej

    pay["💸 Payment-execution agent<br/>authorize → dry-run → commit"]
    pay --> settled(["🏦 Settled in ledger"])
```

Run states follow the A2A vocabulary: `extracted` → `risk_scored` →
`pending_human` / `approved` → `settled` (or `failed` / `rejected`).

---

## 3. The six governance layers

```mermaid
flowchart TB
    subgraph L1["Layer 1 · Identity"]
        direction LR
        kc["Keycloak<br/>who is the human + role"]
        spire["SPIRE<br/>crypto ID per agent/service"]
        inf["Infisical<br/>secrets vault"]
    end
    subgraph L2["Layer 2 · Data"]
        direction LR
        om["OpenMetadata<br/>catalog + sensitivity tags"]
        pres["Presidio<br/>strips PII before audit"]
        gx["Great Expectations<br/>validates loaded data"]
    end
    subgraph L3["Layer 3 · Model"]
        direction LR
        ll["LiteLLM<br/>one gateway, per-agent budget caps"]
        mlf["MLflow<br/>prompt versions"]
        de["DeepEval<br/>pass/fail prompt quality gate"]
    end
    subgraph L4["Layer 4 · Policy"]
        direction LR
        cedar["Cedar<br/>static rules"]
        dw["Dogwood<br/>rules over time"]
    end
    subgraph L5["Layer 5 · Agent Runtime"]
        direction LR
        lg["LangGraph<br/>orchestration"]
        gr["Guardrail model<br/>prompt-injection + fraud text"]
        cap["Capability manifests<br/>second permission check"]
        sb["Sandboxed OCR<br/>Docker + Landlock"]
        ks["Kill switch"]
    end
    subgraph L6["Layer 6 · Operations"]
        direction LR
        otel["OpenTelemetry"]
        lf["Langfuse<br/>LLM traces + cost"]
        prom["Prometheus + Grafana"]
        al["Tamper-proof audit log"]
    end

    L1 --> L2 --> L3 --> L4 --> L5 --> L6
```

| Layer | Question it answers | Tools |
|---|---|---|
| 1 Identity | *Who* is acting — human or agent — and can it be faked? | Keycloak, SPIRE, Infisical |
| 2 Data | Is the data clean, catalogued, and PII-safe? | OpenMetadata, Presidio, Great Expectations |
| 3 Model | Which model/prompt is used, at what cost, and is it good enough? | LiteLLM, MLflow, DeepEval |
| 4 Policy | Is this action *allowed*? | Cedar, Dogwood |
| 5 Agent Runtime | Is the agent behaving safely right now? | LangGraph, guardrail, manifests, sandbox, kill switch |
| 6 Operations | Can we see and *prove* what happened? | OpenTelemetry, Langfuse, Prometheus, Grafana, audit log |

---

## 4. What happens before money moves

The payment-execution agent must pass **two independent permission checks**
(capability manifest + Cedar) and a **ledger dry-run** before anything is
committed. Either check alone denying is enough to block.

```mermaid
sequenceDiagram
    autonumber
    participant PE as Payment-execution agent
    participant CM as Capability manifest
    participant PS as policy-service
    participant CP as control-plane
    participant LD as custodian-ledger
    participant AL as audit-log

    PE->>CM: may I CreateLedgerEntry?
    CM-->>PE: yes (write-ledger tier only)
    PE->>PS: /authorize CreateLedgerEntry
    PS->>CP: kill switch tripped?
    CP-->>PS: no
    PS->>PS: evaluate Cedar + Dogwood (default deny)
    PS->>AL: record decision
    PS-->>PE: ALLOW
    PE->>LD: POST /transactions/dry-run
    LD-->>PE: valid
    PE->>LD: POST /transactions (idempotency key)
    LD-->>PE: committed
    PE->>AL: append payment step
    Note over PE,AL: Any DENY or invalid dry-run → status "blocked", nothing written to the ledger
```

---

## 5. Sandboxed OCR for invoice images

Invoice images are untrusted. Each one is read in a brand-new, locked-down
container that is thrown away afterwards.

```mermaid
flowchart LR
    img["🖼️ Uploaded invoice image"] --> be["custodian-backend<br/>/runs/from-image"]
    be --> auth{"policy-service<br/>InvokeTool allowed?"}
    auth -->|deny| stop(["⛔ Blocked"])
    auth -->|allow| run["custodian-sandbox-runner<br/>/ocr"]
    run --> box

    subgraph box["Fresh container per image — custodian-sandbox-ocr"]
        direction TB
        d["Docker limits<br/>--rm · --network none · read-only FS<br/>non-root · cap-drop ALL · CPU/mem/pid caps · timeout"]
        ll["Landlock (kernel)<br/>OCR process can touch only its exact files"]
        ocr["OCR engine"]
        d --> ll --> ocr
    end

    box -->|"extracted text"| gr["Guardrail model scan"] --> ex["Extraction agent"]
```

---

## 6. Approval rules (Cedar + Dogwood)

Files in `policies/`. Everything is **denied by default**; these rules carve
out what is allowed and force escalation where it matters.

```mermaid
flowchart TD
    req["Agent requests ApprovePayment"] --> amt{"Amount > $5,000?"}
    amt -->|yes| h1["⛔ Agent forbidden<br/>human approver tier ≥ 2 needed<br/>(controller / CFO)"]
    amt -->|no| fv{"First-seen vendor<br/>bank account?"}
    fv -->|yes| h2["⛔ Dual human approval required<br/>(2 distinct approvers)"]
    fv -->|no| rate{"Dogwood: too many approvals<br/>to this vendor in 24h?"}
    rate -->|yes| h3["⛔ Escalate to human"]
    rate -->|no| ok["✅ Agent may approve"]
```

| File | Rule |
|---|---|
| `01-agent-approval-threshold.cedar` | No agent may approve a payment over $5,000 |
| `02-first-seen-vendor-dual-approval.cedar` | New vendor bank account → two human approvals |
| `03-payment-execution-exclusive-ledger-write.cedar` | Only the payment-execution agent may write to the ledger |
| `04-tool-allowlist.cedar` | Each agent may call only its listed tools |
| `05-flagging-and-documentation-actions.cedar` | Who may flag for review / request docs / update vendors |
| `temporal/05-vendor-approval-rate-limit.dw` | Pattern-over-time limit on approvals per vendor |

---

## 7. Kill switch and tamper-proof audit log

```mermaid
flowchart LR
    subgraph triggers["Automatic trip conditions"]
        t1["Stale heartbeat<br/>(agent stopped reporting)"]
        t2["Repeated policy denials"]
        t3["Canary vendor touched"]
    end
    manual["🧑‍💼 Manual trip<br/>console button"]

    t1 --> cp
    t2 --> cp
    t3 --> cp
    manual --> cp
    cp["custodian-control-plane<br/>KILL SWITCH"] -->|"tripped → every /authorize denies"| ps["policy-service"]

    subgraph log["custodian-audit-log"]
        direction LR
        e1["entry n-1<br/>hash"] --> e2["entry n<br/>hash = SHA-256(prev hash + data)"] --> e3["entry n+1"]
    end
    ps --> log
    log --> minio[("MinIO bucket<br/>Object Lock: cannot be edited or deleted")]
    verify["Console: Verify chain"] -->|"/verify recomputes every hash"| log
```

If anyone edits an old audit entry, its hash no longer matches the next
entry's `prev hash`, and `/verify` reports exactly where the chain broke.
