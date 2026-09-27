# Forward Deployment Engineer Learning and Interview Guide

## Audience

An engineer with approximately two years of experience in software engineering,
cloud services, APIs, AWS, payments, production support, and troubleshooting.

## Purpose

Build the ability to:

1. Understand an unfamiliar customer problem.
2. Translate business requirements into technical requirements.
3. Write useful, maintainable integration code.
4. Design and deploy reliable systems.
5. Investigate production incidents systematically.
6. Explain technical trade-offs to customers and interviewers.

This guide contains original practice problems and illustrative deployment
scenarios. They are not claimed to be questions actually asked by any named
company or accounts of specific customer deployments.

Interview expectations vary by role and company. Treat the rubrics here as
preparation guidance, not confidential hiring criteria.

## Document status

This installment contains:

- Module 1: Forward Deployment Engineer overview.
- Module 2: Python, including 20 examples, 20 exercises, and 20 interview questions.
- Module 3: SQL and databases, including 30 interview questions.
- Module 4: REST APIs and integration.

Remaining modules:

5. AWS
6. Docker
7. Kubernetes
8. Linux
9. Git
10. CI/CD
11. System design
12. Observability
13. Production support
14. Networking
15. Behavioral interviews
16. Fifty coding questions with solutions
17. One hundred mock interview questions with answers
18. Sixty-day study plan

## How to use this guide

For each topic:

1. Explain it aloud without looking.
2. Run the example in an isolated development environment.
3. Complete the exercise without copying the solution.
4. Introduce a failure and investigate it.
5. Answer the interview question in two minutes.
6. Answer the follow-up with a concrete trade-off.

Do not memorize scripts without understanding their assumptions.

## Lab conventions

- Python examples target Python 3.11 or later.
- SQL examples use PostgreSQL unless another database is named.
- Use synthetic payment data, not real customer information.
- Monetary amounts are integers in the currency's minor unit.
- Never assume every currency uses two decimal places.
- API URLs using example.com are placeholders.
- Production-oriented examples are teaching examples, not complete audited
  payment implementations.
- Examples have not been executed in your environment.

## Running Python examples

Create a directory and virtual environment:

```bash name=setup.sh
mkdir -p fde-lab
cd fde-lab
python3 -m venv .venv
source .venv/bin/activate

# Required only for the HTTP examples:
python -m pip install requests
```

On Windows PowerShell, activate with:

```powershell name=setup.ps1
.\.venv\Scripts\Activate.ps1
```

Save each example under the filename shown and run:

```bash name=run_example.sh
python example01_variables.py
```

For a real project, select supported dependency versions, lock them, scan them,
and update them through a tested process.

---

# Module 1 — Forward Deployment Engineer Overview

## 1.1 What is an FDE?

### Beginner explanation

A Forward Deployment Engineer helps a customer make a software product work
in the customer's real environment.

You do not stop at:

> "The API works on my laptop."

You work toward:

> "The customer's workflow works reliably, their team can operate it,
> and we can demonstrate the intended business result."

### Industry-level explanation

An FDE combines hands-on engineering with customer discovery and deployment
ownership.

The work can include:

- Understanding business workflows.
- Integrating APIs and data sources.
- Building application features or customer-specific adapters.
- Investigating infrastructure and security constraints.
- Deploying and validating a solution.
- Troubleshooting incidents.
- Feeding reusable requirements back into the product.

The exact balance differs across employers. Some roles emphasize application
development; others emphasize data platforms, enterprise integrations,
infrastructure, or AI deployments.

## 1.2 Role comparison

These are useful preparation distinctions, not rigid organizational boundaries.

| Role | Main emphasis | Typical deliverable | Useful success measure |
|---|---|---|---|
| Software Engineer | Building software capabilities | Tested application code | Correctness, maintainability, product outcomes |
| DevOps Engineer | Delivery and operational automation | Pipelines, infrastructure automation | Deployment speed and repeatability |
| SRE | Engineering for reliability | SLOs, reliability improvements, automation | Reliability and reduced operational toil |
| Solutions Architect | Designing a suitable solution | Architecture and integration design | Fit to requirements and constraints |
| FDE | Making a solution succeed with a customer | Working customer deployment | Adoption, business impact, reliability |

An FDE may perform work from all five columns.

The distinguishing preparation question is:

> Can you connect engineering decisions to a customer's actual workflow?

## 1.3 Responsibilities across a deployment

