
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
