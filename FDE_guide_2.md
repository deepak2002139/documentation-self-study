---

# Module 2 — Python for FDE

## 2.1 Mental model

Python is useful for expressing integration logic, transforming data,
automating diagnostics, and writing services.

For this guide, think of a Python integration as a pipeline:

```text
Configuration
     |
     v
Read input -> Validate -> Transform -> Call dependency -> Record outcome
                 |                         |
                 v                         v
          Reject / quarantine        Classify failure
                                           |
                                           v
                                  Retry safely or stop
```

## 2.2 Core concepts

### Variables

**Beginner:** A variable is a name associated with a value.

**Industry:** Python names refer to objects. Assignment does not automatically
copy an object.

**Example:** Two variables referencing the same list observe mutations to that
list.

**Production use:** Configuration, request state, counters, and parsed records.

**Mistake:** Assuming assignment creates an independent nested data structure.

**Best practice:** Make ownership and mutation deliberate.

### Loops

**Beginner:** Repeat an operation over items or while a condition is true.

**Industry:** Loops need termination conditions and predictable resource use.

**Example:** Process each row in a settlement file.

**Production use:** Pagination, batch processing, bounded retry loops.

**Mistake:** An unbounded loop that repeatedly calls a failing service.

**Best practice:** Bound attempts, pages, work per batch, and runtime.

### Functions

**Beginner:** A named, reusable operation.

**Industry:** A good function has clear inputs, outputs, side effects, and
failure behavior.

**Example:** Convert a validated payment record to a provider request.

**Production use:** Keep validation separate from network calls.

**Mistake:** One function reads configuration, calls an API, writes files,
and silently handles every failure.

**Best practice:** Prefer small units that are easy to test independently.

### Object-oriented programming

**Beginner:** A class groups related data and behavior.

**Industry:** Use objects to encapsulate state and dependencies. Composition
often makes adapters easier to test than deep inheritance.

**Example:** A payment client contains a session and configuration.

**Mistake:** Putting request-specific mutable state on the class itself.

**Best practice:** Use instance attributes and inject dependencies.

### Exception handling

**Beginner:** Exceptions report operations that could not complete normally.

**Industry:** Catch exceptions where you can add context, recover, translate
the error, or release resources.

**Example:** Distinguish invalid input from an unavailable dependency.

**Mistake:** `except Exception: pass`.

**Best practice:** Catch specific exceptions and preserve the original cause
when translating them.

### File handling

**Beginner:** Open a file and read or write data.

**Industry:** Consider encoding, memory usage, malformed records, partial
writes, permissions, and restart behavior.

**Example:** Read a large JSON Lines file one record at a time.

**Mistake:** Loading a multi-gigabyte file into memory unnecessarily.

**Best practice:** Stream when possible and use context managers.

### API calls

**Beginner:** Send a request to another service and interpret its response.

**Industry:** Network operations need timeouts, authentication, response
validation, safe retries, and observability.

**Mistake:** Assuming valid JSON means the HTTP operation succeeded.

**Best practice:** Check status and validate the response structure.

### JSON processing

**Beginner:** Convert JSON text into Python data, or Python data into JSON.

**Industry:** Parsing is not schema validation.

**Example:** A JSON object can be syntactically valid while containing an
invalid amount or missing payment identifier.

**Mistake:** Treating `null`, a missing field, and an empty string as identical.

**Best practice:** Define their meanings in the contract.

### Logging

**Beginner:** Record useful events while a program runs.

**Industry:** Logs should support correlation and diagnosis without exposing
credentials or unnecessary personal data.

**Example:** Record event name, request ID, outcome, and duration.

**Mistake:** Logging entire request bodies or authorization headers.

**Best practice:** Use an allowlist of safe fields.

### Multithreading

**Beginner:** Let multiple threads make progress within one process.

**Industry:** Threads are useful for overlapping blocking I/O, but shared state
requires coordination.

In ordinary GIL-enabled CPython, threads do not generally parallelize
CPU-bound Python bytecode. Free-threaded builds and native extensions can
change the analysis; identify the actual runtime.

