# Module 4 — REST APIs and Integration

## 4.1 Beginner explanation

An API defines how software components interact.

An HTTP API commonly receives:

- A method.
- A URL.
- Headers.
- An optional body.

It returns:

- A status code.
- Headers.
- An optional body.

The API contract defines what those messages mean.

## 4.2 Industry-level explanation

An integration must handle more than a successful request.

It needs agreement on:

- Authentication and authorization.
- Request and response schemas.
- Timeouts and retry behavior.
- Idempotency.
- Pagination.
- Rate limits.
- Error categories.
- Versioning.
- Event delivery.
- Observability.
- Ownership.

A reliable client understands both the protocol and the business operation.

## 4.3 Architecture

```text name=api_integration_architecture.txt
Customer application
       |
       | HTTPS + credential
       v
API gateway
  - Routing
  - Request limits
  - Authentication integration
       |
       v
Application service
  - Resource authorization
  - Input validation
  - Business rules
  - Idempotency handling
       |
       +----> Database
       |
       +----> External provider
       |
       +----> Durable event publication
```

The gateway does not eliminate the need for application-level authorization.

---

## 4.4 REST

### Beginner

REST is an architectural style commonly used to design resource-oriented APIs.

### Practical example

- `/payments` represents a collection.
- `/payments/payment_123` identifies one payment.
- HTTP methods describe the intended operation.

### Industry considerations

- Consistent resource naming.
- Clear representations.
- Stateless request handling.
- Appropriate method semantics.
- Cache behavior where applicable.
- Compatible evolution.

### Mistake

Assuming any endpoint returning JSON is automatically a well-designed REST API.

### Exercise

Design endpoints for:

1. Creating a payment.
2. Retrieving a payment.
3. Listing payments.
4. Creating a refund.
5. Listing refunds for a payment.

Explain which operations require business-level idempotency.

---

## 4.5 SOAP

### Beginner

SOAP is a structured messaging protocol using XML envelopes.

### Industry

A SOAP integration may involve formal service contracts, namespaces, generated
clients, and message-level standards selected by the service.

SOAP is not simply "REST with XML." REST is an architectural style; SOAP is a
messaging protocol.

### Example envelope

```xml name=payment_status_request.xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope
    xmlns:soap="http://www.w3.org/2003/05/soap-envelope"
    xmlns:pay="urn:example:payments">
  <soap:Body>
    <pay:GetPaymentStatus>
      <pay:PaymentId>payment_demo_001</pay:PaymentId>
    </pay:GetPaymentStatus>
  </soap:Body>
</soap:Envelope>
```

### Production scenario

A customer's existing enterprise system exposes payment status through SOAP.
Your adapter translates between that contract and your application's internal
model.

### Mistakes

- Ignoring XML namespaces.
- Treating HTTP success as proof that no SOAP fault occurred.
- Parsing untrusted XML with unsafe parser settings.
- Hand-building complex production contracts when a suitable generated client
  is available.

### Interview follow-up

What would you inspect if the response is HTTP 200 but the operation failed?

Inspect the documented message body and fault/error contract.

---

## 4.6 HTTP methods

| Method | Typical purpose | Idempotent by HTTP semantics? |
|---|---|---|
| GET | Retrieve a representation | Yes |
| HEAD | Retrieve response metadata without the response content | Yes |
| PUT | Create or replace target resource state | Yes |
| PATCH | Apply a partial modification | Not inherently |
| DELETE | Remove target resource association/state | Yes |
| POST | Perform a resource-specific operation | Not inherently |

### Important distinction

Idempotent does not mean the response is always identical.

A repeated DELETE can return a different status while leaving the same intended
resource state.

### PUT versus PATCH

- PUT supplies the desired replacement representation under the API contract.
- PATCH describes a partial modification.
- Setting `enabled` to `true` can be idempotent.
- Incrementing a counter is not inherently idempotent.

### Concurrency consideration

Use version checks or conditional requests where needed to prevent one client
from silently overwriting another client's changes.

---

## 4.7 HTTP status codes

