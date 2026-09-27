---

# Module 3 — SQL and Databases

## 3.1 Beginner explanation

A database stores information so applications can retrieve and change it
reliably.

SQL is a language for describing which relational data you want and how it
should be combined, filtered, or changed.

Instead of describing every loop yourself, you usually describe the result.
The database chooses an execution plan.

## 3.2 Industry-level explanation

Database engineering includes:

- Data modeling.
- Constraints.
- Transactions and isolation.
- Query design.
- Index selection.
- Concurrency control.
- Recovery.
- Capacity planning.
- Access control.
- Operational diagnosis.

A query that returns the correct answer on five rows may perform poorly on
five hundred million rows.

A fast query can still be wrong if it duplicates money through an incorrect join.

## 3.3 Payment data architecture

```text name=payment_database_architecture.txt
Customer request
       |
       v
Payment API
       |
       | Short transaction
       v
Relational database
  ├── merchants
  ├── payments
  ├── payment_events
  └── refunds
       |
       +----> Reporting / reconciliation
       |
       +----> Event publishing through a durable outbox design

External provider calls:
Must be coordinated with local state, but are not automatically part
of the database transaction.
```

---

## 3.4 Runnable PostgreSQL lab schema

Run this only in an empty, disposable database.

```sql name=payment_schema.sql
CREATE TABLE merchants (
    merchant_id BIGINT PRIMARY KEY,
    merchant_name TEXT NOT NULL
);

CREATE TABLE payments (
    payment_id BIGINT PRIMARY KEY,
    merchant_id BIGINT NOT NULL REFERENCES merchants(merchant_id),
    idempotency_key TEXT NOT NULL,
    amount_minor BIGINT NOT NULL CHECK (amount_minor > 0),
    currency TEXT NOT NULL CHECK (currency IN ('USD', 'INR', 'EUR')),
    status TEXT NOT NULL CHECK (
        status IN ('pending', 'succeeded', 'failed', 'cancelled')
    ),
    created_at TIMESTAMPTZ NOT NULL,
    UNIQUE (merchant_id, idempotency_key)
);

CREATE TABLE payment_events (
    event_id BIGINT PRIMARY KEY,
    payment_id BIGINT NOT NULL REFERENCES payments(payment_id),
    provider_event_id TEXT NOT NULL UNIQUE,
    status TEXT NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE refunds (
    refund_id BIGINT PRIMARY KEY,
    payment_id BIGINT NOT NULL REFERENCES payments(payment_id),
    amount_minor BIGINT NOT NULL CHECK (amount_minor > 0),
    status TEXT NOT NULL CHECK (
        status IN ('pending', 'succeeded', 'failed')
    )
);

INSERT INTO merchants VALUES
    (1, 'Merchant Alpha'),
    (2, 'Merchant Beta'),
    (3, 'Merchant Gamma');

INSERT INTO payments VALUES
    (101, 1, 'order-1', 1000, 'USD', 'succeeded', '2026-09-20T10:00:00Z'),
    (102, 1, 'order-2', 2000, 'USD', 'failed',    '2026-09-20T11:00:00Z'),
    (103, 2, 'order-3', 3000, 'USD', 'succeeded', '2026-09-20T12:00:00Z'),
    (104, 2, 'order-4', 4000, 'INR', 'pending',   '2026-09-21T10:00:00Z'),
    (105, 1, 'order-5', 1500, 'USD', 'succeeded', '2026-09-21T11:00:00Z');

INSERT INTO payment_events VALUES
    (1, 101, 'evt-1', 'pending',   '2026-09-20T10:00:00Z'),
    (2, 101, 'evt-2', 'succeeded', '2026-09-20T10:00:03Z'),
    (3, 102, 'evt-3', 'failed',    '2026-09-20T11:00:02Z'),
    (4, 103, 'evt-4', 'succeeded', '2026-09-20T12:00:04Z'),
    (5, 104, 'evt-5', 'pending',   '2026-09-21T10:00:00Z'),
    (6, 105, 'evt-6', 'succeeded', '2026-09-21T11:00:02Z');

INSERT INTO refunds VALUES
    (1, 101, 200, 'succeeded'),
    (2, 101, 100, 'succeeded'),
    (3, 103, 500, 'pending');
```

