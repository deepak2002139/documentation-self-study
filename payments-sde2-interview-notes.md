# Payments Domain — Complete Notes for SDE-2 (Backend / Payments Engineer)

---

# PART 1 — DOMAIN FOUNDATIONS

## 1.1 What a payment actually is

A payment is a **transfer of value from a payer (debtor) to a payee (creditor)**, executed through one or more intermediaries, resulting in:
- **Clearing** — exchange of payment instructions + calculating who owes whom.
- **Settlement** — actual movement of funds (finality of value).

Key distinction interviewers probe:
> "Clearing is information. Settlement is money. A payment is only *final* after settlement."

**Finality** = irrevocability. Once settled in an RTGS system, the payment cannot be recalled unilaterally (only a *recall request* can be raised, which the beneficiary bank may reject).

## 1.2 The actors

| Actor | Role |
|---|---|
| Debtor / Payer | Originates payment |
| Debtor Agent | Payer's bank (ordering institution) |
| Intermediary Agent(s) | Correspondent banks in the chain |
| Creditor Agent | Beneficiary bank |
| Creditor / Payee | Final recipient |
| Clearing & Settlement Mechanism (CSM) | TARGET2, Fedwire, CHAPS, NPP, UPI switch |
| Scheme / Network | SWIFT, SEPA, Visa/Mastercard, NPCI |

## 1.3 End-to-end payment lifecycle (memorize this)

```
1.  Initiation        → channel (mobile, net-banking, host-to-host file, API, branch)
2.  Ingestion         → gateway/channel adapter, auth, schema validation
3.  Duplicate check   → idempotency / dedupe window
4.  Enrichment        → BIC/IBAN lookup, routing codes, purpose codes, charges
5.  Validation        → mandatory fields, format, currency/country rules, limits
6.  Screening         → sanctions (OFAC/UN/EU), PEP, embargo (real-time, blocking)
7.  Fraud / AML       → scoring, velocity checks, behavioural rules (may be async)
8.  Authorization     → customer limits, entitlements, dual auth for high-value
9.  Funds control     → balance check, hold/earmark, overdraft check
10. Accounting        → debit customer, credit nostro/suspense (double-entry)
11. Routing           → choose rail: RTGS / ACH / instant / on-us / correspondent
12. Format & Transform→ pacs.008 / MT103 / NACH file / UPI API
13. Dispatch          → to CSM / SWIFT network / scheme
14. Ack/Nack          → pacs.002 status report, MT012/MT019, network ack
15. Settlement        → RTGS debit-credit at central bank, or net batch settlement
16. Confirmation      → pacs.002 ACSC / MT910/MT950, camt.054 credit notification
17. Reconciliation    → nostro statement (camt.053/MT940) vs internal ledger
18. Post-processing   → notifications, fee posting, reporting, archival
19. Exceptions        → repair, return (pacs.004), recall (camt.056), investigation (camt.027)
```

**Interview line:** "A payment engine is essentially a long-running, durable state machine over an ordered event log, with hard idempotency and hard auditability requirements."

---

# PART 2 — PAYMENT TYPES & RAILS

## 2.1 Classification

| Dimension | Types |
|---|---|
| Geography | Domestic, Cross-border |
| Speed | RTGS (real-time gross), Instant/Faster (24×7 seconds), ACH/Batch (T+0/T+1/T+2) |
| Value | High-value (wholesale), Low-value (retail/bulk) |
| Settlement | Gross (one by one), Net (multilateral netting) |
| Direction | Credit transfer (push), Direct debit (pull) |
| Customer | C2B, B2B, B2C, P2P, G2C |

## 2.2 Gross vs Net settlement

- **RTGS (Gross)**: each payment settled individually, immediately, irrevocably. High liquidity need, zero settlement risk. e.g. Fedwire, TARGET2, CHAPS, RTGS India.
- **DNS (Deferred Net Settlement)**: payments accumulated, netted multilaterally, settled at cycles. Low liquidity need, but **settlement risk** exists until cycle completes. e.g. ACH, NEFT, BACS, SEPA SCT via STEP2.
- **Instant (hybrid)**: instant credit to beneficiary + deferred net settlement between banks, backed by **prefunded liquidity** at the central bank. e.g. SEPA Inst (TIPS/RT1), UPI, FedNow, RTP, NPP.

## 2.3 The major rails

### SWIFT
- Not a payment system — a **secure messaging network** (~11,000+ institutions).
- Identifies institutions by **BIC** (8 or 11 chars: BANK + CC + LL + BBB).
- Legacy **MT** (FIN) messages; migrating to **ISO 20022 MX**.
- **SWIFT gpi**: end-to-end tracking via **UETR** (UUIDv4), transparency of fees, same-day use of funds, Tracker API, stop-and-recall (gSRP), gCase.
- **CBPR+** = Cross-Border Payments and Reporting Plus — the ISO 20022 usage guidelines for cross-border SWIFT traffic. MT→MX coexistence ended **Nov 2025**; pacs/camt now mandatory for cross-border interbank.

**Key MT messages:**
| MT | Meaning |
|---|---|
| MT101 | Request for transfer (customer initiation) |
| MT103 | Single customer credit transfer |
| MT103 STP | Straight-through-processing variant |
| MT202 | General financial institution transfer (bank-to-bank cover) |
| MT202 COV | Cover payment carrying underlying customer details |
| MT200/210 | Own account transfer / notice to receive |
| MT900/910 | Debit / Credit confirmation |
| MT940/942/950 | End-of-day / intraday / statement |
| MT192/196/199 | Cancellation request / answer / free format |
| MT012/MT019 | Ack / Nack from SWIFT network |

### ISO 20022
XML (and now JSON in some schemes) standard. Message name format: `pacs.008.001.08`
→ `businessArea.messageId.variant.version`

**Business areas:**
| Prefix | Domain |
|---|---|
| `pain` | Payments Initiation (customer→bank) |
| `pacs` | Payments Clearing & Settlement (bank→bank) |
| `camt` | Cash Management (statements, investigations) |
| `acmt` | Account Management |
| `remt` | Remittance advice |
| `admi` | Administration (system events) |
| `head` | Business Application Header (BAH — head.001) |

**Must-know messages:**
| Message | Purpose |
|---|---|
| pain.001 | Customer Credit Transfer Initiation |
| pain.002 | Payment Status Report to customer |
| pain.007/008 | Reversal / Direct Debit initiation |
| pacs.008 | FI-to-FI Customer Credit Transfer (≈ MT103) |
| pacs.009 | FI Credit Transfer (≈ MT202); `COV` variant ≈ MT202COV |
| pacs.002 | FI-to-FI Payment Status Report (ACCP/ACSC/RJCT/PDNG) |
| pacs.004 | Payment Return |
| pacs.003 | FI-to-FI Direct Debit |
| pacs.028 | Payment Status Request |
| camt.052/053/054 | Intraday report / EoD statement / Debit-Credit notification |
| camt.056 | FI to FI Payment Cancellation Request (recall) |
| camt.029 | Resolution of Investigation |
| camt.027/026/087 | Claim non-receipt / unable to apply / request to modify |
| camt.998 | Proprietary |

**Why ISO 20022 matters (classic interview Q):**
1. Richer structured data (structured addresses, 140→9000 char remittance).
2. Better AML/sanctions screening (fewer false positives — party data is structured).
3. Interoperability across schemes.
4. Extensibility & versioning.
5. Enables analytics, reconciliation automation, purpose codes, LEI.

**Structural anatomy of pacs.008:**
```
<Document>
 <FIToFICstmrCdtTrf>
  <GrpHdr>  MsgId, CreDtTm, NbOfTxs, SttlmInf(SttlmMtd: INDA/INGA/CLRG/COVE)
  <CdtTrfTxInf>
    <PmtId> InstrId, EndToEndId, TxId, UETR
    <PmtTpInf> InstrPrty, SvcLvl, LclInstrm, CtgyPurp
    <IntrBkSttlmAmt Ccy="EUR">, <IntrBkSttlmDt>
    <ChrgBr> DEBT | CRED | SHAR | SLEV
    <Dbtr>, <DbtrAcct>, <DbtrAgt>
    <IntrmyAgt1..3>
    <CdtrAgt>, <Cdtr>, <CdtrAcct>
    <RmtInf> Ustrd / Strd
    <RgltryRptg>, <Purp>
```
**Key IDs:** `MsgId` (per message), `InstrId` (agent-to-agent), `EndToEndId` (debtor→creditor, preserved end to end), `TxId`, `UETR` (unique across whole chain, gpi tracking).

### Scheme cheat-sheet

