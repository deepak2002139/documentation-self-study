# Forward Deployment Engineer Learning Guide
## Engineering Foundations, Production Practice, and Interview Preparation

**Audience:** A software engineer with approximately two years of experience in software development, AWS, APIs, payments, troubleshooting, or production support.

**Prepared:** September 27, 2026  
**Approach:** Learn → implement → break safely → investigate → explain → improve.

---

# How to Use This Guide

This is a single-file study workbook covering an entire FDE preparation curriculum. It combines explanations, implementations, exercises, interview answer keys, and a study plan.

It is not a substitute for production experience, an organization’s security requirements, or product documentation when deploying real systems. No guide can guarantee selection at a particular company.

## Your learning outcomes

By the end, you should be able to:

1. Turn an ambiguous customer problem into measurable requirements.
2. Build and debug Python integrations.
3. Query and reason about transactional data.
4. Explain cloud, container, networking, and deployment choices.
5. Investigate incidents without making the situation worse.
6. Design reliable payment and event-processing workflows.
7. Communicate trade-offs to engineers and nontechnical stakeholders.
8. Demonstrate your decisions through a working portfolio project.

## Environment

Use:

- Python 3.12 or newer for the examples.
- PostgreSQL for executable SQL examples.
- Linux, macOS, or Windows with WSL for shell exercises.
- Docker for container exercises.
- An optional local Kubernetes cluster.
- A separate AWS sandbox only when performing cloud exercises.

Examples intentionally avoid depending on newly introduced language features.

## Safety rules

- Use synthetic payment data only.
- Never store card numbers, CVVs, passwords, bearer tokens, or private keys in examples or logs.
- Do not run load tests against systems you do not own or have permission to test.
- Use a sandbox AWS account with billing alerts; alerts are not spending caps.
- Never assume a resource is free.
- Do not open databases or administrative ports to the internet.
- Commands that modify state belong in disposable labs first.
- Keep production investigation read-only until a mitigation is approved.
- Treat all code here as learning code unless explicitly hardened and tested.

## The interview answer framework

For technical questions:

1. Clarify the requirement.
2. State assumptions.
3. Explain the simplest correct solution.
4. Identify failure modes.
5. Discuss trade-offs.
6. Explain how you would test and observe it.
7. Connect the answer to customer impact.

For incidents:

**Impact → scope → evidence → hypotheses → safe mitigation → verification → prevention**

For customer requests:

**User workflow → business problem → constraints → smallest useful solution → acceptance criteria**

## Assessment rubric

Score each response from 0–4:

- **0:** Incorrect or unsafe.
- **1:** Can define terminology.
- **2:** Can implement a basic solution.
- **3:** Handles failure, security, and trade-offs.
- **4:** Connects technical decisions to measurable customer outcomes.

A two-year candidate should aim for consistent level 2–3 answers, not pretend to have principal-level experience.

---

# MODULE 1 — FORWARD DEPLOYMENT ENGINEER OVERVIEW

## 1.1 Beginner explanation

An FDE is an engineer who works close to customers to make software solve a real operational problem.

A traditional request might be:

> “Deploy this application.”

An FDE asks:

> “Who will use it, what decision should it improve, which systems must it integrate with, and how will we know the deployment succeeded?”

## 1.2 Industry-level explanation

An FDE combines implementation, integration, technical discovery, operational ownership, and customer communication.

The work often includes:

- Understanding incomplete requirements.
- Mapping customer systems and data.
- Building adapters, workflows, APIs, and applications.
- Deploying into constrained environments.
- Debugging issues across organizational boundaries.
- Teaching users and transferring operational ownership.
- Feeding reusable product improvements back to engineering.

The exact balance varies by employer. Some roles are implementation-heavy, some data-heavy, and some application-development-heavy.

## 1.3 Role comparison

| Role | Primary focus | Typical deliverable | Typical success measure |
|---|---|---|---|
| Software Engineer | Product capabilities and maintainable code | Service, feature, library | Correctness, adoption, maintainability |
| DevOps Engineer | Delivery and infrastructure automation | Pipeline, environment automation | Repeatability, deployment speed |
| SRE | Reliability through engineering | SLOs, reliability improvements | User-visible reliability, reduced toil |
| Solutions Architect | Technical fit and system design | Architecture, integration plan | Feasibility, risk reduction |
| FDE | Customer outcome through hands-on engineering | Working deployment and integration | Adoption, operational value |

These boundaries overlap. DevOps is also a collaborative operating model, not only a job title.

## 1.4 Example customer deployment

**Scenario:** A merchant has an API payment system, a legacy accounting system, and settlement CSV files. Support employees reconcile transactions manually.

**Beginner solution:** Read both data sources and compare payment IDs.

**Production solution:**

1. Discover the meaning of authorization, capture, settlement, refund, and chargeback.
2. Establish stable identifiers across systems.
3. Normalize currencies, timestamps, and status mappings.
4. Import records with checkpoints and validation.
5. Store mismatches in a review queue.
6. Prevent duplicate imports.
7. Expose an operator dashboard.
8. Audit manual actions.
9. Measure reconciliation completeness and time saved.
10. Document failure recovery and ownership.

```text name=diagrams/customer-reconciliation.txt
Merchant API ──┐
              ├──> Import adapters ──> Canonical records ──> Reconciler
Settlement CSV┘                                          │
                                                         v
Accounting system <── Approved actions <── Review dashboard
                                         │
                                         v
                                   Audit history
```

## 1.5 Example daily activities

- 09:00: Review overnight failures and customer impact.
- 09:30: Clarify a workflow with a customer operator.
- 10:30: Implement an integration adapter.
- 12:00: Review API and security assumptions.
- 13:00: Test malformed records and retry behavior.
- 14:00: Investigate a staging deployment issue.
- 15:00: Demonstrate the workflow to users.
- 16:00: Update the runbook and engineering backlog.

This is an illustrative schedule, not a universal expectation.

## 1.6 Coding example: measurable acceptance criteria

```python name=acceptance.py
def deployment_accepted(total, reconciled, p95_seconds, audit_enabled):
    if total <= 0:
        return False
    return (
        reconciled / total >= 0.99
        and p95_seconds <= 60
        and audit_enabled
    )

assert deployment_accepted(1000, 995, 40, True)
assert not deployment_accepted(1000, 995, 40, False)
```

The numerical targets are example customer requirements, not industry standards.

## 1.7 Hands-on exercise

Write a one-page discovery document containing:

- Business problem.
- Users and current workflow.
- Source systems.
- Data ownership.
- Authentication requirements.
- Volume and latency expectations.
- Failure consequences.
- Acceptance criteria.
- Out-of-scope work.
- Support owner.

**Acceptance check:** Another engineer should be able to explain what you are building and why without asking you to restate the problem.

## 1.8 Interview questions and answers

**Q: How is an FDE different from a support engineer?**

An FDE may troubleshoot, but the role usually extends into designing and implementing customer-specific solutions. I would distinguish reactive issue resolution from owning the discovery-to-deployment lifecycle, while recognizing that actual responsibilities vary.

**Q: A customer requests a feature you cannot deliver this month. What do you do?**

I clarify the underlying workflow and urgency, identify a safe workaround or smaller deliverable, explain constraints, and agree on milestones. I avoid committing engineering dates without confirming dependencies.

**Q: How do you avoid creating an unmaintainable customer fork?**

I separate reusable product behavior from configuration and adapters. I document intentional deviations, define owners, and move repeated requirements into shared product capabilities.

## 1.9 Scenario

**Customer:** “Your integration is broken.”

**Strong response:**

- Ask what outcome failed and when.
- Obtain a sanitized request ID and example.
- Establish affected tenants and workflows.
- Trace the operation across boundaries.
- Distinguish data, configuration, network, and product failures.
- Provide a clear next update time.
- Avoid assigning blame before collecting evidence.

## 1.10 Common mistakes

- Coding before understanding the workflow.
- Treating deployment as the finish line.
- Agreeing to every customer request.
- Hiding uncertainty.
- Building one-off logic without an owner.
- Measuring infrastructure health but not customer success.

## 1.11 Best practices and cheat sheet

**Discover → design → implement → validate → deploy → observe → transfer ownership**

A successful deployment needs:

- Technical correctness.
- Operational usability.
- Security.
- Measurable value.
- Clear maintenance ownership.

**Interviewers expect:** Curiosity, engineering depth, customer empathy, and reliable follow-through.

**Likely follow-ups:** What did you personally implement? How did users adopt it? What failed? What would you generalize?

Reference: R1.

---

# MODULE 2 — PYTHON FOR FDE

## 2.1 Concept explanations

| Topic | Beginner explanation | Industry application | Main pitfall |
|---|---|---|---|
| Variables | Names referring to values | Configuration and state | Confusing mutation with rebinding |
| Loops | Repeat work | Process records or pages | Unbounded iteration |
| Functions | Named reusable operations | Testable business logic | Hidden side effects |
| OOP | Group state and behavior | Client adapters and domain models | Inheritance without a useful abstraction |
| Exceptions | Signal failed operations | Separate validation, transient, and permanent failures | Catching everything silently |
| Files | Read or write stored data | Imports, reports, checkpoints | Loading huge files entirely |
| API calls | Exchange messages over HTTP | Customer integrations | Missing timeout and retry policy |
| JSON | Structured interchange format | API payloads and events | Parsing without schema validation |
| Logging | Record diagnostic events | Incident investigation | Logging credentials |
| Threads | Concurrent execution with shared memory | Blocking I/O fan-out | Races and unbounded workers |
| Async | Cooperative task scheduling | Many concurrent I/O operations | Blocking the event loop |

### Important distinction

Concurrency means tasks can make progress during overlapping periods.

Parallelism means tasks execute simultaneously.

In conventional GIL-enabled CPython, Python-bytecode-heavy threads generally do not provide CPU parallelism. Native extensions may release the GIL, and free-threaded builds differ. State the interpreter and workload instead of saying “Python can never run threads in parallel.”

## 2.2 Production architecture

```text name=diagrams/python-importer.txt
Configuration
     │
     v
API/File reader → Validator → Transformer → Durable writer
     │                │                         │
     │                v                         v
     └──────────> Error report             Checkpoint
                         │
                         v
                  Logs and metrics
```

## 2.3 Twenty Python coding examples

The following file contains exactly twenty numbered examples. API examples use the local server introduced in Module 4.

```python name=python_examples.py
import asyncio
import csv
import json
import logging
import os
import threading
from collections import Counter
from concurrent.futures import ThreadPoolExecutor
from dataclasses import dataclass
from decimal import Decimal
from pathlib import Path
from urllib.request import Request, urlopen

# P01: Variables and currency-aware integer minor units.
amount_minor = 1250
currency = "USD"

# P02: Loop and filtering.
statuses = ["succeeded", "failed", "succeeded"]
successful_count = sum(1 for status in statuses if status == "succeeded")

# P03: Functions and validation.
def fee_minor(amount, basis_points):
    if type(amount) is not int or type(basis_points) is not int:
        raise TypeError("integer inputs required")
    if amount < 0 or basis_points < 0:
        raise ValueError("nonnegative inputs required")
    return (amount * basis_points + 5000) // 10000

# P04: Reverse a string by Unicode code points.
def reverse_string(text):
    return text[::-1]

# P05: Find duplicate hashable values.
def duplicates(values):
    counts = Counter(values)
    return {value for value, count in counts.items() if count > 1}

# P06: Count frequency.
def frequencies(values):
    return dict(Counter(values))

# P07: OOP with an immutable data container.
@dataclass(frozen=True)
class Payment:
    payment_id: str
    amount_minor: int
    currency: str

# P08: Targeted exception handling.
def parse_integer(text):
    try:
        return int(text)
    except (TypeError, ValueError) as exc:
        raise ValueError("invalid integer") from exc

# P09: Stream a large text file.
def matching_lines(path, needle):
    with open(path, encoding="utf-8") as handle:
        for line_number, line in enumerate(handle, start=1):
            if needle in line:
                yield line_number, line.rstrip("\n")

# P10: Parse JSON and validate its shape.
def parse_payment(raw):
    obj = json.loads(raw)
    if not isinstance(obj, dict):
        raise ValueError("JSON object required")
    if not isinstance(obj.get("payment_id"), str):
        raise ValueError("payment_id required")
    if type(obj.get("amount_minor")) is not int:
        raise ValueError("amount_minor must be an integer")
    if obj["amount_minor"] <= 0:
        raise ValueError("positive amount required")
    if obj.get("currency") not in {"USD", "EUR", "JPY"}:
        raise ValueError("unsupported currency in this lab")
    return Payment(
        obj["payment_id"], obj["amount_minor"], obj["currency"]
    )

# P11: Structured logging with a strict field allowlist.
def log_event(event, request_id, status):
    logging.getLogger("fde").info(json.dumps({
        "event": event,
        "request_id": request_id,
        "status": status,
    }))

# P12: GET request with a timeout and response-size bound.
def get_json(url, max_bytes=1_000_000):
    request = Request(url, headers={"Accept": "application/json"})
    with urlopen(request, timeout=3) as response:
        raw = response.read(max_bytes + 1)
        if len(raw) > max_bytes:
            raise ValueError("response exceeds configured limit")
        return json.loads(raw)

# P13: CSV streaming.
def csv_rows(path):
    with open(path, newline="", encoding="utf-8") as handle:
        yield from csv.DictReader(handle)

# P14: Generator transformation.
def successful_ids(payments):
    for payment in payments:
        if payment["status"] == "succeeded":
            yield payment["payment_id"]

# P15: Thread pool for a small, bounded collection.
def fetch_many(urls):
    with ThreadPoolExecutor(max_workers=4) as pool:
        return list(pool.map(get_json, urls))

# P16: Protect a compound operation.
class SafeCounter:
    def __init__(self):
        self.value = 0
        self._lock = threading.Lock()

    def increment(self):
        with self._lock:
            self.value += 1

# P17: Async tasks with bounded active work.
async def async_demo():
    semaphore = asyncio.Semaphore(3)

    async def task(number):
        async with semaphore:
            await asyncio.sleep(0.01)
            return number * number

    return await asyncio.gather(*(task(n) for n in range(10)))

# P18: Move blocking I/O off the event-loop thread.
async def async_get_json(url):
    return await asyncio.to_thread(get_json, url)

# P19: Replace a file atomically within the same filesystem.
# Single-writer teaching example; not a crash-durable database.
def write_checkpoint(path, value):
    path = Path(path)
    temporary = path.with_name(path.name + ".tmp")
    with temporary.open("w", encoding="utf-8") as handle:
        json.dump(value, handle)
        handle.flush()
        os.fsync(handle.fileno())
    os.replace(temporary, path)

# P20: Decimal arithmetic from strings.
def decimal_total(values):
    return sum((Decimal(value) for value in values), Decimal("0"))

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    assert fee_minor(10000, 250) == 250
    assert reverse_string("fde") == "edf"
    assert duplicates([1, 2, 2, 3]) == {2}
    assert frequencies(["a", "a"]) == {"a": 2}
    assert decimal_total(["0.10", "0.20"]) == Decimal("0.30")
    assert asyncio.run(async_demo())[-1] == 81
    print("Python example checks passed")
```

### Production qualifications

- P12 uses trusted URLs. Do not pass arbitrary customer-controlled URLs without SSRF protections.
- A socket timeout is not necessarily a complete end-to-end deadline.
- P15 bounds worker count, not arbitrary input size. Use producer-consumer backpressure for huge streams.
- P17 is safe for ten tasks; creating millions of waiting tasks still consumes memory.
- P18 cancellation does not forcibly stop the underlying thread.
- P19 uses a fixed temporary filename and assumes one writer. Stronger durability also requires platform-specific directory synchronization.
- P20 does not define currency rounding rules. Those must come from the business contract.

## 2.4 Twenty exercises

| ID | Exercise | Acceptance criterion |
|---|---|---|
| E01 | Find duplicate IDs | Empty list and repeated values pass |
| E02 | Reverse text | Explain code points versus grapheme clusters |
| E03 | Count status frequency | Missing statuses handled explicitly |
| E04 | Validate payment JSON | Reject booleans as amounts |
| E05 | Stream a large log | Memory does not grow with file size |
| E06 | Build a CSV importer | Report bad row numbers |
| E07 | Add API timeout handling | Network failure produces useful output |
| E08 | Add safe GET retries | Attempts are capped |
| E09 | Follow pagination | Detect repeated cursors |
| E10 | Add structured logs | No secrets or full payment payloads |
| E11 | Build a payment class | Invalid state transitions rejected |
| E12 | Add unit tests | Include empty and invalid input |
| E13 | Compare threads and sequential I/O | Report measured results |
| E14 | Build async fan-out | Limit concurrency |
| E15 | Demonstrate a race | Fix with synchronization |
| E16 | Save and resume checkpoints | Restart does not skip uncommitted data |
| E17 | Reconcile two CSV files | Report missing and mismatched records |
| E18 | Deduplicate events | State the deduplication scope |
| E19 | Implement graceful shutdown | Stop accepting new work, finish bounded work |
| E20 | Build a CLI | Invalid arguments return nonzero exit status |