| Code | Meaning for integration diagnosis |
|---|---|
| 200 | Request succeeded according to endpoint semantics |
| 201 | Resource created |
| 202 | Accepted for processing; completion is not implied |
| 204 | Successful response without content |
| 400 | Invalid request |
| 401 | Valid authentication credentials are missing or unacceptable |
| 403 | Request is understood but refused |
| 404 | Target resource not found, or existence may be concealed |
| 409 | Conflict with current resource state |
| 429 | Rate limiting |
| 500 | Unexpected server-side failure |
| 502 | Gateway received an invalid upstream response |
| 503 | Service temporarily unavailable |
| 504 | Gateway did not receive a timely upstream response |

### Production lesson

An HTTP 200 response containing `"payment_status": "pending"` does not mean the
payment has succeeded.

Similarly, an HTTP timeout does not prove that a charge failed.

---

## 4.8 JSON contracts

### Beginner

JSON represents objects, arrays, strings, numbers, booleans, and null.

### Industry

Specify:

- Required and optional fields.
- Field types.
- Limits.
- Enumerated values.
- Timestamp format and time zone.
- Monetary representation.
- Missing-versus-null behavior.
- Compatibility rules.

### Example request

```json name=create_payment_request.json
{
  "merchant_reference": "order_demo_001",
  "amount_minor": 2500,
  "currency": "USD",
  "payment_method_token": "sandbox_token"
}
```

### Example accepted response

```json name=create_payment_response.json
{
  "payment_id": "payment_demo_001",
  "status": "pending",
  "request_id": "request_demo_001"
}
```

### Common mistake

Returning `null` for an optional field in one version and removing it in another
without checking client expectations.

---

## 4.9 Authentication versus authorization

### Authentication

Who or what is making the request?

Examples:

- API key.
- Access token.
- Client certificate.

### Authorization

Is this authenticated identity allowed to perform this action on this resource?

Example:

A valid merchant token must not permit reading another merchant's payment.

### Production authorization check

```text name=authorization_flow.txt
Validate credential
       |
       v
Identify principal and tenant
       |
       v
Load or identify target resource
       |
       v
Check action permission AND resource ownership
       |
       v
Execute operation
```

### Mistake

Checking that a JWT is valid but never checking whether its principal owns
the requested payment.

---

## 4.10 OAuth 2.0

### Beginner

OAuth 2.0 provides a framework for granting access to protected resources.

### Industry

Choose the flow for the client and trust model.

Examples:

- Authorization Code with PKCE for appropriate user-delegated applications.
- Client Credentials for suitable machine-to-machine use.

OAuth alone is not a user-login protocol. OpenID Connect adds an identity layer
for authentication use cases.

### Authorization Code with PKCE: simplified flow

```text name=oauth_pkce_flow.txt
Application ---- authorization request + challenge ----> Authorization server
User        ---- authenticates / approves -------------> Authorization server
Application <--- authorization code -------------------- Authorization server
Application ---- code + verifier ----------------------> Token endpoint
Application <--- access token -------------------------- Token endpoint
Application ---- access token -------------------------> Resource API
```

### Production practices

- Use a maintained OAuth/OIDC library.
- Register and validate redirect URIs carefully.
- Use PKCE where required and appropriate.
- Protect tokens from logs and unintended storage.
- Validate scopes and audience.
- Do not use an ID token as a substitute for an API access token.
- Follow the authorization server's supported flow and security guidance.

### Exercise

Explain why a browser application must not contain a confidential client secret.

---

## 4.11 JWT

### Beginner

A JSON Web Token is a format for representing claims.

A commonly encountered signed compact JWT has:

```text name=jwt_structure.txt
encoded_header.encoded_payload.signature
```

### Industry

A signed JWT is not automatically encrypted.

Validation can include:

- Signature.
- Explicitly permitted algorithms.
- Trusted key source.
- Issuer.
- Audience.
- Expiration and relevant time claims.
- Token type and application context.
- Required claims.

After validation, authorization is still necessary.

### Mistakes