| Scheme | Region | Type | Currency | Notes |
|---|---|---|---|---|
| **Fedwire** | US | RTGS | USD | Fed-operated, ISO 20022 migrated Jul 2025, ABA routing number |
| **CHIPS** | US | Netting, real-time final | USD | Private (TCH), ~large-value, UID participants |
| **ACH (NACHA)** | US | Batch net | USD | Same-day ACH windows, R-codes for returns |
| **RTP (TCH)** | US | Instant | USD | 24×7, ISO 20022, $10M limit, credit push only |
| **FedNow** | US | Instant RTGS | USD | Launched 2023, 24×7, ISO 20022 |
| **TARGET2 / T2** | EU | RTGS | EUR | ECB; consolidated with T2S; ISO 20022 (Mar 2023) |
| **EURO1 / STEP2** | EU | Net / bulk | EUR | EBA Clearing |
| **SEPA SCT** | EU | Batch (D+1) | EUR | IBAN + BIC, pain/pacs |
| **SEPA SCT Inst** | EU | Instant, ≤10s | EUR | via TIPS or RT1; Instant Payments Regulation mandates VoP |
| **SEPA SDD** | EU | Direct debit | EUR | Core (8-week no-Q refund) & B2B |
| **CHAPS** | UK | RTGS | GBP | BoE, ISO 20022 since Jun 2023 |
| **BACS** | UK | 3-day batch | GBP | Direct debit/credit |
| **FPS** | UK | Instant | GBP | 24×7, ~seconds |
| **UPI** | India | Instant | INR | NPCI, VPA-based, 24×7, API/ISO-ish |
| **IMPS / NEFT / RTGS** | India | Instant / batch-ish / RTGS | INR | NEFT now 24×7 half-hourly batches |
| **NPP / PayTo** | AU | Instant | AUD | PayID, ISO 20022 |
| **CIPS** | CN | Cross-border | CNY | |
| **SIC / SEPA-like** | CH | RTGS | CHF | |

### UPI specifics (commonly asked in India interviews)
- **Participants:** Payer PSP, Payee PSP, Remitter Bank, Beneficiary Bank, NPCI switch, TPAP (GPay/PhonePe).
- **VPA** (`user@bank`) as an addressing layer over account numbers → privacy + portability.
- **Flows:** Pay (push), Collect (pull, request), Mandate (recurring/autopay), UPI Lite (on-device, low value), UPI 123Pay, Credit-line-on-UPI, RuPay CC on UPI.
- **Key APIs:** ReqPay/RespPay, ReqAuthDetails, ReqChkTxn (status check), ReqComplaint, ReqValAdd (address resolution), ReqListAccount.
- **2-factor:** device binding + UPI PIN (encrypted via NPCI common library, never leaves the secure element unencrypted).
- **Settlement:** deferred net settlement at RBI in cycles, but credit is instant → banks bear intraday exposure.
- **Deemed-approved / TAT rules:** RBI TAT circular — auto-reversal within T+1 else ₹100/day penalty.
- **Big engineering problem:** **status ambiguity**. Timeout ≠ failure. Must run **ReqChkTxn** reconciliation loops; never assume failure until network confirms.

---

# PART 3 — PAYMENT SYSTEM ARCHITECTURE

## 3.1 Reference architecture (payment hub)

```
                        ┌──────────────────────────────────────────┐
 Channels               │           CHANNEL / API LAYER            │
 mobile, web, H2H file, │  REST/gRPC APIs, SFTP file ingest,       │
 ERP, branch, ATM/POS   │  ISO20022 adapters, MT adapters          │
                        └───────────────┬──────────────────────────┘
                                        │ (normalize → Canonical Payment Model)
                        ┌───────────────▼──────────────────────────┐
                        │            INGESTION SERVICE             │
                        │  authN/Z, schema validation, dedupe,     │
                        │  idempotency key, persist RAW + CANON    │
                        └───────────────┬──────────────────────────┘
                                        │  Kafka: payments.received
        ┌───────────────────────────────┼─────────────────────────────────┐
        ▼                ▼              ▼              ▼                  ▼
 ┌────────────┐  ┌──────────────┐ ┌───────────┐ ┌────────────┐   ┌──────────────┐
 │ Enrichment │  │  Validation  │ │ Sanctions │ │  Fraud/AML │   │   Limits/    │
 │ (BIC,IBAN, │  │ (rules eng.) │ │ Screening │ │  scoring   │   │ Entitlements │
 │ routing)   │  │              │ │ (sync)    │ │  (async)   │   │              │
 └─────┬──────┘  └──────┬───────┘ └─────┬─────┘ └─────┬──────┘   └──────┬───────┘
       └────────────────┴───────────────┴─────────────┴─────────────────┘
                                        │  Kafka: payments.validated
                        ┌───────────────▼──────────────────────────┐
                        │        ORCHESTRATOR / STATE MACHINE      │
                        │  saga coordinator, retries, timeouts,    │
                        │  compensations, SLA timers               │
                        └───┬────────────┬──────────────┬──────────┘
                            ▼            ▼              ▼
                   ┌────────────┐ ┌────────────┐ ┌──────────────┐
                   │ Accounting │ │  Liquidity │ │   Routing    │
                   │ Ledger     │ │ / Position │ │   Engine     │
                   │(double-ent)│ │  manager   │ │(least cost)  │
                   └─────┬──────┘ └─────┬──────┘ └──────┬───────┘
                         └──────────────┴───────────────┘
                                        │
                        ┌───────────────▼──────────────────────────┐
                        │   FORMATTER / SCHEME ADAPTERS (egress)   │
                        │ pacs.008 | MT103 | NACH | RTP | UPI | FPS│
                        └───────────────┬──────────────────────────┘
                                        ▼
                            SWIFT / CSM / Scheme network
                                        │
                        ┌───────────────▼──────────────────────────┐
                        │  ACK/NACK + camt.053 + Recon + Exception │
                        │  Investigations | Repair UI | Reporting  │
                        └──────────────────────────────────────────┘
```

**Cross-cutting:** audit store (immutable), config/rules service, reference data (BIC/IBAN/calendars/currency), observability, secrets/HSM, archival.

## 3.2 Payment gateway vs payment hub vs payment engine

| | Gateway | Hub | Engine |
|---|---|---|---|
| Scope | Edge, protocol translation, card auth | Bank-wide central processing across rails | Core execution: validate, route, settle |
| Concerns | Tokenization, PCI, 3DS, acquirer routing | Canonical model, single sanctions/AML point, common reference data | State machine, ledger, scheme dispatch |
| Example | Stripe/Adyen/Razorpay, or bank's internal API gateway | Finastra GPP, Volante, Icon IPF, ACI, Temenos, FIS, internal hub | Underlying processing module |

**Why banks build a Payment Hub (frequent interview question):**
- Previously N channels × M rails = N×M silos; each had its own screening, formats, ledger touchpoints.
- Hub gives: one canonical model, one sanctions integration, one reference data, one audit, one SLA/monitoring, cheaper rail onboarding, consistent regulatory reporting.

## 3.3 Canonical Payment Model (CPM)

Never let scheme formats leak into the core.

```
Ingress format ──(adapter)──▶ CPM ──(adapter)──▶ Egress format
```
CPM entity essentials:
```
PaymentId (internal UUID), UETR, EndToEndId, InstructionId
Direction (OUTBOUND/INBOUND), PaymentType, Rail, Priority
Debtor{name,address,account,agentBIC}, Creditor{...}
Amount{value, currency}, FxRate?, ChargeBearer
ValueDate, SettlementDate, CutoffRef
Status, SubStatus, StatusReasonCode
IdempotencyKey, SourceSystem, Channel, TenantId
CreatedAt, Version (optimistic locking)
```
**Trade-off:** CPM must be a *superset* but not a dumping ground. Use a typed core + `Map<String,String> schemeExtensions` for rail-specific fields, plus keep the **raw original message** verbatim for audit/legal (you may need to reproduce it exactly).

## 3.4 Message transformation & enrichment

**Transformation approaches:**
1. **Code-based mappers** (MapStruct/hand-written) — fast, testable, but change = deploy.
2. **Rules/DSL-driven** (Drools, Camel, XSLT, Volante/Trace mapping designer) — business-editable, slower, harder to debug.
3. **Hybrid** (most real systems): structural mapping in code, business rules externalized.

**Practical concerns:**
- Character set: SWIFT `X` charset restrictions; transliteration of non-Latin names (mandatory for MT, avoidable in MX).
- Field truncation: MT103 field 70 = 4×35 chars vs ISO 140/9000 → **truncation on MX→MT downgrade** is a real production issue; you must record "truncated" flags.
- Amount/decimal handling: **always `BigDecimal` or minor-units `long`, never `double`**. Currency exponent from ISO 4217 (JPY=0, BHD=3, USD=2).
- Date/time: `IntrBkSttlmDt` is a business date, not instant; always store timezone-aware and the scheme's business date separately.

**Enrichment sources:** BIC directory (SWIFTRef), IBAN Plus, national clearing codes (ABA, SortCode, IFSC, BSB), SSI (Standard Settlement Instructions), currency & country calendars, purpose codes, correspondent routing table, charge/fee schedules, FX rate service.

---

# PART 4 — COMPLIANCE: SCREENING, SANCTIONS, AML

## 4.1 Sanctions screening (real-time, blocking)