Suggested solutions are in P01–P20, Module 4, and Module 16.

## 2.5 Twenty interview questions and answers

1. **List versus tuple?** A list is mutable; a tuple is immutable as a container. A tuple containing a mutable object is not deeply immutable.
2. **Dictionary lookup complexity?** Average O(1), with pathological worst cases. Hashing and equality costs also matter.
3. **`is` versus `==`?** `is` compares identity; `==` compares value. Use `is None`.
4. **Why avoid mutable default arguments?** The default object is created once and reused. Use `None` and create a new object inside.
5. **What is a generator?** An iterator that produces values lazily, preserving execution state between yields.
6. **How do you read a huge file?** Stream records, bound line or record sizes where needed, and avoid collecting all results.
7. **What does `with` do?** It enters and exits a context manager, enabling cleanup even when exceptions occur.
8. **When should you catch an exception?** When you can recover, add meaningful context, translate it at a boundary, or perform cleanup.
9. **Why not `except: pass`?** It hides failures and may suppress termination signals.
10. **Thread or process?** Threads often suit blocking I/O; processes can suit Python CPU work. Measure serialization and startup overhead.
11. **What does `await` mean?** Wait for an awaitable while allowing cooperative scheduling when it suspends; it does not inherently create parallel execution.
12. **Does calling `async def` execute the function?** It creates a coroutine object. It must be awaited or scheduled.
13. **Why is a synchronous HTTP call inside async code problematic?** It blocks the event-loop thread unless offloaded or replaced with an async client.
14. **How do you validate JSON?** Parse syntax, then validate types, required fields, ranges, allowed values, and unknown-field policy.
15. **How do you represent money?** Integer minor units plus currency, or Decimal with explicit precision and rounding. Currency scales differ.
16. **How do you test API code?** Inject transport, time, and dependencies; test success, timeout, malformed response, rate limiting, and duplicate operations.
17. **What should a log contain?** Event, timestamp, severity, correlation identifiers, safe context, and actionable error information.
18. **Why bound concurrency?** To protect memory, sockets, database pools, downstream quotas, and latency.
19. **What is dependency injection?** Passing dependencies into code instead of constructing hidden globals, improving testing and replacement.
20. **What makes an automation script production-worthy?** Validation, idempotency, timeouts, bounded retries, observability, resumability, tests, and clear exit behavior.

## 2.6 Scenario, mistakes, and expectations

**Scenario:** An importer crashes after writing 700 records but before saving its checkpoint.

**Answer:** Resume from the last durable checkpoint and tolerate reprocessing with stable record keys and transactional writes. Saving the checkpoint before the data commit can lose data.

**Mistakes:** Retrying permanent errors, swallowing exceptions, using floats for money, logging secrets, and assuming an in-memory set gives durable deduplication.

**Best practice:** Make side effects explicit and test failure boundaries.

**Cheat sheet:** `dict`, `set`, `Counter`, generators, context managers, `logging`, executors, `asyncio`, explicit deadlines.

**Interviewers expect:** Correct code plus handling of empty input, bad data, network failures, and scale.

**Follow-ups:** What happens after restart? How much memory is used? What if two workers process the same record?

References: R2–R4.

---

# MODULE 3 — SQL AND DATABASES

## 3.1 Beginner and industry explanations

| Topic | Simple explanation | Production interpretation |
|---|---|---|
| SELECT | Read columns | Retrieve the smallest useful result |
| JOIN | Combine related tables | Match records with explicit cardinality |
| GROUP BY | Form groups | Aggregate business metrics |
| HAVING | Filter groups | Apply aggregate conditions |
| Window functions | Calculate across related rows without collapsing them | Ranking, latest-event selection, running totals |
| Index | Lookup structure | Faster reads with write and storage costs |
| Normalization | Separate facts cleanly | Reduce update anomalies |
| ACID | Transaction guarantees | Protect invariants under failures and concurrency |
| Transactions | All-or-nothing unit of work | Group related changes |
| Optimization | Reduce unnecessary work | Measure plans, waits, data access, and workload |

### Logical query order

A useful mental model:

**FROM/JOIN → WHERE → GROUP BY → HAVING → window calculations → SELECT → ORDER BY → LIMIT**

The optimizer may execute a physically different plan.

## 3.2 ACID

- **Atomicity:** Related changes commit or roll back together.
- **Consistency:** Correct transactions preserve defined constraints and invariants.
- **Isolation:** Concurrent work is separated according to the isolation level.
- **Durability:** Committed data survives the failures covered by the configured durability model.

ACID consistency is not the same concept as distributed replica consistency.

## 3.3 Database selection

| Database | Core model | Good fit | Important trade-off |
|---|---|---|---|
| PostgreSQL | Relational | Transactional systems, rich queries | Index, vacuum, connection, and workload management |
| MySQL/InnoDB | Relational | Web transactional applications | Engine and isolation semantics matter |
| Oracle | Relational | Existing enterprise ecosystems | Operational complexity and licensing constraints |
| DynamoDB | Key-value/document | Known access patterns at scale | Partition-key design and secondary-index behavior |
| Neptune | Graph | Relationship traversal and fraud-link analysis | Not a replacement for a transaction ledger |
| MongoDB | Document | Aggregate-oriented document models | Schema discipline and transaction boundaries still matter |

MongoDB supports multi-document transactions. “NoSQL databases cannot provide transactions” is incorrect.

## 3.4 Lab schema and data

```sql name=sql/schema.sql
CREATE TABLE merchants (
    merchant_id BIGINT PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE payments (
    payment_id BIGINT PRIMARY KEY,
    merchant_id BIGINT NOT NULL REFERENCES merchants(merchant_id),
    idempotency_key TEXT NOT NULL,
    amount_minor BIGINT NOT NULL CHECK (amount_minor > 0),
    currency CHAR(3) NOT NULL,
    status TEXT NOT NULL CHECK (
        status IN ('pending', 'succeeded', 'failed')
    ),
    created_at TIMESTAMPTZ NOT NULL,
    UNIQUE (merchant_id, idempotency_key)
);

CREATE TABLE payment_events (
    event_id BIGINT PRIMARY KEY,
    payment_id BIGINT NOT NULL REFERENCES payments(payment_id),
    status TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL
);

INSERT INTO merchants VALUES
    (1, 'Example Shop'),
    (2, 'Demo Travel');

INSERT INTO payments VALUES
    (101, 1, 'key-101', 1000, 'USD', 'succeeded', '2026-09-01T10:00:00Z'),
    (102, 1, 'key-102', 2000, 'USD', 'failed',    '2026-09-01T11:00:00Z'),
    (103, 1, 'key-103', 1500, 'USD', 'succeeded', '2026-09-02T10:00:00Z'),
    (104, 2, 'key-104', 5000, 'EUR', 'pending',   '2026-09-02T11:00:00Z'),
    (105, 2, 'key-105', 3000, 'EUR', 'succeeded', '2026-09-03T10:00:00Z'),
    (106, 2, 'key-106', 4000, 'EUR', 'failed',    '2026-09-03T11:00:00Z');

INSERT INTO payment_events VALUES
    (1, 101, 'pending',   '2026-09-01T09:59:00Z'),
    (2, 101, 'succeeded', '2026-09-01T10:00:00Z'),
    (3, 104, 'pending',   '2026-09-02T11:00:00Z');
```

## 3.5 Practical payment queries

```sql name=sql/payment_queries.sql
-- Q1: Read only required columns.
SELECT payment_id, amount_minor, currency
FROM payments
WHERE status = 'succeeded'
ORDER BY payment_id;

-- Q2: Join payments with merchants.
SELECT p.payment_id, m.name, p.status
FROM payments p
JOIN merchants m ON m.merchant_id = p.merchant_id;

-- Q3: Revenue-like captured totals, separated by currency.
-- "succeeded" is a simplified lab status, not accounting revenue.
SELECT merchant_id, currency, SUM(amount_minor) AS total_minor
FROM payments
WHERE status = 'succeeded'
GROUP BY merchant_id, currency;

-- Q4: HAVING filters aggregate groups.
SELECT merchant_id, COUNT(*) AS payment_count
FROM payments
GROUP BY merchant_id
HAVING COUNT(*) >= 3;

-- Q5: Success rate with an explicit denominator.
SELECT merchant_id,
       COUNT(*) FILTER (WHERE status = 'succeeded')::numeric
       / NULLIF(COUNT(*), 0) AS success_rate
FROM payments
GROUP BY merchant_id;

-- Q6: Latest event per payment, with deterministic tie-breaking.
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY payment_id
               ORDER BY created_at DESC, event_id DESC
           ) AS rn
    FROM payment_events
)
SELECT payment_id, status, created_at
FROM ranked
WHERE rn = 1;

-- Q7: Running successful amount per merchant and currency.
SELECT payment_id, merchant_id, currency,
       SUM(amount_minor) OVER (
           PARTITION BY merchant_id, currency
           ORDER BY created_at, payment_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total
FROM payments
WHERE status = 'succeeded';

-- Q8: Merchants without successful payments.
SELECT m.merchant_id
FROM merchants m
WHERE NOT EXISTS (
    SELECT 1
    FROM payments p
    WHERE p.merchant_id = m.merchant_id
      AND p.status = 'succeeded'
);

-- Q9: Pending payments older than a chosen evaluation time.
SELECT payment_id
FROM payments
WHERE status = 'pending'
  AND created_at < TIMESTAMPTZ '2026-09-04T00:00:00Z'
                   - INTERVAL '30 minutes';

-- Q10: Efficient half-open timestamp range.
SELECT payment_id
FROM payments
WHERE created_at >= TIMESTAMPTZ '2026-09-01T00:00:00Z'
  AND created_at <  TIMESTAMPTZ '2026-09-02T00:00:00Z';

-- Q11: Keyset pagination; cursor contains BOTH values.
SELECT payment_id, created_at
FROM payments
WHERE (created_at, payment_id) >
      (TIMESTAMPTZ '2026-09-01T10:00:00Z', 101)
ORDER BY created_at, payment_id
LIMIT 20;

-- Q12: Detect duplicate business keys in imported staging data.
-- On the constrained payments table this should return zero rows.
SELECT merchant_id, idempotency_key, COUNT(*)
FROM payments
GROUP BY merchant_id, idempotency_key
HAVING COUNT(*) > 1;
```

Expected checks:

- Successful USD total for merchant 1: `2500`.
- Successful EUR total for merchant 2: `3000`.
- Merchant 1 success rate: `2/3`.
- Latest status for payment 101: `succeeded`.

Do not add USD and EUR amounts together without an explicit conversion model.

## 3.6 Transactions and concurrent updates

```sql name=sql/conditional_transition.sql
BEGIN;

UPDATE payments
SET status = 'succeeded'
WHERE payment_id = 104
  AND status = 'pending'
RETURNING payment_id;

-- The application must inspect whether a row was returned.
-- Zero rows means the precondition did not hold.

COMMIT;
```

This demonstrates an atomic state transition, not a complete payment processor.

A production workflow also needs authorized transition rules, durable events, processor reconciliation, and auditability.

## 3.7 Performance tuning

```sql name=sql/tuning.sql
CREATE INDEX payments_merchant_time_idx
ON payments (merchant_id, created_at DESC, payment_id DESC);

EXPLAIN (ANALYZE, BUFFERS)
SELECT payment_id, amount_minor
FROM payments
WHERE merchant_id = 1
ORDER BY created_at DESC, payment_id DESC
LIMIT 20;
```

`EXPLAIN ANALYZE` executes the statement. Use it carefully, especially for writes and expensive queries.

### Investigation sequence

1. Identify the slow query and its frequency.
2. Separate CPU, I/O, lock waits, and connection waits.
3. Inspect estimated versus actual row counts.
4. Check filtering and join cardinality.
5. Examine indexes and sort operations.
6. Consider stale statistics.
7. Reduce returned data.
8. Retest with realistic volume and concurrency.

A sequential scan is not automatically wrong. It can be optimal for small tables or queries reading most rows.

## 3.8 Architecture

```text name=diagrams/database-access.txt
API replicas → Bounded connection pools → Primary database
                                          │
                                          ├── Transaction log
                                          └── Replica → Reporting
                                               │
                                               └── Possible replication lag
```

## 3.9 Thirty SQL/database interview questions

1. **WHERE versus HAVING?** WHERE filters rows before aggregation; HAVING filters grouped results.
2. **INNER versus LEFT JOIN?** INNER keeps matches; LEFT preserves every left row and fills missing right columns with NULL.
3. **Why did a join multiply rows?** The relationship was one-to-many or many-to-many. Check keys before aggregating.
4. **COUNT(*) versus COUNT(column)?** COUNT(*) counts rows; COUNT(column) excludes NULL values.
5. **Why not `column = NULL`?** NULL represents unknown; use `IS NULL`.
6. **GROUP BY versus window functions?** GROUP BY collapses rows; window functions preserve row detail.
7. **ROW_NUMBER versus RANK?** ROW_NUMBER gives distinct sequence positions; RANK gives ties the same rank and leaves gaps.
8. **RANK versus DENSE_RANK?** DENSE_RANK does not leave gaps after ties.
9. **Find the latest row per entity?** Use ROW_NUMBER partitioned by entity, ordered by timestamp and a deterministic tie-breaker.
10. **What is an index?** A maintained access structure that can reduce lookup and ordering work.
11. **Why not index every column?** Indexes consume storage and increase insert, update, and maintenance costs.
12. **How do you order composite-index columns?** Start from actual predicates and ordering; equality, range, selectivity, and engine behavior matter.
13. **What is a covering index?** An index containing the values a query needs; whether table access is avoided depends on the engine and visibility.
14. **What is normalization?** Organizing dependencies to reduce duplicate facts and update anomalies.
15. **What is denormalization?** Deliberate duplication or precomputation to serve reads, with a consistency-maintenance cost.
16. **What does ACID consistency mean?** Transactions preserve defined invariants; the database cannot invent missing business rules.
17. **What is a dirty read?** Reading uncommitted data from another transaction.
18. **What is a nonrepeatable read?** Reading the same row twice and observing a committed change.
19. **What is a phantom?** A repeated predicate query observes a changed set of qualifying rows.
20. **What is a deadlock?** Transactions wait cyclically for resources. The database typically aborts one; retry the entire safe transaction.
21. **Optimistic versus pessimistic concurrency?** Optimistic checks versions or conditions; pessimistic locks before conflicting work.
22. **Why keep transactions short?** Long transactions hold resources, extend contention, and complicate maintenance.
23. **Why is OFFSET pagination expensive?** Large offsets may require scanning and discarding many rows. Keyset pagination uses a stable cursor.
24. **What causes a slow query besides missing indexes?** Bad estimates, locks, spills, I/O, excessive rows, network transfer, or connection saturation.
25. **How do you prevent SQL injection?** Use parameterized statements; identifiers require separate validation or safe composition.
26. **Why can a replica return stale data?** Replication can lag. Read-after-write requirements may need primary reads or another consistency strategy.
27. **How would you model DynamoDB?** Start from access patterns and distribute partition-key traffic; do not design it as a relational schema first.
28. **When is Neptune useful?** When traversing relationships is central, such as linked accounts and devices in fraud analysis.
29. **Does MongoDB support ACID transactions?** Yes; the topology and read/write concerns affect behavior and operational trade-offs.
30. **Can database transactions make an external payment call atomic?** Not by themselves. Use durable intent, provider idempotency, state machines, and reconciliation.

## 3.10 Exercises and scenario

- Add an index and compare plans on a larger synthetic dataset.
- Introduce a one-to-many join and show how totals become inflated.
- Run competing conditional updates in two sessions.
- Explain how a refund should be represented without overwriting historical facts.

**Scenario:** Database CPU reaches 100% after a deployment.

**Answer:** Correlate with the release, identify high-total-cost queries, inspect plans and concurrency, and consider safe rollback. Do not assume a larger database fixes the root cause.

**Cheat sheet:** Filter early logically, aggregate at the right grain, parameterize, inspect plans, preserve currency, keep transactions short.

**Interviewers expect:** Correct results first, then performance and concurrency.

**Follow-ups:** What happens at ten times the data? What if timestamps tie? Can this query double-count? What is the consistency requirement?

References: R5–R8, R12, R13.

---

# MODULE 4 — REST APIs AND INTEGRATION

## 4.1 Foundations

| Concept | Beginner explanation | Production detail |
|---|---|---|
| HTTP | Request-response protocol | Methods, headers, status codes, caching, connection behavior |
| REST | Resource-oriented architectural style | Stateless interactions and uniform interface constraints |
| SOAP | Structured messaging protocol | XML envelopes and contracts, often with WSDL |
| JSON | Structured text format | Syntax validation is separate from schema validation |
| Authentication | Establish identity | Credentials, sessions, tokens, identity providers |
| Authorization | Decide permitted actions | Tenant, object, operation, and policy checks |
| OAuth 2.0 | Delegated authorization framework | Scope, audience, client type, and secure flow selection |
| JWT | Token format | Signature verification and claim validation; not encryption by default |
| API Gateway | Entry layer for APIs | Routing, limits, authentication integration, and observability |
| Error handling | Describe failures consistently | Stable codes, request IDs, retryability, and safe messages |