- Decoding without verifying.
- Trusting the token's algorithm choice without policy.
- Accepting a token intended for another API.
- Logging the complete token.
- Assuming every OAuth access token is a JWT.

### Interview follow-up

How do you handle key rotation?

Use the issuer's trusted key-distribution mechanism, cache appropriately,
handle key identifiers safely, and follow a bounded refresh strategy.

---

## 4.12 API gateway

### Beginner

A gateway is an entry point that routes API requests to services.

### Industry

Depending on the product and configuration, it may provide:

- Authentication integration.
- Routing.
- Request limits.
- Rate limiting.
- Access logging.
- Transformation.
- Version routing.

### Production scenario

The gateway reports 504 responses after a release.

Investigate:

1. Gateway timeout.
2. Backend duration.
3. Dependency duration.
4. Network path.
5. Connection and concurrency limits.
6. Whether the backend continues processing after the client sees a timeout.

### Mistake

Increasing the gateway timeout without understanding why the dependency slowed.

---

## 4.13 Error handling

### Proposed error contract

```json name=api_error.json
{
  "error": {
    "code": "provider_temporarily_unavailable",
    "message": "Payment processing is temporarily unavailable.",
    "request_id": "request_demo_002",
    "retryable": true
  }
}
```

### Design principles

- Stable machine-readable code.
- Safe human-readable message.
- Correlation identifier.
- No stack traces or secrets.
- Documented handling instructions.

### Retry caution

Even if an error is marked retryable, the client must consider whether the
operation can be repeated safely.

A payment-creation request needs appropriate idempotency protection.

---

## 4.14 Idempotency

### Beginner

Repeating the same intended operation should not create duplicate business
effects.

### Industry-level payment example

The client creates one stable key for a business operation and reuses it when
retrying that same operation.

The server coordinates concurrent requests using durable state.

```text name=idempotent_payment_creation.txt
POST payment + merchant identity + idempotency key
                    |
                    v
        Atomically claim scoped key
                    |
          +---------+----------+
          |                    |
        New key            Existing key
          |                    |
          v                    v
 Record request hash      Compare request hash
 and operation state           |
          |             +------+------+
          v             |             |
 Execute/recover     Same request   Different request
 operation              |             |
          |             v             v
          v        Replay result /   Reject conflict
 Store result       report pending
```

### Required questions

1. What scopes the key: merchant, endpoint, or operation?
2. How is the request fingerprint defined?
3. What happens when the same key has a different payload?
4. What happens during concurrent requests?
5. What happens if the process crashes?
6. How long is the key retained?
7. What happens after retention expires?
8. Can the provider accept the same idempotency identifier?
9. How is an unknown provider outcome reconciled?

### Critical mistake

Generating a new key for every retry defeats the purpose.

### Another critical mistake

Using a local database transaction as if it could atomically include an external
provider call.

The external boundary still requires recovery design.

---

## 4.15 Python integration example: one payment-creation attempt

This example deliberately does not automatically retry a write.
It exposes ambiguous outcomes to the caller.

```python name=payment_creation_client.py
import requests


class PaymentOutcomeUnknown(RuntimeError):
    pass


def create_payment_once(session, base_url, token, payload, idempotency_key):
    if not idempotency_key:
        raise ValueError("A stable idempotency key is required")

    try:
        response = session.post(
            f"{base_url.rstrip('/')}/payments",
            headers={
                "Authorization": f"Bearer {token}",
                "Idempotency-Key": idempotency_key,
                "Accept": "application/json",
            },
            json=payload,
            timeout=(3.05, 10),
            allow_redirects=False,
        )
    except requests.exceptions.SSLError:
        raise
    except (requests.Timeout, requests.ConnectionError) as exc:
        raise PaymentOutcomeUnknown(
            "No definitive response; reconcile or retry with the same key "
            "only according to the provider contract"
        ) from exc

    if response.status_code >= 500:
        raise PaymentOutcomeUnknown(
            "Server error does not prove the payment operation was not applied"
        )

    if response.status_code not in {200, 201, 202}:
        # A real adapter maps the provider's documented error contract.
        raise RuntimeError(f"Unexpected HTTP status: {response.status_code}")

    try:
        result = response.json()
    except ValueError as exc:
        raise PaymentOutcomeUnknown(
            "Successful HTTP response had an unreadable result; reconcile"
        ) from exc

    if not isinstance(result, dict) or "payment_id" not in result:
        raise PaymentOutcomeUnknown(
            "Response did not contain the expected payment identifier"
        )

    return result
```