**Mistake:** Assuming every operation on shared data is safe because a GIL
exists.

**Best practice:** Avoid shared mutable state, bound concurrency, and inspect
worker exceptions.

### Async programming

**Beginner:** Let a task yield while waiting so another task can run.

**Industry:** Async I/O uses cooperative scheduling. Blocking work inside the
event loop can delay all tasks.

**Mistake:** Calling `time.sleep()` or a synchronous HTTP client inside an async
request handler.

**Best practice:** Use async-compatible libraries, cancellation, and bounded
concurrency.

---

## 2.3 Twenty Python coding examples

### Example 1 — Variables and a payment record

```python
payment_id = "pay_001"
amount_minor = 12500
currency = "INR"
successful = True

payment = {
    "payment_id": payment_id,
    "amount_minor": amount_minor,
    "currency": currency,
    "successful": successful,
}

print(payment)
```

**Production connection:** Preserve amount and currency together.

**Practice:** Add a merchant identifier and validate that it is non-empty.

---

### Example 2 — Loops and filtering

```python
payments = [
    {"id": "p1", "status": "succeeded", "amount_minor": 1000},
    {"id": "p2", "status": "failed", "amount_minor": 2000},
    {"id": "p3", "status": "succeeded", "amount_minor": 3000},
]

total = 0

for payment in payments:
    if payment["status"] == "succeeded":
        total += payment["amount_minor"]

assert total == 4000
print(total)
```

**Assumption:** Every amount belongs to the same currency.

**Mistake:** Adding USD, INR, and EUR amounts together.

---

### Example 3 — Functions and validation

```python
def validate_amount(amount_minor: int) -> int:
    if type(amount_minor) is not int:
        raise TypeError("amount_minor must be an integer")

    if amount_minor <= 0:
        raise ValueError("amount_minor must be positive")

    return amount_minor


assert validate_amount(100) == 100

try:
    validate_amount(True)
except TypeError:
    print("Boolean rejected correctly")
```

**Important:** Type hints do not enforce validation at runtime.

---

### Example 4 — Find duplicates

```python
def find_duplicates(values):
    seen = set()
    duplicates = set()

    for value in values:
        if value in seen:
            duplicates.add(value)
        else:
            seen.add(value)

    return duplicates


assert find_duplicates([1, 2, 3, 2, 4, 1]) == {1, 2}
assert find_duplicates([]) == set()
```

**Brute force:** Compare each value with every other value: O(n²) time.

**This approach:** Expected O(n) time and O(n) additional space.

**Assumption:** Values are hashable. Output order is unspecified.

**Follow-up:** How would you preserve first duplicate occurrence order?

---

### Example 5 — Reverse a string

```python
def reverse_string(value: str) -> str:
    return value[::-1]


assert reverse_string("payment") == "tnemyap"
assert reverse_string("") == ""
```

**Complexity:** O(n) time and O(n) space for the result.

**Caveat:** This reverses Unicode code points, not necessarily user-perceived
characters containing combining marks or multi-code-point emoji.

---

### Example 6 — Count frequency

```python
from collections import Counter

statuses = ["success", "failed", "success", "pending", "failed", "success"]
counts = Counter(statuses)

assert counts["success"] == 3
print(counts)
```

**Complexity:** Expected O(n) time; O(k) space for k distinct values.

**Production connection:** Count failures by error category.

---

### Example 7 — Group payments by merchant

```python
from collections import defaultdict

payments = [
    {"merchant_id": "m1", "amount_minor": 100},
    {"merchant_id": "m2", "amount_minor": 200},
    {"merchant_id": "m1", "amount_minor": 300},
]

totals = defaultdict(int)

for payment in payments:
    totals[payment["merchant_id"]] += payment["amount_minor"]

assert dict(totals) == {"m1": 400, "m2": 200}
```

**Extension:** Group by `(merchant_id, currency)`.

---

### Example 8 — Parse and serialize JSON