### Limitations of the teaching schema

It is not a complete financial ledger.

A production design may need:

- Separate payment attempts.
- Provider account identifiers.
- Request hashes for idempotency validation.
- Authorization, capture, reversal, refund, and dispute entities.
- Immutable accounting entries.
- Stronger state-transition enforcement.
- Refund-total and currency invariants.
- Retention and audit policies.
- Tenant-aware access controls.

---

## 3.5 SELECT and WHERE

### Beginner

`SELECT` chooses columns. `WHERE` filters rows.

### Industry

Retrieve only needed columns and define time boundaries explicitly.

```sql name=select_successful_payments.sql
SELECT payment_id, merchant_id, amount_minor, currency
FROM payments
WHERE status = 'succeeded'
ORDER BY payment_id;
```

### Payment-system example

Find pending payments older than a threshold:

```sql name=stale_pending_payments.sql
SELECT payment_id, merchant_id, created_at
FROM payments
WHERE status = 'pending'
  AND created_at < TIMESTAMPTZ '2026-09-22T00:00:00Z'
ORDER BY created_at, payment_id;
```

### Mistake

Treating old pending payments as definitively failed without checking the
provider or workflow.

### Exercise

Return only Merchant Alpha's successful USD payments.

---

## 3.6 JOINs

### Beginner

A join combines related rows from different tables.

### Industry

Before joining, identify the cardinality:

- One-to-one.
- One-to-many.
- Many-to-many.

Cardinality determines whether totals are multiplied.

### INNER JOIN

Return payments with merchant names:

```sql name=payments_with_merchants.sql
SELECT
    p.payment_id,
    m.merchant_name,
    p.amount_minor,
    p.currency
FROM payments AS p
JOIN merchants AS m
  ON m.merchant_id = p.merchant_id;
```

### LEFT JOIN

Keep merchants even if they have no payments:

```sql name=merchant_payment_counts.sql
SELECT
    m.merchant_id,
    m.merchant_name,
    COUNT(p.payment_id) AS payment_count
FROM merchants AS m
LEFT JOIN payments AS p
  ON p.merchant_id = m.merchant_id
GROUP BY m.merchant_id, m.merchant_name
ORDER BY m.merchant_id;
```

Merchant Gamma should have a count of zero.

### Important mistake

With a left join, `COUNT(*)` counts the preserved merchant row even when the
payment side is NULL. Count a non-null payment identifier instead.

### Another mistake

A payment with two refunds appears twice after a direct join to refunds.
Summing payment amounts after that join can double-count the original payment.

---

## 3.7 GROUP BY and HAVING

### Beginner

`GROUP BY` forms groups. Aggregate functions summarize them.
`HAVING` filters the groups.

### Industry

Choose grouping dimensions carefully, especially currency and tenant.

```sql name=merchant_success_totals.sql
SELECT
    merchant_id,
    currency,
    COUNT(*) AS successful_payments,
    SUM(amount_minor) AS total_minor
FROM payments
WHERE status = 'succeeded'
GROUP BY merchant_id, currency
HAVING SUM(amount_minor) >= 2000
ORDER BY merchant_id, currency;
```

### WHERE versus HAVING

- `WHERE`: Filter input rows.
- `HAVING`: Filter aggregate groups.

### Exercise

Find merchants with at least two successful payments in the same currency.

---

## 3.8 Window functions

### Beginner

A window function calculates across related rows while keeping each row in
the output.

### Industry

Distinguish:

- Partition: Which rows belong together?
- Order: In what sequence are they considered?
- Frame: Which subset contributes to the current calculation?

### Latest event per payment

```sql name=latest_payment_event.sql
WITH ranked_events AS (
    SELECT
        event_id,
        payment_id,
        status,
        occurred_at,
        ROW_NUMBER() OVER (
            PARTITION BY payment_id
            ORDER BY occurred_at DESC, event_id DESC
        ) AS row_number
    FROM payment_events
)
SELECT payment_id, status, occurred_at
FROM ranked_events
WHERE row_number = 1
ORDER BY payment_id;
```

**Caveat:** The latest timestamp is not automatically the authoritative business
state. Provider ordering, event versions, and valid transitions still matter.

### Running successful volume

