🎯 Step 1 --- What Happens When You Tap Your Card?
================================================

Let's walk through the real-world flow:

1.  Customer taps card at merchant terminal.

2.  Merchant sends payment request to acquiring bank.

3.  Acquirer sends to card network (Visa/Mastercard).

4.  Card network routes to issuing bank (customer's bank).

5.  Issuer:

    -   Checks balance

    -   Runs fraud checks

    -   Approves or declines

6.  Response flows back to merchant.

This is called **Authorization**.

Important:\
Money is NOT moved yet.

* * * * *

🧠 Core Concepts You Must Know
==============================

1️⃣ Authorization vs Capture
----------------------------

Authorization:

-   Reserve funds

-   Check validity

-   Temporary hold

Capture:

-   Actually move money

-   Happens later (often end of day batch)

> "Does authorization move money?"

> No. It places a hold. Settlement happens later.

* * * * *

2️⃣ Actors in the System
------------------------

For JPMC design, simplify to:

-   Merchant

-   Payment Gateway (your system)

-   Issuer (bank core system)

-   Card Network (abstracted)

We will focus on designing:

> The Payment Gateway system.

* * * * *

🎯 Step 2 --- Functional Requirements
===================================

> Design a card payment processing system.

You start with:

### Functional Requirements

-   Authorize a payment

-   Capture a payment

-   Refund a payment

-   Query payment status

-   Ensure idempotency

-   Handle duplicate requests

-   Store transaction history

* * * * *

🎯 Step 3 --- Non-Functional Requirements
=======================================

-   High availability

-   Low latency (authorization < 300ms ideally)

-   Strong consistency for transaction state

-   Idempotency (no double charge)

-   Auditability

-   Observability

-   Resilience under downstream failure

You must explicitly mention these.

* * * * *

🎯 Step 4 --- High-Level Architecture (Simple First)
==================================================

For now, keep it simple:

Client → Payment API → Payment Service → Database\
↓\
Issuer Bank API

We are not adding Kafka yet.

* * * * *

1️⃣ What data must we store for a payment authorization?
--------------------------------------------------------

For a payment authorization record, you typically store:

-   **payment_id** (internal ID)

-   **merchant_id / terminal_id**

-   **amount + currency**

-   **timestamp**

-   **card token** (not raw card number), plus **last4 + brand** for display

-   **authorization_id** from the network/issuer

-   **status** (AUTHORIZED / DECLINED / PENDING / EXPIRED)

-   **risk/fraud decision** (optional) + reason codes

-   **idempotency_key** (critical)

-   **trace/correlation_id** for audit/debug

-   Optional: **billing zip/postal** (sometimes), but not required everywhere

* * * * *

2️⃣ Why is idempotency critical in payment systems?
---------------------------------------------------

> Because retries are normal (timeouts, network glitches). Without idempotency, a retried authorization/capture could create **duplicate charges**. Idempotency ensures "same request" produces the same outcome and prevents double-charging.

* * * * *

3️⃣ What is the most dangerous failure scenario in card payments?
-----------------------------------------------------------------

For card payments, the biggest real danger is usually:

-   **Customer gets charged twice** (duplicate authorization/capture)\
    or

-   **Merchant gets paid without a valid capture** (incorrect settlement)

Because authorization and capture are distinct steps, and retries/timeouts can cause duplicates.

A classic worst-case scenario is:

> The issuer approved, but the gateway timed out before persisting the result, so the client retries and causes a second authorization/capture.

That's why idempotency + state machine matter.

* * * * *

4️⃣ Should authorization and capture use the same database table?
-----------------------------------------------------------------
Best practice:

-   Use **one payment/transaction table** representing the lifecycle (AUTH → CAPTURED → REFUNDED/VOIDED),\
    plus separate tables for **events/attempts**.

A common schema pattern:

-   `payments` (one row per payment intent): amount, merchant, token, status, idempotency_key

-   `payment_operations` (many rows): AUTH attempt, CAPTURE attempt, REFUND attempt with timestamps and results

-   `outbox_events` (later weeks): reliable event publishing

So: **Yes, they often share the same "payments" record**, because it's one lifecycle.