OAuth 2.0 is not itself an identity protocol. OpenID Connect adds an identity layer.

For interactive clients, understand authorization code flow with PKCE. For suitable machine-to-machine integrations, understand client credentials. Do not use insecure legacy flows simply because a tutorial does.

## 4.2 HTTP cheat sheet

| Operation | Typical meaning | Idempotency considerations |
|---|---|---|
| GET | Retrieve representation | Defined as safe and idempotent |
| POST | Submit/create according to endpoint semantics | Not inherently idempotent |
| PUT | Create/replace state at a known target | Defined as idempotent |
| PATCH | Partial modification | Depends on patch operation |
| DELETE | Remove target association | Idempotent effect; repeated response may differ |

Status codes:

- 200: Success.
- 201: Created.
- 202: Accepted, not necessarily completed.
- 204: Success without response content.
- 400: Invalid request.
- 401: Authentication credentials missing or invalid.
- 403: Refused/forbidden.
- 404: Resource not found, or existence intentionally not disclosed.
- 409: Conflict with current state.
- 429: Rate limited.
- 500: Unexpected server failure.
- 502: Invalid upstream response.
- 503: Temporarily unavailable.
- 504: Gateway timed out.

## 4.3 API architecture

```text name=diagrams/api.txt
Client
  │ HTTPS
  v
Gateway → Authentication → Authorization → Validation
                                             │
                                             v
                                       Business service
                                        │          │
                                        v          v
                                     Database   External API
```

## 4.4 Runnable local mock service

This is a standard-library teaching server. It has no real authentication, persistence, payment processing, or production HTTP hardening.

```python name=lab_server.py
import json
import logging
import os
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        routes = {
            "/healthz": {"status": "alive"},
            "/readyz": {"status": "ready"},
            "/payments": {
                "items": [
                    {"payment_id": "p1", "status": "succeeded"},
                    {"payment_id": "p2", "status": "pending"},
                ],
                "next_cursor": None,
            },
        }
        payload = routes.get(self.path)
        code = 200 if payload is not None else 404
        if payload is None:
            payload = {"error": {"code": "not_found"}}
        body = json.dumps(payload).encode()
        self.send_response(code)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def log_message(self, format, *args):
        # Do not log arbitrary URLs, tokens, or request bodies.
        logging.info("http_request")

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    host = os.environ.get("HOST", "127.0.0.1")
    port = int(os.environ.get("PORT", "8080"))
    server = ThreadingHTTPServer((host, port), Handler)
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        pass
    finally:
        server.server_close()
```

Run:

```bash name=scripts/run-local-api.sh
python3 lab_server.py
```

In another terminal:

```bash name=scripts/check-local-api.sh
curl --fail --max-time 3 http://127.0.0.1:8080/healthz
curl --fail --max-time 3 http://127.0.0.1:8080/payments
```

## 4.5 Python integration with bounded retries

```python name=api_client.py
import json
import random
import time
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen

RETRYABLE = {429, 502, 503, 504}

def fetch_json(url, attempts=3, max_bytes=1_000_000):
    if attempts < 1:
        raise ValueError("attempts must be positive")

    for attempt in range(attempts):
        delay = random.uniform(0, min(2.0, 0.25 * 2**attempt))
        try:
            request = Request(
                url,
                headers={"Accept": "application/json"}
            )
            with urlopen(request, timeout=3) as response:
                content_type = response.headers.get_content_type()
                if content_type != "application/json":
                    raise ValueError("unexpected content type")
                raw = response.read(max_bytes + 1)
                if len(raw) > max_bytes:
                    raise ValueError("response too large")
                return json.loads(raw)

        except HTTPError as exc:
            try:
                if exc.code not in RETRYABLE or attempt == attempts - 1:
                    raise
                retry_after = exc.headers.get("Retry-After")
                if retry_after:
                    # Lab supports delta-seconds, not HTTP-date form.
                    try:
                        server_delay = float(retry_after)
                    except ValueError:
                        server_delay = None
                    if server_delay is not None:
                        if not 0 <= server_delay <= 30:
                            raise RuntimeError(
                                "server retry delay exceeds this lab's budget"
                            )
                        delay = max(delay, server_delay)
            finally:
                exc.close()

        except (URLError, TimeoutError):
            if attempt == attempts - 1:
                raise

        time.sleep(delay)

    raise RuntimeError("unreachable")

if __name__ == "__main__":
    print(fetch_json("http://127.0.0.1:8080/payments"))
```

Production additions:

- End-to-end deadline.
- Connection pooling.
- Full Retry-After parsing.
- Validated schemas and pagination.
- Destination allowlisting where relevant.
- TLS verification.
- Correlation IDs.
- Metrics.
- Provider-specific retry semantics.

Do not reuse this GET retry policy blindly for payment creation.

## 4.6 Idempotency in depth

**Simple explanation:** Repeating the same logical operation does not create an additional effect.

**Production design:**

1. Scope the key by tenant and operation.
2. Canonicalize relevant request fields.
3. Store a request fingerprint.
4. Atomically claim the key using a uniqueness constraint.
5. Reject the same key with a different payload.
6. Persist operation state and response information.
7. Reuse a stable provider idempotency key.
8. Reconcile ambiguous external outcomes.

A database record and an external payment call do not share one transaction. A crash after the provider succeeds but before your commit requires reconciliation.

## 4.7 Interview answers

**PUT versus PATCH?** PUT supplies replacement state for the target representation. PATCH applies partial changes. “Set status to active” can be idempotent; “increment balance” generally is not.

**404 versus 500?** A 404 concerns the requested resource or its visibility. A 500 indicates an unexpected server-side failure. Neither should expose internal stack traces.

**Authentication versus authorization?** Authentication identifies the caller. Authorization checks whether that caller can perform this operation on this resource.

**What should JWT verification include?** Allowed algorithms, trusted keys, signature, issuer, audience, expiration, and relevant time claims. Merely decoding the token is not verification.

**A payment request times out. Is it safe to retry?** The outcome is unknown. Retry only using the provider’s idempotency contract or query/reconcile the operation. A timeout does not prove failure.

## 4.8 Exercise and scenario

Build a mock endpoint that returns:

- A success.
- Invalid JSON.
- 429.
- 503.
- A slow response.

Verify retry counts and elapsed-time bounds.

**Mistakes:** Unlimited retries, disabled TLS verification, exposing stack traces, accepting arbitrary JWT algorithms, and missing object-level authorization.

**Best practice:** Define a contract for every endpoint: input, output, auth, errors, idempotency, limits, and versioning.

**Interviewers expect:** Protocol correctness and reasoning about failure boundaries.

**Follow-ups:** What if the response is lost? What if two requests arrive together? How do you rotate signing keys? How is tenant isolation enforced?

References: R14–R16.

---

# MODULE 5 — AWS

## 5.1 Mental model

AWS provides managed building blocks. Your job is to select the simplest set that satisfies the workload’s reliability, security, cost, and operational requirements.

Managed does not mean “no operational responsibility.”

## 5.2 Service-by-service guide

| Service | What it is / why it exists | Real-world use | Production scenario | Interview question and answer |
|---|---|---|---|---|
| EC2 | Virtual compute for OS-level control | Legacy adapter | Instance fails; replace from an image rather than hand-repairing forever | **EC2 or Lambda?** Choose based on runtime, control, workload shape, and operations |
| S3 | Object storage | Settlement files and exports | Bad overwrite; recover through configured versioning and retention | **Is S3 a normal filesystem?** No; design around object semantics |
| RDS | Managed relational database service | Payment records | Failover interrupts connections; app reconnects safely | **Multi-AZ versus replica?** Availability and read-scaling roles depend on deployment type |
| Lambda | Event-driven managed function execution | Webhook processing | Timeouts cause retried work; make effects idempotent | **Why timeouts?** Dependency delay, networking, resource pressure, or oversized work |
| IAM | Identity and policy system | Workload access | AccessDenied after role change | **Role versus access key?** Prefer workload roles and temporary credentials |
| CloudWatch | AWS telemetry and alarms | Lambda errors and queue age | Alarm fires; correlate logs, metrics, and deployment time | **Metric versus log?** Numeric trend versus detailed event record |
| ECS | AWS container orchestration | Long-running API | Task is healthy locally but fails load-balancer checks | **ECS versus EC2?** ECS orchestrates tasks; compute can be EC2 or another supported capacity option |
| EKS | Managed Kubernetes control plane | Kubernetes-based platform | Pods pending because capacity or constraints prevent scheduling | **Is EKS fully hands-off?** No; workloads, networking, access, upgrades, and data need ownership |
| DynamoDB | Managed key-value/document database | Idempotency records | Hot partition throttles one tenant | **How choose keys?** Access patterns and traffic distribution |
| SNS | Pub/sub fan-out | Notify several subscribers | One consumer fails independently | **SNS versus SQS?** Fan-out distribution versus durable work queue |
| SQS | Managed message queue | Payment-processing jobs | Backlog grows; processing capacity is below arrival rate | **Can messages repeat?** Design consumers for duplicates |
| EventBridge | Event routing using rules and buses | Route domain events | Schema change breaks a target | **Why not only SQS?** Routing and integration requirements differ from work buffering |
| API Gateway | Managed API front door | Public integration API | Requests are throttled or rejected before reaching the app | **What belongs there?** Routing and cross-cutting controls, not all business logic |
| Secrets Manager | Managed secret storage and rotation support | Database credentials | Rotation succeeds but app retains stale credentials | **How use safely?** Least privilege, bounded caching, rotation-aware clients |
| VPC | Logically isolated network | Private application and database tiers | Private workload cannot reach a dependency | **What to inspect?** Routes, endpoints, DNS, security groups, NACLs, and egress |

### Service-specific exercises

1. **EC2:** Draw boot, health-check, replacement, and patching procedures.
2. **S3:** Design a prefix layout and lifecycle policy for synthetic reports.
3. **RDS:** Write a connection-recovery and backup-restore test plan.
4. **Lambda:** Process a duplicate event without duplicate side effects.
5. **IAM:** Write a policy allowing only a specific operation on a specific resource.
6. **CloudWatch:** Define an alarm with an owner and response.
7. **ECS:** Identify task role versus task execution role responsibilities.
8. **EKS:** Map service accounts to workload permissions.
9. **DynamoDB:** Design keys for tenant-scoped idempotency records.
10. **SNS:** Draw fan-out to two independent queues.
11. **SQS:** Simulate a poison message and DLQ recovery.
12. **EventBridge:** Define an event envelope and schema evolution rule.
13. **API Gateway:** Specify request limits and authentication behavior.
14. **Secrets Manager:** Describe rotation without restarting all consumers.
15. **VPC:** Draw public and private subnet routing.

## 5.3 Reference architecture

```text name=diagrams/aws-payment-platform.txt
Internet
   │
   v
API Gateway → Lambda/API service → RDS
                       │             │
                       │             └── Durable outbox
                       │                       │
                       v                       v
                 Secrets Manager         Outbox relay
                                               │
                                               v
                                         SQS → Workers
                                               │
                                               v
                                         Payment provider

S3: import/export files
IAM: workload permissions
CloudWatch: metrics, logs, alarms
VPC: network boundaries
```

Alternative: An ALB fronts an ECS service. Do not add both API Gateway and ALB without a reason.

## 5.4 SQS partial-batch example

```python name=lambda_handler.py
def handler(event, context):
    failures = []
    for record in event["Records"]:
        try:
            process_record(record)  # Implement durable idempotent processing.
        except Exception:
            # Boundary catch: report this item as failed.
            # Emit sanitized diagnostics in a real implementation.
            failures.append({"itemIdentifier": record["messageId"]})
    return {"batchItemFailures": failures}

def process_record(record):
    raise NotImplementedError("connect to a durable processor")
```

This is an interface skeleton, not a ready-to-deploy handler.

Requirements:

- Enable partial batch responses in the event source mapping.
- Configure visibility timeout and batch settings appropriately.
- For FIFO ordering requirements, stop and report the relevant unprocessed records after a failure.
- Do not acknowledge a message before its required durable effect.
- A DLQ needs an owner and replay procedure.

## 5.5 Common mistakes

- Giving workloads administrator access.
- Assuming a private subnet automatically has internet access.
- Assuming placing Lambda in a public subnet gives it a public IP.
- Treating read replicas as synchronous primary replacements.
- Ignoring service quotas and downstream connection limits.
- Scaling consumers until the database fails.
- Treating TTL expiration as immediate deletion.
- Assuming managed queues produce exactly-once business effects.

## 5.6 Best practices

- Prefer temporary credentials.
- Separate environments.
- Use infrastructure as code.
- Tag resources with owner and purpose.
- Design for retries and duplicates.
- Test restores, not only backup creation.
- Monitor business lag, not only CPU.
- Delete sandbox resources when finished.

**Scenario:** Queue depth grows after increasing Lambda concurrency.

**Answer:** Higher concurrency may have saturated database connections or a provider limit. Inspect processing duration, throttling, retries, connection waits, and oldest-message age. Bound concurrency to downstream capacity.

**Cheat sheet:** Compute → storage → state → messaging → identity → network → observability.

**Interviewers expect:** Reasons for choosing services, not a list of product names.

**Follow-ups:** How does it fail? What costs money while idle? What is the recovery point? How are credentials rotated?

References: R9–R13, R17–R21.

---

# MODULE 6 — DOCKER

## 6.1 Explanations

- **Image:** Packaged filesystem layers and runtime metadata.
- **Container:** A running or stopped instance with an isolated process environment.
- **Network:** Connectivity between containers and external systems.
- **Volume:** Storage managed separately from a container’s writable layer.
- **Dockerfile:** Build instructions for an image.

Containers normally share the host kernel. They are not miniature virtual machines.

## 6.2 Architecture

```text name=diagrams/docker.txt
Source + Dockerfile → Build → Image → Registry
                                         │
                                         v
                                   Container runtime
                                      │       │
                                      v       v
                                   Network  Volume
```

## 6.3 Dockerfile for the local API

```dockerfile name=Dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    HOST=0.0.0.0 \
    PORT=8080

WORKDIR /app

RUN groupadd --gid 10001 app \
    && useradd --uid 10001 --gid app --no-create-home app

COPY --chown=app:app lab_server.py /app/lab_server.py

USER 10001:10001

EXPOSE 8080

CMD ["python", "lab_server.py"]
```

The tag is a readable lab choice, not a claim that it is the newest image. Production builds should use a reviewed digest and an update process.

```text name=.dockerignore
.git
.venv
__pycache__
*.pyc
.env
*.pem
```

```bash name=scripts/docker-lab.sh
docker build -t fde-lab:local .
docker run --rm --name fde-lab \
  --read-only \
  --tmpfs /tmp \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  -p 127.0.0.1:8080:8080 \
  fde-lab:local
```

## 6.4 Fifteen interview questions

1. **Image versus container?** An image is the template; a container is an instance with runtime state.
2. **Why are image layers useful?** They enable reuse and caching; build ordering affects cache efficiency.
3. **COPY versus ADD?** Prefer COPY for straightforward copies; ADD has additional behavior that should be intentional.
4. **CMD versus ENTRYPOINT?** ENTRYPOINT defines the executable pattern; CMD supplies defaults or a default command.
5. **Why exec-form commands?** They avoid an unnecessary shell and improve signal delivery behavior.
6. **Does EXPOSE publish a port?** No. It documents the intended port; runtime publishing is separate.
7. **Why is localhost confusing?** Inside a container, localhost refers to that container’s network namespace.
8. **Volume versus bind mount?** A volume is runtime-managed storage; a bind mount maps a host path.
9. **Why run non-root?** Reduce privilege and impact of compromise; this is one layer, not complete security.
10. **Why use multi-stage builds?** Keep compilers and build tools out of the final runtime image.
11. **Can deleting a secret in a later layer remove it safely?** No; it may remain in earlier layers. Use build-secret mechanisms.
12. **Why is an image not fully reproducible just because a tag is pinned?** Tags can move, and dependencies may still be unpinned.
13. **How do you inspect a crashing container?** Check exit code, logs, command, environment, filesystem permissions, and resource constraints.
14. **Why can a container disappear without losing a volume?** The volume has a separate lifecycle.
15. **What should a health check prove?** A narrowly defined health property, without creating harmful dependency cascades.

## 6.5 Troubleshooting examples

| Symptom | Check | Likely issue |
|---|---|---|
| Port unreachable | Bind address, published port, process | App binds only to container localhost |
| Permission denied | UID, file ownership, mount permissions | Non-root user cannot access file |
| Container exits immediately | Logs and command | Process exits or startup fails |
| Data disappears | Storage mount | Data lived only in writable layer |
| Build unexpectedly slow | Layer ordering and context | Large build context or invalidated cache |

