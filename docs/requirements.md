# PayWallet: Requirements & Design (v1)

Version 1.1 · Status: Ready for build · Target: 4 weeks

## 1. Purpose

PayWallet is a backend-only wallet service. Customers hold a balance in INR, add money, and transfer it to other customers. Operations staff can reverse a transfer that failed or is disputed. There is no frontend: everything is a REST API.

The point of the project is correctness under real conditions: retries, concurrent requests, failures and audits. These are the problems payment backends are hired to solve.

## 2. Users and roles

| Role     | Who                | Can do                                                                       |
| -------- | ------------------ | ---------------------------------------------------------------------------- |
| CUSTOMER | End user           | Register, log in, view own wallet, top up, transfer, view own history        |
| OPS      | Support/operations | View any transaction, reverse a transfer                                     |
| SYSTEM   | Internal           | A seeded funding wallet that is the source of every top-up. It has no login. |

## 3. Scope

**In scope (v1):**

1. Register and log in with JWT; roles CUSTOMER and OPS
2. One wallet per user, created automatically at registration
3. Top-up (simulated, no real bank)
4. Transfer between wallets, safe against retries via an `Idempotency-Key`
5. Transaction history with pagination and filters
6. Reversal of a transfer by OPS
7. Double-entry ledger as the source of truth

**Out of scope:** frontend, real bank or UPI integration, KYC, multi-currency, notifications, Redis, Kafka, microservices. Redis and Kafka are only considered after v1 is done.

## 4. Glossary

- **Wallet:** an account holding a balance for one user.
- **Transaction:** one business operation (TOPUP, TRANSFER, REVERSAL) with a status.
- **Ledger entry:** one debit or credit line against one wallet. Every SUCCESS transaction produces exactly two entries (one debit, one credit) of equal amount. A FAILED transaction moves no money and produces none.
- **Idempotency key:** a client-generated unique value sent with a top-up or transfer. Sending the same request twice with the same key must not move money twice.
- **Minor units:** money stored as integer paise (₹1.00 = 100). Never use `double` or `float` for money.

## 5. Functional requirements

| ID   | Requirement                                                                                                                                                                                                                                                                                                                                                               |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FR-1 | A visitor can register with email and password. Registration creates the user and a wallet with zero balance in one database transaction. Registration always creates a CUSTOMER; OPS users are seeded by a migration and cannot self-register.                                                                                                                           |
| FR-2 | A user can log in and receives a JWT. All endpoints except register and login require a valid JWT.                                                                                                                                                                                                                                                                        |
| FR-3 | A CUSTOMER can view their own wallet and balance.                                                                                                                                                                                                                                                                                                                         |
| FR-4 | A CUSTOMER can top up their wallet. The amount is credited to the customer and debited from the SYSTEM funding wallet. A top-up requires an Idempotency-Key, with the same rules as FR-6.                                                                                                                                                                                 |
| FR-5 | A CUSTOMER can transfer an amount to another wallet by wallet ID.                                                                                                                                                                                                                                                                                                         |
| FR-6 | A top-up or transfer request must carry an `Idempotency-Key` header. A repeat with the same key and the same body returns the original result without moving money again. The same key with a different body is rejected with 409. This also holds when the original transfer FAILED: the same FAILED result is returned, and the client must use a new key to try again. |
| FR-7 | A CUSTOMER can list their own transactions, paginated, filterable by type, status and date range.                                                                                                                                                                                                                                                                         |
| FR-8 | OPS can view any transaction.                                                                                                                                                                                                                                                                                                                                             |
| FR-9 | OPS can reverse a SUCCESS transfer. The reversal is a new transaction with opposite ledger entries. The original is marked REVERSED.                                                                                                                                                                                                                                      |

## 6. Business rules