```python
import json

raw = '{"payment_id": "p1", "amount_minor": 1000, "currency": "USD"}'
payment = json.loads(raw)

assert isinstance(payment, dict)
assert payment["amount_minor"] == 1000

serialized = json.dumps(payment, sort_keys=True)
print(serialized)
```

**Distinction:**

- `loads`: JSON string to Python object.
- `dumps`: Python object to JSON string.
- `load`: Read JSON from a file-like object.
- `dump`: Write JSON to a file-like object.

---

### Example 9 — Validate parsed JSON

```python
import json

ALLOWED_CURRENCIES = {"USD", "INR", "EUR"}


def parse_payment(raw: str) -> dict:
    payment = json.loads(raw)

    if not isinstance(payment, dict):
        raise ValueError("Expected a JSON object")

    payment_id = payment.get("payment_id")
    amount = payment.get("amount_minor")
    currency = payment.get("currency")

    if not isinstance(payment_id, str) or not payment_id:
        raise ValueError("Invalid payment_id")

    if type(amount) is not int or amount <= 0:
        raise ValueError("Invalid amount_minor")

    if currency not in ALLOWED_CURRENCIES:
        raise ValueError("Unsupported currency")

    return payment


print(parse_payment(
    '{"payment_id":"p1","amount_minor":100,"currency":"USD"}'
))
```

**Production extension:** Limit input size before parsing and define field
lengths, allowed values, and schema version.

---

### Example 10 — Specific exceptions and exception chaining

```python
import json


class InvalidConfiguration(Exception):
    pass


def read_configuration(raw: str) -> dict:
    try:
        configuration = json.loads(raw)
    except json.JSONDecodeError as exc:
        raise InvalidConfiguration("Configuration is not valid JSON") from exc

    if not isinstance(configuration, dict):
        raise InvalidConfiguration("Configuration must be an object")

    return configuration


try:
    read_configuration("{bad json}")
except InvalidConfiguration as exc:
    print(str(exc))
```

**Production connection:** Translate a low-level parser failure into a meaningful
configuration error without losing its cause.

---

### Example 11 — Read a large JSON Lines file

```python
import json


def iter_json_records(path):
    with open(path, "r", encoding="utf-8") as handle:
        for line_number, line in enumerate(handle, start=1):
            if not line.strip():
                continue

            try:
                yield line_number, json.loads(line)
            except json.JSONDecodeError as exc:
                raise ValueError(
                    f"Malformed JSON on line {line_number}"
                ) from exc


if __name__ == "__main__":
    with open("payments.jsonl", "w", encoding="utf-8") as handle:
        handle.write('{"id":"p1","status":"succeeded"}\n')
        handle.write('{"id":"p2","status":"failed"}\n')

    for number, record in iter_json_records("payments.jsonl"):
        print(number, record)
```

**Memory:** Approximately bounded by the largest line and parsed record, not
the entire file, provided the caller does not accumulate all results.

**Production extension:** Enforce a maximum line length and decide whether a
malformed record stops processing or enters a quarantine workflow.

---

### Example 12 — Parse a CSV settlement file

```python
import csv
import io

data = """payment_id,amount_minor,currency
p1,1000,USD
p2,2500,USD
"""

totals = {}

for row in csv.DictReader(io.StringIO(data)):
    amount = int(row["amount_minor"])
    currency = row["currency"]

    if amount <= 0:
        raise ValueError("Expected a positive settlement amount")

    totals[currency] = totals.get(currency, 0) + amount

assert totals == {"USD": 3500}
print(totals)
```

**Production extension:** Handle duplicate references, malformed headers,
unexpected encodings, and reconciliation totals.

---

### Example 13 — Structured logging with safe fields

```python
import json
import logging

logging.basicConfig(level=logging.INFO, format="%(message)s")
logger = logging.getLogger("payment_integration")


def log_outcome(request_id, outcome, duration_ms):
    event = {
        "event": "provider_request_completed",
        "request_id": request_id,
        "outcome": outcome,
        "duration_ms": duration_ms,
    }
    logger.info(json.dumps(event))


log_outcome("request_demo_001", "success", 145)
```