```sql name=running_payment_volume.sql
SELECT
    payment_id,
    merchant_id,
    currency,
    created_at,
    amount_minor,
    SUM(amount_minor) OVER (
        PARTITION BY merchant_id, currency
        ORDER BY created_at, payment_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total_minor
FROM payments
WHERE status = 'succeeded'
ORDER BY merchant_id, currency, created_at, payment_id;
```

### Ranking functions

- `ROW_NUMBER`: Unique sequential position within the window ordering.
- `RANK`: Equal values share a rank; later ranks may have gaps.
- `DENSE_RANK`: Equal values share a rank; later ranks have no gaps.

### Exercise

Return the two largest payments per merchant and currency.
Decide whether ties should increase the number of returned rows.

---

## 3.9 Practical payment queries

### Query A — Daily success rate

```sql name=daily_success_rate.sql
SELECT
    (created_at AT TIME ZONE 'UTC')::date AS payment_day,
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE status = 'succeeded') AS successful,
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE status = 'succeeded')
        / NULLIF(COUNT(*), 0),
        2
    ) AS success_percentage
FROM payments
GROUP BY (created_at AT TIME ZONE 'UTC')::date
ORDER BY payment_day;
```

Define the denominator with the business.

Does it include pending payments, cancelled payments, or repeated attempts?
A mathematically valid ratio can still represent the wrong product metric.

### Query B — Payments with no events

```sql name=payments_without_events.sql
SELECT p.payment_id
FROM payments AS p
WHERE NOT EXISTS (
    SELECT 1
    FROM payment_events AS e
    WHERE e.payment_id = p.payment_id
);
```

### Query C — Net successful payment amount after successful refunds

```sql name=net_payment_amount.sql
WITH successful_refunds AS (
    SELECT payment_id, SUM(amount_minor) AS refunded_minor
    FROM refunds
    WHERE status = 'succeeded'
    GROUP BY payment_id
)
SELECT
    p.payment_id,
    p.currency,
    p.amount_minor,
    COALESCE(r.refunded_minor, 0) AS refunded_minor,
    p.amount_minor - COALESCE(r.refunded_minor, 0) AS net_minor
FROM payments AS p
LEFT JOIN successful_refunds AS r
  ON r.payment_id = p.payment_id
WHERE p.status = 'succeeded'
ORDER BY p.payment_id;
```

Pre-aggregation avoids multiplying payment amounts through the one-to-many join.

This is a reporting example, not proof that refund invariants are enforced.

### Query D — Keyset pagination

```sql name=keyset_pagination.sql
SELECT payment_id, created_at, status
FROM payments
WHERE merchant_id = 1
  AND (created_at, payment_id) >
      (TIMESTAMPTZ '2026-09-20T10:00:00Z', 101)
ORDER BY created_at, payment_id
LIMIT 100;
```

The cursor needs both fields because timestamps can tie.

---

## 3.10 Indexes

### Beginner

An index is an additional structure that can help locate rows without examining
every row.

### Industry

An index is a trade-off:

- It can improve selected reads.
- It consumes storage.
- It adds work to inserts, updates, and deletes.
- It may not help a query that reads most of the table.

### Example

```sql name=payment_index.sql
CREATE INDEX payments_merchant_created_idx
ON payments (merchant_id, created_at, payment_id);
```

This is a reasonable candidate for queries that filter by merchant and then
scan payments in creation order.

### Production questions

- Which query is slow?
- How selective is the predicate?
- What are the row counts and data distribution?
- Is ordering required?
- What is the write rate?
- How will index creation affect the workload?

### Common mistakes

- Creating an index on every column.
- Assuming the presence of an index guarantees its use.
- Ignoring the order of composite index columns.
- Ignoring write amplification.
- Applying index changes to production without an operational plan.

---

## 3.11 Query optimization walkthrough

### Problem

A merchant-history query is slow.

```sql name=slow_payment_query.sql
SELECT *
FROM payments
WHERE merchant_id = 1
  AND DATE(created_at) = DATE '2026-09-20';
```

### Investigation

1. Confirm the slow query and parameter values.
2. Determine whether the query is waiting or consuming CPU.
3. Inspect the execution plan.
4. Compare estimated and actual row counts.
5. Check index fit, statistics, sort work, and rows scanned.
6. Test a targeted change on realistic data.

### Candidate rewrite

