1️⃣ API Design --- Core Endpoints
===============================

### 🔹 Authorize Payment

`POST /payments/authorize`

Request:

`{
  "merchantId": "m123",
  "amount": 100.00,
  "currency": "USD",
  "cardToken": "tok_abc123",
  "idempotencyKey": "unique-request-id"
}`

Response:

`{
  "paymentId": "pay_001",
  "status": "AUTHORIZED",
  "authorizationId": "auth_789",
  "approvedAmount": 100.00
}`

Important:

-   Client supplies idempotency key

-   We return stable paymentId

* * * * *

### 🔹 Capture Payment

`POST /payments/{paymentId}/capture`

Request:

`{
  "amount": 100.00,
  "idempotencyKey": "capture-request-id"
}`

Response:

`{
  "paymentId": "pay_001",
  "status": "CAPTURED"
}`

* * * * *

### 🔹 Refund Payment

`POST /payments/{paymentId}/refund`

* * * * *

### 🔹 Query Payment Status

`GET /payments/{paymentId}`

* * * * *

2️⃣ Payment State Machine (VERY IMPORTANT)
==========================================

This is where mid-level candidates shine.

A payment is not just a row.\
It is a lifecycle.

### Basic State Flow

`CREATED
   ↓
AUTHORIZED
   ↓
CAPTURED
   ↓
REFUNDED (optional)`

Also possible:

`CREATED → DECLINED
AUTHORIZED → VOIDED
AUTHORIZED → EXPIRED`

Important rule:

> State transitions must be validated and atomic.

You cannot:

-   Capture if not AUTHORIZED.

-   Refund if not CAPTURED.

This is critical.

* * * * *

3️⃣ Minimal Database Schema
===========================

### 🔹 payments table

| Column | Purpose |
| --- | --- |
| payment_id (PK) | Internal ID |
| merchant_id | Who initiated |
| amount | Requested amount |
| currency | Currency |
| card_token | Token reference |
| status | Current state |
| authorization_id | From issuer |
| idempotency_key | For request dedupe |
| created_at | Audit |
| updated_at | Audit |
| version | Optimistic locking |

Important:

-   version column for concurrency control

-   idempotency_key unique constraint

* * * * *

### 🔹 payment_operations table (optional but realistic)

Tracks every action:

| id | payment_id | operation_type | status | created_at |

So we have history.

* * * * *

4️⃣ Idempotency Strategy (First Layer)
======================================

For each API call:

-   Store idempotency_key with payment_id

-   Add unique constraint on (merchant_id, idempotency_key)

If duplicate request arrives:

-   Return previous result

-   Do NOT reprocess

This prevents double charge.

* * * * *

5️⃣ Atomic State Transition
===========================

When capturing:

```java
UPDATE payments
SET status = 'CAPTURED'
WHERE payment_id = ?
  AND status = 'AUTHORIZED';
```

If rows affected = 0:

-   Either already captured

-   Or invalid state

This prevents race conditions.

1️⃣ Why must state transitions be validated at the database level, not just application level?
----------------------------------------------------------------------------------------------
> Because the system may run multiple threads and multiple instances. Application-level checks are not enough to prevent race conditions across processes. The database is the shared source of truth, so state transitions must be enforced atomically in the DB (e.g., conditional updates) to guarantee correctness under concurrency.

Key phrase: **atomic conditional update**.

* * * * *

2️⃣ Where should idempotency_key uniqueness be enforced --- app or DB? Why?
-------------------------------------------------------------------------

> Enforce uniqueness in the database with a unique constraint (e.g., `(merchant_id, idempotency_key)`). App-level checks can race under concurrency. The DB constraint provides a single, authoritative gate that guarantees "at most once effect" even with retries, duplicate requests, and multiple instances.

Key phrase: **single authoritative gate**.

* * * * *

3️⃣ Why do we use a version column in payments table?
-----------------------------------------------------

A version column is mainly for:

-   **optimistic locking** (detect concurrent updates)

-   preventing lost updates

-   safe state transitions under concurrency

> We use a version column for optimistic locking so concurrent updates don't overwrite each other. Each update checks the expected version and increments it, allowing us to detect and reject conflicting writes (e.g., two workers trying to transition the same payment state).

Example pattern:

-   read version = 5

-   update `... WHERE payment_id=? AND version=5` and set version=6

-   if 0 rows updated → conflict happened

* * * * *

4️⃣ What happens if two capture requests arrive simultaneously?
---------------------------------------------------------------

> If we implement capture as an atomic conditional update (e.g., `UPDATE ... WHERE status='AUTHORIZED'` or `WHERE version=?`), only one request will successfully transition the payment to CAPTURED. The other will update 0 rows and should return an idempotent response (already captured) or a conflict depending on semantics.