**Do not add:** Authorization headers, card numbers, passwords, or complete
customer payloads.

**Exercise:** Add a safe error category without logging the raw provider body.

---

### Example 14 — OOP through dependency injection

```python
from dataclasses import dataclass
from typing import Protocol


class Provider(Protocol):
    def get_status(self, payment_id: str) -> str:
        ...


@dataclass
class PaymentService:
    provider: Provider

    def is_complete(self, payment_id: str) -> bool:
        return self.provider.get_status(payment_id) == "succeeded"


class FakeProvider:
    def get_status(self, payment_id: str) -> str:
        return "succeeded"


service = PaymentService(provider=FakeProvider())
assert service.is_complete("p1")
```

**Production connection:** Replace the fake with an HTTP adapter without changing
the business rule.

**Follow-up:** How would you test provider timeouts?

---

### Example 15 — A bounded HTTP request

```python
import os
import requests


def get_payment(session, base_url, token, payment_id):
    response = session.get(
        f"{base_url.rstrip('/')}/payments/{payment_id}",
        headers={"Authorization": f"Bearer {token}"},
        timeout=(3.05, 10),
        allow_redirects=False,
    )

    if response.status_code != 200:
        raise RuntimeError(f"Unexpected HTTP status: {response.status_code}")

    payload = response.json()

    if not isinstance(payload, dict) or "id" not in payload:
        raise ValueError("Unexpected response structure")

    return payload


if __name__ == "__main__":
    # Use a trusted configuration value, not an arbitrary customer-supplied URL.
    base_url = os.environ["PAYMENTS_BASE_URL"]
    token = os.environ["PAYMENTS_API_TOKEN"]

    with requests.Session() as session:
        print(get_payment(session, base_url, token, "payment_demo"))
```

**Important:**

- Requests verifies TLS certificates by default.
- The tuple configures connect and read timeouts.
- These are not a complete end-to-end wall-clock deadline.
- Validate or encode identifiers before inserting arbitrary values into paths.
- Do not print real payment payloads in a production script.

---

### Example 16 — Bounded retries for a safe read

```python
import random
import time
import requests


def get_with_retry(session, url, attempts=3):
    if attempts < 1:
        raise ValueError("attempts must be positive")

    for attempt in range(attempts):
        try:
            response = session.get(
                url,
                timeout=(3.05, 5),
                allow_redirects=False,
            )
        except requests.exceptions.SSLError:
            # Certificate failures require investigation, not blind retries.
            raise
        except (requests.Timeout, requests.ConnectionError):
            if attempt == attempts - 1:
                raise
        else:
            if response.status_code not in {502, 503, 504}:
                response.raise_for_status()
                return response

            if attempt == attempts - 1:
                response.raise_for_status()

            response.close()

        delay = random.uniform(0, min(2.0, 0.25 * (2 ** attempt)))
        time.sleep(delay)

    raise RuntimeError("Unreachable")
```

**Scope:** A teaching retry policy for GET.

**Not implemented:** A total deadline, provider-specific rate limiting,
`Retry-After` handling, circuit breaking, or retry metrics.

**Never generalize this automatically to payment-creation POST requests.**

---

### Example 17 — Cursor pagination with a page limit

```python
def iter_payments(session, url, headers, max_pages=100):
    cursor = None
    seen_cursors = set()

    for _ in range(max_pages):
        params = {"limit": 100}
        if cursor is not None:
            params["cursor"] = cursor

        response = session.get(
            url,
            headers=headers,
            params=params,
            timeout=(3.05, 10),
            allow_redirects=False,
        )

        if response.status_code != 200:
            raise RuntimeError(f"Unexpected HTTP status: {response.status_code}")

        payload = response.json()

        if not isinstance(payload, dict):
            raise ValueError("Expected response object")

        items = payload.get("items")
        if not isinstance(items, list):
            raise ValueError("Expected items list")

        yield from items

        cursor = payload.get("next_cursor")

        if cursor is None:
            return

        if not isinstance(cursor, str) or not cursor:
            raise ValueError("Invalid cursor")

        if cursor in seen_cursors:
            raise ValueError("Repeated pagination cursor")

        seen_cursors.add(cursor)

    raise RuntimeError("Pagination limit exceeded")
```