| Stage | Questions to answer | Deliverable |
|---|---|---|
| Discovery | What problem matters? Who experiences it? | Problem statement |
| Requirements | What does success mean? What is out of scope? | Acceptance criteria |
| Integration | What systems, schemas, identities, and networks exist? | Integration contract |
| Implementation | What is the smallest useful end-to-end workflow? | Working vertical slice |
| Validation | Does it work under failure and realistic load? | Test evidence |
| Rollout | How do we limit risk and reverse a bad change? | Rollout and rollback plan |
| Operations | Who responds when it fails? | Dashboard and runbook |
| Adoption | Are users receiving value? | Outcome review |

### Example discovery questions

For a merchant reporting delayed payment status:

1. Is payment execution delayed, or only the dashboard?
2. When did it begin?
3. Which merchants, regions, payment methods, or providers are affected?
4. What is the source of truth?
5. Is there a reconciliation process?
6. What happens if the same event arrives twice?
7. What does the customer currently do manually?
8. What latency is acceptable for the business workflow?

## 1.4 Illustrative customer deployment

### Customer problem

A merchant's payment provider sends status updates, but the merchant's support
dashboard sometimes displays stale information.

### Proposed architecture

```text name=customer_payment_integration.txt
Payment provider
      |
      | Signed webhook
      v
Webhook receiver
      |
      | Verify + validate + durably record
      v
Event inbox / queue
      |
      v
Payment status worker
      |
      +----> Payment database
      |
      +----> Audit records
      |
      v
Customer dashboard

Scheduled reconciliation:
Provider API ----> Reconciliation worker ----> Exception report / repair workflow
```

### Why this is an FDE problem

The implementation needs more than a webhook endpoint:

- The customer must expose a reachable endpoint.
- Security teams must approve the authentication model.
- Provider identifiers must map to internal payment identifiers.
- Duplicate and out-of-order events need defined handling.
- Operations needs visibility into delays.
- Support teams need a clear explanation of intermediate payment states.

### Proposed acceptance criteria

These are hypothetical targets to negotiate, not universal requirements:

- Valid events become visible within the agreed latency target.
- Duplicate events do not create duplicate business effects.
- Invalid signatures are rejected.
- Unknown payment identifiers are quarantined for investigation.
- Customer support can trace a payment using a correlation identifier.
- A failed deployment can be rolled back without losing accepted events.

## 1.5 Coding example: validate a customer's integration record

```python name=validate_customer_record.py
def validate_record(record):
    required = {"merchant_id", "provider_payment_id", "amount_minor", "currency"}
    missing = required - record.keys()

    if missing:
        raise ValueError(f"Missing fields: {sorted(missing)}")

    if not isinstance(record["merchant_id"], str) or not record["merchant_id"]:
        raise ValueError("merchant_id must be a non-empty string")

    amount = record["amount_minor"]

    # bool is an int subclass in Python; reject it explicitly.
    if type(amount) is not int or amount <= 0:
        raise ValueError("amount_minor must be a positive integer")

    if record["currency"] not in {"USD", "INR", "EUR"}:
        raise ValueError("Unsupported currency for this integration")

    return record


if __name__ == "__main__":
    payment = {
        "merchant_id": "merchant_demo",
        "provider_payment_id": "provider_payment_001",
        "amount_minor": 12500,
        "currency": "INR",
    }
    print(validate_record(payment))
```

Production extension:

- Validate identifier lengths and character rules.
- Validate every field, not just a subset.
- Apply schema versioning.
- Reject or handle unknown fields deliberately.
- Define whether normalization is allowed.
- Avoid including sensitive payloads in error messages.

## 1.6 Example daily schedule

This is a suggested working pattern, not a claim about a particular employer.

| Time | Activity |
|---|---|
| Morning | Check rollout health and customer blockers |
| Discovery session | Clarify a workflow or acceptance criterion |
| Engineering block | Implement and test an integration |
| Coordination | Resolve networking, credentials, or data ownership |
| Validation | Run customer acceptance tests |
| End of day | Document progress, risks, decisions, and next steps |

## 1.7 Hands-on exercise

Write a one-page deployment proposal for the payment-status integration.

Include:

1. Customer problem.
2. Current workflow.
3. Proposed workflow.
4. Data contract.
5. Authentication model.
6. Failure modes.
7. Success measures.
8. Rollout plan.
9. Rollback plan.
10. Operational owner.

### Self-check

Your proposal should answer:

- What happens during a provider outage?
- What happens if the webhook arrives before the payment record?
- How does a support engineer investigate one affected payment?
- How do you prove that the customer benefit was achieved?

## 1.8 Interview questions and detailed answers

### Q1. What does an FDE do?

**Answer**

