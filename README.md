
# 💳 Cross-Border Instant Merchant Payment & Anti-Fraud Engine

> **System & Business Analysis Case Study**  
> *Compliant with Visa Direct Standards, ISO 20022 Messaging, and Real-Time Risk Scoring SLAs (<200ms).*

---

## 📌 Executive Summary & Business Context

Cross-border e-commerce transactions often suffer from high transaction drop rates, delayed FX (foreign exchange) settlement, and elevated fraud risks. 

This project defines the complete business and technical architecture for a **Digital Banking Cross-Border Payment Gateway**. The system evaluates transaction risk in real-time, reserves multi-currency funds, and settles payments via the Visa Direct Network.

---

## 👥 Roles & Access Control (RBAC)

| Role | System Rights | Triggered Events |
| :--- | :--- | :--- |
| **Retail User** | Initiate Payment, View Status | `POST /api/v1/payments/cross-border` |
| **Anti-Fraud Engine (Service)** | Evaluate Risk, Flag Device | `POST /evaluate-risk` |
| **Core Banking Ledger** | Reserve Funds, Commit Debit | `HOLD_FUNDS`, `DEBIT_ACCOUNT` |
| **Visa Direct Gateway** | Settle Cross-Border OCT | `EXECUTE_OCT_SETTLEMENT` |

---

## 📄 Functional Requirements & User Stories (Gherkin Format)

```gherkin
Feature: Cross-Border Instant Merchant Payment & Anti-Fraud Evaluation

  Background:
    Given the system SLA for risk evaluation is < 200ms
    And foreign exchange rates are dynamically fetched via Central Bank API

  Scenario: Successful cross-border payment with low fraud risk score
    Given a user holds an active account "acc_eur_12345" with a balance of 150.00 EUR
    When the user submits a payment of 100.00 USD to merchant "Amazon US"
    And the Anti-Fraud Engine returns a Risk Score of 0.12 (Low Risk)
    Then the system reserves 92.59 EUR on the user's account
    And submits an OCT payout request to the Visa Direct Network
    And updates transaction status to "APPROVED"
    And returns HTTP 200 OK with the transaction payload.

  Scenario: Transaction halted due to high fraud score
    Given a user submits a payment of 500.00 USD from IP "192.168.1.1"
    When the Anti-Fraud Engine flags suspicious activity (Risk Score: 0.89)
    Then the system halts execution immediately without debiting funds
    And updates status to "DECLINED_FRAUD"
    And returns HTTP 422 Unprocessable Entity ("FRAUD_RISK_EXCEEDED").
```
## 🔄 Business Architecture: As-Is vs. To-Be Analysis

### Problem Statement & Quantified Business Impact
Legacy cross-border settlement architectures suffer from asynchronous batch processing, leading to high transaction abandonment and fraud exposure.

* **Target Metrics & Value Drivers:**
  * **FX Settlement Latency:** Reduced from **T+2 Days** (batch settlement) to **Real-Time (<2 seconds)**.
  * **False-Positive Authorization Declines:** Targeted reduction by **18%** through risk-score thresholding.
  * **Cart Abandonment Rate:** Decreased by **12%** by replacing redirect-heavy 3DS triggers with silent anti-fraud evaluation.

---

### Process Mapping (BPMN 2.0 Conceptual Overview)

#### ❌ As-Is State (Legacy Batch Processing):
`User Payment Request` ➔ `Manual/Batch Fraud Review` ➔ `T+2 Clearing` ➔ `High Timeout Rate (12% Drop-off)`

#### ✅ To-Be State (Event-Driven API Gateway):
`User Initiates Request` ➔ `Sub-200ms Async Anti-Fraud Scoring` ➔ `Automated Ledger Hold` ➔ `Instant Visa Direct OCT Settlement`

---

## ⚖️ Architectural Trade-off Analysis (Design Rationale)

System design decisions were evaluated against strict Financial Industry Performance and Compliance Standards:

### 1. Protocol Selection: REST/HTTPS vs. gRPC vs. GraphQL
* **Decision:** Adopted **REST over HTTPS (JSON)** for external edge-gateway ingestion.
* **Trade-off & Rationale:** While gRPC provides superior binary serialization speed, REST is the universal standard for public FinTech networks (Visa Direct, Stripe API). It simplifies merchant onboarding and third-party developer integration without requiring specialized Proto-buf client builds.

### 2. Risk Scoring Architecture: Async Risk SLA (<200ms) vs. Deep Synchronous Inspection
* **Decision:** Implemented a **strict 200ms SLA timeout** for the Anti-Fraud Engine.
* **Trade-off & Rationale:** Deep ML inspection models offer higher fraud detection accuracy but introduce latency > 800ms. In e-commerce checkout, latency above 300ms causes a 12% drop in conversion. We prioritize low-latency heuristic scoring at checkout, deferring heavy asynchronous graph checks post-authorization.

### 3. Database Paradigm: Relational (ACID) vs. NoSQL (Eventual Consistency)
* **Decision:** Selected **Relational MySQL/PostgreSQL** with ACID compliance.
* **Trade-off & Rationale:** NoSQL databases (e.g., MongoDB) offer horizontal scaling, but their eventual consistency model poses unacceptable risks of double-spending or balance mismatch during multi-currency holds. Relational isolation guarantees strict double-entry ledger execution.

---

## 🛡️ Resilience, Failure Modes & Edge Case Mitigation

System stability is maintained through explicit fallback policies for network degrades and incomplete data payloads:

```plantuml
@startuml
autonumber
title Circuit Breaker & Fallback Flow

participant "Payment Engine" as Engine
participant "Anti-Fraud API" as Fraud
participant "Core Ledger" as Ledger

Engine -> Fraud: POST /evaluate-risk (Timeout: 200ms)

alt Scenario A: Service Timeout (>200ms)
    Fraud --x Engine: [Gateway Timeout / Circuit Open]
    Engine -> Ledger: Rollback Balance Hold
    Engine --> App: HTTP 500 (Gateway Timeout - Safe Retry)
else Scenario B: Insufficient Telemetry Data
    Fraud --> Engine: Risk Score: UNKNOWN (Missing Device Metadata)
    Engine -> Engine: Trigger Soft Decline (Step-Up 3DS Challenge)
    Engine --> App: HTTP 402 (Payment Challenge Required)
end
@enduml