**Production connection:** Avoid an endless export if a provider returns the
same cursor repeatedly.

**Security detail:** The request URL stays fixed; the script does not follow an
arbitrary next-page URL carrying credentials to another host.

---

### Example 18 — Replace a local checkpoint file

```python
import json
import os
import tempfile
from pathlib import Path


def write_checkpoint(path, state):
    target = Path(path)
    target.parent.mkdir(parents=True, exist_ok=True)
    temporary = None

    try:
        with tempfile.NamedTemporaryFile(
            mode="w",
            encoding="utf-8",
            dir=target.parent,
            prefix=f".{target.name}.",
            delete=False,
        ) as handle:
            temporary = Path(handle.name)
            json.dump(state, handle)
            handle.flush()
            os.fsync(handle.fileno())

        os.replace(temporary, target)
    finally:
        if temporary is not None and temporary.exists():
            temporary.unlink()


write_checkpoint("checkpoint.json", {"last_completed_page": 7})
```

**Assumption:** A single writer and filesystem semantics supporting the intended
same-filesystem replacement behavior.

**Caveat:** Atomic replacement is not a complete distributed coordination or
crash-durability protocol. Directory syncing and filesystem-specific behavior
matter for stronger durability requirements.

**Critical design question:** Is the checkpoint advanced before or after the
business operation is durably complete?

---

### Example 19 — Bounded thread workers

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time


def simulated_status_check(payment_id):
    time.sleep(0.05)  # Simulated blocking I/O.
    return {"payment_id": payment_id, "status": "succeeded"}


payment_ids = ["p1", "p2", "p3", "p4"]

with ThreadPoolExecutor(max_workers=2) as executor:
    futures = {
        executor.submit(simulated_status_check, payment_id): payment_id
        for payment_id in payment_ids
    }

    for future in as_completed(futures):
        payment_id = futures[future]
        try:
            print(future.result())
        except Exception as exc:
            # In production, classify the failure and return a non-success
            # batch outcome when required.
            print(payment_id, type(exc).__name__)
```

**Production caveat:** A fixed worker count does not automatically bound the
number of queued futures. For millions of inputs, submit work in bounded batches
or use a bounded producer-consumer queue.

---

### Example 20 — Async tasks with bounded concurrency

```python
import asyncio


async def simulated_status_check(payment_id, semaphore):
    async with semaphore:
        await asyncio.sleep(0.05)
        return {"payment_id": payment_id, "status": "succeeded"}


async def main():
    semaphore = asyncio.Semaphore(2)
    payment_ids = ["p1", "p2", "p3", "p4"]

    async with asyncio.timeout(2):
        async with asyncio.TaskGroup() as group:
            tasks = [
                group.create_task(
                    simulated_status_check(payment_id, semaphore)
                )
                for payment_id in payment_ids
            ]

    print([task.result() for task in tasks])


if __name__ == "__main__":
    asyncio.run(main())