### Production extensions

- Validate all inputs and responses.
- Obtain secrets from an approved source.
- Persist the key before the request.
- Persist the provider reference and result.
- Implement the provider's documented replay semantics.
- Add a total operational deadline where appropriate.
- Record safe request identifiers and durations.
- Reconcile ambiguous outcomes.
- Never submit real payment details through this teaching example.

---

## 4.16 Webhooks

### Beginner

A webhook lets another system send your application an event when something
happens.

### Industry workflow

```text name=webhook_processing.txt
Receive raw request bytes
          |
          v
Enforce size limit
          |
          v
Verify provider signature and timestamp rules
          |
          v
Validate supported event format
          |
          v
Durably record event with deduplication
          |
          v
Acknowledge according to provider contract
          |
          v
Asynchronous processing
          |
          v
Apply permitted transition / reconcile / quarantine
```

### Important practices

- Use the provider's official signature-verification procedure or SDK.
- Preserve the raw request body when verification requires it.
- Expect duplicate delivery.
- Do not assume event ordering.
- Avoid long synchronous processing before acknowledgement.
- Do not acknowledge success before responsibility for the event is durable.
- Track failed processing and replay safely.

### Why no homemade signature implementation here?

Signature formats and replay rules are provider-specific. A generic HMAC example
can create a false impression that it is compatible with every provider.

### Hands-on exercise

Build a fake webhook producer and receiver using synthetic events.

Demonstrate:

1. A valid event.
2. An invalid signature.
3. A duplicate event.
4. An event arriving out of order.
5. A database outage before durable acceptance.
6. A worker crash after acceptance.

---

## 4.17 Interview questions and answers

### Q1. PUT versus PATCH?

**Answer:** PUT requests creation or replacement of the target resource state
according to the contract. PATCH applies a partial modification. PUT is
idempotent by HTTP semantics; PATCH can be designed to be idempotent but is not
inherently so.

**Expect:** An example such as setting a value versus incrementing it.

**Follow-up:** How do you avoid lost updates?

Use conditional requests or application-level version checks.

### Q2. 404 versus 500?

**Answer:** A 404 means the target resource is not found or its existence is not
being disclosed. A 500 indicates an unexpected server-side error.

**Expect:** Distinguish client-visible semantics from internal root cause.

**Follow-up:** Can a permissions policy intentionally return 404?

Yes, depending on the API's disclosure policy.

### Q3. Authentication versus authorization?

**Answer:** Authentication establishes identity. Authorization determines
whether that identity can perform the action on the resource.

**Expect:** A cross-merchant access example.

**Follow-up:** Is gateway authentication enough?

No; resource-level authorization may still belong in the application.

### Q4. What is idempotency?

**Answer:** Repeating the same intended operation does not create additional
business effects. For payment POSTs, implement this through a durable,
appropriately scoped key and defined replay behavior.

**Expect:** Concurrency and crash recovery.

**Follow-up:** What if the payload changes but the key does not?

Reject or otherwise handle it explicitly according to the contract.

### Q5. A payment request times out. What do you do?

**Answer:** Treat the outcome as unknown until evidence resolves it. Use the
provider reference, lookup API, webhook, or the same supported idempotency key
to reconcile. Do not issue a new independent charge.

**Expect:** Protect against double charging.

**Follow-up:** What should the customer see meanwhile?

A truthful pending/unknown state with an appropriate recovery workflow.

### Q6. What does HTTP 202 mean?

**Answer:** The request was accepted for processing, not necessarily completed.
The API should define how the client learns the eventual outcome.

**Expect:** Polling or event-based completion.

**Follow-up:** How do you handle permanent failure after acceptance?

Expose a durable operation state and a documented failure response path.

