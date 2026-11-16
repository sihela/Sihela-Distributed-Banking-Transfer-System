# 🏦 Distributed Banking Transfer System

A three-tier banking system built in Python, where customers log in and transfer money between accounts over **Pyro5 RMI**, backed by a **SQLite** data tier. The focus is on what makes distributed money movement hard: **crashes, retries, duplicate requests, concurrency and exact arithmetic.**

> **📌 Showcase repository.** This was built for a university Distributed Systems unit (CSI3344), so the source code is kept private to respect academic integrity. This repo documents the architecture, design decisions and results.

![Python](https://img.shields.io/badge/Python-3.10%2B-blue)
![Pyro5](https://img.shields.io/badge/RPC-Pyro5-informational)
![SQLite](https://img.shields.io/badge/DB-SQLite%20(WAL)-lightgrey)
![Tests](https://img.shields.io/badge/tests-152%20passing-brightgreen)

---

## ✨ Highlights

- **Three-tier architecture** with strict isolation: client, application server and database server are separate processes.
- **Idempotent transfers:** retrying a request never double-charges.
- **Crash-safe two-stage transfers** (`PENDING` → `COMPLETED`/`FAILED`) with automatic recovery.
- **No floating-point money anywhere:** all values are integer cents.
- **Tiered fee engine** with exact integer rounding and per-transfer caps.
- **Secure by design:** PBKDF2 password hashing, expiring session tokens, a shared-secret handshake between tiers.
- **152 automated tests**, including a contract suite proving three storage backends behave identically.

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────┐
│  Tier 1: Banking Client (BC)                 │
│  Interactive terminal UI, retry loop,        │
│  idempotency key per transfer                │
└───────────────────┬──────────────────────────┘
                    │  Pyro5 RMI
┌───────────────────▼──────────────────────────┐
│  Tier 2: Bank Application Server (BAS)       │
│  Auth & sessions, validation, fee calc,      │
│  transfer lifecycle, background reconciler   │
└───────────────────┬──────────────────────────┘
                    │  Pyro5 RMI + HMAC handshake secret
┌───────────────────▼──────────────────────────┐
│  Tier 3: Bank Database Server (BDB)          │
│  SQLite (WAL), ACID transactions,            │
│  DB-level constraints, audit log, CSV export │
└──────────────────────────────────────────────┘
```

The client can never reach the database, and the application server never touches SQLite directly. Each boundary is a network call.

The BAS also supports a **Phase 1 two-tier mode** that uses a local JSON state file instead of the BDB. It was the first iteration of the assignment and is kept to show the evolution of the design.

---

## 🧠 Key Design Decisions

### 1. Money is integers, always
Every amount, fee, debit and balance is a 64-bit integer number of **cents**. User input is parsed with a strict ASCII-only pattern, which rejects `1e3`, `NaN`, `-5`, `1,000` and even look-alike Unicode digits (Arabic, Devanagari, full-width). Rounding is half-up using integer arithmetic only.

### 2. Idempotency
The client generates a unique `request_id` (UUID) for each transfer. The database enforces `UNIQUE(sender_account_id, request_id)`, so if a network timeout makes the client retry, the server returns the *original* result instead of moving money twice.

### 3. Two-stage transfers and crash recovery
A transfer is first recorded as `PENDING`, then settled atomically in one database transaction. Anything that goes wrong in between is cleaned up three ways:

| Mechanism | When it runs |
|---|---|
| **Startup recovery** | On BAS restart, any stranded `PENDING` transfers are settled |
| **Background reconciler** | A daemon thread periodically settles stragglers and purges expired sessions |
| **Settle-on-read** | Asking for the status of a stale `PENDING` transfer settles it on the spot |

### 4. Defence in depth
- **Passwords:** PBKDF2-HMAC-SHA256 (200k iterations, per-user salt), constant-time comparison.
- **User enumeration resistance:** unknown usernames still pay the cost of a hash, so response time doesn't reveal whether an account exists.
- **Sessions:** random URL-safe tokens with a sliding 15-minute expiry. The sender is always derived from the token, **never** from client input.
- **Inter-tier trust:** the BDB rejects any connection that doesn't present the shared secret (checked with a constant-time compare).
- **Database constraints:** `CHECK (balance_cents >= 0)` acts as a last line of defence even if application logic had a bug.
- **Privacy in views:** recipients only see completed incoming transfers, and the sender's fee details are hidden from them.

### 5. Backend contract parity
One shared `StorageContract` test suite runs against three interchangeable backends: **Local (JSON)**, **SQLite** and **Remote (RPC to BDB)**. If they all pass the same contract, the layers above can be swapped without behavioural surprises.

---

## 💸 Fee Schedule

The sender pays the fee: `debit = amount + fee`, `credit = amount`. The fee is the tier rate, rounded half-up to the cent, then capped.

| Transfer amount | Rate | Fee cap |
|---|---|---|
| Up to $2,000 | Free | n/a |
| Up to $10,000 | 0.25% | $20 |
| Up to $20,000 | 0.20% | $25 |
| Up to $50,000 | 0.125% | $40 |
| Up to $100,000 | 0.08% | $60 |
| Above $100,000 | 0.06% | $250 |

---

## 🗄️ Data Model

`users` · `accounts` · `transfers` · `fees` · `audit_log`

Every login, logout, transfer creation, completion, failure and rejection is written to the audit log. Tables can be exported to CSV with a single command.

---

## 🔌 Error Handling

All API responses use a uniform `{ok, data}` / `{ok: false, error, message}` envelope with a fixed set of error codes: `INVALID_INPUT`, `AUTH_FAILED`, `INVALID_TOKEN`, `INVALID_RECIPIENT`, `SELF_TRANSFER`, `INSUFFICIENT_FUNDS`, `TRANSFER_NOT_FOUND`, `SERVICE_UNAVAILABLE`, `INTERNAL_ERROR`.

If the database goes down, clients get a friendly `SERVICE_UNAVAILABLE` rather than a stack trace. The BAS-to-BDB link retries once on connection failure and reports a clear message if the shared secret is misconfigured. All servers are configured via environment variables with validated bounds.

---

## 🧪 Testing

**152 tests** run with `python -m unittest discover -s tests -v`, covering:

- Fee tier boundaries and rounding edge cases
- Storage contract parity across Local, SQLite and Remote backends
- Concurrent transfers and race conditions
- Idempotent retries
- Crash recovery and settle-on-read
- Network fault tolerance (server down, timeouts, wrong secret)
- Client UI behaviour, using injected input/output so the interactive flow is testable without a terminal

---

## 🖥️ Demo (illustrative)

```
=== Bank Client ===
  1. Log in
  2. Exit
Choose an option: 1
Username: alice
Password: ********

=== Main Menu (ACC1001) ===
  1. View balance
  2. Transfer money
  3. Transaction history
  4. Log out
Choose an option: 2
Recipient account ID: ACC1002
Amount: 5000.00
Reference: Rent

  Transfer TXN-000001 COMPLETED
  Amount: $5,000.00   Fee: $12.50   Total debited: $5,012.50
```

*(Layout is representative; screenshots can be added to `/docs`.)*

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3.10+ |
| RPC | Pyro5 (RMI over TCP) |
| Database | SQLite (WAL mode, `BEGIN IMMEDIATE` transactions) |
| Crypto | `hashlib` PBKDF2-HMAC-SHA256, `hmac`, `secrets` |
| Testing | `unittest` |

---

## 📚 What I Learned

- Designing for **at-least-once delivery with idempotent handling** instead of hoping the network behaves.
- Why **money must never be a float**, and how to keep rounding deterministic.
- Making multi-step operations **recoverable** after a crash at any point.
- Using **contract tests** to keep multiple implementations of the same interface honest.
- Treating each tier as untrusted by the next: validate everywhere, derive identity server-side.

---

## 📬 Code Access

The code is private because this is assessed coursework. If you're a recruiter or interviewer, I'm happy to walk you through the codebase live or share it privately on request.

**Contact:** `<your-email>` · `<your-linkedin>`

---