```

**Production extension:** Use an async HTTP client and reuse its connection pool.

**Caveat:** A semaphore bounds concurrent operations, not the number of tasks
allocated. Large inputs need bounded task creation too.

---

## 2.4 Twenty coding exercises

Do these without looking at the examples.

| # | Exercise | Acceptance criteria |
|---|---|---|
| 1 | Find duplicate payment IDs | Empty input works; each duplicate appears once |
| 2 | Reverse a string | Explain the Unicode limitation |
| 3 | Count error-code frequencies | Missing codes handled explicitly |
| 4 | Group amounts by merchant and currency | Never combine currencies |
| 5 | Validate payment JSON | Reject booleans as amounts |
| 6 | Read a large JSONL file | Do not load the complete file |
| 7 | Parse settlement CSV | Report malformed line numbers |
| 8 | Compare two payment-ID collections | Return missing and unexpected IDs |
| 9 | Remove duplicates while preserving order | Expected linear time |
| 10 | Find the most frequent failure category | Define tie behavior |
| 11 | Build a GET API client | Timeouts and response validation |
| 12 | Implement safe read retries | Bounded attempts and jitter |
| 13 | Implement cursor pagination | Detect repeated cursors |
| 14 | Generate structured logs | Use an allowlist of fields |
| 15 | Implement a provider interface | Test using a fake dependency |
| 16 | Run ten blocking checks concurrently | At most three active workers |
| 17 | Run ten async checks concurrently | At most three active requests |
| 18 | Save a checkpoint safely | Do not advance it before completion |
| 19 | Reconcile local and provider states | Produce a report before changing data |
| 20 | Write tests for the reconciliation job | Include duplicates, missing data, and timeouts |

### Suggested test matrix

For each exercise, consider:

- Empty input.
- One record.
- Duplicate records.
- Missing fields.
- Wrong types.
- Unexpected encoding.
- Large input.
- Dependency timeout.
- Partial success.
- Restart after interruption.

---

## 2.5 Twenty Python interview questions

### Q1. List, tuple, set, and dictionary: when do you use each?

**Answer:** Use a list for an ordered mutable collection, a tuple for a fixed
record-like grouping, a set for uniqueness and membership, and a dictionary
for key-to-value lookup. Select based on operations, not habit.

**Expect:** A use case and complexity discussion.

**Follow-up:** Can a tuple be a dictionary key?

Only if all its contents are hashable.

### Q2. What is the difference between mutable and immutable objects?

**Answer:** A mutable object's contents can change without replacing the object.
Lists and dictionaries are mutable; strings and integers are immutable.
Aliasing mutable objects can create unexpected shared changes.

**Expect:** An example involving two references to one list.

**Follow-up:** Is a tuple containing a list fully immutable?

The tuple's references cannot change, but the referenced list can mutate.

### Q3. Why are mutable default arguments dangerous?

**Answer:** Default argument expressions are evaluated when the function is
defined, so a default list can be shared across calls. Use `None` and construct
a new collection inside the function when that is the intended behavior.

**Expect:** Explanation of lifetime, not just a memorized rule.

**Follow-up:** When might a shared default be intentional?

A deliberate cache is possible, but explicit state is usually clearer.

### Q4. What is a generator?

**Answer:** A generator produces values incrementally. It can avoid keeping an
entire dataset in memory and supports pipelines such as reading and validating
records one at a time.

**Expect:** Mention that downstream accumulation can eliminate the memory benefit.

**Follow-up:** Can you iterate an exhausted generator again?

Not without creating another generator or otherwise recreating the source.

### Q5. How do you read a 10 GB file?

**Answer:** First determine the format. For line-oriented data, iterate through
the file and process bounded records. Track progress, handle malformed lines,
and avoid collecting every result. A single giant JSON array may require an
incremental parser rather than line iteration.

**Expect:** Format awareness, memory reasoning, and restart behavior.

**Follow-up:** What if a single line is extremely large?

Enforce record-size limits or use a parser with appropriate streaming limits.

### Q6. What is a context manager?

**Answer:** It defines setup and cleanup around a block. File handles and network
clients can be closed reliably even when an exception occurs.

**Expect:** Resource lifecycle reasoning.

**Follow-up:** Does a context manager automatically make writes transactional?

No. That depends on the resource and its implementation.

### Q7. What is the GIL?

**Answer:** In ordinary GIL-enabled CPython, the global interpreter lock restricts
concurrent execution of Python bytecode by threads. Threads can still overlap
blocking I/O. CPU-bound execution may require processes or suitable native
code; free-threaded builds require runtime-specific analysis.

**Expect:** Avoid the absolute claim "Python cannot run in parallel."

**Follow-up:** Does the GIL prevent application race conditions?

No.

### Q8. Threads or async for API integration?

**Answer:** Threads fit blocking libraries and moderate concurrency. Async fits
an async-compatible stack with many waiting operations. Both need timeouts,
bounded concurrency, and careful failure handling.

**Expect:** Workload and library constraints.

**Follow-up:** What if an async handler must call a blocking library?

Use an appropriate executor or `asyncio.to_thread`, with limits and awareness
that cancelling the await does not necessarily stop the underlying thread.

### Q9. Concurrency versus parallelism?

**Answer:** Concurrency concerns overlapping progress; parallelism concerns
simultaneous execution. A single event loop can provide concurrency while
waiting for I/O without executing Python tasks simultaneously.

**Expect:** A concrete network-I/O example.

**Follow-up:** Which helps a CPU-heavy compression task?

Usually parallel execution or optimized native code, subject to measurement.

### Q10. How do you handle exceptions in a batch integration?

**Answer:** Separate invalid records from transient dependency failures and
programming errors. Decide whether each error stops the batch, retries, or
quarantines a record. Preserve a summary and meaningful exit status.

**Expect:** No silent failure.

**Follow-up:** Should you catch every exception at the top level?

A top-level boundary can log and terminate cleanly, but should not claim success.

### Q11. How do you find duplicates efficiently?

**Answer:** Maintain a set of seen values and another collection for duplicates.
This has expected linear time for hashable values, with additional memory
proportional to distinct values.

**Expect:** Assumptions about hashing and output order.

**Follow-up:** What if the dataset exceeds memory?

Consider external sorting, partitioning, or a database-backed approach.

### Q12. Shallow copy versus deep copy?

**Answer:** A shallow copy creates a new outer container but retains references
to nested objects. A deep copy recursively copies supported nested objects.
Deep copying is not always correct for resources such as connections or locks.

**Expect:** Nested data example.

**Follow-up:** Can immutable structures simplify this?

Yes, by reducing unintended shared mutation.

### Q13. What is wrong with `except Exception: pass`?

**Answer:** It hides failures and can let downstream work proceed with invalid
assumptions. Catch specifically where possible and record or propagate a
meaningful outcome.

**Expect:** Connect silent failure to production impact.

**Follow-up:** Is catching `Exception` ever reasonable?

At an isolation boundary, provided failure remains visible and correctly handled.

### Q14. How do you test an API client?

**Answer:** Inject the transport or session and simulate success, invalid JSON,
wrong schema, authentication failure, rate limits, timeout, and server errors.
Verify retry count, timeout configuration, and that secrets are not logged.

**Expect:** Deterministic tests without real charges.

**Follow-up:** What should integration tests add?

Validation against a sandbox or controlled server using real protocol behavior.

### Q15. How would you implement retries?

**Answer:** Retry only failures and operations that are safe under the contract.
Use bounded attempts, backoff, jitter, and a total budget where supported.
Respect provider guidance and expose retry metrics.

**Expect:** Idempotency and overload awareness.

**Follow-up:** Why can retries worsen an outage?

They increase load on an already struggling dependency.

### Q16. How do you protect credentials in automation?

**Answer:** Obtain them from an approved secret source, use least privilege,
avoid command-line exposure where possible, redact logs, and rotate credentials.
Do not commit secrets to source control.

**Expect:** Lifecycle and operational access controls.

**Follow-up:** Are environment variables a complete secret-management solution?

No. Their exposure and lifecycle depend on the runtime and operational controls.

### Q17. How do you make a script restartable?

**Answer:** Identify completed work durably, use stable identifiers, and design
processing to tolerate replay. Coordinate checkpoint advancement with durable
completion or make duplicate processing harmless.

**Expect:** Recognition of the gap between side effect and checkpoint.

**Follow-up:** What happens if the process crashes after the side effect?

Recovery may replay it, so deduplication or reconciliation is required.

### Q18. How do you improve a slow Python job?

**Answer:** Measure first. Separate CPU time, network waiting, database waiting,
and serialization. Remove redundant work, batch operations, select better
algorithms, and introduce bounded concurrency only where useful.

**Expect:** Profiling before rewriting.

**Follow-up:** Why might more threads make it slower?

Contention, scheduling overhead, rate limits, or downstream saturation.

### Q19. Why use structured logs?

**Answer:** Consistent fields make filtering, aggregation, and correlation easier.
A request identifier and outcome are more useful than an unstructured sentence
that changes between versions.

**Expect:** Safe fields and correlation.

**Follow-up:** Should payment IDs be metric labels?

Usually not; their high cardinality is better suited to controlled logs or traces.

### Q20. How do you make automation production-ready?

**Answer:** Add configuration validation, timeouts, safe retries, tests,
structured logs, meaningful exit codes, bounded resource use, access controls,
and a runbook. Establish ownership and a controlled execution process.

**Expect:** More than "put it in a cron job."

**Follow-up:** What is your first production rollout?

A limited, observable run with a dry-run or report-only mode where feasible.

---

## 2.6 Production automation project: payment reconciliation

### Problem

Local payment status and provider status may disagree after delayed or missed
notifications.

### Proposed workflow

```text
Load configuration
       |
       v