```sql name=improved_payment_query.sql
SELECT payment_id, amount_minor, currency, status
FROM payments
WHERE merchant_id = 1
  AND created_at >= TIMESTAMPTZ '2026-09-20T00:00:00Z'
  AND created_at <  TIMESTAMPTZ '2026-09-21T00:00:00Z'
ORDER BY created_at, payment_id;
```

This expresses a UTC time interval directly and avoids applying `DATE()` to
the indexed timestamp column.

It is not equivalent if the original query intentionally used a different
session time zone. Establish the required business time zone first.

### Inspect the plan

```sql name=explain_payment_query.sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT payment_id, amount_minor, currency, status
FROM payments
WHERE merchant_id = 1
  AND created_at >= TIMESTAMPTZ '2026-09-20T00:00:00Z'
  AND created_at <  TIMESTAMPTZ '2026-09-21T00:00:00Z'
ORDER BY created_at, payment_id;
```

**Safety:** `EXPLAIN ANALYZE` executes the statement. Do not casually run it on
modifying statements or expensive production queries.

**Lab limitation:** Five rows are too few to demonstrate realistic index
benefits. Generate a larger synthetic dataset and compare plans.

---

## 3.12 Normalization

### Beginner

Normalization organizes data to reduce avoidable duplication and inconsistency.

### Practical explanation

Suppose every payment stores the merchant's name and address. A merchant address
change could require updating thousands of payment rows.

A normalized design stores the merchant separately and references its identifier.

### Useful normal forms

- First normal form: Use well-defined attribute values rather than repeating
  column groups.
- Second normal form: With a composite candidate key, non-key attributes should
  not depend on only part of that key.
- Third normal form: Avoid non-key attributes depending transitively on a key
  through another non-key attribute.

### Production nuance

Historical snapshots can be intentional.

For example, the invoice address used at the time of purchase may need to remain
unchanged even if the merchant's current address changes.

### Interview expectation

Explain the invariant and access pattern. Do not say "always normalize" or
"always denormalize."

---

## 3.13 ACID and transactions

### Atomicity

A transaction commits its changes together or aborts them.

### Consistency

Declared constraints and correctly implemented application invariants remain
valid. The database does not automatically know every business rule.

### Isolation

Concurrent transactions interact according to the selected isolation level.

### Durability

Committed data is preserved according to the database's configured durability
guarantees and failure model.

### Example transaction

```sql name=payment_status_transaction.sql
BEGIN;

UPDATE payments
SET status = 'succeeded'
WHERE payment_id = 104
  AND status = 'pending';

-- In an application, verify the affected-row count and record the
-- corresponding event/outbox entry within the same transaction.

COMMIT;
```

This demonstrates conditional state change, not a complete event-processing
implementation.

### Isolation concepts

- Dirty read: Reading another transaction's uncommitted changes.
- Non-repeatable read: Re-reading a row and seeing a committed change.
- Phantom: Repeating a predicate query and seeing a changed matching set.
- Serialization anomaly: A concurrent result that cannot be explained by a
  valid serial execution.

Database implementations differ. Learn the actual behavior of your chosen
engine rather than assuming isolation-level names are identical everywhere.

### Production warning

A local database transaction cannot automatically roll back an already accepted
external payment-provider operation.

Use explicit state, idempotency, and reconciliation across that boundary.

---

## 3.14 Database comparison

### PostgreSQL

**Beginner:** A relational database with SQL, constraints, and transactions.

**Industry use:** Transactional applications requiring expressive queries and
relational integrity.

**Practice:** Use it for this guide's payment and reconciliation lab.

**Mistake:** Running large analytical scans on the transactional primary without
checking impact.

**Interview follow-up:** How would you separate reporting from operational
traffic while accounting for replica lag?

### MySQL

**Beginner:** A relational database widely used for application workloads.

**Industry:** With InnoDB, analyze transactions, indexes, locking, and isolation
using MySQL's actual behavior.

**Practice:** Port the payment schema and document syntax differences.

**Mistake:** Copying PostgreSQL-specific SQL unchanged.

**Interview follow-up:** How does your chosen storage engine affect guarantees?

### Oracle

**Beginner:** A relational database used in enterprise systems.