**Exercise:** Intentionally bind the server to `127.0.0.1` inside the container, observe failure through the published port, and fix it.

**Mistakes:** Shipping secrets, using privileged mode as a default, and treating containers as durable machines.

**Cheat sheet:** Build → run → logs → inspect → resource checks → network checks.

**Interviewers expect:** Understanding of process lifecycle and reproducibility.

**Follow-ups:** How do signals reach PID 1? What survives replacement? How do you patch the base image?

Reference: R22.

---

# MODULE 7 — KUBERNETES

## 7.1 Concepts

| Object | Beginner explanation | Production responsibility |
|---|---|---|
| Pod | Smallest deployable workload unit | Containers share networking and specified volumes |
| Deployment | Maintains replicated Pods | Rollouts and desired state |
| Service | Stable access to selected Pods | Correct labels and endpoints |
| Ingress | HTTP routing configuration | Requires a compatible controller |
| ConfigMap | Nonsecret configuration | Update and restart behavior |
| Secret | Sensitive configuration object | Access control and encryption configuration |
| HPA | Adjusts replica count | Metrics, requests, capacity, stabilization |
| Namespace | Logical grouping | Not a complete security boundary by itself |

Base64-encoded Secret data is not encryption.

## 7.2 Architecture

```text name=diagrams/kubernetes.txt
Client → Ingress controller → Service → Ready Pods
                                         ^
                                         │
Deployment → ReplicaSet ─────────────────┘
     ^
     │
    HPA ← Metrics API

Scheduler → Nodes
ConfigMap/Secret → Workload configuration
```

## 7.3 Deployment YAML

The image must be available to cluster nodes. For a local cluster, load the image using that cluster’s supported mechanism; otherwise push it to a registry and update the image reference.

```yaml name=k8s/lab.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fde-lab
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fde-config
  namespace: fde-lab
data:
  HOST: "0.0.0.0"
  PORT: "8080"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fde-api
  namespace: fde-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fde-api
  template:
    metadata:
      labels:
        app: fde-api
    spec:
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: api
          image: fde-lab:local
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: fde-config
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "500m"
              memory: "128Mi"
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 2
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: fde-api
  namespace: fde-lab
spec:
  selector:
    app: fde-api
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

Optional HPA:

```yaml name=k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fde-api
  namespace: fde-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fde-api
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

CPU utilization targets depend on CPU requests and an available metrics pipeline. HPA creates workload demand; it does not itself add nodes.

Optional Ingress, requiring a controller whose class is `nginx`:

```yaml name=k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fde-api
  namespace: fde-lab
spec:
  ingressClassName: nginx
  rules:
    - host: fde.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: fde-api
                port:
                  number: 80
```

This is local HTTP routing, not an internet-ready TLS configuration.

Secret shape, with a dummy value only:

```yaml name=k8s/dummy-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-secret
  namespace: fde-lab
type: Opaque
stringData:
  EXAMPLE_TOKEN: "dummy-local-value-not-a-real-secret"
```

Do not commit actual secrets. A production system should integrate an approved secret-management process.

## 7.4 Walkthrough

```bash name=scripts/kubernetes-lab.sh
kubectl apply -f k8s/lab.yaml
kubectl rollout status deployment/fde-api -n fde-lab --timeout=120s
kubectl get pods,services -n fde-lab
kubectl port-forward service/fde-api 8081:80 -n fde-lab
```

Then call `http://127.0.0.1:8081/healthz`.

Cleanup only after confirming this is your disposable lab:

```bash name=scripts/kubernetes-cleanup.sh
kubectl delete namespace fde-lab
```

## 7.5 Interview questions and answers

**Pod versus Deployment?** A Pod runs containers. A Deployment manages replacement and rollout of interchangeable Pods.

**Readiness versus liveness?** Readiness controls eligibility for traffic; liveness can trigger restart. Startup probes protect slow initialization from premature checks.

**Why not fail liveness when the database is down?** Restarting every application instance may worsen a dependency outage. Liveness should reflect a condition restart can help.

**Service unreachable?** Check DNS, Service selector, EndpointSlices, readiness, target port, application binding, and network policy.

**Why is a Pod Pending?** Scheduling constraints, resource shortage, storage binding, taints, affinity, or other placement requirements.

**Why does HPA not scale?** Missing metrics, missing requests, configuration errors, unsuitable metrics, or limits. Check status and events.

**Are namespaces tenant isolation?** They organize resources, but meaningful isolation also requires RBAC, network policies, quotas, and potentially stronger separation.

## 7.6 Troubleshooting exercise

Inject one fault at a time:

- Invalid image name.
- Wrong Service selector.
- Wrong readiness path.
- Unwritable file path.
- Excessive memory use.
- Impossible resource request.

For each, record symptom, evidence, diagnosis, and fix.

**Mistakes:** Restarting before collecting previous logs, missing resource requests, overaggressive probes, and exposing Services unnecessarily.

**Cheat sheet:** `get` → `describe` → events → current/previous logs → endpoints → resource usage.

**Interviewers expect:** Controller-based reasoning rather than memorized commands.

**Follow-ups:** What happens during a node failure? How do you drain safely? What if replicas exist but none are ready?

References: R23–R26.

---

# MODULE 8 — LINUX

## 8.1 Mental model

Linux troubleshooting connects:

- Processes.
- CPU and memory.
- Filesystems.
- Permissions.
- Network sockets.
- Service management.
- Logs.

A command is useful only if you know which hypothesis it tests.

## 8.2 Command concepts

| Command | Purpose | Production caution |
|---|---|---|
| grep | Match lines | Prefer literal matching for literal identifiers |
| find | Locate files | Be careful with deletion and symlinks |
| awk | Process fields and records | Know the actual log format |
| sed | Transform text | Preview before in-place modification |
| top | Interactive resource view | CPU alone is not the whole problem |
| ps | Process snapshot | Snapshot can miss short-lived behavior |
| netstat | Legacy socket information | May not be installed; `ss` is often available |
| chmod | Change permissions | Avoid `777` as a shortcut |
| curl | Make HTTP requests | Set time limits and retain TLS verification |
| tail | Show recent lines | Follow rotation appropriately |

## 8.3 Thirty real commands

Assume `app.log` is a synthetic log and `access.log` uses a known common access-log layout.

```bash name=scripts/linux-commands.sh
# 01: Literal request ID search.
grep -nF 'request_id=req-123' app.log

# 02: Error-like lines.
grep -nE 'ERROR|CRITICAL' app.log

# 03: Case-insensitive timeout count.
grep -ic 'timeout' app.log

# 04: Recent log files.
find ./logs -type f -name '*.log' -mtime -1

# 05: Large files; inspect only.
find ./logs -type f -size +100M -print

# 06: Count HTTP statuses; assumes status is field 9.
awk '{count[$9]++} END {for (s in count) print s, count[s]}' access.log

# 07: Show 5xx records under that same format assumption.
awk '$9 ~ /^5[0-9][0-9]$/ {print}' access.log

# 08: Print a line range without changing the file.
sed -n '100,120p' app.log

# 09: Demonstrate a transformation without in-place editing.
sed 's/status=pending/status=queued/g' sample.txt

# 10: Interactive process view.
top

# 11: Highest-CPU processes.
ps -eo pid,ppid,%cpu,%mem,cmd --sort=-%cpu | head

# 12: Find the lab service.
pgrep -af lab_server.py

# 13: Listening TCP sockets.
ss -lntp

# 14: Legacy alternative, if installed.
netstat -lntp

# 15: Make a local script executable by its owner.
chmod u+x scripts/check-local-api.sh

# 16: Restrict a synthetic credentials file.
chmod 600 credentials.example

# 17: HTTP check with bounded time.
curl --fail --show-error --max-time 3 http://127.0.0.1:8080/healthz

# 18: HTTP timing breakdown.
curl -sS -o /dev/null --max-time 5 \
  -w 'dns=%{time_namelookup} connect=%{time_connect} first=%{time_starttransfer} total=%{time_total}\n' \
  http://127.0.0.1:8080/healthz

# 19: Recent log lines.
tail -n 100 app.log

# 20: Follow log across replacement/rotation.
tail -F app.log

# 21: Filesystem capacity.
df -h

# 22: Inode capacity.
df -i

# 23: Directory disk usage.
du -sh ./logs

# 24: Memory overview.
free -h

# 25: CPU, memory, and I/O scheduling indicators.
vmstat 1 5

# 26: Systemd service logs, where applicable.
journalctl -u fde-api --since '30 minutes ago' --no-pager

# 27: Resolver lookup through the system name-service path.
getent hosts example.com

# 28: Route selection.
ip route get 1.1.1.1

# 29: Current shell file-descriptor limit.
ulimit -n

# 30: Current user and groups.
id
```

Commands vary by distribution; `top`, `ss`, `ip`, `journalctl`, and `netstat` may require different packages or privileges.

## 8.4 Architecture

```text name=diagrams/linux-debug.txt
User request
    │
    v
DNS → Network → Socket → Process → Files/Dependencies
                          │
                          v
                     CPU / Memory
```

## 8.5 Interview answers

**Disk full but `du` is smaller than `df`?** Possible deleted-but-open files, inaccessible paths, mount differences, or filesystem overhead. Check evidence before deleting data.

**High load with low CPU?** Tasks may be waiting in uninterruptible I/O states. Inspect storage latency and process states.

**Permission denied?** Check user/group, file mode, parent-directory traversal permission, mount flags, and mandatory access controls.

**Kill versus kill -9?** A graceful signal allows cleanup; SIGKILL does not. Use force only when justified.

**Why is grep insufficient for JSON logs?** Text matching ignores structure and escaping. Use a JSON parser for field-aware analysis.

## 8.6 Exercises and scenario

- Find the most frequent error code.
- Compare DNS, connection, and server response timing.
- Create a permission problem in a temporary directory.
- Explain a service that listens on the wrong interface.
- Analyze a large file without loading it into memory.

**Mistakes:** Deleting logs before preserving evidence, running everything as root, using `chmod 777`, and interpreting one snapshot as a trend.

**Best practice:** Record timestamps, commands, outputs, and hypotheses.

**Cheat sheet:** CPU → memory → disk space/inodes → I/O → sockets → DNS → permissions → logs.

**Interviewers expect:** A logical investigation sequence.

**Follow-ups:** What would you check next? Is the command read-only? How would you avoid exposing secrets?

---

# MODULE 9 — GIT

## 9.1 Concepts

- **Branch:** A movable reference to a commit.
- **Merge:** Combine histories.
- **Rebase:** Replay commits on a new base, producing new commit identities.
- **Cherry-pick:** Apply a selected commit’s change elsewhere.
- **Revert:** Add a new commit reversing an earlier change.
- **Reset:** Move references and optionally change the index and working tree.

## 9.2 Diagram

```text name=diagrams/git.txt
Before:
A──B──C  base
    \
     D──E  feature

Merge:
A──B──C────M
    \     /
     D───E

Rebase feature onto C:
A──B──C──D'──E'
```

## 9.3 Safe lab commands

Use a disposable repository. `BASE` must name the actual integration branch.

```bash name=scripts/git-lab.sh
git status
git log --oneline --graph --all

BASE=your-actual-base-branch

git switch -c feature/reconciliation
git add reconciliation.py
git commit -m "Add reconciliation helper"

# Update knowledge of remote history.
git fetch origin

# Rebase only when appropriate for your branch ownership.
git rebase "origin/$BASE"

# If conflicts cannot be resolved safely:
# git rebase --abort

# Apply a selected known commit:
# git cherry-pick <commit-sha>

# Undo a shared bad commit through a new commit:
# git revert <commit-sha>

# Unstage a file without discarding its working-tree content:
git restore --staged reconciliation.py
```

The final command requires that the file is staged at the time you use it; commands are demonstrations, not one mandatory sequence.

## 9.4 Reset modes

- `--soft`: Move HEAD; preserve index and working tree.
- `--mixed`: Move HEAD and reset index; preserve working tree.
- `--hard`: Also overwrite tracked working-tree state; can destroy uncommitted work.

Do not use reset as a default way to undo shared history.

## 9.5 Interview questions

**Merge or rebase?** Merge preserves the branch structure; rebase creates a linearized history by rewriting commits. Follow team policy and avoid rewriting others’ shared work.

**Revert or reset?** Revert is generally appropriate for undoing a shared commit while retaining history. Reset is useful for deliberate local history manipulation.

**What happens during cherry-pick?** Git applies the selected change as a new commit; conflicts and missing dependencies are possible.

**Can Git remove an exposed secret?** History cleanup alone is insufficient. Revoke or rotate the secret immediately, then follow the incident process.

**How do you recover a lost local commit?** Inspect reflog and create a reference to the commit before retention removes it.

## 9.6 Scenario

A bad deployment came from a shared commit.

1. Assess whether rollback is compatible with current data.
2. Revert or prepare a corrective commit.
3. Run tests.
4. Deploy the approved artifact.
5. Verify customer recovery.
6. Investigate why the change escaped controls.

**Mistakes:** Force-pushing shared branches, treating Git revert as database rollback, and committing generated secrets.

**Cheat sheet:** Inspect first; preserve work; use revert for shared undo; understand all three trees.

**Interviewers expect:** Collaboration safety, not only syntax.

**Follow-ups:** What if it was a merge commit? What if the schema already changed? What does the reflog retain?

References: R27–R28.

---

# MODULE 10 — CI/CD

## 10.1 Foundations

**Continuous integration:** Frequently integrate changes and automatically check them.

**Continuous delivery:** Keep changes releasable through an automated delivery process.

**Continuous deployment:** Automatically release qualifying changes to production.

A pipeline is a chain of evidence, not merely a shell script that copies files.

## 10.2 Architecture

```text name=diagrams/cicd.txt
Commit → Unit tests → Integration tests → Security checks → Build
                                                            │
                                                            v
                                                     Immutable artifact
                                                            │
                                                            v
Staging → Smoke tests → Approval/policy → Canary → Full rollout
                                           │
                                           └── Stop/rollback
```

## 10.3 Shared test file

```python name=test_python_examples.py
import unittest
from python_examples import duplicates, fee_minor, parse_payment

class ExampleTests(unittest.TestCase):
    def test_duplicates(self):
        self.assertEqual(duplicates([1, 1, 2]), {1})

    def test_fee(self):
        self.assertEqual(fee_minor(10000, 250), 250)

    def test_reject_boolean_amount(self):
        with self.assertRaises(ValueError):
            parse_payment(
                '{"payment_id":"p1","amount_minor":true,"currency":"USD"}'
            )

if __name__ == "__main__":
    unittest.main()
```

## 10.4 GitHub Actions

```yaml name=.github/workflows/ci.yml
name: CI

on:
  push:
  pull_request:

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: python -m unittest discover -v
      - run: docker build -t fde-lab:ci .
```

These are readable example action tags, not “latest version” claims. In production, review and pin third-party actions to full commit SHAs and maintain them through an update process.

Do not expose production credentials to untrusted pull-request code.

## 10.5 Jenkins

Assumes a Jenkins agent with Python available.

```groovy name=Jenkinsfile
pipeline {
    agent any
    options {
        timeout(time: 10, unit: 'MINUTES')
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Test') {
            steps {
                sh 'python3 -m unittest discover -v'
            }
        }
    }
    post {
        always {
            echo 'Publish diagnostics without secrets'
        }
    }
}
```

## 10.6 GitLab CI

```yaml name=.gitlab-ci.yml
image: python:3.12-slim

stages:
  - test

unit-tests:
  stage: test
  timeout: 10 minutes
  script:
    - python -m unittest discover -v
```

## 10.7 Azure DevOps

```yaml name=azure-pipelines.yml
trigger:
  branches:
    include:
      - "*"

pool:
  vmImage: ubuntu-latest

steps:
  - task: UsePythonVersion@0
    inputs:
      versionSpec: "3.12"

  - script: python -m unittest discover -v
    displayName: Run unit tests
```

These are minimal CI examples. Production delivery additionally requires registry access, artifact identity, environment permissions, deployment targets, and policy decisions.

## 10.8 Deployment strategies

### Blue-green

Run old and new environments side by side, validate the new environment, then switch traffic.

Advantages:

- Clear cutover.
- Fast traffic reversal when compatible.

Costs:

- Duplicate capacity.
- Shared data still needs migration compatibility.

### Canary

Send a small portion of representative traffic to the new release, compare outcomes, and expand gradually.

Example policy:

- 1% → 5% → 25% → 100%.
- Require minimum sample size.
- Compare error rate, latency, and business success.
- Stop automatically when agreed thresholds are breached.

These percentages are an example, not a universal standard.

### Rollback

Rollback must consider:

- Application image.
- Configuration.
- Database schema.
- Queue/event schema.
- External side effects.
- Feature flags.

Prefer expand-and-contract schema changes:

1. Add compatible fields.
2. Deploy readers/writers that tolerate both forms.
3. Backfill safely.
4. Switch usage.
5. Remove old fields only after compatibility windows close.

## 10.9 Exercise and interview questions

**Exercise:** Make a unit test fail. Confirm every pipeline rejects the change. Then explain how the same tested image would be promoted without rebuilding.

**Why build once?** Rebuilding can change dependencies or artifacts, so staging evidence may no longer apply to production.

**Why is rollback sometimes unsafe?** Old code may not understand new data or schemas, and external effects cannot simply be undone.

**What should deployment permissions look like?** Separate from test permissions, short-lived where possible, least-privilege, and restricted to trusted environments.

**Mistakes:** Mutable tags, secrets in logs, deployment from untrusted code, and treating a green build as proof of business correctness.

**Cheat sheet:** Test → build once → identify artifact → promote → observe → stop safely.

**Interviewers expect:** Release safety and clear failure gates.

**Follow-ups:** How do you handle migrations? Who approves production? How do you prove what is running?

References: R29–R32.

---

# MODULE 11 — SYSTEM DESIGN

## 11.1 Beginner building blocks

| Component | Simple explanation | Trade-off |
|---|---|---|
| Load balancer | Distributes traffic | Needs health checks and appropriate routing |
| Cache | Stores reusable results | Staleness and invalidation |
| Queue | Buffers asynchronous work | Delay, duplication, ordering |
| Database | Stores durable state | Consistency, throughput, operations |
| API gateway | Front door for APIs | Adds policy and another dependency |
| Scaling | Add capacity | Bottlenecks move elsewhere |
| High availability | Continue through failures | Redundancy and operational complexity |

## 11.2 Design process

1. Clarify users and operations.
2. Define latency, throughput, durability, and consistency needs.
3. Estimate volume.
4. Define API and data model.
5. Draw a minimal architecture.
6. Walk through one successful request.
7. Walk through failure cases.
8. Discuss scaling, security, and cost.
9. State trade-offs and future evolution.

## 11.3 Useful calculations

- Average requests/second = requests/day ÷ 86,400.
- Approximate concurrency = arrival rate × average time in system.
- Storage/day = records/day × average record size, plus indexes and replication.
- Queue drain time ≈ backlog ÷ (processing rate − arrival rate), only when processing rate exceeds arrival rate.

Example:

- Arrival: 100 messages/second.
- Processing: 150 messages/second.
- Backlog: 30,000.
- Idealized drain time: 30,000 ÷ 50 = 600 seconds.

Real systems add variance, retries, batching, and changing traffic.

## 11.4 Design 1: URL shortener

### Requirements

- Create short links.
- Redirect reads.
- Optional expiry.
- Protect against abuse.
- Track analytics asynchronously.

```text name=diagrams/url-shortener.txt
Create request → API → ID generator → Mapping database
                                      │
Redirect request → Cache ──────────────┘
        │
        └── Analytics event → Queue → Aggregator
```

### Data

`short_code`, `destination`, `owner`, `created_at`, `expires_at`.

### Design decisions

- A unique sequence encoded in base62 is simple but can reveal volume.
- Random codes need collision handling through a uniqueness constraint.
- Cache popular mappings.
- Choose redirect status based on desired caching and mutability.
- Do not fetch arbitrary destination URLs server-side without SSRF controls.
- Rate-limit creation and handle malicious destinations.

### Interview discussion

**Question:** Why not store everything only in cache?

**Answer:** If mappings must survive eviction or restart, a durable source of truth is needed.

**Follow-up:** How do you change a destination without stale redirects?

Discuss TTLs, invalidation, and browser/CDN caching.

### Exercise

Implement create/get operations in memory, then add SQLite persistence and collision tests.

## 11.5 Design 2: Notification system

```text name=diagrams/notifications.txt
Domain event → Router → Preference check → Channel queues
                                          │     │     │
                                          v     v     v
                                        Email  SMS   Push
                                          │
                                          v
                                    Provider adapters
                                          │
                                          v
                                    Delivery tracking
```

### Requirements

- Respect user preferences.
- Support multiple channels.
- Retry transient errors.
- Prevent duplicates within a defined scope.
- Enforce provider rate limits.
- Record delivery attempts.

### Production reasoning

“Accepted by provider” is not the same as “delivered to user.”

Use a stable notification ID. Preserve attempt history. Route permanent failures to review rather than retrying forever.

### Interview discussion

**Question:** How do you handle a provider outage?

**Answer:** Buffer work, apply bounded retries and circuit breaking, expose delay, and use a fallback only if consent and duplicate-delivery risks are addressed.

### Exercise

Model notification states and simulate a timeout after provider acceptance.

## 11.6 Design 3: Payment processing platform

```text name=diagrams/payments.txt
Client → Auth/API → Idempotency record + Payment intent
                              │
                              v
                         Database commit
                              │
                              v
                          Outbox relay
                              │
                              v
                            Queue
                              │
                              v
                       Payment worker → Provider
                              │             │
                              │             v
                              │          Webhook
                              └──────┬──────┘
                                     v
                           State + Ledger + Audit
                                     │
                                     v
                              Reconciliation
```

### Required distinctions

- Authorization: reserve/approve payment capability.
- Capture: request collection of authorized funds.
- Settlement: funds move through the settlement process.
- Refund: separate compensating financial operation.
- Chargeback: dispute-related reversal process.

Provider terminology and contracts vary.

### Core invariants

- Stable logical operation identifiers.
- Currency-aware amounts.
- No duplicate business effect for a retried logical request.
- Auditable state transitions.
- Ledger entries balanced within each currency.
- Reconciliation for unknown outcomes.

### Idempotency flow

1. Receive tenant-scoped idempotency key.
2. Validate payload and compute fingerprint.
3. Atomically insert intent or load existing intent.
4. Return conflict if the same key has a different fingerprint.
5. Commit intent and outbox record together.
6. Worker calls provider with stable operation identity.
7. Record outcome through valid state transitions.
8. Reconcile timeouts and incomplete local state.

### Why an outbox?

Writing to the database and publishing to a queue as independent operations creates a dual-write gap. An outbox stores both local intent and publication intent in one database transaction. The relay can still publish duplicates, so consumers remain idempotent.

### Interview discussion

**Question:** Can you guarantee exactly-once payment execution?

**Answer:** I would define the boundary. At-least-once delivery plus durable idempotency can produce effectively-once business effects within a specified contract, but an external provider introduces uncertain outcomes requiring provider support and reconciliation.

### Exercise

Simulate crashes:

- Before committing intent.
- After commit but before queue publication.
- After provider success but before local persistence.
- After persistence but before acknowledgment.

Explain recovery for each.

## 11.7 Design 4: Real-time transaction tracking

```text name=diagrams/tracking.txt
Payment state changes → Event stream → Projection workers
                                          │
                                          v
                                   Read-optimized store
                                          │
                                          v
Client ← SSE/WebSocket gateway ← Update distribution
```

### Requirements

- Users see recent status quickly.
- Events can arrive late, duplicated, or out of order.
- Reconnecting clients can recover missed updates.
- Each user sees only authorized transactions.

### Production reasoning

Use entity versions or sequence numbers rather than trusting arrival order. The projection is a derived view, not the financial source of truth.

SSE can suit server-to-client updates; WebSockets suit bidirectional communication. Polling may be sufficient for less demanding latency.

### Interview discussion

**Question:** What if a client reconnects after missing events?

**Answer:** Use a cursor or version, replay retained updates where available, and fall back to a current-state snapshot if the cursor is too old.

### Exercise

Implement version-based event application; prove an old event cannot overwrite newer state.

## 11.8 Common mistakes and best practices

**Mistakes:**

- Starting with ten services before defining requirements.
- Claiming caches or queues solve every scaling problem.
- Ignoring tenant isolation.
- Confusing availability with durability.
- Ignoring operational ownership.
- Treating consistency as one global yes/no choice.

**Best practice:** Begin with a modular service and a suitable database. Add distributed components only when requirements justify them.

**Cheat sheet:** Requirements → numbers → data → flow → failures → trade-offs.

**Interviewers expect:** A coherent design that changes when assumptions change.

**Follow-ups:** What breaks first? What can be stale? What must never be duplicated? How do you recover a region?

---

# MODULE 12 — OBSERVABILITY

## 12.1 Concepts

- **Logging:** Discrete event records.
- **Metrics:** Numeric measurements aggregated over time.
- **Tracing:** A request’s path through operations and services.
- **Monitoring:** Evaluating known health indicators.
- **Alerting:** Notifying someone when action is needed.
- **Observability:** The ability to understand system behavior from emitted signals.

## 12.2 Tool roles

| Tool | Main role |
|---|---|
| Prometheus | Metric collection and querying |
| Grafana | Visualization and related observability workflows |
| Elasticsearch | Search and analysis of indexed data |
| Logstash | Data ingestion and transformation |
| Kibana | Exploration and visualization in the Elastic ecosystem |
| ELK | Elasticsearch, Logstash, Kibana stack |
| CloudWatch | AWS telemetry and alarm services |
| OpenTelemetry | Instrumentation and telemetry collection/export conventions |

Grafana is not automatically the storage backend for everything it displays.

## 12.3 Architecture

```text name=diagrams/observability.txt
Application
  ├── Metrics ──> Collector/Prometheus ──> Dashboard
  ├── Logs ─────> Log pipeline ─────────> Search
  └── Traces ───> OTel collector ───────> Trace backend
                         │
                         v
                    Alert rules
                         │
                         v
                   On-call + Runbook
```

## 12.4 What to measure

For APIs:

- Request rate.
- Error rate.
- Latency distribution.
- Saturation.

For payment processing:

- Accepted-to-completed latency.
- Unknown outcomes.
- Reconciliation mismatches.
- Duplicate-suppression counts.
- Oldest pending payment age.

For queues:

- Arrival rate.
- Processing rate.
- Oldest-message age.
- Retry rate.
- DLQ growth.

## 12.5 PromQL examples

Assume the application exports these metric names.

```promql name=observability/queries.promql
# Request rate.
sum(rate(http_requests_total[5m]))

# 5xx ratio, when the denominator is nonzero.
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# p95 using classic histogram buckets.
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

Do not average instance-level p95 values to obtain a global p95.

Do not label metrics with unbounded values such as raw request IDs or payment IDs.

## 12.6 SLI, SLO, SLA

- **SLI:** Measured reliability indicator.
- **SLO:** Target for that indicator over a window.
- **SLA:** Contractual commitment with specified terms.

Example SLO:

> 99.9% of eligible payment-status reads succeed within the chosen latency threshold over 30 days.

Define eligibility, exclusions, measurement source, and low-traffic behavior.

A request-based error budget is a number/fraction of bad requests; it is not automatically equivalent to downtime minutes.

## 12.7 Scenario

The dashboard shows average latency unchanged, but customers report slowness.

Investigate:

1. p95/p99, not only the mean.
2. Affected tenant, route, region, and release.
3. Queueing versus execution time.
4. Downstream trace spans.
5. Client-side and network time.
6. Sampling gaps.

## 12.8 Exercises and interview answers

**Exercise:** Add structured request logs and a histogram to the lab service. Explain which labels are bounded.

**Logs versus traces?** Logs explain individual events; traces connect operations along a request path. Correlation IDs make them more useful together.

**What makes an alert good?** It signals actionable user impact, has an owner, includes context, and points to a response.

**Why not alert on every CPU spike?** A spike may be harmless. Page for actionable risk or impact, and use resource alerts when they provide meaningful early warning.

**Mistakes:** Alert fatigue, missing ownership, secret leakage, high cardinality, and dashboard-only observability.

**Cheat sheet:** Rate, errors, duration, saturation; then business success and lag.

**Interviewers expect:** Clear measurement definitions.

**Follow-ups:** What is the denominator? How do you handle no traffic? Can you compare old and new releases?

References: R20, R33–R35.

---

# MODULE 13 — PRODUCTION SUPPORT

## 13.1 Incident operating procedure

1. Confirm the alert.
2. Determine customer impact and severity.
3. Assign coordination and communication ownership.
4. Freeze unrelated risky changes.
5. Capture a timeline.
6. Form ranked hypotheses.
7. Mitigate safely.
8. Verify technical and business recovery.
9. Reconcile delayed or uncertain work.
10. Create a blameless review with owners and deadlines.

Do not wait for a perfect root cause before restoring service when a safe mitigation is available.

## 13.2 Incident 1: API latency increased

**Evidence:** p95 increased after a release; CPU is normal.

**Investigation:**

- Segment by route and tenant.
- Compare release versions.
- Inspect traces for downstream time and pool waits.
- Check request volume and payload changes.

**Possible confirmed cause:** A new code path performs one database query per item.

**Resolution:** Roll back the path or replace N+1 queries with an appropriate batched query.

**Prevention:** Representative load tests, query-count assertions, and canary latency gates.

**Interviewer follow-up:** What if rollback cannot be done?  
Use a feature flag, disable the expensive path, or reduce its scope with explicit customer communication.

## 13.3 Incident 2: Database CPU at 100%

**Investigation:**

- Identify top queries by total resource use.
- Separate CPU from lock or I/O waits.
- Inspect plans and row estimates.
- Check connection concurrency and recent changes.

**Possible cause:** A frequently executed query lost an efficient access path.

**Resolution:** Reduce harmful traffic, roll back the query change, or add a reviewed index through an appropriate safe procedure.

**Prevention:** Plan regression checks, query budgets, realistic data tests.

**Avoid:** Killing random database sessions without understanding the effect.

## 13.4 Incident 3: Kubernetes Pod crashing

**Investigation:**

- Inspect Pod status and last termination reason.
- Read previous-container logs.
- Review events, probes, configuration, and resource limits.

**Possible cause:** The new version exceeds its memory limit.

**Resolution:** Roll back or fix the memory behavior; temporarily increase limits only with capacity analysis.

**Prevention:** Memory profiling, resource tests, and safe rollout gates.

**Important:** `CrashLoopBackOff` describes restart backoff, not the root cause.

## 13.5 Incident 4: Lambda timeout

**Investigation:**

- Compare duration distribution with timeout.
- Inspect dependency timing.
- Check DNS, routes, egress, connection reuse, and resource utilization.
- Review batch size and concurrency.

**Possible cause:** A function moved into a VPC without the required route or endpoint to its dependency.

**Resolution:** Correct network access through the approved architecture.

**Prevention:** Deployment connectivity tests and dependency deadlines.

**Avoid:** Increasing timeout as the only fix.

## 13.6 Incident 5: Payment processing delayed

**Investigation:**

- Determine whether payments are accepted, submitted, authorized, or settled.
- Inspect oldest pending age.
- Compare provider status and internal records.
- Check duplicate suppression and retry patterns.

**Possible cause:** Provider requests timed out but some succeeded externally.

**Resolution:** Mark unresolved outcomes explicitly and reconcile by stable provider identifiers before initiating new attempts.

**Prevention:** Provider idempotency, durable state, replay-safe webhooks, reconciliation jobs.

**Avoid:** Blindly retrying every payment with a new key.

## 13.7 Incident 6: Queue backlog

**Investigation:**

- Arrival versus completion rate.
- Oldest-message age.
- Processing duration.
- Downstream throttling.
- Poison-message distribution.
- Retry and DLQ metrics.

**Possible cause:** Consumers repeatedly process malformed events.

**Resolution:** Isolate the bad schema/version, route poison messages through the defined failure process, fix parsing, and replay safely.

**Prevention:** Schema compatibility tests, bounded retries, DLQ ownership.

**Avoid:** Purging the queue to make a graph look healthy.

## 13.8 Diagnostic commands

```bash name=scripts/incident-kubernetes.sh
kubectl get pods -n fde-lab
kubectl describe deployment fde-api -n fde-lab
kubectl get events -n fde-lab --sort-by=.metadata.creationTimestamp

# Replace POD with a real Pod name.
kubectl logs POD -n fde-lab --previous
kubectl describe pod POD -n fde-lab
```

## 13.9 Customer update template

> We are investigating delayed payment processing affecting [scope].  
> The issue began at [time and timezone].  
> We have confirmed [facts].  
> We are currently [mitigation].  
> Please [safe customer action, if any].  
> Our next update will be at [time].  
> We have not yet confirmed [important uncertainty].

Never assert “no data loss” or “no duplicate charges” without evidence.

## 13.10 Post-incident review

Record:

- Summary and customer impact.
- Detection and recovery times.
- Timeline.
- Trigger.
- Root cause and contributing conditions.
- Why safeguards did not prevent impact.
- What helped.
- Actions with owners, deadlines, and verification.

**Exercise:** Run a 20-minute tabletop for each scenario. One person acts as customer, one as investigator, one as incident coordinator.

**Cheat sheet:** Restore safely, verify business outcomes, then explain the causal chain.

**Interviewers expect:** Calm prioritization and evidence-based decisions.

**Follow-ups:** What did you personally do? How did you rule out alternatives? How did you know recovery was complete?

---

# MODULE 14 — NETWORKING

## 14.1 Concepts

| Topic | Simple explanation | Production significance |
|---|---|---|
| TCP/IP | Addressing and transport foundations | Routing, connection establishment, retransmission |
| DNS | Maps names to records | Caching, TTLs, resolver failures |
| HTTP | Application protocol | Methods, status, headers, timeouts |
| HTTPS | HTTP protected by TLS | Confidentiality, integrity, peer authentication |
| SSL | Historical protocol family name | Modern deployments use TLS |
| Load balancer | Distributes traffic | L4/L7 behavior and health checks |
| Firewall | Traffic policy enforcement | Direction, ports, protocols, statefulness |
| VPN | Protected network tunnel | Routing, identity, MTU, overlapping networks |
| CIDR | Address prefix notation | Address range and network planning |
| Subnet | Network segment | Routing and placement boundaries |

HTTP/1.1 and HTTP/2 commonly use TCP; HTTP/3 uses QUIC over UDP. Do not claim all HTTP uses TCP.

## 14.2 Request journey

```text name=diagrams/network-path.txt
URL
 │
 v