### Q7. How do you handle 429?

**Answer:** Follow the provider's rate-limit contract, honor Retry-After when
provided and applicable, reduce request pressure, and apply bounded retry
behavior. Coordinate concurrency across workers where necessary.

**Expect:** Do not blindly retry immediately.

**Follow-up:** Is a per-process limiter enough for twenty replicas?

Not necessarily.

### Q8. JWT versus OAuth?

**Answer:** JWT is a token format. OAuth is an authorization framework.
An OAuth access token may be JWT-shaped or opaque.

**Expect:** Do not confuse encoding, protocol, and authorization.

**Follow-up:** Is a signed JWT encrypted?

Not merely because it is signed.

### Q9. How do you validate a JWT?

**Answer:** Use a maintained library with trusted keys, permitted algorithms,
issuer and audience checks, relevant time checks, and required claims. Then
perform authorization.

**Expect:** Verification, not just decoding.

**Follow-up:** Where do verification keys come from?

A configured trusted issuer or key-distribution mechanism.

### Q10. How do you secure webhooks?

**Answer:** Verify the provider signature using its prescribed method, apply
timestamp/replay rules where supported, limit request size, validate input,
deduplicate durably, and process under least privilege.

**Expect:** Raw-body handling and replay awareness.

**Follow-up:** Why is an unguessable URL insufficient?

It does not replace message authentication.

### Q11. REST versus SOAP?

**Answer:** REST is an architectural style, commonly applied to HTTP resources.
SOAP is an XML-based messaging protocol. Choose integration tooling according
to the actual contract.

**Expect:** No simplistic "old versus new" answer.

**Follow-up:** What is a SOAP fault?

A structured failure message defined within the SOAP framework.

### Q12. How do you troubleshoot intermittent 502 responses?

**Answer:** Correlate gateway and backend requests, check upstream resets,
deployment events, connection reuse, backend availability, and response
validity. Compare affected instances and time windows.

**Expect:** Layered evidence.

**Follow-up:** Why might only one instance be affected?

Configuration drift, resource exhaustion, or an instance-specific defect.

### Q13. How do you version an API?

**Answer:** Define compatibility expectations, prefer additive changes where
safe, test existing clients, and use an explicit versioning/deprecation strategy
for breaking changes.

**Expect:** Consider enum changes, required fields, and semantics—not just URLs.

**Follow-up:** Is adding an enum value always safe?

No; clients may exhaustively match known values.

### Q14. Polling versus webhooks?

**Answer:** Polling gives the client control but creates repeated requests and
may delay detection. Webhooks reduce polling but require reachable receivers,
delivery recovery, authentication, and deduplication.

**Expect:** A hybrid reconciliation strategy where appropriate.

**Follow-up:** How do you recover a missed webhook?

Fetch authoritative state or events through the provider's supported mechanism.

### Q15. How do you design useful API errors?

**Answer:** Provide stable codes, safe messages, request identifiers, and
documented handling. Separate business rejection from infrastructure failure.

**Expect:** Consumer usability without leaking internals.

**Follow-up:** Should every failure be HTTP 200 with an error field?

Explain why this can undermine standard client, gateway, and monitoring behavior.

---

## 4.18 Production scenario: duplicate customer charges

### Observed symptom

A customer reports two charges for one order after a temporary network issue.

### Investigation

1. Establish the affected order and provider references.
2. Compare client, service, and provider timestamps.
3. Identify all submitted requests.
4. Compare their idempotency keys and payloads.
5. Determine whether retries happened at multiple layers.
6. Check whether the first request succeeded before the connection failed.
7. Preserve evidence and coordinate customer remediation.

### Illustrative root cause

The client generated a new idempotency key after every timeout. The provider
therefore treated each retry as a new payment operation.

### Immediate resolution

- Stop the unsafe retry behavior.
- Reconcile affected operations.
- Follow the approved duplicate-charge remediation process.
- Communicate scope and next steps.

### Prevention

- Persist one stable key per business operation.
- Test response-loss scenarios.
- Review retry behavior across client, gateway, service, and SDK layers.
- Maintain a durable operation state.
- Alert on suspicious duplicate business references.