**Industry:** Understand its transaction behavior, read consistency, tooling,
and operational environment rather than treating it as interchangeable SQL.

**Practice:** Rewrite pagination and date expressions for the target Oracle
version.

**Mistake:** Assuming DDL and transaction behavior match PostgreSQL.

**Interview follow-up:** What could an implicit commit mean for a migration?

### DynamoDB

**Beginner:** An AWS-managed key-value and document database.

**Industry:** Start with access patterns and partition-key design. Use
conditional writes for appropriate invariants and transactions when needed.

**Practice design:**

```text name=dynamodb_payment_keys.txt
PK = MERCHANT#<merchant_id>
SK = PAYMENT#<payment_id>

Alternative high-volume design:
Partition keys may need distribution beyond a single large merchant.

Choose the key from actual access patterns and traffic distribution.
```

**Important:** Tables and local secondary indexes can support strongly
consistent reads; global secondary index reads are eventually consistent.

**Mistake:** Assuming a successful write must immediately appear in a GSI query.

**Follow-up:** How would you read authoritative status immediately after a write?

### Neptune

**Beginner:** An AWS graph database for connected data.

**Industry:** Model relationships when traversals are central to the problem.

**Example:** Investigate accounts connected by shared devices, addresses,
or payment instruments.

```text name=fraud_graph.txt
Account A ----uses----> Device X <----uses---- Account B
    |                                           |
 pays_to                                     pays_to
    |                                           |
    +--------------> Merchant M <---------------+
```

**Query languages:** Property graphs can use Gremlin or openCypher; RDF data
uses SPARQL.

**Mistake:** Selecting a graph database solely because the data has relationships.
Relational databases also represent relationships effectively.

**Follow-up:** Which query becomes simpler or more efficient with the graph model?

### MongoDB

**Beginner:** A database storing document-shaped records.

**Industry:** Choose embedding and references based on update boundaries,
document growth, and access patterns.

**Example:** An integration configuration document with provider-specific options.

**Important:** MongoDB supports multi-document transactions in supported
deployments. "NoSQL means no transactions" is incorrect.

**Mistake:** Allowing unbounded arrays to grow inside one document.

**Follow-up:** When should a child collection replace embedding?

---

## 3.15 Thirty database interview questions

### Q1. WHERE versus HAVING?

**Answer:** WHERE filters rows before grouping; HAVING filters groups after
aggregation.

**Expect:** A query example.

**Follow-up:** Can both appear in one query? Yes.

### Q2. INNER JOIN versus LEFT JOIN?

**Answer:** INNER JOIN keeps matching combinations. LEFT JOIN preserves every
left-side row and supplies NULLs when no right-side match exists.

**Expect:** Explain missing relationships.

**Follow-up:** How can a WHERE predicate on the right table remove unmatched rows?

### Q3. COUNT(*) versus COUNT(column)?

**Answer:** COUNT(*) counts rows. COUNT(column) counts non-NULL values.

**Expect:** Apply this to a left join.

**Follow-up:** What does COUNT(DISTINCT column) measure?

### Q4. What is NULL?

**Answer:** NULL represents missing or unknown information. Use IS NULL rather
than equality comparison to test for it.

**Expect:** Awareness of three-valued logic.

**Follow-up:** Why can NOT IN behave unexpectedly when its input contains NULL?

### Q5. What is a primary key?

**Answer:** A chosen unique, non-NULL row identifier enforced by the database.

**Expect:** Separate identity from business uniqueness.

**Follow-up:** Why might an idempotency key need a separate unique constraint?

### Q6. What is a foreign key?

**Answer:** A constraint connecting referencing values to valid referenced
values, subject to its defined update and deletion behavior.

**Expect:** Referential integrity.

**Follow-up:** Should deleting a merchant cascade into payment history?

Discuss retention and business requirements rather than assuming yes.

### Q7. What is a window function?

**Answer:** It calculates across related rows while preserving individual rows,
unlike ordinary grouping that collapses rows.

**Expect:** Partition and ordering.

**Follow-up:** What is a window frame?

### Q8. ROW_NUMBER versus RANK versus DENSE_RANK?

**Answer:** ROW_NUMBER assigns positions. RANK shares ranks for ties with gaps.
DENSE_RANK shares ranks without gaps.

**Expect:** Explain what "top three" means under ties.