"An FDE takes responsibility for turning a customer's problem into a working
deployment. I would start by understanding the business workflow and constraints,
then design and build the required integration, validate it with the customer,
and establish monitoring and an operational handover.

The role needs strong coding and troubleshooting, but also the ability to
explain trade-offs and discover whether we are solving the right problem."

**Interviewer preparation rubric**

- Customer outcome, not just deployment activity.
- Evidence of hands-on engineering.
- End-to-end thinking.
- Clear communication.

**Follow-up**

How would you distinguish a product defect from customer misconfiguration?

**Strong direction**

Reproduce the behavior, compare working and failing environments, inspect the
contract and configuration, gather evidence, and avoid assigning blame before
isolating the cause.

### Q2. A customer requests a feature that is not on the roadmap. What do you do?

**Answer**

"I would first ask what workflow the feature is intended to enable. I would
separate the desired outcome from the proposed implementation, assess existing
capabilities, and identify security and operational implications.

Then I would compare configuration, a reusable extension, and a product change.
I would make the trade-offs explicit, agree on an owner and scope, and avoid
promising an unsupported delivery date."

**Expectations**

Discovery, judgment, scope control, and a path to action.

**Follow-up**

When is a temporary workaround acceptable?

**Strong direction**

When its risk is understood, the customer agrees, ownership is explicit,
monitoring exists, and there is an expiry or replacement plan.

### Q3. Your integration works in staging but fails for the customer.

**Answer**

"I would compare the environments systematically: DNS, routing, proxies, TLS,
credentials, permissions, configuration, schema, payload size, and traffic
patterns.

I would reproduce the smallest failing request and correlate it across client,
gateway, application, and dependency logs. I would mitigate customer impact
while collecting evidence, rather than repeatedly redeploying without a
hypothesis."

**Expectations**

Structured investigation and an understanding of environmental differences.

**Follow-up**

What if the customer cannot share logs?

**Strong direction**

Provide a redacted diagnostic procedure, specific fields to collect, and a
customer-run test package that avoids exposing credentials or business data.

### Q4. How do you measure deployment success?

**Answer**

"I would agree on a baseline and acceptance criteria before implementation.
Technical measures might include error rate, processing delay, and recovery
behavior. Business measures might include reduced manual reconciliation or
faster support resolution.

I would also verify that the customer's team can operate the solution.
A successful demo without adoption or support ownership is not enough."

**Expectations**

Baseline, measurable outcome, operational readiness, and adoption.

### Q5. Why are you interested in FDE work?

**Answer template**

"My experience with APIs, AWS, payments, and production troubleshooting has
shown me that difficult problems often cross application, infrastructure, and
customer-workflow boundaries. I enjoy following a problem across those
boundaries and explaining the result clearly.

I want an FDE role because it combines implementation with direct responsibility
for whether the solution works for the customer."

Adapt this to your genuine experience.

## 1.9 Common mistakes

- Starting implementation before understanding the workflow.
- Confusing a successful API response with a successful business transaction.
- Promising deadlines before checking dependencies.
- Creating customer-specific code without an ownership plan.
- Treating security review as a final administrative step.
- Handing over a system without a runbook.
- Claiming customer impact without a baseline.

## 1.10 Best practices

- Write down assumptions and unresolved questions.
- Build an end-to-end vertical slice early.
- Prefer reusable integration boundaries.
- Demonstrate progress using the customer's workflow.
- Make failure states visible.
- Separate temporary mitigation from permanent correction.
- Keep a decision log.
- Define operational ownership before launch.

## 1.11 Cheat sheet

```text name=fde_cheatsheet.txt
DISCOVER  -> What outcome matters?
DEFINE    -> What does success mean?
DESIGN    -> What are the constraints and trade-offs?
BUILD     -> What is the smallest useful end-to-end workflow?
VALIDATE  -> Does it work under failure and load?
DEPLOY    -> How do we limit and reverse risk?
OPERATE   -> Who detects and fixes problems?
IMPROVE   -> What should become reusable product capability?
```

## 1.12 Scenario practice

**Scenario:** A customer wants a full production deployment in one week, but
network access and security approval are incomplete.

**Suggested answer structure:**

1. Identify what can safely be demonstrated now.
2. List critical dependencies and owners.
3. Propose a limited pilot using approved synthetic data.
4. Do not bypass security controls.
5. Define conditions for production readiness.
6. Communicate a dependency-based schedule.

**Follow-ups:**

- What if an executive asks you to bypass the approval?
- What would you remove from scope?
- How would you explain the delay without blaming another team?