- Lists: **OFAC SDN/CAPTA**, UN, EU consolidated, UK HMT/OFSI, local regulators, internal watchlists.
- Screened attributes: debtor/creditor names, addresses, BICs, countries, vessel/goods names, free-text remittance, intermediary agents.
- Technique: **fuzzy matching** — Levenshtein/Jaro-Winkler, phonetic (Soundex/Metaphone/Double Metaphone), token-based, transliteration handling, name-order permutations, "good guy"/whitelist lists to suppress known false positives.
- Result: **HIT → payment goes to HOLD/PENDING_INVESTIGATION**, human analyst adjudicates → Release / Reject / Block(freeze funds).
- Vendors: Fircosoft, FircoSoft Continuity, Bridger, NICE Actimize, Oracle Mantas/FCCM, LexisNexis, Accuity.

**Engineering requirements:**
- Must be **synchronous & blocking** before dispatch (regulatory). Design for <100–200 ms p99.
- Must be **deterministic & replayable** — store screening request/response + list version + rule version for audit (regulators re-run historic cases).
- **Rescreening** on list update: batch re-screen of in-flight payments.
- **False positive rate** is the operational cost driver (typically 95–99% of hits are FP).

**Interview answer on FP reduction:** structured ISO 20022 party data, better address parsing, LEI usage, whitelisting, ML re-ranking on top of the deterministic engine (ML *never replaces* the rule engine for regulatory defensibility).

## 4.2 AML / Transaction monitoring (mostly async)
- KYC/CDD/EDD, beneficial ownership.
- Scenarios: structuring/smurfing, rapid movement of funds, round-amount patterns, high-risk jurisdictions, unusual velocity, mule account patterns.
- Output: **Alert → Case → SAR/STR** filing.
- Typically batch/near-real-time on a data lake; not in the critical path (unlike sanctions).

## 4.3 Other regulatory checks
- **Travel Rule** (FATF R16 / FinCEN): complete originator & beneficiary info must travel with the payment. Drives ISO 20022 mandatory-field enforcement.
- **Wolfsberg** / correspondent banking due diligence.
- **PSD2/PSD3, SCA** (Strong Customer Authentication — 2 of: knowledge, possession, inherence), Open Banking.
- **Verification of Payee (VoP) / Confirmation of Payee (CoP)** — name-account match before sending; mandatory for EU instant payments (Oct 2025) and UK.
- **PCI-DSS** for card data.
- **Regulatory reporting**: BoP reporting, FATCA/CRS, large-value reporting.

---

# PART 5 — ACCOUNTS, SETTLEMENT, LIQUIDITY

## 5.1 Nostro / Vostro / Loro
- **Nostro** — "our account with you", held by *our* bank at a foreign correspondent, in *foreign* currency. E.g. HDFC's USD account at Citi New York.
- **Vostro** — "your account with us", the mirror: Citi's books show HDFC's account. (In India: *Special Rupee Vostro Accounts* for INR trade settlement.)
- **Loro** — third-party's account, referenced by a bank that is neither owner nor holder.
- **Mirror account** — internal shadow ledger of the nostro, updated when we *instruct*, reconciled against the *actual* nostro statement (camt.053/MT940).

**Why cross-border is slow/expensive:** you need a correspondent chain; each hop adds fees, cut-offs, screening, and FX. Correspondent de-risking has shrunk the number of available chains.

## 5.2 Correspondent banking chain (serial vs cover)
- **Serial**: MT103/pacs.008 passes hop-by-hop through each intermediary.
- **Cover**: MT103 goes *directly* debtor agent → creditor agent (information), while MT202COV/pacs.009COV moves the *funds* through the correspondents. Faster info, but the cover leg must carry underlying customer data (post-2009 rule) for screening.