### What a strong interview answer demonstrates

- A timeout does not prove failure.
- Correctness takes priority over blind retries.
- Customer impact is addressed alongside technical diagnosis.
- Remediation uses evidence, not speculative data changes.

### Follow-up questions

- What if the provider does not support idempotency?
- What if the idempotency retention period has expired?
- What if two requests with the same key arrive simultaneously?
- What if the first process crashes after the provider accepts the charge?
- How would you test this without making real charges?

---

## 4.19 Hands-on integration lab

Build a small client against a local fake payment server.

The server should support scripted responses:

1. Successful creation.
2. Accepted but pending.
3. Validation error.
4. Authentication error.
5. Rate limit.
6. Server error.
7. Slow response.
8. Connection loss after simulated acceptance.
9. Duplicate key with the same payload.
10. Duplicate key with a different payload.

### Deliverables

- Client code.
- Synthetic fixtures.
- Tests.
- Error-classification table.
- Retry policy.
- Idempotency design.
- README explaining assumptions.
- Runbook for unknown payment outcomes.

### Evaluation rubric

Score each dimension from 0 to 2:

| Dimension | 0 | 1 | 2 |
|---|---|---|---|
| Correctness | Happy path only | Some error handling | Failure semantics are explicit |
| Safety | Blind retries | Partial safeguards | Ambiguous writes handled safely |
| Tests | None | Basic tests | Failure and replay tests |
| Observability | Prints only | Some useful logs | Safe correlated events |
| Communication | Undocumented | Basic README | Assumptions and trade-offs explained |

A high score means the lab is well prepared, not that it is production-certified.

## 4.20 Common mistakes

- No timeout.
- Retrying writes without checking idempotency.
- Generating a new key per retry.
- Treating a timeout as payment failure.
- Trusting decoded JWT claims without verification.
- Checking identity but not resource ownership.
- Logging credentials.
- Assuming webhook ordering.
- Acknowledging events before durable acceptance.
- Increasing timeouts instead of diagnosing latency.

## 4.21 Cheat sheet

```text name=api_cheatsheet.txt
HTTP success != business success
Timeout != operation failed
JWT decoding != JWT verification
Authentication != authorization
OAuth != JWT
Signed != encrypted
POST != automatically idempotent
PATCH != always non-idempotent
DELETE idempotent != identical response every time
Webhook delivery != exactly-once business effect
Gateway auth != complete resource authorization
Safe retry = operation semantics + failure classification + bounded policy
```

---

# Part 1 Completion Checklist

## Module 1

- [ ] Explain the FDE role in two minutes.
- [ ] Compare FDE, SWE, DevOps, SRE, and Solutions Architect responsibilities.
- [ ] Produce a deployment proposal with measurable acceptance criteria.
- [ ] Explain a customer problem without immediately selecting a technology.

## Module 2

- [ ] Run all 20 Python examples that do not require an external API.
- [ ] Complete the 20 exercises.
- [ ] Explain thread and async trade-offs.
- [ ] Build a report-only reconciliation job.
- [ ] Test malformed input and unknown outcomes.

## Module 3

- [ ] Run the PostgreSQL schema.
- [ ] Explain join cardinality and prevent double-counted totals.
- [ ] Write a window-function query.
- [ ] Inspect a query plan.
- [ ] Explain transactions across an external-provider boundary.
- [ ] Answer all 30 database questions.

## Module 4

- [ ] Explain authentication, authorization, OAuth, and JWT distinctly.
- [ ] Explain HTTP idempotency.
- [ ] Design business-level payment idempotency.
- [ ] Handle timeout outcomes safely.
- [ ] Design authenticated, deduplicated webhook processing.
- [ ] Complete the fake-server integration lab.

# Next Section

Module 5 — AWS

Planned coverage:
EC2, S3, RDS, Lambda, IAM, CloudWatch, ECS, EKS, DynamoDB, SNS, SQS,
EventBridge, API Gateway, Secrets Manager, and VPC.