| ID    | Rule                                                                                                                                                                                                                                       |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| BR-1  | Amounts must be greater than zero and are integers in paise.                                                                                                                                                                               |
| BR-2  | A wallet balance can never go negative. No overdraft.                                                                                                                                                                                      |
| BR-3  | A user cannot transfer to their own wallet.                                                                                                                                                                                                |
| BR-4  | Maximum single transfer: ₹1,00,000. Maximum single top-up: ₹50,000. Both limits live in configuration, not in code.                                                                                                                        |
| BR-5  | A transfer can be reversed only once, and only if the receiver still has enough balance to cover it. Otherwise the reversal is rejected.                                                                                                   |
| BR-6  | Idempotency keys are scoped per user.                                                                                                                                                                                                      |
| BR-7  | Ledger entries are never updated or deleted. Corrections are made by new entries.                                                                                                                                                          |
| BR-8  | A wallet balance must always equal the sum of its ledger entries (credits minus debits).                                                                                                                                                   |
| BR-9  | A top-up or transfer that breaks a rule (insufficient funds, limit exceeded) is stored as a FAILED transaction with a reason and no ledger entries. A rejected reversal is not stored: it returns an error and the original stays SUCCESS. |
| BR-10 | Only OPS can reverse, and only transfers. Top-ups cannot be reversed in v1.                                                                                                                                                                |

## 7. Non-functional requirements

- **Correctness under concurrency:** two simultaneous transfers from the same wallet must never overdraw it or lose an update. This is proven by an automated test, not by reasoning.
- **Consistency:** a transaction row and its two ledger entries are written in a single database transaction. Either everything is saved or nothing is.
- **Auditability:** every transaction records who initiated it and when. All timestamps are stored in UTC.
- **Security:** passwords hashed with BCrypt. No secrets in the repository. All configuration through environment variables. JWTs expire after 15 minutes; there are no refresh tokens in v1.
- **Observability:** structured logs that include a correlation ID per request. Health endpoint available.
- **Testing:** service-layer unit tests, plus integration tests against a real PostgreSQL.
- **Reproducibility:** one command (`docker compose up`) starts the app and database.

## 8. Domain model

**users**: id (UUID), email (unique), password\_hash, role (CUSTOMER or OPS), created\_at

**wallets**: id (UUID), user\_id (unique, nullable only for the SYSTEM wallet), balance\_minor (BIGINT, CHECK >= 0 except SYSTEM), created\_at

**transactions**: id (UUID), type (TOPUP, TRANSFER, REVERSAL), status (PENDING, SUCCESS, FAILED, REVERSED), from\_wallet\_id, to\_wallet\_id, amount\_minor (BIGINT, CHECK > 0), idempotency\_key (required for TOPUP and TRANSFER, null for REVERSAL), request\_hash (hash of the request body, to detect a reused key with a different body), initiated\_by (user id), reversal\_of (nullable transaction id), failure\_reason, created\_at. Unique constraint on (initiated\_by, idempotency\_key); PostgreSQL ignores rows where the key is null.

**ledger\_entries**: id, transaction\_id, wallet\_id, direction (DEBIT or CREDIT), amount\_minor, balance\_after\_minor, created\_at

The `balance_minor` column on wallets is a cached value for speed. The ledger is the source of truth, and BR-8 is verified by a test.

## 9. Transaction lifecycle

```
PENDING --> SUCCESS --> REVERSED
   |
   +------> FAILED
```

- A transfer starts PENDING, becomes SUCCESS when both ledger entries are written, or FAILED with a reason (for example insufficient funds).
- Only SUCCESS transfers can become REVERSED (BR-5).
- FAILED and REVERSED are terminal.

### Transfer flow

One request, one database transaction:

1. Reject a missing `Idempotency-Key` with 400.
2. Look up (initiated\_by, key). Found with the same `request_hash`: return the stored result and stop. Found with a different hash: return 409.
3. Insert the transaction as PENDING. The unique constraint makes this the claim on the key: if two identical requests arrive together, one insert fails, and that request re-reads the winner's result and returns it.
4. Lock both wallet rows, in wallet ID order.
5. Check BR-1 to BR-4. If a rule fails, mark the transaction FAILED with a reason, write no ledger entries, commit and return 422 (BR-9).
6. Write the debit and credit entries, update both cached balances, mark SUCCESS, commit.