## 5.3 Settlement mechanics
- **RTGS**: central bank account debited/credited immediately; needs intraday liquidity; queuing + **gridlock resolution** algorithms (offsetting/bilateral/multilateral optimization), reservations, priority levels.
- **Net settlement**: netting cycles; **settlement risk** (Herstatt risk in FX — one leg settles, the other doesn't) → solved by **CLS** (Continuous Linked Settlement, PvP for FX).
- **PvP** = payment vs payment (FX). **DvP** = delivery vs payment (securities). **On-us** = both parties at same bank → internal book transfer, no external rail.

## 5.4 Liquidity management
Concepts to be able to speak to:
- **Intraday liquidity**: buffer at central bank/correspondents to fund outgoing payments before incoming arrive.
- **Throughput rules**: e.g., CHAPS/Fedwire expect X% of value by certain hours.
- **Queue management & prioritization**: urgent vs normal; hold-release; sequencing to avoid deadlock.
- **Liquidity bridging/pooling**, sweeping, cash concentration, notional pooling.
- **Reservations/earmarks** for critical settlements (CLS pay-ins, CCP margin calls).
- **BCBS 248** intraday liquidity monitoring metrics.
- **Prefunding** for instant schemes (TIPS/RTP/FedNow) — the balance is finite and 24×7, so you need **auto-replenishment** logic and low-balance alerts, otherwise instant payments start failing at 3 AM Sunday.

**Engineering note:** liquidity/position service is a **hot single-writer per account** problem → partition Kafka by account, use per-account serialization, optimistic locking, or an in-memory position cache with a durable log (LMAX-style).

## 5.5 Accounting / Ledger design
Double-entry, always:
```
Outbound customer payment:
  DR Customer Account       100
  CR Nostro/Clearing Suspense 100
  DR Customer Account (fee)    2
  CR Fee Income                2
```
Ledger rules:
- **Immutable, append-only** postings. No UPDATE/DELETE. Corrections are *reversal + new posting*.
- Every posting has: `postingId, transactionId (groups the balanced set), accountId, DR/CR, amount, currency, valueDate, bookingDate, narrative, sourceEventId`.
- **Balances are derived**, but you maintain materialized balance snapshots for performance + periodic recomputation to verify.
- **Suspense/clearing accounts** must be reconciled to zero daily — a non-zero suspense is a production incident.

## 5.6 Cut-offs & business day processing
- Each **currency × rail × correspondent** has its own cut-off time (in the *scheme's* timezone).
- Payment after cut-off → **warehoused** for next business day (value-dated forward).
- **Business calendars**: currency holiday, country holiday, scheme holiday, bank holiday. Value date must be a valid business day for *both* currency and both agents.
- **Forward-dated / warehoused payments**: stored, re-validated on the release day (limits, sanctions rescreen, account still open).
- **EOD process**: cut-off freeze → net position calc → settlement file/statement generation → date roll → reopen.
- Testing tip you can mention: never use `LocalDate.now()` in code — inject a `BusinessDateProvider`/`Clock` so you can simulate date roll and cut-offs in tests.

---

# PART 6 — STATUS MODEL & STATE MACHINE

## 6.1 ISO 20022 external status codes
`ACTC` accepted technical validation · `ACCP` accepted customer profile · `ACSP` accepted settlement in process · `ACSC` accepted settlement completed · `ACWC` accepted with change · `ACWP` accepted without posting · `PDNG` pending · `RJCT` rejected · `CANC` cancelled · `RCVD` received · `PART` partially accepted.

**Common reason codes:** AC01 incorrect account number, AC04 closed account, AC06 blocked account, AM04 insufficient funds, AM05 duplication, AG01 transaction forbidden, BE01 inconsistent debtor name, MD07 end customer deceased, RR04 regulatory reason, DUPL, NARR, FOCR (following cancellation request), CUST (requested by customer), TM01 cut-off time.

## 6.2 Internal state machine (typical)

```
RECEIVED
  → VALIDATED ──(fail)──▶ REJECTED
  → DUPLICATE_DETECTED
  → SCREENING_IN_PROGRESS
        ├─ hit ─▶ COMPLIANCE_HOLD ─▶ (release) ─▶ AUTHORIZED
        │                          └▶ BLOCKED / REJECTED
        └─ clear ─▶ AUTHORIZED
  → FUNDS_RESERVED ──(insufficient)──▶ ON_HOLD_INSUFFICIENT_FUNDS
  → POSTED (accounting done)
  → SENT_TO_SCHEME
  → ACK_RECEIVED / NACK_RECEIVED(→ REPAIR)
  → SETTLED  (terminal, immutable)
  → RETURNED / RECALLED / REVERSED (post-settlement, new linked payment)
REPAIR ⇄ RESUBMITTED
```

**Design rules:**
- Only **forward transitions**, defined in an explicit transition table; illegal transitions throw.
- Persist **every** transition as an event: `(paymentId, fromStatus, toStatus, reasonCode, actor, timestamp, correlationId)`.
- `SETTLED` is terminal. "Undo" is a **new compensating payment** (return/reversal), never a status rollback. This is the core "no distributed rollback in payments" point.
- Store status *externally reportable* (ISO code) separately from *internal* sub-status.

## 6.3 Exceptions, repair, investigations
- **Repair queue / STP failure**: payment missing/invalid data (bad BIC, missing purpose code). Ops repairs via UI → resubmit. Track **STP rate** (% of payments needing zero manual touch) — a headline KPI (target 95–99%).
- **Return** (`pacs.004`): beneficiary bank cannot apply funds → sends money back with reason code.
- **Recall/Cancellation** (`camt.056` → `camt.029`): sender requests cancellation; only succeeds if funds not yet credited or beneficiary consents.
- **Investigations**: camt.027 (claim non-receipt), camt.026 (unable to apply), camt.087 (request to modify), SWIFT gpi **gCase**.
- **Non-STP / NAK reprocessing**: automated retry for transient scheme errors; manual for business errors.
- **Duplicate handling**: dedupe on `(sender, MsgId)` or a business hash `(debtorAcct, creditorAcct, amount, ccy, valueDate, ref)` within an N-day window.

---

# PART 7 — RECONCILIATION

## 7.1 Types
1. **Nostro recon** — internal mirror vs correspondent's camt.053/MT940.
2. **Scheme/CSM recon** — our sent/received vs scheme settlement report.
3. **Internal recon** — ledger vs payment engine vs channel system.
4. **Suspense/clearing account recon** — must net to zero.
5. **Fee/FX recon**.

## 7.2 Matching engine design
```
Stage 1: Exact match on unique ref (UETR / EndToEndId / bank ref)     ~85–95%
Stage 2: Rule-based match (amount + ccy + date window ± 2d + counterparty)
Stage 3: One-to-many / many-to-one (bulk credit vs individual items)
Stage 4: Fuzzy match on narrative (regex extraction of refs)
Stage 5: Unmatched → investigation queue, aging buckets (T+1, T+3, T+30)
```
**Break categories:** timing differences (in-transit), amount differences (fees deducted by intermediary), missing entries, duplicates, wrong-account.

**Engineering points:** run as a bulk/batch job (Spring Batch / Spark) with partitioning by account+date; make matching **idempotent and re-runnable**; keep an audit of match decisions; support manual match/unmatch with maker-checker.

---

# PART 8 — STREAMING & EVENT-DRIVEN ARCHITECTURE

## 8.1 Why event-driven for payments
- Natural fit: a payment *is* a sequence of events.
- Decouples the 8–12 processing stages → independent scaling (screening is CPU-heavy, formatting is not).
- Durable log gives replay, audit, and recovery.
- Backpressure absorption for bulk file spikes (a 500k-record NACH file at 6 PM).

## 8.2 Kafka in payments — the details that matter

**Topic design**
```
payments.inbound.raw.v1
payments.canonical.v1
payments.validated.v1
payments.screening.request.v1 / .response.v1
payments.accounting.command.v1
payments.dispatch.v1
payments.status.v1        (status change events; compacted mirror: payments.status.latest)
payments.dlq.v1
```
Naming: `<domain>.<entity>.<event>.<version>`. Version in the topic name so you can run v1/v2 in parallel.

**Partitioning — the #1 interview question**
- Key by **paymentId** for per-payment ordering.
- Key by **accountId** when you need per-account ordering (balance/limit updates, position).
- Key by **debtorAccount** for velocity/fraud aggregation.
- Trade-off: account-keying creates **hot partitions** for corporate accounts pushing 100k payments. Mitigations: composite key `accountId#bucket` with application-level sequencing, or route hot accounts to a dedicated topic/consumer.

**Ordering guarantee** = only within a partition. Never assume global ordering. If cross-partition ordering matters, either (a) re-key, or (b) design consumers to be **order-insensitive** using version numbers/state machine guards (reject out-of-order transitions).

**Delivery semantics**
- Producer: `acks=all`, `enable.idempotence=true`, `max.in.flight.requests.per.connection<=5`, `retries=MAX`, `min.insync.replicas=2` with `replication.factor=3`.
- Consumer: `enable.auto.commit=false`, commit **after** business processing; this yields at-least-once → **your handler MUST be idempotent**.
- **Exactly-once (EOS)**: Kafka transactions (`transactional.id`, `read_committed`) work for Kafka→Kafka. But Kafka→DB is *not* covered → use **transactional outbox** or idempotent upsert.

**Transactional Outbox pattern (must know)**
```
BEGIN TX
  INSERT INTO payments ...            -- business state
  INSERT INTO outbox(id, aggregate_id, type, payload, created_at)
COMMIT
-- separate relay (Debezium CDC or polling publisher) reads outbox → Kafka
```
Solves the dual-write problem (DB commit + Kafka publish can't be atomic). Debezium reads the DB WAL/binlog → guaranteed at-least-once publish in commit order.
**Inbox pattern** on the consumer side: store processed `messageId` in a table with a unique constraint → dedupe.

**Consumer groups & scaling:** parallelism ≤ partition count. Over-provision partitions upfront (e.g. 60) since increasing partitions later breaks key→partition affinity.

**Retries & DLQ**
```
main-topic → (transient failure) → retry.5s → retry.30s → retry.5m → DLQ
```
- Non-blocking retry topics (Spring Kafka `@RetryableTopic`) avoid head-of-line blocking.
- Classify errors: **retryable** (timeout, 503, connection reset) vs **non-retryable** (validation error, business rejection) — never retry a business rejection.
- DLQ must carry: original payload, headers, exception, stack, attempt count, original topic/partition/offset. Needs an ops **replay tool** with dedupe protection.
- **DLQ alerting is mandatory** — a silent DLQ is how payments get lost.

**Other Kafka items:** log compaction for reference data / latest-state topics, tiered storage for retention, Schema Registry with **Avro/Protobuf** and **BACKWARD compatibility**, `MirrorMaker2`/cluster linking for DR, consumer lag monitoring (Burrow / `kafka_consumergroup_lag`).

**Kafka Streams / ksqlDB uses:** velocity windows for fraud, real-time position aggregation, enrichment joins with KTable reference data, dedupe windows, recon streaming joins.

## 8.3 Kinesis vs Kafka (if AWS shop)
| | Kafka (MSK) | Kinesis Data Streams |
|---|---|---|
| Unit | partition | shard |
| Retention | configurable, days–infinite (tiered) | 24h default, up to 365d |
| Ordering | per partition | per shard (per partition key) |
| Scaling | add partitions (rebalance) | shard split/merge, or On-Demand |
| Consumers | consumer groups | KCL + enhanced fan-out (2MB/s/consumer) |
| Ops | heavy (or MSK) | fully managed |
| Throughput limits | broker-bound | 1MB/s or 1000 rec/s write per shard |

Also: **SQS FIFO** (exactly-once-ish, message group ordering, 300 TPS/group or 3000 batched) for command queues; **EventBridge** for routing; **SNS** fan-out.

## 8.4 Event-Driven patterns
- **Event Notification** (thin event, consumer calls back) vs **Event-Carried State Transfer** (fat event, no callback — preferred in payments to avoid chatty coupling and to enable replay).
- **Choreography** (services react to events) — loose coupling, but the flow becomes implicit and hard to debug. Good for 2–4 steps.
- **Orchestration** (central saga orchestrator) — explicit flow, easier timeouts/compensation/visibility. **Preferred for payments** because you must answer "where is my payment right now?" instantly. Tools: Temporal, Camunda/Zeebe, AWS Step Functions, or a homegrown state machine + scheduler.

---

# PART 9 — DISTRIBUTED SYSTEMS FOR PAYMENTS

## 9.1 Consistency
- **CAP**: during a partition, payments almost always choose **CP for the ledger** (never double-spend) and **AP for read/reporting** views.
- **Strong consistency** required: balance/limit checks, ledger postings, dedupe, sequence numbers.
- **Eventual consistency** acceptable: notifications, reporting, analytics, search index, status dashboards.
- **2PC/XA**: technically available (JTA over DB + JMS), used in legacy bank stacks, but avoided in modern microservices — blocking coordinator, poor availability, bad latency. Mention you know it and why it's avoided.

## 9.2 Saga pattern (the standard answer)
Long-running transaction split into local transactions, each with a **compensating action**.

```
Reserve funds ──▶ Screen ──▶ Post ledger ──▶ Dispatch ──▶ Settle
     │              │            │              │
  release        (no comp)   reverse entry   recall/return
```
Key point: **compensation ≠ rollback**. Post-settlement you cannot undo; you issue a *return*. Also **semantic locks** (funds earmark) prevent double-spend while the saga runs.

## 9.3 Idempotency (asked in nearly every payments interview)
Layers of defense:
1. **Client-supplied `Idempotency-Key`** header (UUID), stored with a unique index; scoped to `(clientId, key)`; TTL 24h–7d.
2. Store the **response** against the key → replay returns the identical response (including the original HTTP status), not a new payment.
3. **Business-level dedupe hash**: `sha256(debtorAcct|creditorAcct|amount|ccy|valueDate|reference)` within a rolling window → catches genuine client double-submits without a key.
4. **Downstream idempotency**: `INSERT ... ON CONFLICT DO NOTHING`, conditional updates `UPDATE ... WHERE status='X' AND version=n`.
5. **Scheme-level**: UETR / EndToEndId reuse detection.

Race handling: rely on the DB unique constraint, not a read-then-write check. On duplicate-key exception, fetch and return the stored response. If the first request is still in-flight, return `409 Conflict` / `425 Too Early` with a retry hint.

```java
@Transactional
public PaymentResponse create(String idemKey, PaymentRequest req) {
    var hash = hash(req);
    try {
        idempotencyRepo.insert(idemKey, hash, IN_PROGRESS);   // unique index
    } catch (DuplicateKeyException e) {
        var rec = idempotencyRepo.get(idemKey);
        if (!rec.hash().equals(hash))
            throw new IdempotencyKeyReuseException();          // 422
        if (rec.status() == IN_PROGRESS) throw new InProgressException(); // 409
        return rec.response();                                 // replay
    }
    var resp = process(req);
    idempotencyRepo.complete(idemKey, resp);
    return resp;
}
```

## 9.4 Retries
- **Exponential backoff + full jitter** (`sleep = random(0, min(cap, base*2^n))`) to avoid thundering herd.
- **Retry budgets** — cap retries as a % of total traffic; unbounded retries cause retry storms and turn a brownout into an outage.
- **Circuit breaker** (Resilience4j): CLOSED → OPEN (fail fast) → HALF_OPEN (probe). Config: failure-rate threshold, sliding window, wait duration.
- **Bulkheads**: separate thread pools/connection pools per downstream so a slow sanctions engine can't exhaust all threads.
- **Timeouts everywhere** — connect, read, total. A missing timeout is the most common cause of cascading failure.
- **Never blindly retry a non-idempotent debit.** Retry only with the same idempotency key, or query status first (`pacs.028`, UPI `ReqChkTxn`) — this is the "**query-before-retry**" rule.

## 9.5 Other distributed concerns
- **Distributed locking**: Redis Redlock (caveats), ZooKeeper/etcd, or DB row locks / advisory locks. Prefer partitioned single-writer over distributed locks.
- **Leader election** for singleton jobs (EOD, cut-off trigger) — ShedLock, ZooKeeper, K8s lease.
- **Clock skew**: never order events by wall-clock across nodes; use sequence numbers, Kafka offsets, or Hybrid Logical Clocks.
- **Exactly-once end-to-end doesn't exist** — you get at-least-once delivery + idempotent processing = effectively-once.
- **Outbox + CDC** for reliable integration; **Saga** for consistency; **Event sourcing** for auditability.

---

# PART 10 — DATA & DATABASE DESIGN

## 10.1 Choices
| Store | Use |
|---|---|
| PostgreSQL / Oracle / DB2 | Core payment state, ledger — ACID, strong consistency |
| Cassandra / DynamoDB | High-volume append-only event/audit store, status history, huge scale |
| Redis | Idempotency cache, rate limits, reference data cache, distributed locks |
| Elasticsearch / OpenSearch | Payment search/investigation UI (by name, amount, ref) |
| S3 / HDFS + Parquet | Archive, regulatory retention (7–10 yrs), analytics |
| Snowflake/BigQuery/Redshift | Reporting, recon analytics |

## 10.2 Schema sketch
```sql
payments(
  payment_id UUID PK, uetr UUID UNIQUE, end_to_end_id, instruction_id,
  tenant_id, channel, direction, payment_type, rail,
  debtor_account, debtor_agent_bic, creditor_account, creditor_agent_bic,
  amount NUMERIC(23,5), currency CHAR(3),
  value_date DATE, settlement_date DATE,
  status VARCHAR(32), sub_status, status_reason_code,
  idempotency_key, dedupe_hash,
  version BIGINT,                        -- optimistic locking
  created_at TIMESTAMPTZ, updated_at TIMESTAMPTZ
) PARTITION BY RANGE (created_at);

payment_events(payment_id, seq BIGINT, event_type, payload JSONB, occurred_at,
               PRIMARY KEY(payment_id, seq));

ledger_postings(posting_id, transaction_id, account_id, dr_cr CHAR(1),
                amount NUMERIC, currency, value_date, booking_ts, source_event_id);
-- immutable; UNIQUE(source_event_id, account_id, dr_cr) for idempotency

outbox(id, aggregate_id, event_type, payload, created_at, published_at NULL);
idempotency(key PK, request_hash, status, response JSONB, created_at, expires_at);
```

**Indexing:** `(status, created_at)` for work queues, `(debtor_account, value_date)`, `uetr`, `end_to_end_id`, `dedupe_hash`. Beware: too many indexes kill write throughput on the hot table.

**Partitioning:** range by date (daily/monthly) → cheap archival via `DETACH PARTITION`; hash by account for large tenants.

**Hot/cold separation:** keep only in-flight + last N days in the OLTP table; move settled payments to an archive store. Payment tables grow to billions of rows; this is the single biggest performance lever.

**Locking:** optimistic (`version`) for payments; pessimistic (`SELECT ... FOR UPDATE`) only for balance rows, and always in a consistent lock order to avoid deadlocks.

## 10.3 Event Sourcing & CQRS

**Event sourcing:** state = fold(events). Store `PaymentInitiated, PaymentValidated, ScreeningCleared, FundsReserved, LedgerPosted, SentToScheme, AckReceived, Settled`.

Pros: perfect audit trail (regulator-friendly), temporal queries ("what did we know at 14:05?"), replay to rebuild projections, natural fit for the domain.
Cons: complexity, schema/event versioning (upcasting), snapshotting needed for long streams, eventual consistency in read models, harder onboarding, GDPR erasure conflicts with immutability (→ crypto-shredding: store PII encrypted, delete the key).

**CQRS:** write model = aggregate + event store; read models = denormalized projections (payment search index, ops dashboard, recon view, customer statement). Different scaling and schema per read model.

**Pragmatic answer for interviews:** "Full event sourcing for the payment aggregate, plus a materialized current-state table for queries. I'd apply CQRS at the boundary — a Postgres write model with an outbox feeding Elasticsearch/Cassandra read models. I would *not* event-source everything; reference data and config are better as CRUD."

**Snapshotting:** every N events store an aggregate snapshot to bound replay time.

---

# PART 11 — MICROSERVICES, APIs, INTEGRATION

## 11.1 Service decomposition (by bounded context, not by layer)
`payment-initiation`, `validation`, `enrichment`, `screening-adapter`, `fraud-adapter`, `limits`, `accounting/ledger`, `liquidity`, `routing`, `scheme-adapter-swift`, `scheme-adapter-sepa`, `scheme-adapter-upi`, `status-service`, `notification`, `recon`, `investigation/case`, `reference-data`, `reporting`.

Anti-patterns to call out: shared database across services, chatty synchronous chains (latency multiplies, availability multiplies down: 5 services @99.9% = 99.5%), distributed monolith (services that must deploy together).

## 11.2 API design
- **REST** for external/partner; **gRPC** for internal low-latency; **async webhooks/Kafka** for status.
- Resource design:
```
POST /v1/payments                 (Idempotency-Key header) → 201 + paymentId
GET  /v1/payments/{id}
GET  /v1/payments/{id}/status
GET  /v1/payments?status=&from=&to=&cursor=   (cursor pagination, not offset)
POST /v1/payments/{id}/cancel
POST /v1/payments/batch
POST /v1/payments/{id}/return
GET  /v1/accounts/{id}/balance
```
- **202 Accepted** for async processing + `Location` header for polling; plus webhook callbacks with HMAC signature and retry-with-backoff.
- **Versioning**: URI (`/v1`) for major, media type or field-level additive changes for minor. Never break: additive only, tolerant reader.
- **Error model**: RFC 7807 problem+json with a **stable machine-readable code** mapped to ISO reason codes.
- **Pagination**: cursor-based (keyset) — offset pagination collapses on billion-row tables.
- **Rate limiting**: token bucket per client, 429 + `Retry-After`.
- **Contract testing**: Pact / Spring Cloud Contract; schema registry for events; OpenAPI as source of truth.

## 11.3 File-based (still huge in banking)
- H2H SFTP, bulk pain.001 / NACH / BACS / positive-pay files.
- Concerns: **file-level idempotency** (checksum + filename + control record), **control totals** (record count + sum of amounts must match trailer), partial-failure semantics (accept-all-or-nothing vs per-record), PGP signing/encryption, watermarking, large file streaming (never load 1GB XML in memory — use StAX/SAX streaming parsing), chunked processing with Spring Batch + restartability.

---

# PART 12 — BATCH VS REAL-TIME

| | Batch | Real-time |
|---|---|---|
| Trigger | Schedule/file arrival | Event/API |
| Latency | Minutes–hours | ms–seconds |
| Throughput | Very high (millions/run) | High but per-message overhead |
| Error handling | Reject file/record, rerun | DLQ, retry, compensate |
| Examples | ACH, BACS, NEFT batches, EOD recon, interest accrual, statements | RTGS, UPI, RTP, FedNow, card auth |
| Tech | Spring Batch, Control-M/Autosys, Spark, COBOL/mainframe | Kafka, microservices, in-memory grids |

**Spring Batch specifics worth naming:** Job/Step/Chunk, `ItemReader/Processor/Writer`, partitioned steps + remote chunking for parallelism, `JobRepository` for restartability, skip/retry policies, `JobParameters` for idempotent reruns.

**Modern trend:** "**mini-batch as streaming**" — decompose the file at ingest into individual events on Kafka, process each through the same real-time pipeline, then re-aggregate for the scheme file. Gives one code path for both, better observability, per-record error isolation. Trade-off: loses natural batch-level atomicity → you must add a batch-tracking aggregate (`batchId`, expected count, completion detection).

---

# PART 13 — SECURITY

| Concern | Mechanism |
|---|---|
| Transport | TLS 1.2/1.3; **mTLS** for all bank-to-bank and internal service-to-service |
| Client auth | OAuth2 **client_credentials** (M2M), **authorization_code + PKCE** (user), FAPI profile for open banking |
| Token | **JWT** — validate `iss`, `aud`, `exp`, `nbf`, signature (RS256/ES256 via JWKS); short TTL; **never** put PAN/PII in claims; prefer opaque + introspection for high-security |
| Message integrity | **JWS** detached signature; SWIFT RMA + PKI; scheme-specific signing (e.g., UPI XML signature) |
| Non-repudiation | Digital signatures + immutable audit log + WORM storage |
| Data at rest | AES-256 (AWS KMS / Azure Key Vault / **HSM** for PIN & keys) |
| Card data | **Tokenization** (network tokens, PCI scope reduction), format-preserving encryption, PAN never logged |
| PIN | ISO 9564 PIN block, **DUKPT**, translated only inside HSM |
| Secrets | Vault / Secrets Manager, rotation, no secrets in config or env dumps |
| AuthZ | RBAC + **maker-checker (4-eyes)** for high-value release, segregation of duties |
| Replay protection | nonce + timestamp window + idempotency key |
| App sec | OWASP Top 10, input validation, XXE-safe XML parsing (**disable DTD** — critical for ISO 20022 XML), SQL injection prevention, dependency scanning (SCA), SAST/DAST |
| Network | Private subnets, VPC endpoints, WAF, allowlisted IPs, dedicated MPLS/leased line to SWIFT |
| Compliance | PCI-DSS, SWIFT **CSP/CSCF** attestation, SOC2, ISO 27001, GDPR/DPDP |
| Logging | **Mask PAN, IBAN, name, PIN, CVV**; log correlation IDs, not payloads |

**SCA / 3DS2:** risk-based authentication, exemptions (low value, TRA, whitelisting), frictionless vs challenge flow, liability shift.

---

# PART 14 — CLOUD, HA, DR

## 14.1 AWS mapping
| Need | Service |
|---|---|
| Compute | EKS / ECS Fargate, Lambda (for adapters/lightweight) |
| Streaming | MSK (Kafka), Kinesis, SQS FIFO, SNS, EventBridge |
| DB | Aurora PostgreSQL (Multi-AZ, Global DB), DynamoDB (global tables), ElastiCache Redis |
| Storage | S3 (+Object Lock/WORM for regulatory), Glacier |
| Orchestration | Step Functions, MWAA |
| Security | KMS, CloudHSM, Secrets Manager, IAM, PrivateLink, WAF/Shield |
| Observability | CloudWatch, X-Ray, OpenSearch, Managed Prometheus/Grafana |
| Batch | AWS Batch, EMR, Glue |
| Networking | Direct Connect to on-prem/SWIFT, Transit Gateway |

Azure equivalents: AKS, Event Hubs (Kafka-compatible), Service Bus, Azure SQL/Cosmos DB, Key Vault + Managed HSM, App Insights/Log Analytics, Data Factory, ExpressRoute.

## 14.2 High availability
- **Active-Active multi-AZ** minimum; **Active-Active or Active-Passive multi-region** for tier-0 payments.
- Stateless services + externalized state → horizontal scale.
- Kafka: RF=3 across AZs, `min.insync.replicas=2`, rack awareness.
- DB: synchronous replica in another AZ; async cross-region.
- Graceful degradation: if the fraud service is down → apply a conservative rule (hold high-value, allow low-value under a threshold) rather than failing everything. **Never** fail open on sanctions.
- Health checks: liveness vs readiness distinction; readiness should reflect downstream dependency health for critical deps only.

## 14.3 Disaster recovery
- **RPO** (data loss tolerance) — for payments typically **0 or near-0** → synchronous replication for the ledger.
- **RTO** (time to recover) — typically 15 min–2 h for tier-0.
- Strategies: backup/restore → pilot light → warm standby → **active-active** (choose active-active for payments).
- Must-haves: regular **DR drills / failover tests** (regulators require evidence), runbooks, data reconciliation post-failover, **split-brain protection** (fencing tokens, quorum), idempotency so replayed in-flight messages don't double-pay.
- **Message replay safety after failover is the crux**: you will re-deliver; if handlers are idempotent, you're fine. This is why idempotency is non-negotiable.

---

# PART 15 — OBSERVABILITY & OPERATIONS

## 15.1 Three pillars + payments-specific fourth
1. **Metrics** (Prometheus/Grafana, Dynatrace, CloudWatch)
2. **Logs** (Splunk, ELK/EFK, OpenSearch)
3. **Traces** (OpenTelemetry, Jaeger, Dynatrace PurePath, X-Ray)
4. **Business/flow monitoring** — the one that matters most in payments.

## 15.2 Metrics to define (say these in interviews)
**Technical:** throughput (TPS), latency p50/p95/**p99/p99.9**, error rate, consumer lag, DLQ depth, thread pool saturation, GC pause, DB connection pool usage, circuit breaker state.

**Business:**
- **STP rate** (% straight-through) — target ≥97%
- Payments by status (stuck-in-status counts with age buckets)
- **Repair queue depth & aging**
- Sanctions hit rate & false-positive rate
- Cut-off breach count / near-cut-off backlog
- Settled value vs expected value per rail
- Nostro break count and aging
- Return/reject rate by reason code
- Time-to-settle distribution per corridor
- Liquidity buffer utilization on instant rails

**Alerts that actually catch incidents:**
- "No payments received on rail X for 10 minutes during business hours" (silence detection — catches upstream outages that error-rate alerts miss).
- "Payments in `SENT_TO_SCHEME` older than 15 min > N".
- "DLQ non-empty".
- "Consumer lag > X and increasing for 5 min".
- "Suspense account non-zero at EOD".
- "Prefunded balance < 20% of daily average".

## 15.3 Tooling notes
- **Correlation/trace ID** propagated across HTTP headers, Kafka headers, and into logs (MDC in Spring). Also carry `paymentId` and `UETR` as first-class log fields — ops search by UETR.
- **Splunk**: SPL queries, dashboards for payment flow funnels, alerts. Common query pattern: `index=payments uetr=X | transaction uetr | table _time status service`.
- **ELK**: Filebeat → Logstash (grok) → Elasticsearch → Kibana; ILM for retention.
- **Dynatrace**: OneAgent auto-instrumentation, PurePath end-to-end traces, Davis AI anomaly detection, service flow maps, SLO burn-rate alerts. Widely used in banks — good to name.
- **Prometheus**: pull model, Micrometer in Spring Boot, `histogram_quantile()` for p99, recording rules, Alertmanager routing. Note the cardinality trap: **never label a metric with paymentId/accountId**.
- **SLO/SLI + error budgets**; on-call runbooks; blameless postmortems.

## 15.4 Structured log example
```json
{"ts":"...","level":"INFO","service":"screening","traceId":"a1b2","paymentId":"...",
 "uetr":"...","event":"SCREENING_COMPLETED","result":"CLEAR","durationMs":87,
 "listVersion":"OFAC-2026-08-30","debtorAccount":"****4321"}
```

---

# PART 16 — PERFORMANCE & SCALABILITY

**Where payments systems actually bottleneck:**
1. Sanctions screening (CPU + external vendor call) → cache clear results for repeat counterparties (carefully, with list-version invalidation), batch requests, scale horizontally.
2. Database hot rows (account balance) → partition by account, single-writer per partition, in-memory position with WAL.
3. Large XML parsing → StAX streaming, avoid DOM, reuse JAXB contexts (they're expensive to create, thread-safe once built; `Marshaller`/`Unmarshaller` are NOT thread-safe → pool them).
4. Synchronous chains → convert to async where regulation allows.
5. Chatty reference-data lookups → local caches (Caffeine) + Kafka-compacted topic for invalidation.
6. Cut-off spikes (80% of daily volume in the last hour) → autoscaling by consumer lag, pre-warmed capacity, queue-based load leveling.

**Techniques:** connection pooling (HikariCP sizing = `cores*2 + effective_spindles`, not 200), batching DB writes, prepared statements, async I/O (WebFlux/virtual threads), read replicas for queries, CQRS read models, avoiding N+1 queries, pagination, compression on Kafka (`lz4`/`zstd`), `linger.ms`+`batch.size` tuning, JVM tuning (G1/ZGC, heap sizing, off-heap caches), avoiding `BigDecimal` allocation churn in hot loops.

**Capacity planning example to quote:** "10M payments/day, 80% in a 4-hour window → ~550 TPS average, peak 3× → ~1,700 TPS. At 100 ms per payment per stage and 8 stages, with 200 ms budget headroom, I'd size ~60 Kafka partitions and 20–30 consumer pods per hot stage, and load-test to 3× peak."

---

# PART 17 — PRODUCTION ISSUES & TROUBLESHOOTING

| Issue | Likely cause | Approach |
|---|---|---|
| Payment stuck in `SENT_TO_SCHEME` | No ack; network/scheme outage; message rejected silently | Check scheme connectivity, MT012/019, gpi Tracker by UETR, send pacs.028 status request; never resend blindly |
| Duplicate payment sent | Retry without idempotency; failover replay; ops resubmit | Confirm via UETR/dedupe hash; raise recall (camt.056); fix idempotency at source |
| Consumer lag spiking | Slow downstream, poison message, GC, rebalance loop | Check DLQ, downstream latency, `max.poll.interval.ms` vs processing time, partition skew |
| Poison message / infinite retry | Non-retryable error retried | Classify errors, move to DLQ, add schema validation at ingest |
| Rebalance storm | Long processing > `max.poll.interval.ms`, frequent deploys | Reduce `max.poll.records`, increase interval, use cooperative-sticky assignor, static group membership |
| Nostro break | Fee deducted by intermediary, timing, missing entry | Match by UETR; check charge bearer; raise investigation |
| Cut-off missed | Backlog, screening queue, upstream late file | Prioritized queue, cut-off dashboard, expedite path, warehouse to next day + notify |
| Balance mismatch | Non-atomic posting, missing compensation, double posting | Reconcile ledger vs payments; check outbox unpublished rows; verify uniqueness constraint on postings |
| Sanctions engine down | Vendor outage | **Do not bypass.** Queue payments in HOLD, alert, invoke fallback secondary engine if contracted |
| High false positives after list update | New list version, tuning change | Compare list versions, review threshold config, whitelist replay |
| Slow API p99 | DB lock contention, pool exhaustion, GC, N+1 | Trace with APM, check DB waits, connection pool metrics, heap dump |
| Out-of-order status updates | Cross-partition events | Guard with state machine + event sequence/version; ignore stale transitions |
| Memory spike on file ingest | DOM parsing of big XML | Switch to StAX streaming, chunked processing |
| Timezone/date bug at month-end | `LocalDate.now()` on server TZ | Central business-date service, UTC storage, explicit scheme TZ |

**Structured troubleshooting narrative (use this in interviews):**
1. Assess blast radius (how many payments, what value, which rail, is money at risk?).
2. Stop the bleeding (pause consumer, disable rail, throttle) — **containment before diagnosis**.
3. Establish facts via UETR trace across services + scheme ack evidence.
4. Determine: are funds *actually* moved? (ledger + scheme confirmation, not our internal status).
5. Remediate: replay from DLQ (idempotent), manual repair, or recall.
6. Reconcile end-of-day; confirm suspense nets to zero.
7. Postmortem + guardrail (alert, test, code fix).

---

# PART 18 — DESIGN QUESTIONS (HLD)

## 18.1 "Design a payment gateway/processing system"
**Clarify:** volume/TPS, rails, sync or async, latency SLA, multi-currency, multi-tenant, consistency requirements, regulatory region.

**Answer skeleton:** API layer (idempotency, authN/Z, rate limit) → ingestion (persist raw + canonical, outbox) → Kafka → parallel validation/enrichment → sanctions (sync, blocking) → orchestrator saga → ledger (ACID, double-entry) → routing → scheme adapters → status service + webhooks → recon + DLQ + ops console.
**Emphasize:** idempotency, exactly-once-effect, state machine, audit immutability, DR, cut-offs.

## 18.2 "Design a distributed ledger / wallet system"
- Double-entry postings, immutable.
- Balance = snapshot + delta, or event-sourced with periodic snapshots.
- Per-account single-writer (partition by accountId) to avoid locks.
- Handle concurrent debits: `UPDATE balances SET bal = bal - :amt WHERE acct=:a AND bal >= :amt` (conditional update returns rows-affected; 0 ⇒ insufficient funds) — atomic, no read-modify-write race.
- Reservations/holds with expiry.
- Reconciliation job: sum(postings) must equal snapshot.

## 18.3 "Design UPI / instant payments"
- Sub-second SLA end-to-end → tight timeout budget per hop (e.g., 3s total: 300ms validate, 500ms screen, 500ms debit, 1s network).
- 24×7 → no maintenance window → rolling deploys, online schema changes, prefunded liquidity with auto-top-up.
- **Timeout ambiguity**: implement status-check reconciliation with the switch; deemed-approval and auto-reversal within TAT.
- Idempotency across retries with the same RRN/txnId.
- Very high TPS spikes (festival/sale) → horizontal scale, caching, circuit breakers, graceful shedding of low-priority traffic.

## 18.4 "Design cross-border payments"
- Corridor routing, correspondent selection (least-cost + speed + SLA), FX rate sourcing & margin, charge bearer (OUR/BEN/SHA), nostro funding, cut-offs per currency, sanctions on all parties, gpi tracking by UETR, MT/MX coexistence and truncation handling, investigations.

## 18.5 "Design a reconciliation system"
- Ingest sources (camt.053, scheme reports, internal ledger extract) → normalize → staged matching engine → break management → aging & escalation → auto-resolution rules → reporting. Must be idempotent and re-runnable for a given date.

## 18.6 "Design notifications / webhooks at scale"
- Outbox → Kafka → dispatcher; per-tenant rate limiting; exponential backoff retry with a max window; HMAC signature; ordering per subscriber (partition by subscriberId); dead-letter + manual replay; delivery receipt tracking.

## 18.7 "Design idempotent payment API" — covered in §9.3.

## 18.8 "Design fraud detection in real time"
- Kafka Streams windowed aggregations (velocity: N txns / M minutes per account/device/IP), feature store (Redis), rule engine + ML model scoring service with a strict latency budget, shadow mode for new models, feedback loop from confirmed fraud, explainability for regulators.

---

# PART 19 — LOW-LEVEL DESIGN (LLD)

## 19.1 Patterns you should be able to code
| Pattern | Payments use |
|---|---|
| **State** | Payment status transitions |
| **Strategy** | Per-rail routing/validation/fee calculation |
| **Chain of Responsibility** | Validation pipeline / processing stages |
| **Factory + Registry** | `SchemeAdapterFactory.get(rail)` |
| **Builder** | Immutable canonical payment construction |
| **Template Method** | Common processing skeleton, rail-specific hooks |
| **Observer / Pub-Sub** | Status change listeners |
| **Decorator** | Add retry/metrics/logging around adapters |
| **Adapter** | MT/MX/proprietary → canonical |
| **Specification** | Composable business rules |
| **Saga / Process Manager** | Orchestration |
| **Outbox / Inbox** | Reliable messaging |
| **Circuit Breaker / Bulkhead** | Resilience |
| **Money / Value Object** | Amount + currency, never a raw double |

## 19.2 Money type (classic LLD ask)
```java
public final class Money implements Comparable<Money> {
    private final long minorUnits;     // avoid float entirely
    private final Currency currency;

    public Money add(Money o) { requireSameCurrency(o);
        return new Money(Math.addExact(minorUnits, o.minorUnits), currency); }
    public Money multiply(BigDecimal rate, RoundingMode rm) { ... }
    // equals/hashCode on both fields; no default currency; immutable
}
```
Talk about: rounding rules (HALF_EVEN for financial), currency exponent, allocation/splitting without losing cents (largest-remainder), never comparing across currencies.

## 19.3 State machine
```java
enum Status { RECEIVED, VALIDATED, SCREENED, POSTED, SENT, SETTLED, REJECTED, RETURNED }

static final Map<Status, Set<Status>> ALLOWED = Map.of(
  RECEIVED,  EnumSet.of(VALIDATED, REJECTED),
  VALIDATED, EnumSet.of(SCREENED, REJECTED),
  SCREENED,  EnumSet.of(POSTED, REJECTED),
  POSTED,    EnumSet.of(SENT, REJECTED),
  SENT,      EnumSet.of(SETTLED, RETURNED),
  SETTLED,   EnumSet.noneOf(Status.class));   // terminal

void transition(Payment p, Status to, String reason) {
    if (!ALLOWED.getOrDefault(p.status(), Set.of()).contains(to))
        throw new IllegalStateTransition(p.status(), to);
    int rows = repo.updateStatus(p.id(), p.status(), to, p.version()); // CAS
    if (rows == 0) throw new ConcurrentModificationException();
    events.append(p.id(), new StatusChanged(p.status(), to, reason));
}
```
Note the **compare-and-swap update** — this makes concurrent duplicate consumers safe without locks.

## 19.4 Validation chain
```java
interface ValidationRule { Optional<Violation> validate(Payment p); }
List.of(new MandatoryFieldsRule(), new IbanChecksumRule(), new CurrencyCountryRule(),
        new AmountLimitRule(), new CutoffRule(), new SanctionsCountryRule())
    .stream().map(r -> r.validate(p)).flatMap(Optional::stream).toList();
```
Discuss: fail-fast vs collect-all (payments prefer **collect-all** so ops repairs everything in one pass), rule versioning, externalizing thresholds to config, and unit-testability.

## 19.5 IBAN validation (frequently asked coding question)
```java
boolean isValid(String iban) {
    String s = iban.replaceAll("\\s","").toUpperCase();
    if (s.length() < 15 || s.length() > 34) return false;
    s = s.substring(4) + s.substring(0,4);                 // move first 4 to end
    StringBuilder n = new StringBuilder();
    for (char c : s.toCharArray())
        n.append(Character.isDigit(c) ? c-'0' : c-'A'+10);
    // mod-97 in chunks to avoid BigInteger
    int rem = 0;
    for (char c : n.toString().toCharArray()) rem = (rem*10 + (c-'0')) % 97;
    return rem == 1;
}
```
Related: **Luhn** for card numbers, **mod-97-10** for Creditor Reference (RF), BIC format regex `^[A-Z]{6}[A-Z0-9]{2}([A-Z0-9]{3})?$`.

## 19.6 Other likely coding rounds
- Rate limiter (token bucket / sliding window).
- LRU cache.
- Producer-consumer with `BlockingQueue`.
- Concurrency: `ConcurrentHashMap.compute`, `AtomicLong`, `ReentrantLock`, `CompletableFuture` composition, virtual threads.
- Deduplicate a stream of transactions within a time window.
- Merge/match two sorted transaction lists (reconciliation).
- Design a scheduler for warehoused/future-dated payments.

---

# PART 20 — RAPID-FIRE Q&A

**Q: Clearing vs settlement?** Clearing exchanges instructions and computes obligations; settlement is the actual irrevocable transfer of funds.

**Q: RTGS vs NEFT/ACH?** RTGS = gross, immediate, irrevocable, high value; ACH/NEFT = netted batches, cheaper, delayed, used for bulk/low value.

**Q: MT103 vs MT202 vs MT202COV?** MT103 = customer credit transfer; MT202 = bank-to-bank transfer with no underlying customer detail; MT202COV = cover payment carrying the underlying customer info (required so intermediaries can screen).

**Q: pacs.008 vs pacs.009?** pacs.008 is FI-to-FI *customer* credit transfer (≈MT103); pacs.009 is FI-to-FI own/bank transfer (≈MT202).

**Q: What is UETR and why does it matter?** A UUIDv4 attached at origination and preserved across the whole chain; enables SWIFT gpi end-to-end tracking, deduplication, and investigation correlation. In your systems it's the best log/search key.

**Q: Nostro vs Vostro?** Same account, two viewpoints — "our account with them" vs "their account with us."

**Q: How do you guarantee no double payment?** Layered idempotency (client key + DB unique constraint + business dedupe hash + downstream conditional updates), CAS status transitions, and query-before-retry against the scheme. Accept at-least-once delivery, enforce effectively-once processing.

**Q: How do you handle a failure after debiting the customer but before sending?** Never rollback silently. Either complete forward (retry dispatch, since state is durable) or issue a compensating credit through the ledger with a linked reversal reference, and record both entries. The saga's compensation must itself be idempotent and auditable.

**Q: Why is exactly-once a myth?** Network failures make it impossible to distinguish "not delivered" from "delivered, ack lost." You can only combine at-least-once delivery with idempotent, deduplicating consumers.

**Q: Kafka ordering guarantee?** Only within a partition. Choose the key to match your ordering requirement, and design state transitions to reject stale/out-of-order events.

**Q: How do you handle a poison message?** Classify the error; non-retryable → DLQ immediately with full context; retryable → tiered retry topics with backoff; alert on DLQ depth; provide an idempotent replay tool.

**Q: What is the outbox pattern and why do you need it?** To make "update DB" and "publish event" atomic. Write both in one DB transaction; a CDC relay publishes the outbox rows. Avoids lost events and phantom events.

**Q: Saga vs 2PC?** 2PC gives atomicity but blocks and hurts availability; saga gives availability with eventual consistency plus explicit compensations. Payments use sagas plus semantic locks (fund earmarks).

**Q: CAP for a payment system?** Ledger side is CP — reject rather than risk a double-spend during partition. Reporting/notifications are AP.

**Q: How do you screen efficiently?** Deterministic fuzzy-matching engine, structured ISO 20022 party data, whitelists, list-version pinning for replayability, caching of stable counterparties, horizontal scale, and offline tuning to reduce false positives.

**Q: What's the STP rate and how do you improve it?** % of payments processed with zero manual intervention. Improve via better upstream validation, richer reference data, auto-repair rules, CoP/VoP, and analysing top repair reason codes.

**Q: How do you handle cut-off times?** Config-driven per currency/rail/correspondent in the scheme's timezone; a business-date service; pre-cut-off alerting; automatic warehousing with re-validation and re-screening on the release day.

**Q: Explain charge bearer.** OUR/DEBT — payer pays all charges; BEN/CRED — beneficiary bears them (deducted en route); SHA/SLEV — shared. Drives amount reconciliation differences on nostro.

**Q: Difference between EndToEndId and TransactionId?** EndToEndId is assigned by the debtor and must be passed unchanged to the creditor (used for the payer's reconciliation); TxId/InstrId are assigned by agents for their leg.

**Q: How do you test a payment system?** Unit + contract tests, testcontainers for Kafka/Postgres, scheme simulators/stubs (SWIFT simulator, scheme sandbox), golden-file tests for message mapping, property-based tests for amount/rounding, chaos tests (kill broker, delay downstream), replay of production-like traffic, date/cut-off simulation, and mandatory regression on all reason codes.

**Q: Zero-downtime deploy for a 24×7 payment rail?** Rolling/blue-green with backward-compatible schemas (expand-contract migrations), consumer group version-tolerant deserialization, feature flags, drain-then-stop consumers with graceful shutdown hooks, and idempotency so in-flight redelivery is safe.

**Q: How do you handle PII/GDPR with an immutable event store?** Crypto-shredding — encrypt PII fields with a per-subject key; delete the key to render data unrecoverable while preserving the event structure and hashes.

---

# PART 21 — TECH STACK CHEAT SHEET (say these fluently)

**Language/Framework:** Java 17/21 (records, sealed types, virtual threads), Spring Boot 3, Spring Cloud, Spring Batch, Spring Kafka, Spring Data JPA, WebFlux/reactive, Resilience4j, MapStruct, JAXB/StAX for ISO 20022, Prowide/JPMorgan libs for SWIFT MT parsing.

**Messaging:** Kafka/MSK, Confluent Schema Registry (Avro/Protobuf), Kafka Streams, IBM MQ (very common in banks for SWIFT connectivity), Solace, RabbitMQ, Kinesis, SQS.

**Data:** PostgreSQL/Oracle, Cassandra/DynamoDB, Redis, Elasticsearch, S3/Parquet, Flyway/Liquibase.

**Platform:** Docker, Kubernetes/OpenShift, Helm, Terraform, ArgoCD, Jenkins/GitHub Actions.

**Vendors banks actually run:** Finastra Global PAYplus, Volante VolPay, ACI Enterprise Payments, Icon IPF, Temenos, FIS Open Payment Framework, Form3, Thought Machine Vault (core), Fircosoft (screening), NICE Actimize (AML), CLS (FX settlement), Swift Alliance Access/Gateway, SWIFT Alliance Lite2, Microgen/Intellect for recon, SmartStream TLM (reconciliation).

---

# PART 22 — 10-DAY REVISION PLAN

| Day | Focus |
|---|---|
| 1 | Lifecycle, actors, clearing vs settlement, payment types |
| 2 | ISO 20022 message set + MT equivalents + reason codes |
| 3 | Rails: SWIFT/gpi, SEPA, TARGET2, Fedwire, CHAPS, RTP/FedNow, UPI |
| 4 | Nostro/vostro, settlement, liquidity, cut-offs, reconciliation |
| 5 | Screening/AML/fraud, exceptions, repair, investigations |
| 6 | Kafka deep dive: partitions, EOS, outbox, DLQ, Streams |
| 7 | Distributed systems: saga, idempotency, retries, consistency, HA/DR |
| 8 | Data: schema design, event sourcing, CQRS, ledger design |
| 9 | HLD practice (gateway, wallet, UPI, recon, cross-border) |
| 10 | LLD + coding (Money, state machine, IBAN, rate limiter) + rapid-fire Q&A |

---

## Final framing for interviews

When asked *anything* open-ended in a payments interview, structure your answer around these five pillars — it signals seniority immediately:

1. **Correctness** — double-entry, idempotency, no double-spend, reconciliation proves it.
2. **Auditability** — immutable events, who/what/when/why for every transition, replayable compliance decisions.
3. **Resilience** — at-least-once + idempotent, sagas with compensation, DLQ, circuit breakers, DR with RPO≈0.
4. **Compliance** — sanctions blocking and never bypassed, travel rule data, regulatory reporting, data residency.
5. **Operability** — status visibility by UETR, STP rate, stuck-payment alerts, repair tooling, cut-off dashboards.

> "In payments, being slow is an incident. Being wrong is a regulatory event. Design for wrong-never, slow-rarely."