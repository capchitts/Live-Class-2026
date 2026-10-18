# Database learning resources for Senior Engineer interviews

Use these resources in order, alongside the database course discussed. Start with the first two; you do not need to understand everything immediately.

| Order | Resource | What to focus on | Practice afterward |
|---|---|---|---|
| **1. SQL basics** | [SQLBolt interactive lessons](https://www.sqlbolt.com/) | `SELECT`, `INSERT`, `UPDATE`, `DELETE`, and creating tables. Skip this if you already write SQL comfortably. | Create a `time_entries` table and insert/query entries. |
| **2. Transactions** | [PostgreSQL: Transactions tutorial](https://www.postgresql.org/docs/18/tutorial-transactions.html) | `BEGIN`, `COMMIT`, `ROLLBACK`, and why related changes belong together. | Insert two records inside a transaction, then roll back. Verify neither remains. |
| **3. Database constraints** | [PostgreSQL: Constraints](https://www.postgresql.org/docs/18/ddl-constraints.html) | Read **Not-Null**, **Unique Constraints**, and **Primary Keys** first. | Add `UNIQUE(user_id, operation_key)` and attempt a duplicate insert. |
| **4. Concurrent transactions** | [PostgreSQL: Transaction Isolation](https://www.postgresql.org/docs/14/transaction-iso.html) | Start with the introduction and **Read Committed**. Return to Repeatable Read and Serializable later. | Open two database sessions and observe overlapping inserts. |
| **5. Idempotency** | [Stripe: Designing robust APIs with idempotency](https://stripe.com/blog/idempotency) | Why a failed connection leaves the outcome uncertain, and how retrying the same operation can be safe. | Explain the “database committed, response lost” scenario in your own words. |
| **6. Locks and deadlocks** | [PostgreSQL: Explicit Locking](https://www.postgresql.org/docs/17/explicit-locking.html) | Focus on **Row-Level Locks** and **Deadlocks**; leave the full lock-mode table for later. | Have two transactions update the same row and observe the waiting behavior. |
| **7. Retry policies** | [AWS: Timeouts, retries, and backoff with jitter](https://d1.awsstatic.com/builderslibrary/pdfs/timeouts-retries-and-backoff-with-jitter.pdf) | Timeouts, bounded retries, backoff, and jitter. | Explain why immediately retrying every failed request can worsen an outage. |

The PostgreSQL links cover different documentation versions. Use the documentation matching your installed version when checking exact behavior.

## Recommended learning path

**Transactions → unique constraints → concurrent requests → idempotency** is the most useful path for the interview questions discussed.

Use one small project throughout: a time-entry API. Gradually make it satisfy these requirements:

1. A valid request creates an entry.
2. A failed transaction leaves no partial records.
3. Retrying the same operation returns the original entry.
4. Two simultaneous retries create exactly one entry.
5. Reusing the key with different input produces an error.

## First study session

Start with the PostgreSQL transactions tutorial and its rollback exercise. Once you can explain what commit and rollback do, move on to unique constraints and concurrent requests.

For the broader preparation plan, see [Senior Engineer interview preparation](SENIOR_ENGINEER_PREPARATION.md).