An unexpected technical error rolls back everything, including the PENDING row, so the client can safely retry with the same key.

### Worked example

Wallet A holds ₹1,000.00 and wallet B holds ₹200.00. A sends B ₹300.00. B then spends ₹400.00 elsewhere, and OPS tries to reverse the first transfer.

| Step                  | Transaction                                                     | Ledger entries (paise)                                                    | Balances after (paise) |
| --------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------- | ---------------------- |
| Start                 | none                                                            | none                                                                      | A 100000, B 20000      |
| A sends B ₹300        | T1 TRANSFER, SUCCESS                                            | A DEBIT 30000 (balance after 70000); B CREDIT 30000 (balance after 50000) | A 70000, B 50000       |
| B sends C ₹400        | T2 TRANSFER, SUCCESS                                            | B DEBIT 40000 (balance after 10000); C CREDIT 40000                       | B 10000                |
| OPS reverses T1       | request rejected: REVERSAL\_NOT\_ALLOWED (BR-5), nothing stored | none                                                                      | unchanged              |
| B tops up ₹250        | T3 TOPUP, SUCCESS                                               | B CREDIT 25000; SYSTEM DEBIT 25000                                        | B 35000                |
| OPS reverses T1 again | T4 REVERSAL, SUCCESS; T1 becomes REVERSED                       | B DEBIT 30000 (balance after 5000); A CREDIT 30000 (balance after 100000) | A 100000, B 5000       |

In every row that moves money, the debit and the credit are equal, and each wallet's balance equals the sum of its ledger entries (BR-8).

## 10. API (v1)

| Method | Path                                   | Role                        | Notes                                                 |
| ------ | -------------------------------------- | --------------------------- | ----------------------------------------------------- |
| POST   | /api/v1/auth/register                  | public                      | creates user and wallet                               |
| POST   | /api/v1/auth/login                     | public                      | returns JWT                                           |
| GET    | /api/v1/wallets/me                     | CUSTOMER                    | own wallet and balance                                |
| POST   | /api/v1/wallets/me/topups              | CUSTOMER                    | header Idempotency-Key; body: amountMinor             |
| POST   | /api/v1/transfers                      | CUSTOMER                    | header Idempotency-Key; body: toWalletId, amountMinor |
| GET    | /api/v1/transactions                   | CUSTOMER                    | page, size, type, status, from, to                    |
| GET    | /api/v1/transactions/{id}              | CUSTOMER (own) or OPS (any) |                                                       |
| POST   | /api/v1/ops/transactions/{id}/reversal | OPS                         |                                                       |

**Standard error body** for every failure:

```
{ "code": "INSUFFICIENT_FUNDS", "message": "...", "timestamp": "...", "path": "..." }
```

| Code                                         | HTTP | When                          |
| -------------------------------------------- | ---- | ----------------------------- |
| VALIDATION\_ERROR                            | 400  | malformed or invalid input    |
| UNAUTHENTICATED                              | 401  | missing or invalid JWT        |
| FORBIDDEN                                    | 403  | role or ownership check fails |
| WALLET\_NOT\_FOUND / TRANSACTION\_NOT\_FOUND | 404  | unknown ID                    |
| EMAIL\_ALREADY\_REGISTERED                   | 409  | duplicate email               |
| IDEMPOTENCY\_KEY\_CONFLICT                   | 409  | same key, different body      |
| INSUFFICIENT\_FUNDS                          | 422  | BR-2                          |
| LIMIT\_EXCEEDED                              | 422  | BR-4                          |
| REVERSAL\_NOT\_ALLOWED                       | 422  | BR-5                          |

## 11. Key design decisions (to be explained back)

