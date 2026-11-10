# Aurora Pay: Real-Time Fraud and Transaction Monitoring

> A cloud-native, event-driven fraud detection platform on AWS, provisioned entirely with Terraform.

**Status:** Requirements complete. Delivery scope defined. Design phase in progress.

> **Disclaimer:** Aurora Pay is a fictional company. All data is synthetic, and all upstream systems (card processor, wallet, transfer service, SMS provider) are mocked. This is a portfolio project.

---

## Table of contents

1. [Company overview](#1-company-overview)
2. [Problem statement](#2-problem-statement)
3. [Business goals](#3-business-goals)
4. [Users and personas](#4-users-and-personas)
5. [Delivery scope](#5-delivery-scope)
6. [Functional requirements](#6-functional-requirements)
7. [Non-functional requirements](#7-non-functional-requirements)
8. [Constraints and assumptions](#8-constraints-and-assumptions)
9. [Out of scope](#9-out-of-scope)
10. [Design phase deliverables](#10-design-phase-deliverables)
11. [Project roadmap](#11-project-roadmap)

---

## 1. Company overview

**Aurora Pay** is a Canadian digital payments company. It issues prepaid and debit cards, runs a mobile wallet, and supports bank-to-bank transfers.

| Metric | Value |
|---|---|
| Active customers | 3 million, growing about 8% per quarter |
| Average volume | About 2 million transactions per day |
| Peak volume | 1,500 TPS during holiday sales and payday spikes |
| Channels | Card purchases (online and in-store), wallet payments, account-to-account transfers |

## 2. Problem statement

Fraud detection today is a **nightly batch job**. By the time suspicious activity is found, the money is gone.

- Fraud losses are about **0.12% of transaction volume** (roughly CAD 9M per year) and rising.
- Fraud rings drain accounts within minutes using stolen credentials and card-testing attacks.
- Legacy rules are hardcoded, and every change needs a release that takes weeks.
- About **9% of flagged transactions are false positives**, so customers get cards blocked at checkout and support costs climb.
- Analysts work from spreadsheets with no prioritization, so high-risk cases wait hours.
- Regulators have asked for a clearer audit trail of why transactions were declined or held.

## 3. Business goals

| Goal | Target |
|---|---|
| Reduce fraud losses | 40% within 12 months |
| Reduce false positives | Below 2% of legitimate transactions |
| Detection speed | Seconds, not hours |
| High-risk case response | Analyst picks up within 15 minutes |
| Rule change lead time | From weeks to hours, with no engineering release |
| Audit readiness | Every decision explainable and reproducible |

## 4. Users and personas

| Persona | Needs |
|---|---|
| **Cardholder** | Smooth approvals, and a quick way to confirm or deny suspicious activity |
| **Fraud analyst (L1)** | Review alerts, investigate, mark fraud or legitimate |
| **Senior analyst / fraud manager** | Handle escalations, tune rules, monitor team SLAs and fraud trends |
| **Compliance officer** | Prepare suspicious transaction reports, respond to regulator requests |
| **Auditor** | Read-only access to decisions and evidence |
| **Platform / SRE engineer** | Keep the system running, respond to incidents |
| **Upstream systems** | Card processor, wallet service, and transfer service that send transactions in |

## 5. Delivery scope

The business requirements below describe the full problem. This project is delivered as a **thin end-to-end slice**: the core decisioning path is built and deployed with Terraform, and the rest is **designed and documented but not built**. The goal is to demonstrate practical AWS and Terraform engineering without building an entire fraud platform.

Every requirement carries one of these tags:

| Tag | Meaning |
|---|---|
| **[BUILD]** | Implemented, deployed, and demonstrable |
| **[LITE]** | Implemented in a reduced form. The reduction is stated next to the requirement |
| **[DESIGN]** | Fully designed and documented in `docs/`, but not implemented |

### Scope summary

| Area | Delivered as |
|---|---|
| Card authorization path (sync) | Built |
| Wallet and transfer path (async, event-driven) | Built |
| Core rules (velocity, amount threshold, deny list) with versioning | Built |
| Decision store, idempotency, event publishing, durable archive | Built |
| Fallback policy and DLQ handling | Built, with documented failure tests |
| Infrastructure as code, CI/CD, security baseline | Built |
| Dashboard, alarms, and a scaled-down load test | Built |
| Advanced rules, shadow mode, case management, verification flow | Designed only |
| Analyst self-service, ad hoc analytics, multi-environment promotion | Designed only |

### Honest-labeling rule

The README and any resume description must state what is implemented and what is designed only. Nothing marked **[DESIGN]** is described as built.

## 6. Functional requirements

### FR1: Transaction ingestion
- FR1.1 **[BUILD]** Accept transactions from three sources (card, wallet, transfer), each with a slightly different payload shape. Validate and normalize them into one canonical event.
- FR1.2 **[BUILD]** Handle duplicates and retries from sources safely. The same transaction must never be scored or acted on twice.
- FR1.3 **[BUILD]** Never lose a transaction, even if downstream components are slow or failing.
- FR1.4 **[BUILD]** Process events for the same account in a sensible order (async path).

### FR2: Real-time decisioning
- FR2.1 **[BUILD]** For card authorizations, the processor asks Aurora Pay for a decision and waits. Possible outcomes: **approve**, **decline**, **hold for review**, or **request step-up verification**.
- FR2.2 **[BUILD]** For wallet and transfer activity, decisions can take a few seconds, since these flows tolerate short delays. The result is returned to the sender through a separate channel, correlated by transaction ID.
- FR2.3 **[BUILD]** Every decision includes a risk score (0 to 100) and the reasons behind it.

### FR3: Detection rules
Support at least these rule types:

| Rule type | Scope |
|---|---|
| Velocity (for example, more than 5 transactions in 2 minutes) | **[BUILD]** |
| Amount thresholds relative to the customer's normal behavior | **[LITE]** (simple threshold or fixed multiple of a running average) |
| Deny lists (merchants, devices, IPs, countries) | **[BUILD]** |
| Impossible travel | **[DESIGN]** |
| New device or new beneficiary combined with a large amount | **[DESIGN]** |
| Card-testing patterns | **[DESIGN]** |

Rules must be:

- FR3.1 **[DESIGN]** **Configurable by analysts** without a code deployment. *Delivered as:* rules are versioned config files in the repo, shipped through the CI/CD pipeline. The analyst self-service flow (validation, approval, rule status) is designed only.
- FR3.2 **[BUILD]** **Versioned**, with every decision recording which rule versions fired.
- FR3.3 **[DESIGN]** **Testable in shadow mode**: a new rule runs against live traffic and logs what it would have done without affecting customers.

### FR4: Alerts and case management
- FR4.1 **[LITE]** Held or high-risk transactions generate an alert event and a notification. Grouping into cases is designed only.
- FR4.2 **[DESIGN]** Cases are prioritized by risk and amount, assigned to analysts, and tracked against SLAs (**15 minutes** for high risk, **4 hours** for medium).
- FR4.3 **[DESIGN]** Analysts can add notes, attach evidence, and close with a disposition: *confirmed fraud*, *false positive*, or *inconclusive*.
- FR4.4 **[DESIGN]** Unactioned cases past their SLA escalate automatically.

### FR5: Customer verification
- FR5.1 **[LITE]** For held transactions, the customer gets a notification ("Was this you?"). A simple email or SNS-based notification stands in for SMS and push.
- FR5.2 **[DESIGN]** Their reply releases or blocks the transaction and updates the case.
- FR5.3 **[DESIGN]** If they do not reply within a defined window, a default action applies.

### FR6: Protective actions
- FR6.1 **[DESIGN]** Analysts and automated rules can freeze a card, block a merchant or device, or lower an account's limits.
- FR6.2 **[DESIGN]** These actions must take effect quickly and apply to all subsequent decisions.

### FR7: Feedback loop
- FR7.1 **[DESIGN]** Case dispositions feed back into metrics (false positive rate per rule, detection rate) so managers can retire noisy rules and strengthen effective ones.

### FR8: Reporting and analytics
- FR8.1 **[LITE]** One live dashboard covering decision volume, decision latency, error rate, and queue depth. Fraud rate and analyst SLA views are designed only.
- FR8.2 **[DESIGN]** Ad hoc querying of historical transactions and decisions (for example, "all declined transactions from device X over the last 6 months").
- FR8.3 **[DESIGN]** Compliance report generation for suspicious transactions.

### FR9: Access control
- FR9.1 **[LITE]** Least-privilege IAM roles for services and engineers. A user identity layer for analysts, compliance, and auditors is designed only.
- FR9.2 **[LITE]** Customer PII is masked or tokenized at write time, so stores hold masked values by default. A logged, role-restricted unmask action is designed only.

## 7. Non-functional requirements

Targets stay the same as the business case. The **Delivery** column states how each is addressed in this project.

### Performance
| ID | Requirement | Delivery |
|---|---|---|
| NFR-P1 | Card authorization decision: **p99 under 300 ms** end to end, p50 under 100 ms | **[BUILD]** Measured with a scaled-down load test, with real numbers published |
| NFR-P2 | Stream monitoring (non-blocking detection): alert raised within **5 seconds** of the transaction | **[BUILD]** |
| NFR-P3 | Design capacity: **sustained 1,500 TPS, bursts to 5,000 TPS** for 10 minutes without degradation | **[DESIGN]** Quota analysis and extrapolation from the scaled-down test |

### Availability and resilience
| ID | Requirement | Delivery |
|---|---|---|
| NFR-A1 | Decision path availability: **99.95%** | **[DESIGN]** Multi-AZ managed services, documented analysis |
| NFR-A2 | Zero lost transactions, with the ability to replay events after an incident | **[BUILD]** Durable queues, DLQ, and a raw event archive. Replay procedure documented |
| NFR-A3 | Recovery: **RPO under 5 minutes, RTO under 1 hour** | **[DESIGN]** |
| NFR-A4 | Fallback policy when the decision engine is unavailable: **approve transactions under CAD 100, decline above** | **[BUILD]** Demonstrated through a failure test |
| NFR-A5 | A single bad event must never block the processing of others | **[BUILD]** DLQ with an alarm |

### Security and compliance
| ID | Requirement | Delivery |
|---|---|---|
| NFR-S1 | **PCI DSS:** raw card numbers are never stored or logged. Only tokens or masked values | **[BUILD]** |
| NFR-S2 | **PIPEDA / privacy:** data minimization, encryption in transit and at rest, full access logging for personal data | **[LITE]** Encryption with KMS and CloudTrail audit logging |
| NFR-S3 | **AML (FINTRAC-style):** retain decision and case records for **7 years**, immutable and tamper-evident | **[LITE]** Decision archive with immutable object storage and lifecycle rules. Case records are designed only |
| NFR-S4 | **Explainability:** for any past decision, reconstruct the inputs, rule versions, score, and outcome | **[BUILD]** |
| NFR-S5 | Least-privilege access everywhere, no shared credentials, secrets managed centrally | **[BUILD]** CI uses short-lived federated credentials, with no long-lived keys |
| NFR-S6 | Dev, staging, and prod fully isolated | **[DESIGN]** One deployed environment. Terraform is parameterized so more environments can be added, and the multi-account layout is documented |

### Data retention tiers
| ID | Requirement | Delivery |
|---|---|---|
| NFR-D1 | Last 90 days: fast query access | **[DESIGN]** |
| NFR-D2 | 90 days to 7 years: cheaper storage, slower queries acceptable | **[LITE]** Archive storage classes and lifecycle rules |
| NFR-D3 | All retention tiers are automated | **[LITE]** Lifecycle rules for what is built |

### Operations
| ID | Requirement | Delivery |
|---|---|---|
| NFR-O1 | Full observability: structured logs, metrics, traces, and alerting on decision latency, error rates, queue backlogs, and SLA breaches | **[LITE]** Structured logs, one dashboard, and alarms on latency, errors, and DLQ depth. SLA-breach alerting is designed only |
| NFR-O2 | Everything deployable through automated pipelines | **[BUILD]** Plan on pull request, apply on merge, with lint and security scanning |
| NFR-O3 | Cost ceiling for the demo environment: **under CAD 150/month**, with a documented estimate for production | **[BUILD]** Budget alarm, cost breakdown, and production estimate in the README |

## 8. Constraints and assumptions

- All data is **synthetic**. A transaction generator simulates normal customers and a few fraud patterns, and produces traffic bursts.
- Upstream systems (card processor, wallet, transfers) and the SMS provider are **mocked**.
- No mobile app or analyst UI is required. The built slice is API-driven.
- ML scoring is out of scope. Rules-based decisioning is the baseline.
- Load testing at the full 5,000 TPS is impractical on a hobby budget. The system is designed for it, with a scaled-down test and a documented extrapolation.
- Wallet and transfer services are assumed to be **internal** systems authenticating with IAM. The card processor is an external caller and needs its own authentication design.
- Compute choice (serverless, containers, or instances) is decided in an ADR, based on the latency, burst, and cost requirements above.
- The demo environment is created and destroyed on demand to stay within budget.

## 9. Out of scope

- Real card network integration
- Real KYC onboarding
- Chargeback processing
- Customer-facing apps
- Analyst UI
- ML model training

## 10. Design phase deliverables

The solution design lives in `docs/` and must include:

1. **Architecture diagram** with end-to-end flows (card authorization path, async path, event fan-out). Designed-only components are drawn dashed.
2. **Event catalog** (events, producers, consumers, schemas, ordering and idempotency strategy)
3. **Data model and storage choices**, with justification for each store
4. **Architecture decision records (ADRs)**: front door, compute, single-store versus multi-store state, rules delivery, and one more as needed
5. **Failure analysis**: what happens when each major component fails, and how the requirements are still met
6. **Security design** (identity, data protection, PCI scope boundary)
7. **Cost estimate** for demo and for production scale
8. **Open questions and assumptions** where the requirements were silent
9. **Designed-only components**: short design notes for shadow mode, case management, customer verification, analytics, and multi-environment promotion

### Known tensions to resolve deliberately

- Latency vs. completeness of checks
- Availability vs. fallback risk
- Auditability vs. cost
- Scope vs. delivery time (resolved in section 5)

## 11. Project roadmap

- [x] Phase 0: Business requirements and delivery scope
- [ ] Phase 1: Solution design and review
- [ ] Phase 2: Terraform foundation (remote state, networking baseline, CI/CD with GitHub Actions OIDC)
- [ ] Phase 3: Card authorization path and decision core (rules, state, idempotency, decision store)
- [ ] Phase 4: Async path and event backbone (queue, DLQ, fan-out, result channel, archive)
- [ ] Phase 5: Observability and security pass (dashboard, alarms, KMS, IAM review)
- [ ] Phase 6: Transaction generator, scaled-down load test, and failure tests
- [ ] Phase 7: Write-up (README, diagram, ADRs, cost report, designed-only notes)

**Publishing plan:** make the repository public after Phase 2, and update the README as each phase completes.

## Repository structure (planned)

```
.
├── README.md
├── REQUIREMENTS.md
├── docs/
│   ├── architecture/
│   ├── adr/
│   ├── events/
│   └── designed-only/
├── infra/
│   ├── modules/
│   └── envs/
│       └── demo/
├── rules/
├── services/
├── tools/
│   └── txn-generator/
└── .github/
    └── workflows/
```