**Follow-up:** How do you make ROW_NUMBER deterministic?

Add a suitable tie-breaker.

### Q9. How do you retrieve the latest record per payment?

**Answer:** Use a window function ordered by a defined event sequence or timestamp
plus a tie-breaker, then select the first row.

**Expect:** Distinguish ordering from authoritative business state.

**Follow-up:** What if provider timestamps are unreliable?

### Q10. What is an index?

**Answer:** An auxiliary structure that can accelerate selected access patterns
at a storage and write-maintenance cost.

**Expect:** A read/write trade-off.

**Follow-up:** Why might a sequential scan be faster?

### Q11. How do you select composite index order?

**Answer:** Start with actual predicates, ordering, selectivity, and supported
access paths. Equality filters followed by range/order columns are a useful
starting point, not a universal rule.

**Expect:** Workload-based reasoning.

**Follow-up:** Can the same index support several queries?

### Q12. What does EXPLAIN show?

**Answer:** The planned execution strategy and estimates. With ANALYZE, it also
executes and reports observed behavior.

**Expect:** Safety warning and estimate-versus-actual comparison.

**Follow-up:** What might a large row-count estimation error suggest?

### Q13. What is normalization?

**Answer:** Organizing dependencies to reduce undesirable duplication and
update anomalies.

**Expect:** An example, not only normal-form names.

**Follow-up:** When is denormalization justified?

### Q14. What does ACID mean?

**Answer:** Atomicity, consistency, isolation, and durability. Explain each in
terms of a transaction and its configured guarantees.

**Expect:** Do not claim consistency means the database invents business rules.

**Follow-up:** Which guarantee prevents partial local updates?

### Q15. What is an isolation level?

**Answer:** A specification of permitted interaction between concurrent
transactions. Stronger isolation can require additional coordination or retries.

**Expect:** Actual engine behavior.

**Follow-up:** Does serializable mean every transaction runs one at a time?

No; implementations can execute concurrently and reject unsafe outcomes.

### Q16. What is a deadlock?

**Answer:** Transactions wait cyclically on resources held by each other.
The database may abort one participant to break the cycle.

**Expect:** Consistent lock ordering and bounded transaction retries.

**Follow-up:** Should you retry only the last statement?

Usually retry the complete transaction under the application's retry policy.

### Q17. Optimistic versus pessimistic concurrency?

**Answer:** Optimistic control detects conflicts using versions or conditions.
Pessimistic control locks relevant resources before conflicting work proceeds.

**Expect:** Contention and workflow trade-offs.

**Follow-up:** How would you prevent a stale status update?

### Q18. How do you prevent duplicate payment creation?

**Answer:** Use a stable, appropriately scoped idempotency key, an atomic
uniqueness mechanism, payload-consistency checks, and durable result tracking.

**Expect:** Concurrency safety, not a check-then-insert race.

**Follow-up:** What if the same key arrives with a different amount?

Reject it according to the API contract.

### Q19. Why can JOINs inflate totals?

**Answer:** A one-to-many join repeats parent values. Summing the parent amount
after joining multiple child rows can overcount.

**Expect:** Pre-aggregation or a query matching the intended grain.

**Follow-up:** Why is SUM(DISTINCT amount) not a general solution?

Different valid payments can have the same amount.

### Q20. OFFSET versus keyset pagination?

**Answer:** OFFSET skips a number of rows and can become costly at depth.
Keyset pagination continues after a stable ordered key.

**Expect:** Tie-breakers and behavior under concurrent changes.

**Follow-up:** How do you implement arbitrary page jumps?

### Q21. What is replication lag?

**Answer:** The delay between a change at the source and its visibility at a
replica or downstream view.

**Expect:** Read-after-write implications.

**Follow-up:** Which reads should use an authoritative source?

### Q22. What is the N+1 query problem?

**Answer:** One query loads parent records, then one additional query is made for
each parent. The repeated round trips can dominate latency.

**Expect:** Batching or suitable joins without overfetching.

**Follow-up:** Can one giant join create other problems?

Yes: row multiplication, larger payloads, and expensive plans.

### Q23. How do you prevent SQL injection?

**Answer:** Bind values using the database driver's parameter mechanism.
Do not construct SQL by concatenating untrusted values.