1. **Money is stored as integer paise.** Decimal and floating-point rounding errors are unacceptable for money.
2. **Ledger plus cached balance, written together.** The ledger gives auditability. The cached balance gives fast reads. A test checks they always agree.
3. **Package by feature**, not by layer: `auth`, `wallet`, `transaction`, `ledger`, `common`. Each feature owns its controller, service, repository and DTOs.
4. **Schema managed by Flyway migrations**, not by Hibernate auto-update. The schema is versioned like code.
5. **Concurrency strategy (decided in week 2):** lock both wallet rows inside the transaction, always in the same order (by wallet ID), so two transfers can neither overdraw nor deadlock each other.
6. **Idempotency enforced by the database.** The unique constraint on (initiated\_by, idempotency\_key) is the safety net, not only application code.

## 12. Backlog: first five tickets

**PAY-1: Project skeleton** Acceptance criteria:

- Spring Boot (Java 21, Maven) with a feature-based package layout
- `docker-compose.yml` runs PostgreSQL; configuration comes from environment variables, no passwords in Git
- `GET /actuator/health` returns UP
- README explains how to run the app

**PAY-2: Database schema (Flyway V1)** Acceptance criteria:

- Tables from section 8 created by a migration, with CHECK and UNIQUE constraints as specified
- SYSTEM funding wallet seeded by a migration, plus a development-only OPS user
- An integration test proves the app starts against a real PostgreSQL and the migration runs

**PAY-3: Registration** Acceptance criteria:

- `POST /auth/register` creates user and zero-balance wallet in one transaction (FR-1)
- Password stored as a BCrypt hash; email validated; duplicate email returns EMAIL\_ALREADY\_REGISTERED
- Tests: success, duplicate email, invalid input, and a rollback test showing no orphan user if wallet creation fails

**PAY-4: Login and JWT** Acceptance criteria:

- `POST /auth/login` returns a JWT; wrong password returns 401
- Every endpoint except register and login requires a valid token
- `GET /wallets/me` returns only the caller's wallet
- Tests: valid token, missing token, tampered token

**PAY-5: Top-up with ledger** Acceptance criteria:

- `POST /wallets/me/topups` creates a TOPUP transaction and two ledger entries (credit customer, debit SYSTEM)
- BR-1 and BR-4 enforced; Idempotency-Key required, and a retry with the same key credits only once
- A test verifies BR-8: balance equals the ledger sum after several top-ups

## 13. Backlog: later tickets (details written when we reach them)

PAY-6 transfer happy path · PAY-7 insufficient funds and limits · PAY-8 idempotency · PAY-9 concurrency test and fix · PAY-10 transaction history with pagination and filters · PAY-11 OPS reversal · PAY-12 change request (daily transfer limit) · PAY-13 bug ticket (planted, to be reproduced with a test first) · PAY-14 Dockerfile, Compose and CI · PAY-15 deployment, logging and README

## 14. Working agreement

- One ticket means one branch, one small pull request, one merge.
- The owner writes the code first. The reviewer comments and requests changes. Code is not dictated.
- A ticket is done only when: acceptance criteria met, tests pass, CI is green (once it exists), and the README or API docs are updated.
- Scope changes only through an explicit change-request ticket. Nothing is added silently.

## 15. Open questions for the owner

Questions 1 to 4 were answered and reviewed before PAY-1 and led to the changes listed in section 16. Question 5 stays open: while building, flag any unclear requirement by raising a ticket instead of guessing.

1. Why is money an integer in paise and not a `double`?
2. Why does a transfer create two ledger entries instead of just updating two balances?
3. What exactly happens if the client sends the same transfer request twice?
4. Why can a reversal be rejected, and why is it a new transaction instead of editing the old one?
5. Which requirement in this document do you find least clear?

## 16. Document history

| Version | Changes                                                                                                                                                                                                                                                                                                        |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.1     | Added a transfer flow and a worked example (section 9), request\_hash, Idempotency-Key on top-ups, OPS seeding, FAILED handling (BR-9), reversal scope (BR-10), JWT expiry and UTC timestamps. Clarified that ledger entries exist only for SUCCESS transactions and how a retry of a FAILED transfer behaves. |
| 1.0     | Initial draft.                                                                                                                                                                                                                                                                                                 |
