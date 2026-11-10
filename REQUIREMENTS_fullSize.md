# Aurora Pay: Real-Time Fraud and Transaction Monitoring

> A cloud-native, event-driven fraud detection platform on AWS, provisioned entirely with Terraform.

**Status:** Requirements complete. Design phase in progress.

> **Disclaimer:** Aurora Pay is a fictional company. All data is synthetic, and all upstream systems (card processor, wallet, transfer service, SMS provider) are mocked. This is a portfolio project.

---

## Table of contents

1. [Company overview](#1-company-overview)
2. [Problem statement](#2-problem-statement)
3. [Business goals](#3-business-goals)
4. [Users and personas](#4-users-and-personas)
5. [Functional requirements](#5-functional-requirements)
6. [Non-functional requirements](#6-non-functional-requirements)
7. [Constraints and assumptions](#7-constraints-and-assumptions)
8. [Out of scope](#8-out-of-scope)
9. [Design phase deliverables](#9-design-phase-deliverables)
10. [Project roadmap](#10-project-roadmap)

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

## 5. Functional requirements

### FR1: Transaction ingestion
- FR1.1 Accept transactions from three sources (card, wallet, transfer), each with a slightly different payload shape.
- FR1.2 Handle duplicates and retries from sources safely. The same transaction must never be scored or acted on twice.
- FR1.3 Never lose a transaction, even if downstream components are slow or failing.
- FR1.4 Process events for the same account in a sensible order.

### FR2: Real-time decisioning
- FR2.1 For card authorizations, the processor asks Aurora Pay for a decision and waits. Possible outcomes: **approve**, **decline**, **hold for review**, or **request step-up verification**.
- FR2.2 For wallet and transfer activity, decisions can take a few seconds, since these flows tolerate short delays.
- FR2.3 Every decision includes a risk score (0 to 100) and the reasons behind it.

### FR3: Detection rules
Support at least these rule types:

- Velocity (for example, more than 5 transactions in 2 minutes)
- Amount thresholds relative to the customer's normal behavior
- Impossible travel (two locations too far apart for the time elapsed)
- New device or new beneficiary combined with a large amount
- Card-testing patterns (many small declined attempts across merchants)
- Deny lists (merchants, devices, IPs, countries)

Rules must be:

- FR3.1 **Configurable by analysts** without a code deployment.
- FR3.2 **Versioned**, with every decision recording which rule versions fired.
- FR3.3 **Testable in shadow mode**: a new rule runs against live traffic and logs what it would have done without affecting customers.

### FR4: Alerts and case management
- FR4.1 Held or high-risk transactions generate an alert. Related alerts are grouped into a case (for example, same card or same device).
- FR4.2 Cases are prioritized by risk and amount, assigned to analysts, and tracked against SLAs (**15 minutes** for high risk, **4 hours** for medium).
- FR4.3 Analysts can add notes, attach evidence, and close with a disposition: *confirmed fraud*, *false positive*, or *inconclusive*.
- FR4.4 Unactioned cases past their SLA escalate automatically.

### FR5: Customer verification
- FR5.1 For held transactions, the customer gets an SMS or push notification ("Was this you?").
- FR5.2 Their reply releases or blocks the transaction and updates the case.
- FR5.3 If they do not reply within a defined window, a default action applies.

### FR6: Protective actions
- FR6.1 Analysts and automated rules can freeze a card, block a merchant or device, or lower an account's limits.
- FR6.2 These actions must take effect quickly and apply to all subsequent decisions.

### FR7: Feedback loop
- FR7.1 Case dispositions feed back into metrics (false positive rate per rule, detection rate) so managers can retire noisy rules and strengthen effective ones.

### FR8: Reporting and analytics
- FR8.1 Live dashboards: fraud rate, alert volume, decision latency, queue depth, analyst SLA performance.
- FR8.2 Ad hoc querying of historical transactions and decisions (for example, "all declined transactions from device X over the last 6 months").
- FR8.3 Compliance report generation for suspicious transactions.

### FR9: Access control
- FR9.1 Role-based access per persona above.
- FR9.2 Customer PII is masked by default. Unmasking is a logged, role-restricted action.

## 6. Non-functional requirements

### Performance
| ID | Requirement |
|---|---|
| NFR-P1 | Card authorization decision: **p99 under 300 ms** end to end, p50 under 100 ms |
| NFR-P2 | Stream monitoring (non-blocking detection): alert raised within **5 seconds** of the transaction |
| NFR-P3 | Design capacity: **sustained 1,500 TPS, bursts to 5,000 TPS** for 10 minutes without degradation |

### Availability and resilience
| ID | Requirement |
|---|---|
| NFR-A1 | Decision path availability: **99.95%** |
| NFR-A2 | Zero lost transactions, with the ability to replay events after an incident |
| NFR-A3 | Recovery: **RPO under 5 minutes, RTO under 1 hour** |
| NFR-A4 | Fallback policy when the decision engine is unavailable: **approve transactions under CAD 100, decline above**. The system degrades safely instead of failing entirely. |
| NFR-A5 | A single bad event must never block the processing of others |

### Security and compliance
| ID | Requirement |
|---|---|
| NFR-S1 | **PCI DSS:** raw card numbers are never stored or logged. Only tokens or masked values. |
| NFR-S2 | **PIPEDA / privacy:** data minimization, encryption in transit and at rest, full access logging for personal data |
| NFR-S3 | **AML (FINTRAC-style):** retain decision and case records for **7 years** (company policy), immutable and tamper-evident |
| NFR-S4 | **Explainability:** for any past decision, reconstruct the inputs, rule versions, score, and outcome |
| NFR-S5 | Least-privilege access everywhere, no shared credentials, secrets managed centrally |
| NFR-S6 | Dev, staging, and prod fully isolated |

### Data retention tiers
| ID | Requirement |
|---|---|
| NFR-D1 | Last 90 days: fast query access |
| NFR-D2 | 90 days to 7 years: cheaper storage, slower queries acceptable |
| NFR-D3 | All retention tiers are automated |

### Operations
| ID | Requirement |
|---|---|
| NFR-O1 | Full observability: structured logs, metrics, traces, and alerting on decision latency, error rates, queue backlogs, and SLA breaches |
| NFR-O2 | Everything deployable through automated pipelines |
| NFR-O3 | Cost ceiling for the demo environment: **under CAD 150/month**, with a documented estimate for production |

## 7. Constraints and assumptions

- All data is **synthetic**. A transaction generator simulates normal customers, fraud patterns (card testing, account takeover, mule accounts), and traffic bursts.
- Upstream systems (card processor, wallet, transfers) and the SMS provider are **mocked**.
- No mobile app or full analyst UI is required. A thin UI or API-only workflow is fine, but the case workflow must be demonstrable.
- ML scoring is optional. Rules-based decisioning is the baseline, and an AI/ML scoring component is a stretch goal.
- Load testing at the full 5,000 TPS may be impractical on a hobby budget. The system is designed for it, with a scaled-down test and a documented extrapolation.

## 8. Out of scope

- Real card network integration
- Real KYC onboarding
- Chargeback processing
- Customer-facing apps
- Production-grade ML model training

## 9. Design phase deliverables

The solution design lives in `docs/` and must include:

1. **Architecture diagram** with end-to-end flows (card authorization path, stream monitoring path, case workflow)
2. **Event catalog** (events, producers, consumers, schemas, ordering and idempotency strategy)
3. **Data model and storage choices**, with justification for each store
4. **Architecture decision records (ADRs)** for the 4 to 5 key trade-offs
5. **Failure analysis**: what happens when each major component fails, and how the requirements are still met
6. **Security design** (identity, data protection, PCI scope boundary)
7. **Cost estimate** for demo and for production scale
8. **Open questions and assumptions** where the requirements were silent

### Known tensions to resolve deliberately

- Latency vs. completeness of checks
- Availability vs. fallback risk
- Auditability vs. cost

## 10. Project roadmap

- [x] Phase 0: Business requirements
- [ ] Phase 1: Solution design and review
- [ ] Phase 2: Terraform foundation (remote state, VPC, CI/CD with GitHub Actions OIDC)
- [ ] Phase 3: Ingestion and event backbone
- [ ] Phase 4: Decision engine and rules
- [ ] Phase 5: Case management and customer verification
- [ ] Phase 6: Analytics, retention tiers, and dashboards
- [ ] Phase 7: Observability, load test, and failure tests
- [ ] Phase 8: Cost report, write-up, and demo

## Repository structure (planned)

```
.
├── README.md
├── docs/
│   ├── architecture/
│   ├── adr/
│   └── events/
├── infra/
│   ├── modules/
│   └── envs/
│       ├── dev/
│       ├── staging/
│       └── prod/
├── services/
├── tools/
│   └── txn-generator/
└── .github/
    └── workflows/
```