**Expect:** Distinguish values from identifiers.

**Follow-up:** How do you safely choose a sort column?

Use a strict allowlist or the driver's identifier composition facilities.

### Q24. How do you represent money?

**Answer:** Use integer minor units or a defined fixed-precision decimal model,
always accompanied by currency and explicit rounding rules.

**Expect:** Currency-specific scale and overflow limits.

**Follow-up:** Why is binary floating point problematic for exact accounting?

### Q25. Can NoSQL databases support transactions?

**Answer:** Yes. Evaluate the specific database, deployment, transaction scope,
and limitations.

**Expect:** Avoid SQL-versus-NoSQL absolutes.

**Follow-up:** What does DynamoDB provide for multi-item atomic operations?

### Q26. Query versus Scan in DynamoDB?

**Answer:** Query uses key-based access conditions. Scan examines items across
a table or index and then applies filtering as configured.

**Expect:** Access-pattern design and capacity impact.

**Follow-up:** Why does a filter not necessarily make a scan cheap?

### Q27. When would you use Neptune?

**Answer:** When relationship traversal is central, such as exploring linked
accounts and devices, and the graph model fits the required queries.

**Expect:** A concrete traversal.

**Follow-up:** Why not use joins in PostgreSQL?

Compare depth, query shape, scale, and operational complexity.

### Q28. When would you use MongoDB embedding?

**Answer:** When related data is commonly read together, has a compatible
lifecycle, and remains within practical document-growth limits.

**Expect:** Update boundaries and bounded growth.

**Follow-up:** What if child records are independently queried and updated?

### Q29. How do you investigate database CPU at 100%?

**Answer:** Identify active work and recent changes, inspect expensive queries,
concurrency, execution plans, and resource saturation. Distinguish CPU-heavy
work from lock or I/O waiting. Mitigate safely and validate recovery.

**Expect:** Evidence before scaling or killing sessions.

**Follow-up:** What would you check after an application deployment?

### Q30. How do you migrate a production schema safely?

**Answer:** Prefer compatible expansion, controlled backfill, application
transition, validation, and later contraction. Account for locks, mixed
application versions, rollback, and data volume.

**Expect:** More than executing ALTER TABLE.

**Follow-up:** What if old and new application versions run simultaneously?

---

## 3.16 Hands-on exercises

1. Run the schema and verify row counts.
2. Return merchants with zero payments.
3. Calculate successful volume by merchant and currency.
4. Calculate refund totals without duplicating payment amounts.
5. Retrieve the latest recorded event per payment.
6. Implement keyset pagination.
7. Generate a larger dataset and inspect query plans.
8. Compare performance before and after a candidate index.
9. Open two sessions and demonstrate a conflicting update.
10. Propose a schema migration that permits old and new application versions.

## 3.17 Scenario: dashboard totals doubled after a release

### Investigation

1. Compare the previous and new SQL.
2. Identify the intended row grain.
3. Inspect join cardinality.
4. Check whether a new one-to-many table was joined.
5. Reconcile a small set of payments manually.
6. Validate whether stored data or only the report is wrong.

### Illustrative root cause

The query joins payments to refunds and sums the original payment amount.
Payments with multiple refund records contribute multiple times.

### Resolution

Pre-aggregate refunds by payment before joining, or calculate each measure in
a query at its correct grain.

### Prevention

- Include one-to-many fixtures in tests.
- Verify totals against independent reconciliation.
- Review query grain during code review.

### Interview follow-ups

- Why would SUM(DISTINCT payment_amount) still be wrong?
- Could the issue affect only certain merchants?
- How would you explain the incident to a non-technical customer?

## 3.18 Cheat sheet

```text name=sql_cheatsheet.txt
Filter rows                  -> WHERE
Filter groups                -> HAVING
Keep unmatched left rows     -> LEFT JOIN
Find missing related rows    -> NOT EXISTS
Aggregate without collapsing -> window function
Latest row                   -> ROW_NUMBER + deterministic order
Prevent duplicate business ID-> UNIQUE constraint
Atomic local changes         -> transaction
Investigate plan             -> EXPLAIN
Measure execution            -> EXPLAIN ANALYZE, with caution
Deep pagination              -> consider keyset pagination
Payment totals               -> group by currency; verify join grain
```