DNS resolution
 │
 v
Route selection
 │
 v
Transport connection
 │
 v
TLS negotiation and certificate validation
 │
 v
HTTP request → Proxy/LB → Application → Dependency
```

## 14.3 CIDR example

IPv4 `/24` leaves 8 host bits:

`2^8 = 256` total addresses.

Usable counts depend on the environment and reservation rules. AWS subnets reserve addresses, so do not assume all 256 are assignable.

## 14.4 Python subnet example

```python name=network_check.py
import ipaddress

network = ipaddress.ip_network("10.0.1.0/24")
assert ipaddress.ip_address("10.0.1.25") in network
assert ipaddress.ip_address("10.0.2.25") not in network

print(network.num_addresses)
```

## 14.5 Interview questions and answers

**TCP versus UDP?** TCP provides an ordered byte stream with reliability and congestion control. UDP provides datagrams without those same guarantees; applications can build additional behavior on top.

**Why can DNS work but HTTPS fail?** Name resolution only proves one stage. Routing, TCP/QUIC, TLS, or HTTP can still fail.

**L4 versus L7 load balancer?** L4 routes using transport-level information; L7 understands application details such as HTTP host or path.

**Security group versus NACL in AWS?** Security groups are stateful resource-level traffic controls; network ACLs are stateless subnet-level rules.

**What does a TLS certificate validate?** It helps authenticate an endpoint identity through a trust chain and hostname checks. Encryption without validation is insufficient.

**Why does a VPN break only some requests?** MTU, routing, DNS, split-tunnel policy, or overlapping CIDRs may affect subsets of traffic.

**What is a subnet route table?** It maps destination prefixes to next hops or targets; a subnet’s name does not determine whether it is public.

## 14.6 Scenario and exercise

**Scenario:** An API works on a laptop but fails from a workload.

Check from the workload’s actual network context:

1. DNS answer.
2. Route.
3. Egress policy.
4. Transport connection.
5. TLS.
6. Proxy settings.
7. Application authorization.

**Exercise:** Draw a two-AZ VPC with application and database subnets. Explain which components need outbound internet access and why.

**Mistakes:** Disabling TLS verification, opening all ports, confusing DNS with routing, and blaming the firewall before testing.

**Cheat sheet:** Resolve → route → connect → secure → request → authorize.

**Interviewers expect:** Layered diagnosis.

**Follow-ups:** What does the timeout duration suggest? Is the failure regional? What does a packet capture reveal, and how would you handle sensitive data?

References: R16, R19.

---

# MODULE 15 — BEHAVIORAL INTERVIEW

## 15.1 Use truthful evidence

The answers below are templates. Replace bracketed details with facts from your experience. Do not invent scale, ownership, metrics, or customer exposure.

STAR:

- Situation.
- Task.
- Action.
- Result.

Add a reflection when useful: what you learned or would change.

## 15.2 Tell me about yourself

**Situation:**  
“I’m a software engineer with about two years of experience working with [actual technologies and domain].”

**Task:**  
“My responsibilities have included [development, integration, support, or cloud operations that are true].”

**Action:**  
“I’ve worked on [specific example], where I used [skills] to address [problem]. I particularly enjoy tracing issues across APIs, data, and infrastructure.”

**Result and direction:**  
“That experience made me interested in FDE work, where I can combine implementation with understanding customer workflows and owning a solution through deployment.”

Keep this around 60–90 seconds.

## 15.3 Explain your current project

**S:** “The system supports [user workflow].”

**T:** “My responsibility is [your actual boundary].”

**A:** “Requests move through [architecture]. I contributed [specific component], and collaborated with [teams]. We handle failures through [actual mechanisms].”

**R:** “The work improved [verified outcome]. The most important trade-off was [trade-off].”

Prepare a diagram. Explain your work separately from the team’s work.

## 15.4 Biggest challenge

**S:** “We faced [specific technical or coordination constraint].”

**T:** “I needed to [goal] while preserving [safety or deadline constraint].”

**A:** “I broke the problem into hypotheses, tested [evidence], and compared [options]. I chose [decision] because [reason].”

**R:** “We achieved [observed outcome]. I learned [lesson], and would improve [specific area].”

## 15.5 Production issue solved

**S:** “Users experienced [actual symptom].”

**T:** “I was responsible for [investigation or mitigation role].”

**A:** “I established scope, correlated [signals], identified [cause], and implemented [approved mitigation]. I communicated [facts and update cadence].”

**R:** “We verified recovery through [technical metric and business check], then added [preventive control].”

Do not describe a hypothetical scenario as your own incident.

## 15.6 Conflict with a team member

**S:** “We disagreed about [technical decision].”

**T:** “We needed a decision that met [shared objective].”

**A:** “I restated the other person’s concern, documented trade-offs, and proposed a small test or decision matrix.”

**R:** “We chose [result]. Even where my preference was not selected, I supported the agreed plan and documented risks.”

Avoid portraying disagreement as proof that the other person was incompetent.

## 15.7 Why Forward Deployment Engineer?

**S:** “My experience has shown me that working code is only valuable when it fits a real workflow.”

**T:** “I want a role where I can connect implementation with that workflow.”

**A:** “I’ve been developing skills in APIs, troubleshooting, cloud deployment, and explaining technical decisions.”

**R:** “FDE work matches the kind of end-to-end ownership I want to grow into.”

Add one real example demonstrating this motivation.

## 15.8 Why should we hire you?

**S:** “This role needs someone who can learn customer context and implement reliably.”

**T:** “The challenge is balancing speed with correctness and communication.”

**A:** “I bring [true strengths], demonstrated by [specific evidence]. I also recognize gaps in [area] and have been addressing them through [work].”

**R:** “I can contribute in [specific scope] immediately while growing into broader deployment ownership.”

## 15.9 Exercise

Record all seven answers.

For each, check:

- Did I answer the actual question?
- Is my personal contribution clear?
- Did I explain a decision?
- Is the result supported by evidence?
- Did I avoid unnecessary jargon?
- Can I answer two follow-ups?

**Mistakes:** Memorized speeches, fabricated metrics, blaming others, and saying “we” without explaining “I.”

**Cheat sheet:** Context briefly; actions specifically; results honestly.

**Interviewers expect:** Ownership, self-awareness, collaboration, and learning.

**Follow-ups:** What would your teammate say? What failed? What did you learn? What did you do differently next time?

---

# MODULE 16 — FIFTY CODING QUESTIONS WITH SOLUTIONS

## 16.1 Instructions

For every problem:

1. Clarify the input contract.
2. Give a simple approach.
3. Explain the optimized approach.
4. Implement.
5. Test empty, normal, boundary, and invalid inputs.
6. State time and space complexity.
7. Discuss production limitations where relevant.

Complexities below assume normal hash-table behavior and constant-cost scalar operations unless noted.

## 16.2 Questions, brute force, and optimized approaches

| # | Difficulty | Problem | Brute force | Implemented approach / time / auxiliary space |
|---|---|---|---|---|
| 1 | Easy | Reverse string | Repeated front concatenation, potentially O(n²) | Slice, O(n), O(n) result |
| 2 | Easy | Palindrome | Reverse-copy compare, O(n) space | Two pointers, O(n), O(1) |
| 3 | Easy | Frequencies | Count each distinct value repeatedly, O(n²) | Counter, O(n), O(u) |
| 4 | Easy | Duplicate values | Pair comparison, O(n²) | Counter, O(n), O(u) |
| 5 | Easy | First unique character | Repeated counts, O(n²) | Count then scan, O(n), O(u) |
| 6 | Easy | Two sum | All pairs, O(n²) | Hash map, O(n), O(n) |
| 7 | Easy | Anagrams | Sort both, O(n log n) | Counters, O(n), O(u) |
| 8 | Easy | Stable deduplication | List membership, O(n²) | Hash-backed order, O(n), O(u) |
| 9 | Easy | Missing number | Search each candidate, O(n²) | Sum identity, O(n), O(1) |
| 10 | Easy | Merge sorted arrays | Concatenate and sort | Two pointers, O(n+m), O(n+m) output |
| 11 | Easy | Binary search | Linear scan, O(n) | Binary search, O(log n), O(1) |
| 12 | Easy | Valid brackets | Repeated removal, O(n²) | Stack, O(n), O(n) |
| 13 | Easy | Move zeros | Repeated deletion/insertion | Stable compaction, O(n), O(1) |
| 14 | Easy | Second-largest distinct value | Sort, O(n log n) | Single pass, O(n), O(1) |
| 15 | Easy | Common prefix | Repeated substring checks | Character scan, O(total chars), output-dependent |
| 16 | Easy | Rotate list | Repeated shifts, O(nk) | Slices, O(n), O(n) |
| 17 | Easy | Flatten one level | Repeated list concatenation | Comprehension, O(total items), output-sized |
| 18 | Easy | Group by status | Scan once per status | One pass, O(n), O(n) output |
| 19 | Easy | Chunk a list | Manual repeated slicing logic | Generator of slices, O(n), O(k) per chunk |
| 20 | Easy | Count matching file lines | Read entire file | Stream, O(bytes), O(max line) |
| 21 | Medium | Longest unique substring | All substrings, O(n³) naive | Sliding window, O(n), O(u) |
| 22 | Medium | Maximum subarray | All sums, O(n²) with running sums | Kadane, O(n), O(1) |
| 23 | Medium | Product except self | Product for each position, O(n²) | Prefix/suffix, O(n), O(1) beyond output |
| 24 | Medium | Merge intervals | Repeated pair merging | Sort and scan, O(n log n), O(n) |
| 25 | Medium | Top-k frequent | Repeated maximum extraction | Counter + heap, O(n+u log k), O(u+k) |
| 26 | Medium | Kth largest | Sort, O(n log n) | Size-k heap, O(n log k), O(k) |
| 27 | Medium | Subarray sum equals k | All subarrays, O(n²) | Prefix counts, O(n), O(n) |
| 28 | Medium | Minimum positive-array window sum | Enumerate windows, O(n²) | Sliding window, O(n), O(1) |
| 29 | Medium | BFS shortest unweighted path | Enumerate paths | BFS, O(V+E), O(V) |
| 30 | Medium | Detect directed cycle | Explore paths repeatedly | Topological removal, O(V+E), O(V+E) representation |
| 31 | Medium | LRU cache | Linear recency list | OrderedDict, expected O(1) operations, O(capacity) |
| 32 | Medium | Merge k sorted lists | Flatten and sort | Heap merge, O(N log k), O(k) beyond output |
| 33 | Medium | Find all anagram starts | Sort every window | Fixed-alphabet counts, O(n+m), O(1) |
| 34 | Medium | Minimum coins | Enumerate combinations | Dynamic programming, O(amount × coin count), O(amount) |
| 35 | Medium | Longest consecutive run | Start from every element | Set starts only, O(n) expected, O(n) |
| 36 | FDE | Validate payment record | Ad hoc checks at call sites | Central validator, O(1) bounded schema |
| 37 | FDE | Parse JSON Lines | Load all lines | Generator, O(bytes), O(max record) |
| 38 | FDE | Reconcile payment collections | Nested comparisons, O(nm) | ID maps, O(n+m), O(n+m) |
| 39 | FDE | In-memory idempotency registry | Re-execute every request | Key + fingerprint, expected O(1) lookup |
| 40 | FDE | Retry transient operation | Immediate/unlimited retries | Capped backoff, O(attempts) calls |
| 41 | FDE | Follow API pagination | Assume one page | Cursor loop with cycle guard, O(items+pages) |
| 42 | FDE | Token bucket | Recount all historical requests | Constant-state refill, O(1) per request |
| 43 | FDE | TTL cache | Scan all entries per read | Per-key expiry, expected O(1) access |
| 44 | FDE | Redact sensitive fields | String replacement | Recursive structured traversal, O(nodes) |
| 45 | FDE | Event deduplication | Scan prior events | Seen-key set, O(n), O(u) |
| 46 | FDE | Aggregate logs | Repeated scans per status | One pass, O(n), O(status count) |
| 47 | FDE | Circuit breaker | Call failing dependency always | State machine, O(1) overhead |
| 48 | FDE | Streaming file checksum | Load full file | Chunked hash, O(bytes), O(chunk size) |
| 49 | FDE | Bounded async map | Launch unbounded tasks | Fixed workers, O(n) scheduling, O(n) result storage |
| 50 | FDE | Apply ordered state updates | Apply arrival order | Version guard, expected O(1) per event |

## 16.3 Solutions 1–20

```python name=coding_01_20.py
from collections import Counter, defaultdict

# 1
def reverse_string(s):
    return s[::-1]

# 2: Exact code-point comparison; no normalization.
def is_palindrome(s):
    left, right = 0, len(s) - 1
    while left < right:
        if s[left] != s[right]:
            return False
        left += 1
        right -= 1
    return True

# 3
def frequencies(values):
    return dict(Counter(values))

# 4
def duplicates(values):
    return {x for x, count in Counter(values).items() if count > 1}

# 5
def first_unique(s):
    counts = Counter(s)
    return next((c for c in s if counts[c] == 1), None)

# 6
def two_sum(values, target):
    seen = {}
    for index, value in enumerate(values):
        wanted = target - value
        if wanted in seen:
            return seen[wanted], index
        seen[value] = index
    return None

# 7
def are_anagrams(a, b):
    return Counter(a) == Counter(b)

# 8: Hashable values.
def stable_unique(values):
    return list(dict.fromkeys(values))

# 9: Contract: distinct integers from 0..n with one missing.
def missing_number(values):
    n = len(values)
    return n * (n + 1) // 2 - sum(values)

# 10
def merge_sorted(a, b):
    i = j = 0
    result = []
    while i < len(a) and j < len(b):
        if a[i] <= b[j]:
            result.append(a[i])
            i += 1
        else:
            result.append(b[j])
            j += 1
    result.extend(a[i:])
    result.extend(b[j:])
    return result

# 11: Sorted ascending input.
def binary_search(values, target):
    left, right = 0, len(values) - 1
    while left <= right:
        middle = (left + right) // 2
        if values[middle] == target:
            return middle
        if values[middle] < target:
            left = middle + 1
        else:
            right = middle - 1
    return -1

# 12: Reject non-bracket characters.
def valid_brackets(text):
    pairs = {")": "(", "]": "[", "}": "{"}
    stack = []
    for char in text:
        if char in "([{":
            stack.append(char)
        elif char in pairs:
            if not stack or stack.pop() != pairs[char]:
                return False
        else:
            return False
    return not stack

# 13: Mutates input; keeps nonzero relative order.
def move_zeros(values):
    write = 0
    for value in values:
        if value != 0:
            values[write] = value
            write += 1
    for index in range(write, len(values)):
        values[index] = 0
    return values

# 14: Numeric, mutually comparable values; excludes NaN.
def second_largest(values):
    first = second = None
    for value in values:
        if first is None or value > first:
            second, first = first, value
        elif value != first and (second is None or value > second):
            second = value
    return second

# 15
def common_prefix(words):
    if not words:
        return ""
    for index, char in enumerate(words[0]):
        for word in words[1:]:
            if index >= len(word) or word[index] != char:
                return words[0][:index]
    return words[0]

# 16: Rotate right.
def rotate(values, k):
    if not values:
        return []
    k %= len(values)
    return values[-k:] + values[:-k] if k else values[:]

# 17
def flatten_one_level(groups):
    return [item for group in groups for item in group]

# 18
def group_by_status(records):
    grouped = defaultdict(list)
    for record in records:
        grouped[record["status"]].append(record)
    return dict(grouped)

# 19
def chunks(values, size):
    if size <= 0:
        raise ValueError("size must be positive")
    for start in range(0, len(values), size):
        yield values[start:start + size]

# 20
def count_matching_lines(path, needle):
    with open(path, encoding="utf-8") as handle:
        return sum(needle in line for line in handle)

