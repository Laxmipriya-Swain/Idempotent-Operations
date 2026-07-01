# T6. Idempotent Operations

**Module:** Fault Tolerance  
**Difficulty:** Easy | **Duration:** 3 Days  
**Intern:** Laxmipriya Swain | **Supervisor:** Vasudha Tayade | **Captain:** Bhavitha Pathapalli

---

## Table of Contents
1. [Definition of Idempotency](#1-definition-of-idempotency)
2. [Examples](#2-examples)
3. [Non-Idempotent Risks](#3-non-idempotent-risks)
4. [Best Practices](#4-best-practices)
5. [Deliverables](#5-deliverables)
6. [References](#6-references)

---

## 1. Definition of Idempotency

An operation is **idempotent** if applying it **multiple times** produces the **same result** as applying it **once**.

**Formal definition:**
```
f(f(x)) = f(x)   for all x
```

In software: executing an operation once or N times must result in the **same final system state**. This is what makes retry logic safe — a client can resend a request without corrupting data or triggering unintended side effects.

**Key characteristics:**
- Same input always produces the same outcome
- No extra side effects on repeated execution
- Safe to retry without counters or guards
- System state after N executions = state after 1 execution

---

## 2. Examples

### ✅ Idempotent Operations

| Operation | Why Idempotent |
|---|---|
| `GET /users/123` | Read-only; never changes state |
| `PUT /users/123 {name: "Alice"}` | Sets resource to a fixed state; repeating changes nothing |
| `DELETE /resource` | Resource stays absent on all subsequent calls |
| `SET status = 'submitted'` | Assigning a fixed value is always safe to repeat |
| `INSERT ... ON CONFLICT DO NOTHING` | Duplicate inserts are silently ignored |

### ❌ Non-Idempotent Operations

| Operation | Why Non-Idempotent |
|---|---|
| `POST /payments` | Each call creates a new payment; retry = double charge |
| `INSERT INTO orders VALUES (...)` | Each call inserts a new row; retry = duplicate order |
| `UPDATE balance = balance - 100` | Each call deducts further; retry = incorrect deduction |
| Send email / push notification | Each call sends a new message; retry = spam |
| `INCREMENT counter` | Each call increases count; retry = over-count |

### Real-World Analogy

> **Light switch to ON** (already ON) → nothing changes ✅ Idempotent  
> **Doorbell press** → rings once more each time ❌ Non-idempotent

In software: setting session status to `"flagged"` is idempotent. Appending a new flag entry to a log on every check is not.

---

## 3. Non-Idempotent Risks

| Risk | Scenario | Impact |
|---|---|---|
| **Duplicate records** | Interview submission times out; client retries; two sessions created | Data corruption, wrong scoring |
| **Double charges** | Payment times out mid-flight; client retries; user charged twice | Financial loss, legal liability |
| **Repeated notifications** | Low-score alert fails; retry sends again | Poor UX, loss of trust |
| **Race conditions** | Two retries arrive together; both pass existence check before either commits | Duplicate DB records |
| **Cascading failures** | Retry storm hits overloaded service; each non-idempotent write amplifies failure | System-wide outage |
| **Audit issues** | Duplicate records make audit trails unreliable | Compliance failures |

---

## 4. Best Practices

### 4.1 Use Idempotency Keys

Assign a **UUID per logical operation** on the client. The server stores the key on first processing and returns the cached result on duplicates.

```
Client → POST /payments
         Header: Idempotency-Key: a1b2c3d4-e5f6-7890-abcd-ef1234567890

Server → Key exists? → return stored result (no re-execution)
         Key absent? → process → store key + result → return result
```

### 4.2 Use the Correct HTTP Methods

| Method | Idempotent? | Notes |
|---|---|---|
| `GET` | ✅ Yes | Read-only |
| `PUT` | ✅ Yes | Full replacement — always safe to repeat |
| `DELETE` | ✅ Yes | Resource stays absent |
| `PATCH` | ⚠️ Conditionally | Safe only with absolute values, not relative changes |
| `POST` | ❌ No | Use idempotency keys to make safe |

### 4.3 Safe Database Operations

```sql
-- UPSERT — safe to repeat
INSERT INTO sessions (id, status, user_id)
VALUES ('abc123', 'active', 42)
ON CONFLICT (id) DO UPDATE SET status = EXCLUDED.status;

-- Conditional update — only applies when expected state exists
UPDATE interviews
SET status = 'submitted'
WHERE id = 'xyz' AND status = 'pending';
```

### 4.4 Retry Policies

```
Retry 1 → wait 1s
Retry 2 → wait 2s
Retry 3 → wait 4s   (exponential backoff)
Retry 4 → wait 8s + random jitter (0–1s)
Max retries: 5
```

- Retryable: timeout, 503, 429
- Non-retryable: 400 Bad Request, 404 Not Found
- Log every retry with its idempotency key

### 4.5 Safe Side Effects

| Side Effect | How to Make Safe |
|---|---|
| Emails | Store `notification_key`; check before sending |
| Push notifications | Use event IDs; deduplicate at consumer |
| Webhooks | Include event IDs; consumers track processed IDs |
| Message queues | Use exactly-once delivery (Kafka idempotent producer, SQS FIFO) |

---

## 5. Deliverables

| Deliverable | Status |
|---|---|
| `README.md` — definition, examples, risks, best practices | ✅ Complete |
| Research documentation report (`.docx`) | ✅ Complete |

**Acceptance criteria met:**
- [x] Definition of idempotency clearly documented
- [x] Idempotent and non-idempotent examples provided
- [x] Non-idempotent risks explained with scenarios
- [x] Best practices documented with techniques and code

---

## 6. References

- [Stripe — Idempotent Requests](https://stripe.com/docs/idempotency)
- [AWS Builders' Library — Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- [RFC 7231 — HTTP/1.1 Method Idempotency](https://datatracker.ietf.org/doc/html/rfc7231#section-4.2.2)
- [PostgreSQL — INSERT ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html)