Read bounded page of candidate payments
       |
       v
Fetch provider status with controlled concurrency
       |
       v
Normalize and compare
       |
       +---- Consistent ----> Record checked
       |
       +---- Different -----> Record discrepancy
       |
       +---- Unknown -------> Preserve for retry / investigation
       |
       v
Write report
       |
       v
Checkpoint safely
```

### Initial implementation scope

Build report-only behavior first.

A report record could contain:

```json
{
  "payment_id": "payment_demo_001",
  "local_status": "pending",
  "provider_status": "succeeded",
  "classification": "status_mismatch",
  "proposed_action": "review_or_apply_validated_transition"
}
```

### Required safeguards

- Do not convert a timeout into a failed payment.
- Do not issue a second charge to resolve uncertainty.
- Do not overwrite a newer state with stale information.
- Store enough evidence to explain a discrepancy.
- Keep a bounded retry queue.
- Define how concurrent reconciliation runs are coordinated.
- Separate report creation from approved repair actions.

### Acceptance tests

1. Matching status creates no repair.
2. A provider timeout remains unknown.
3. Duplicate input does not create duplicate repairs.
4. Restart resumes safely.
5. Unsupported states are escalated, not guessed.
6. Reports exclude sensitive payment credentials.

## 2.7 Common mistakes and best practices

| Mistake | Better approach |
|---|---|
| No timeout | Explicit connect/read limits and an overall budget where needed |
| Retry every exception | Classify the failure and operation |
| Unlimited threads/tasks | Bound active and queued work |
| `json.loads` treated as validation | Validate shape, fields, types, and limits |
| Float used for payment amounts | Integer minor units or a defined decimal model |
| Entire payload logged | Allowlist diagnostic fields |
| Every result accumulated | Stream or batch |
| Checkpoint written too early | Advance only after durable completion |
| Script always exits successfully | Reflect partial or complete failure |
| Network tests charge real cards | Use fakes and approved sandboxes |

## 2.8 Cheat sheet

```text
Uniqueness                -> set
Keyed lookup              -> dict
Frequency                 -> collections.Counter
Grouping                  -> defaultdict
Incremental processing    -> generator
Resource cleanup          -> with
Expected input failure    -> specific exception
I/O concurrency           -> bounded threads or async
CPU work                  -> measure; consider processes/native code
API request               -> timeout + status check + schema validation
Production job            -> safe restart + logs + ownership + tests
```

## 2.9 Scenario interview

**Scenario:** A Python reconciliation script takes six hours to process
100,000 payments.

### Suggested answer

"I would measure where time is spent before selecting an optimization.
If requests are sequential and network-bound, I would test bounded concurrency
within provider limits. I would also check for a bulk endpoint, unnecessary
requests, pagination inefficiencies, and repeated authentication.

I would preserve timeouts and retry budgets, avoid overloading the provider,
and compare throughput and error rate before and after the change. The job
must remain restartable and must not change payment state merely because a
request timed out."

### Follow-ups

- What if the provider permits only 20 requests per second?
- What if one payment request takes 30 seconds?
- How do you prevent one bad record from stopping all useful work?
- How do you compare performance fairly?
- What happens to queued work when the process shuts down?