if __name__ == "__main__":
    assert reverse_string("abc") == "cba"
    assert is_palindrome("")
    assert first_unique("aabbc") == "c"
    assert two_sum([2, 7, 11], 9) == (0, 1)
    assert missing_number([0, 1, 3]) == 2
    assert merge_sorted([1, 3], [2, 4]) == [1, 2, 3, 4]
    assert valid_brackets("([])")
    assert not valid_brackets("([)]")
    assert move_zeros([0, 1, 0, 2]) == [1, 2, 0, 0]
    assert second_largest([3, 3, 2]) == 2
    assert rotate([1, 2, 3], 1) == [3, 1, 2]
```

## 16.4 Solutions 21–35

```python name=coding_21_35.py
import heapq
from collections import Counter, defaultdict, deque, OrderedDict

# 21
def longest_unique_substring(text):
    last_seen = {}
    left = best = 0
    for right, char in enumerate(text):
        if char in last_seen:
            left = max(left, last_seen[char] + 1)
        last_seen[char] = right
        best = max(best, right - left + 1)
    return best

# 22
def maximum_subarray(values):
    if not values:
        raise ValueError("nonempty input required")
    current = best = values[0]
    for value in values[1:]:
        current = max(value, current + value)
        best = max(best, current)
    return best

# 23
def product_except_self(values):
    result = [1] * len(values)
    prefix = 1
    for index, value in enumerate(values):
        result[index] = prefix
        prefix *= value
    suffix = 1
    for index in range(len(values) - 1, -1, -1):
        result[index] *= suffix
        suffix *= values[index]
    return result

# 24: Closed intervals; touching intervals are merged.
def merge_intervals(intervals):
    result = []
    for start, end in sorted(intervals):
        if start > end:
            raise ValueError("invalid interval")
        if not result or start > result[-1][1]:
            result.append([start, end])
        else:
            result[-1][1] = max(result[-1][1], end)
    return result

# 25: Tie order is not part of the contract.
def top_k_frequent(values, k):
    if k < 0:
        raise ValueError("k must be nonnegative")
    counts = Counter(values)
    return heapq.nlargest(k, counts, key=counts.get)

# 26
def kth_largest(values, k):
    if not 1 <= k <= len(values):
        raise ValueError("invalid k")
    heap = []
    for value in values:
        if len(heap) < k:
            heapq.heappush(heap, value)
        elif value > heap[0]:
            heapq.heapreplace(heap, value)
    return heap[0]

# 27
def count_subarrays_sum(values, target):
    counts = {0: 1}
    prefix = total = 0
    for value in values:
        prefix += value
        total += counts.get(prefix - target, 0)
        counts[prefix] = counts.get(prefix, 0) + 1
    return total

# 28: Nonnegative values and strictly positive target.
def min_window_sum(values, target):
    if target <= 0 or any(value < 0 for value in values):
        raise ValueError("positive target and nonnegative values required")
    left = total = 0
    best = len(values) + 1
    for right, value in enumerate(values):
        total += value
        while total >= target:
            best = min(best, right - left + 1)
            total -= values[left]
            left += 1
    return 0 if best == len(values) + 1 else best

# 29
def shortest_path(graph, start, goal):
    queue = deque([start])
    parent = {start: None}
    while queue:
        node = queue.popleft()
        if node == goal:
            path = []
            while node is not None:
                path.append(node)
                node = parent[node]
            return path[::-1]
        for neighbor in graph.get(node, []):
            if neighbor not in parent:
                parent[neighbor] = node
                queue.append(neighbor)
    return None

# 30
def has_directed_cycle(graph):
    adjacency = {node: list(neighbors) for node, neighbors in graph.items()}
    nodes = set(adjacency)
    for neighbors in adjacency.values():
        nodes.update(neighbors)
    indegree = {node: 0 for node in nodes}
    for neighbors in adjacency.values():
        for neighbor in neighbors:
            indegree[neighbor] += 1
    queue = deque(node for node in nodes if indegree[node] == 0)
    removed = 0
    while queue:
        node = queue.popleft()
        removed += 1
        for neighbor in adjacency.get(node, []):
            indegree[neighbor] -= 1
            if indegree[neighbor] == 0:
                queue.append(neighbor)
    return removed != len(nodes)

# 31: Single-threaded in-memory cache.
class LRUCache:
    def __init__(self, capacity):
        if capacity <= 0:
            raise ValueError("capacity must be positive")
        self.capacity = capacity
        self.data = OrderedDict()

    def get(self, key):
        if key not in self.data:
            raise KeyError(key)
        self.data.move_to_end(key)
        return self.data[key]

    def put(self, key, value):
        self.data[key] = value
        self.data.move_to_end(key)
        if len(self.data) > self.capacity:
            self.data.popitem(last=False)

# 32
def merge_k_sorted(groups):
    return list(heapq.merge(*groups))

# 33: Lowercase ASCII letters only.
def anagram_starts(text, pattern):
    if any(not "a" <= c <= "z" for c in text + pattern):
        raise ValueError("lowercase ASCII required")
    if not pattern or len(pattern) > len(text):
        return []
    need = [0] * 26
    window = [0] * 26
    for char in pattern:
        need[ord(char) - 97] += 1
    result = []
    width = len(pattern)
    for index, char in enumerate(text):
        window[ord(char) - 97] += 1
        if index >= width:
            window[ord(text[index - width]) - 97] -= 1
        if index >= width - 1 and window == need:
            result.append(index - width + 1)
    return result

# 34
def minimum_coins(coins, amount):
    if amount < 0 or any(type(c) is not int or c <= 0 for c in coins):
        raise ValueError("invalid amount or coins")
    dp = [0] + [amount + 1] * amount
    for subtotal in range(1, amount + 1):
        for coin in coins:
            if coin <= subtotal:
                dp[subtotal] = min(dp[subtotal], dp[subtotal - coin] + 1)
    return -1 if dp[amount] > amount else dp[amount]

# 35
def longest_consecutive(values):
    numbers = set(values)
    best = 0
    for number in numbers:
        if number - 1 not in numbers:
            end = number
            while end in numbers:
                end += 1
            best = max(best, end - number)
    return best

if __name__ == "__main__":
    assert longest_unique_substring("abba") == 2
    assert maximum_subarray([-3, -1, -2]) == -1
    assert product_except_self([1, 2, 3, 4]) == [24, 12, 8, 6]
    assert merge_intervals([[1, 3], [2, 4]]) == [[1, 4]]
    assert kth_largest([3, 1, 2], 2) == 2
    assert count_subarrays_sum([1, 1, 1], 2) == 2
    assert min_window_sum([2, 3, 1, 2, 4, 3], 7) == 2
    assert shortest_path({"a": ["b"], "b": ["c"]}, "a", "c") == [
        "a", "b", "c"
    ]
    assert has_directed_cycle({"a": ["b"], "b": ["a"]})
    assert anagram_starts("cbaebabacd", "abc") == [0, 6]
    assert minimum_coins([1, 2, 5], 11) == 3
    assert longest_consecutive([100, 4, 200, 1, 3, 2]) == 4
```

## 16.5 Solutions 36–50

These are interview models. In-memory caches, rate limits, deduplication, and circuit breakers are not distributed or durable.

```python name=coding_36_50.py
import asyncio
import hashlib
import json
import random
import time
from collections import Counter

# 36
def validate_payment(record):
    if not isinstance(record, dict):
        raise ValueError("object required")
    if not isinstance(record.get("payment_id"), str) or not record["payment_id"]:
        raise ValueError("payment_id required")
    if type(record.get("amount_minor")) is not int:
        raise ValueError("integer amount required")
    if record["amount_minor"] <= 0:
        raise ValueError("positive amount required")
    if record.get("currency") not in {"USD", "EUR", "JPY"}:
        raise ValueError("unsupported lab currency")
    return dict(record)

# 37: Fail-fast policy; report line number, not sensitive content.
def read_jsonl(path):
    with open(path, encoding="utf-8") as handle:
        for number, line in enumerate(handle, 1):
            if not line.strip():
                continue
            try:
                yield json.loads(line)
            except json.JSONDecodeError as exc:
                raise ValueError(f"invalid JSON at line {number}") from exc

# 38
def reconcile(left, right):
    def index(records):
        result = {}
        for record in records:
            key = record["payment_id"]
            if key in result:
                raise ValueError(f"duplicate payment_id: {key}")
            result[key] = record
        return result

    a, b = index(left), index(right)
    mismatched = []
    for key in a.keys() & b.keys():
        fields = ("amount_minor", "currency", "status")
        if any(a[key].get(field) != b[key].get(field) for field in fields):
            mismatched.append(key)
    return {
        "only_left": sorted(a.keys() - b.keys()),
        "only_right": sorted(b.keys() - a.keys()),
        "mismatched": sorted(mismatched),
    }

# 39: Single-threaded; local deterministic operations only.
# Unsafe as the sole protection for real external side effects.
class IdempotencyRegistry:
    def __init__(self):
        self.entries = {}

    def run(self, key, payload, operation):
        canonical = json.dumps(
            payload, sort_keys=True, separators=(",", ":"), allow_nan=False
        )
        fingerprint = hashlib.sha256(canonical.encode()).hexdigest()
        if key in self.entries:
            old_fingerprint, result = self.entries[key]
            if old_fingerprint != fingerprint:
                raise ValueError("idempotency key reused with different payload")
            return result
        result = operation()
        self.entries[key] = (fingerprint, result)
        return result

# 40
class TransientError(Exception):
    pass

def retry_transient(operation, attempts=3, sleep=time.sleep, rng=random.random):
    if attempts < 1:
        raise ValueError("attempts must be positive")
    for attempt in range(attempts):
        try:
            return operation()
        except TransientError:
            if attempt == attempts - 1:
                raise
            sleep(rng() * min(2.0, 0.1 * 2**attempt))

# 41: fetch_page(cursor) returns {"items": [...], "next_cursor": ...}.
def paginate(fetch_page, max_pages=1000):
    cursor = None
    seen = set()
    for _ in range(max_pages):
        page = fetch_page(cursor)
        yield from page["items"]
        next_cursor = page.get("next_cursor")
        if next_cursor is None:
            return
        if not isinstance(next_cursor, str):
            raise ValueError("string cursor required")
        if next_cursor in seen:
            raise ValueError("pagination cycle")
        seen.add(next_cursor)
        cursor = next_cursor
    raise RuntimeError("page limit exceeded")

# 42: Single-process, single-threaded model.
class TokenBucket:
    def __init__(self, rate, capacity, clock=time.monotonic):
        if rate <= 0 or capacity <= 0:
            raise ValueError("positive rate and capacity required")
        self.rate = rate
        self.capacity = capacity
        self.tokens = float(capacity)
        self.clock = clock
        self.updated = clock()

    def allow(self, cost=1):
        if cost <= 0:
            raise ValueError("positive cost required")
        now = self.clock()
        self.tokens = min(
            self.capacity,
            self.tokens + max(0, now - self.updated) * self.rate
        )
        self.updated = now
        if self.tokens < cost:
            return False
        self.tokens -= cost
        return True

# 43: Lazy expiry; not size-bounded.
class TTLCache:
    def __init__(self, clock=time.monotonic):
        self.clock = clock
        self.data = {}

    def put(self, key, value, ttl):
        if ttl <= 0:
            raise ValueError("positive ttl required")
        self.data[key] = (self.clock() + ttl, value)

    def get(self, key):
        expires, value = self.data[key]
        if self.clock() >= expires:
            del self.data[key]
            raise KeyError(key)
        return value

# 44: JSON-like tree; denylist is illustrative, allowlists are safer for logs.
SENSITIVE = {"password", "token", "authorization", "secret", "cvv", "card_number"}

def redact(value):
    if isinstance(value, dict):
        return {
            key: (
                "[REDACTED]" if str(key).lower() in SENSITIVE
                else redact(item)
            )
            for key, item in value.items()
        }
    if isinstance(value, list):
        return [redact(item) for item in value]
    return value

# 45: Batch-local deduplication with unbounded seen set for the batch.
def deduplicate_events(events):
    seen = set()
    for event in events:
        key = (event["tenant_id"], event["event_id"])
        if key not in seen:
            seen.add(key)
            yield event

# 46
def summarize_logs(records):
    counts = Counter()
    total_ms = 0
    count = 0
    for record in records:
        duration = record["duration_ms"]
        if duration < 0:
            raise ValueError("negative duration")
        counts[record["status"]] += 1
        total_ms += duration
        count += 1
    return {
        "counts": dict(counts),
        "mean_duration_ms": total_ms / count if count else None,
    }

# 47: Sequential circuit-breaker model.
class CircuitOpen(Exception):
    pass

class CircuitBreaker:
    def __init__(self, threshold=3, cooldown=10, clock=time.monotonic):
        if threshold <= 0 or cooldown <= 0:
            raise ValueError("positive threshold and cooldown required")
        self.threshold = threshold
        self.cooldown = cooldown
        self.clock = clock
        self.failures = 0
        self.open_until = None

    def call(self, operation):
        if self.open_until is not None and self.clock() < self.open_until:
            raise CircuitOpen()
        try:
            result = operation()
        except TransientError:
            self.failures += 1
            if self.failures >= self.threshold:
                self.open_until = self.clock() + self.cooldown
            raise
        else:
            self.failures = 0
            self.open_until = None
            return result

# 48
def sha256_file(path, chunk_size=65536):
    if chunk_size <= 0:
        raise ValueError("positive chunk_size required")
    digest = hashlib.sha256()
    with open(path, "rb") as handle:
        while chunk := handle.read(chunk_size):
            digest.update(chunk)
    return digest.hexdigest()

# 49: Finite sequence input; fixed number of worker tasks.
async def bounded_map(items, operation, concurrency=4):
    if concurrency <= 0:
        raise ValueError("positive concurrency required")
    iterator = iter(enumerate(items))
    results = [None] * len(items)

    async def worker():
        while True:
            try:
                index, item = next(iterator)
            except StopIteration:
                return
            results[index] = await operation(item)

    async with asyncio.TaskGroup() as group:
        for _ in range(min(concurrency, len(items))):
            group.create_task(worker())
    return results

# 50: Snapshot events; updates may skip intermediate versions.
def apply_versioned_event(state, event):
    key = (event["tenant_id"], event["entity_id"])
    version = event["version"]
    if type(version) is not int or version < 0:
        raise ValueError("nonnegative integer version required")
    old = state.get(key)
    if old is not None and version <= old["version"]:
        return False
    state[key] = {"version": version, "status": event["status"]}
    return True

if __name__ == "__main__":
    registry = IdempotencyRegistry()
    assert registry.run("k", {"x": 1}, lambda: 42) == 42
    assert registry.run("k", {"x": 1}, lambda: 99) == 42

    assert redact({"token": "hidden", "status": "ok"}) == {
        "token": "[REDACTED]", "status": "ok"
    }

    state = {}
    assert apply_versioned_event(state, {
        "tenant_id": "t1", "entity_id": "p1",
        "version": 2, "status": "succeeded"
    })
    assert not apply_versioned_event(state, {
        "tenant_id": "t1", "entity_id": "p1",
        "version": 1, "status": "pending"
    })
```

## 16.6 Deep follow-ups for FDE problems

- **39:** What happens if the process crashes after an external operation succeeds?
- **40:** Is the operation safe to retry, and what is the total time budget?
- **41:** Can data change between pages? What consistency contract does the API provide?
- **42:** How do multiple replicas share rate-limit state?
- **43:** How do you bound memory and clean expired untouched keys?
- **44:** Can secrets appear in free text or unexpected field names?
- **45:** What is the deduplication retention window and persistence model?
- **47:** How do you restrict concurrent half-open probes?
- **49:** How do you process an infinite stream with bounded result storage?
- **50:** Is the event a snapshot or a delta? A delta may require detecting and repairing gaps.

## 16.7 Common mistakes

- Writing code before defining input contracts.
- Giving complexity without explaining variables.
- Ignoring output storage.
- Assuming a hash map guarantees worst-case O(1).
- Claiming local data structures solve distributed coordination.
- Omitting failure and concurrency tests.

**Interviewers expect:** Clear reasoning, correct implementation, tests, and honest boundaries.

---

# MODULE 17 — ONE HUNDRED MOCK INTERVIEW QUESTIONS

These are representative FDE-style questions with concise model answers, not verified questions from any named company.

Use each as a spoken exercise. Answer first, then compare.

## Python — 1–10

1. **How would you process a 10 GB log?** Stream it, parse only required fields, aggregate bounded state, and emit partial results. If grouping cardinality is large, partition or use external storage.
2. **Why use a generator?** It avoids materializing all results and composes with streaming pipelines. It does not eliminate memory used by downstream consumers.
3. **How would you find duplicate payment IDs?** Use a set or Counter for a bounded dataset. For larger datasets, partition or use a database uniqueness check; define tenant scope.
4. **How do you structure an integration script?** Separate transport, validation, transformation, persistence, and orchestration so each can be tested.
5. **When are threads useful?** For overlapping blocking I/O, with bounded workers and safe shared state. Benchmark instead of assuming speedup.
6. **When is async useful?** Many concurrent I/O operations with compatible libraries; avoid blocking calls and propagate cancellation correctly.
7. **How do you represent money?** Currency plus integer minor units or Decimal, with explicit scale and rounding.
8. **What belongs in a retry loop?** Only failures classified as transient and operations safe to retry, with bounded attempts, jitter, and deadlines.
9. **How do you test time-based code?** Inject a clock and sleep function so tests are deterministic and fast.
10. **How do you make a script resumable?** Persist checkpoints after durable writes and tolerate replay through stable identifiers.

## SQL — 11–20

11. **How do you get the latest payment event?** Partition by payment ID and order by event time plus a tie-breaker using ROW_NUMBER.
12. **Why is a LEFT JOIN returning fewer rows than expected?** A WHERE condition on right-side columns may filter NULL rows and effectively remove unmatched records.
13. **Why are totals too large?** A join may multiply rows. Aggregate at the intended grain and verify relationship cardinality.
14. **How do you calculate payment success rate?** Define eligible attempts and status meaning, then divide successful count by eligible count with zero-denominator handling.
15. **How do you choose an index?** Start with frequent expensive access patterns, inspect plans, and balance read gains against write costs.
16. **What is a lost update?** Concurrent operations overwrite one another’s changes. Use atomic updates, versions, or suitable locking.
17. **How do you handle deadlocks?** Keep transactions short, acquire resources consistently, and retry the entire transaction when safe.
18. **Why not run reports on the primary?** Expensive analytics can compete with transactional work. A replica or warehouse can help, with freshness trade-offs.
19. **What does normalization solve?** Duplicate facts and update anomalies; it does not automatically optimize every query.
20. **Why use keyset pagination?** It avoids large-offset scanning and offers a stable progression when based on a deterministic indexed order.

## AWS — 21–30

21. **EC2, ECS, or Lambda?** Compare control, runtime model, workload duration, traffic pattern, deployment needs, and operational burden.
22. **S3 or RDS?** S3 stores objects; RDS supports relational queries and transactions. They solve different problems.
23. **How do you give a workload AWS access?** Use a suitable role with temporary credentials and narrowly scoped permissions.
24. **Why is an SQS message processed twice?** Delivery or acknowledgment failures can cause redelivery. The consumer must tolerate duplicates.
25. **What is a DLQ for?** Isolating repeatedly failing work for investigation and controlled replay, not hiding failures.
26. **Why can Lambda overwhelm RDS?** Scaled function concurrency can create excessive connections or queries. Bound concurrency and manage connections.
27. **How do you troubleshoot AccessDenied?** Confirm caller identity, action, resource, and applicable policies, including explicit denies and organization boundaries.
28. **What is a DynamoDB hot partition?** Traffic concentrates on a partition-key range. Redesign distribution or access patterns rather than only adding total capacity.
29. **Why use Secrets Manager?** Centralized controlled secret retrieval and rotation support, with application behavior designed around rotation.
30. **What would you monitor for a queue worker?** Completion rate, age, retry rate, errors, saturation, and downstream limits.

## Linux — 31–40

31. **A service is unreachable. First checks?** Confirm process, listening address and port, local request behavior, then network path and policy.
32. **High CPU?** Identify processes and threads, correlate with traffic and releases, and profile the expensive path.
33. **High memory?** Inspect working set, growth over time, limits, caches, and leaks; distinguish usage from pressure.
34. **Disk full?** Check both bytes and inodes, identify growth, preserve evidence, and follow retention policy.
35. **Why can deleting a log fail to free space?** A process may still hold the deleted file open.
36. **What does `chmod 600` mean?** Owner read/write, no group or other permissions; parent-directory and other controls still matter.
37. **Why use `tail -F`?** It follows the filename across replacement/rotation in common implementations.
38. **What is a file descriptor?** A process-local reference to an open resource such as a file or socket; exhaustion can break unrelated operations.
39. **When use SIGKILL?** When graceful shutdown fails and the risk is understood, because cleanup cannot run.
40. **How do you analyze logs safely?** Use minimal authorized access, avoid exporting secrets, preserve timestamps, and parse known structure.

## Docker — 41–50

41. **Why does an image run locally but not in production?** Compare architecture, environment, permissions, dependencies, networking, and runtime policy.
42. **Why minimize the image?** Reduce unnecessary packages, transfer size, and attack surface without sacrificing required functionality.
43. **What is a multi-stage build?** Separate build-time tooling from runtime artifacts.
44. **How should a container persist data?** Use an explicitly managed persistent store or volume rather than assuming its writable layer is durable.
45. **How do you inject secrets?** Through an approved runtime mechanism, not source code or image layers.
46. **Why does port publishing not help?** The application may not be listening on the expected interface or port.
47. **What happens when PID 1 exits?** The container’s main process ends; runtime restart policy determines what follows.
48. **Why use image digests?** They identify immutable content more precisely than mutable tags.
49. **How do you investigate a crash?** Inspect logs, exit status, startup command, resource events, and filesystem permissions.
50. **Is a non-root container completely secure?** No. It reduces privilege but still requires isolation, patched dependencies, and restricted capabilities.

## Kubernetes — 51–60

51. **What does a Deployment reconcile?** Desired replicated workload state, including rollout and replacement.
52. **What causes CrashLoopBackOff?** Repeated container failures; inspect the underlying exit reason rather than treating the status as the cause.
53. **Why is a Service empty?** Selector mismatch or no suitable endpoints; check readiness and EndpointSlices.
54. **Readiness or liveness for a dependency outage?** Readiness may reflect inability to serve, but avoid dependency-coupled liveness restart storms.
55. **Why are requests important?** Scheduling and utilization-based autoscaling depend on them.
56. **Why are limits important?** They constrain usage, but inappropriate limits can cause throttling or memory termination.
57. **How does HPA work?** A controller adjusts replicas from observed metrics and policy; adequate node capacity must also exist.
58. **Are Secrets encrypted because they are base64?** No; base64 is encoding. Storage encryption and access control are separate.
59. **How do you roll back?** Select a known compatible revision or image and verify rollout, health, and business behavior.
60. **Why can a namespace fail to isolate tenants?** RBAC, networking, quotas, node sharing, and workload permissions also affect isolation.

## APIs — 61–70

61. **REST versus SOAP?** REST is an architectural style commonly implemented over HTTP; SOAP is a structured messaging protocol with an XML envelope and associated standards.
62. **PUT versus PATCH?** Replacement-state semantics versus partial modification; patch idempotency depends on the operation.
63. **401 versus 403?** Missing/invalid authentication versus refusal of access; resource-hiding policies can affect responses.
64. **What makes POST retry-safe?** A documented idempotency contract or an operation known to tolerate repetition.
65. **How do you verify a webhook?** Verify the provider’s signature over the required raw bytes, validate timestamp/replay rules, and deduplicate durably.
66. **What should an error response contain?** Stable error code, safe message, request ID, and relevant retry information without internal secrets.
67. **Why validate JWT audience?** A valid token intended for another service should not grant access here.
68. **How do you version APIs?** Prefer compatible evolution, explicit deprecation, contract testing, and a defined versioning policy when breaking changes are unavoidable.
69. **What is rate limiting for?** Protect capacity and fairness; define scope, burst behavior, and retry guidance.
70. **What if a provider returns 200 with malformed JSON?** Treat it as a contract failure, capture sanitized evidence, and avoid assuming success from status alone.

## Troubleshooting — 71–80

71. **What do you do first during an incident?** Confirm impact and scope, establish ownership, and begin an evidence timeline.
72. **How do you form hypotheses?** Use symptom boundaries, recent changes, and telemetry to rank plausible causes, then test discriminating evidence.
73. **Rollback or debug?** Prefer a safe known mitigation when impact is active, while preserving enough evidence for later diagnosis.
74. **What if you do not know the root cause?** State known facts and uncertainty, explain the investigation, and give the next update time.
75. **How do you distinguish root cause from trigger?** The trigger starts the incident; root and contributing conditions explain why the system failed to tolerate it.
76. **Why not restart everything?** It destroys evidence, can amplify load, and may not address the cause.
77. **How do you verify recovery?** Check user-facing metrics, successful workflows, backlog drain, and unresolved data states.
78. **What should an incident update avoid?** Blame, unsupported assurances, speculative causes stated as fact, and promises without evidence.
79. **How do you replay failed events?** Fix the cause, validate schema and idempotency, bound replay rate, and monitor outcomes.
80. **What makes an action item useful?** A concrete change with owner, deadline, and a verification method.

## System design — 81–90

81. **Where do you start a design?** Users, operations, constraints, and measurable requirements—not technology names.
82. **When add a cache?** When measured access patterns benefit and acceptable staleness and invalidation behavior are defined.
83. **When add a queue?** When work can be asynchronous and buffering, decoupling, or retry isolation is useful.
84. **How do you avoid a dual-write gap?** Use an outbox or another suitable transactional coordination pattern; consumers still handle duplicates.
85. **How do you design tenant isolation?** Carry tenant identity through authentication, authorization, storage access, background jobs, logs, and operational tools.
86. **What is backpressure?** Slowing or rejecting incoming work when downstream capacity is insufficient rather than allowing unbounded accumulation.
87. **How do you choose consistency?** Identify which operations require fresh/serialized state and which tolerate lag; choose per workflow.
88. **How do you handle out-of-order events?** Use versions or sequence numbers, distinguish snapshots from deltas, and repair gaps where needed.
89. **How do you design disaster recovery?** Define RTO/RPO, backup and replication strategy, failover procedure, and restoration exercises.
90. **What is your first scaling move?** Identify the measured bottleneck; optimize or add capacity there rather than adding distributed complexity blindly.

## Communication — 91–100

91. **A customer asks for an impossible deadline.** Clarify the essential outcome, propose a smaller safe scope, identify dependencies, and agree on a realistic plan.
92. **A customer says the product is broken.** Translate the statement into a reproducible workflow and impact, then investigate without defensiveness.
93. **How do you explain a technical trade-off to an executive?** Compare business consequences, risk, cost, and timeline, then recommend an option.
94. **How do you disagree with a senior engineer?** State evidence and assumptions respectfully, test the disagreement, and support the final decision.
95. **How do you handle a mistake you caused?** Disclose promptly, mitigate, communicate impact, and improve the system rather than conceal responsibility.
96. **How do you prioritize two customers?** Compare severity, contractual obligations, business impact, and available workarounds with the appropriate owner.
97. **How do you hand over a deployment?** Provide runbooks, access ownership, dashboards, recovery procedures, training, and acceptance evidence.
98. **What if a customer asks for an unsafe workaround?** Explain the risk, decline the unsafe path, and offer a safer alternative or escalation.
99. **Why should an FDE write documentation?** It reduces dependency on individuals, improves supportability, and makes repeated deployments faster.
100. **What does success look like after launch?** Users adopt the workflow, agreed outcomes improve, reliability is measurable, and support ownership is clear.

## 17.1 Mock interview formats

### 45-minute technical mock

- 5 minutes: Project overview.
- 15 minutes: One coding problem.
- 10 minutes: API/SQL discussion.
- 10 minutes: Incident scenario.
- 5 minutes: Questions and reflection.

### 60-minute FDE simulation

- 10 minutes: Customer discovery.
- 15 minutes: Architecture.
- 15 minutes: Implementation sketch.
- 10 minutes: Failure injection.
- 10 minutes: Customer communication and rollout plan.

## 17.2 Scoring

Score each category 0–4:

- Requirement clarification.
- Correctness.
- Failure handling.
- Security.
- Trade-offs.
- Communication.
- Customer outcome.

Do not reward jargon without a coherent explanation.

---

# MODULE 18 — SIXTY-DAY STUDY PLAN

## 18.1 Daily structure

Suggested time: approximately 2–3 hours, adjusted to your schedule.

- 35 minutes: Learn.
- 45 minutes: Hands-on exercise.
- 35 minutes: Coding.
- 15 minutes: Revision.
- 15 minutes: Spoken mock answers.
- 10 minutes: Record mistakes and next actions.

Question references:

- **C1–C50:** Module 16 coding problems.
- **M1–M100:** Module 17 mock questions.
- **P01–P20:** Module 2 examples.
- **E01–E20:** Module 2 exercises.

## 18.2 Day-by-day roadmap

| Day | Topics | Hands-on exercise | Coding practice | Revision task | Mock questions |
|---|---|---|---|---|---|
| 1 | FDE role and discovery | Write customer problem brief | C1 | Explain role differences | M91, M100 |
| 2 | Python values and loops | Run P01–P06 | C2, C3 | Mutability and equality | M3, M4 |
| 3 | Functions and validation | Build fee calculator tests | C4, C5 | Input contracts | M7, M9 |
| 4 | Collections | Group synthetic payments | C6, C7 | Dict/set complexity | M3 |
| 5 | Files and generators | Stream a log | C8, C20 | Memory bounds | M1, M2 |
| 6 | Exceptions and JSON | Reject malformed records | C9, C36 | Error classification | M8 |
| 7 | Python review | Complete E01–E06 | C10 | Re-solve two failures | M1–M5 |
| 8 | OOP and testing | Build Payment model | C11, C12 | State and behavior | M4, M9 |
| 9 | Logging | Add allowlisted JSON logs | C18, C46 | Secret handling | M40 |
| 10 | Threads | Compare bounded I/O workers | C19 | Races and pool limits | M5 |
| 11 | Async | Run bounded async tasks | C49 | Blocking versus awaiting | M6 |
| 12 | HTTP client | Start local server and client | C40 | Retry safety | M64, M70 |
| 13 | Importer design | Add checkpoint and restart test | C37 | Commit/checkpoint ordering | M10 |
| 14 | Python mock | Timed implementation and review | C13, C14 | Weak Python topics | M1–M10 |
| 15 | SQL fundamentals | Load schema and seed data | C15 | SQL logical order | M14 |
| 16 | Joins | Demonstrate row multiplication | C16 | Join cardinality | M12, M13 |
| 17 | Aggregation | Compute per-currency totals | C17 | Denominator definitions | M14 |
| 18 | Window functions | Latest event and running total | C21 | Tie-breaking | M11 |
| 19 | Transactions | Competing conditional updates | C22 | ACID and invariants | M16, M17 |
| 20 | Indexes and plans | Compare query plans | C23 | Read/write trade-offs | M15 |
| 21 | Database selection | Write selection matrix | C24 | SQL versus document/key-value | M18–M20 |
| 22 | REST and HTTP | Define API contract | C25 | Methods/status codes | M61–M64 |
| 23 | Authentication | Draw OAuth and token validation | C26 | Authentication/authorization | M67 |
| 24 | Webhooks and idempotency | Simulate duplicate webhook | C39, C45 | Unknown outcomes | M65 |
| 25 | Pagination and rate limits | Mock paginated API | C41, C42 | Cursor consistency | M69 |
| 26 | Linux processes | Inspect lab process and socket | C27 | Process lifecycle | M31, M32 |
| 27 | Linux files and permissions | Create safe permission failure | C48 | Filesystem checks | M34–M36 |
| 28 | Linux troubleshooting | Diagnose three injected faults | C28 | Layered investigation | M37–M40 |
| 29 | Git | Branch, conflict, revert lab | C29 | Shared-history safety | Explain revert versus reset |
| 30 | Midpoint mock | 60-minute customer simulation | C30 | Review scorecard gaps | M71, M81, M92 |
| 31 | AWS compute | Compare EC2/ECS/Lambda | C31 | Runtime trade-offs | M21 |
| 32 | AWS storage/database | Design S3/RDS responsibilities | C32 | Durability and consistency | M22 |
| 33 | IAM and secrets | Draft least-privilege policy | C44 | Temporary credentials | M23, M27, M29 |
| 34 | VPC and networking | Draw routes and boundaries | C33 | DNS/TLS/egress sequence | Explain private-subnet egress |
| 35 | Messaging | Design SNS/SQS/EventBridge flow | C34 | Duplicates and DLQs | M24, M25 |
| 36 | Lambda and DynamoDB | Analyze concurrency limits | C35 | Downstream protection | M26, M28 |
| 37 | AWS review | Tabletop cloud outage | C38 | Service-selection reasons | M21–M30 |
| 38 | Docker fundamentals | Build and run local image | Revisit C36 | Images versus containers | M41–M44 |
| 39 | Docker hardening | Non-root/read-only lab | Revisit C44 | Secret and image handling | M45–M50 |
| 40 | Kubernetes objects | Deploy API and Service | Revisit C49 | Desired-state model | M51, M53 |
| 41 | Probes and resources | Break readiness and fix it | Revisit C43 | Readiness/liveness | M52, M54–M56 |
| 42 | HPA and troubleshooting | Inspect metrics and events | C50 | Pod versus node scaling | M57–M60 |
| 43 | CI basics | Run unit tests in one pipeline | Revisit C12 | Test boundaries | Explain build once |
| 44 | Delivery safety | Write canary/rollback plan | Revisit C39 | Migration compatibility | Explain rollback risk |
| 45 | Observability | Define
