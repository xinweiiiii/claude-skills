---
name: structure-database
displayName: Structuring SQL Databases
description: Habits for shaping relational schemas so a design that looks correct at launch does not become expensive at scale. Consult while designing tables, keys, indexes, migrations, and background jobs — before the schema ships, not after growth makes it expensive to change.
version: 1.0.0
---

# Structuring Databases That Survive Scale

Reference for how to shape a relational schema. Consult before implementing, not after.

A design that is correct at five thousand rows can be expensive at forty million — soft deletes, random UUID keys, full normalization, and an index per slow query all fall into this trap. Rolling back a primary key strategy on a forty-million-row table is a project; rolling back a bad application class is a rewrite. None of these choices are wrong everywhere — the goal is making each one deliberate, sized against the data volume it will actually meet, not a default nobody costed out.

---

# Defaults

Write these unless there is a specific reason not to:

1. Copy anything that must not change onto the record that depends on it — price, name, and rate at the time of the transaction, not a join to the current value.
2. Give primary keys a sort order — time-ordered (UUIDv7, ULID, or bigint), never random UUIDv4.
3. Prove an index with `EXPLAIN ANALYZE` before adding it, and check `idx_scan` for ones to drop.
4. Constraints describe a valid row; application code decides business behavior. Keep triggers for audit trails only.
5. Soft-delete only what the business restores, with the exclusion filter in one place, not repeated per query.
6. Promote a JSON key to a real column the moment any query filters or reports on it.
7. Keep foreign keys inside one service's database; reconcile across service boundaries on a schedule instead.
8. Write hot-path reads and reports by hand; let an ORM handle ordinary CRUD.
9. Move recurring or high-volume job dispatch off a polled table and onto a real queue; keep the transactional outbox.
10. Run migrations as their own pipeline step, ahead of the deploy — never on application boot.

---

## Pattern 1: Scope Every Soft Delete to One Place

Hiding a row instead of deleting it means every future read, in every service, for the life of the table, must remember to exclude it. Forget once and a deleted product reappears on the storefront; a deleted customer's email collides with their unique constraint when they sign up again.

```sql
-- Not this — every caller must remember the filter
SELECT * FROM products WHERE deleted_at IS NULL;

-- Write this — the exclusion lives in one view, everyone reads through it
CREATE VIEW active_products AS
SELECT * FROM products WHERE deleted_at IS NULL;
```

Apply soft deletes only to rows the business actually restores (orders, customers) and keep the filter behind a single view or repository method, not copy-pasted across queries. Give unique constraints a way to coexist with soft-deleted rows, e.g. a partial unique index (`UNIQUE (email) WHERE deleted_at IS NULL`).

**Skip when:** the table is session state, logs, or anything else nobody has ever asked to bring back — use a real `DELETE`.

---

## Pattern 2: Give Primary Keys an Order

A random UUIDv4 key inserts into a random point in the index rather than the end. On an engine that physically orders rows by primary key, that forces page splits and scattered writes — fine while the index fits in memory, expensive once it does not. It also costs twice the storage of a bigint, multiplied across every secondary index that stores a copy of it.

```sql
-- Not this — random insert point, 16 bytes, page splits at scale
id UUID PRIMARY KEY DEFAULT gen_random_uuid()  -- v4

-- Write this — sorts by time, new rows land at the end of the index
id UUID PRIMARY KEY DEFAULT gen_uuid_v7()  -- or ULID, or bigint
```

Where a schema already committed to UUIDv4 and cannot change the primary key, expose a separate public identifier instead of migrating the key: `ALTER TABLE orders ADD COLUMN public_id UUID NOT NULL UNIQUE;` and generate it as v7 going forward.

**Skip when:** the table will never exceed a few hundred thousand rows, or the storage engine does not organize data by primary key (e.g. a heap-organized table with a separate index structure) — the insert-order cost does not apply there.

---

## Pattern 3: Copy What Must Not Change, Not Just What Is True Now

Normalizing to "one fact, one place" is correct for facts that are still true. It silently rewrites history for anything that captures a moment in time. A price join through `product_id` reports today's price on a six-month-old invoice, not what the customer actually paid.

```sql
-- Not this — order_items joins to products for price, name, tax
SELECT oi.quantity, p.price, p.name FROM order_items oi
JOIN products p ON p.id = oi.product_id;

-- Write this — the transaction records what was true when it happened
CREATE TABLE order_items (
  order_id INT NOT NULL,
  product_id INT NOT NULL,
  unit_price NUMERIC(10,2) NOT NULL,
  product_name TEXT NOT NULL,
  tax_rate NUMERIC(5,2) NOT NULL
);
```

This is deliberate repetition, not a normalization mistake — it also collapses invoice-style reads from several joins to one. Apply it anywhere a row represents something that already happened: orders, invoices, audit entries, signed agreements.

**Skip when:** the referenced value is genuinely current-state (a customer's current shipping address on their profile, not on a past order) — there, the join is correct and copying would let the copy drift from the source without anyone deciding that.

---

## Pattern 4: Prove an Index Before Adding One, and Prune the Ones Nobody Uses

Every index makes writes more expensive: inserting one row updates the table plus every index on it. Indexes accumulate one slow-query alert at a time and nobody revisits them once the alert clears, so a table can end up with a dozen, several of them redundant with each other.

```sql
-- Redundant: this index does nothing (status,created_at) doesn't already cover
CREATE INDEX idx_orders_status ON orders (status);
CREATE INDEX idx_orders_status_created ON orders (status, created_at);

-- Find indexes nobody is using
SELECT indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0;
```

Check `EXPLAIN ANALYZE` before creating an index — the planner may prefer a sequential scan anyway when a column has low cardinality (few distinct values across many rows). Drop indexes a composite already covers as a prefix.

**Skip when:** the table is small enough, or written to rarely enough, that the write cost of an extra index is immaterial — a low-traffic admin table does not need this discipline.

---

## Pattern 5: Keep Foreign Keys Inside a Service, Reconcile Across Services

Application-level validation only runs on the paths someone remembered to route through it — not bulk imports, not admin scripts, not a one-off fix run at midnight. Dropping a foreign key to save the write cost of one index lookup trades a small, known cost for an unbounded, undetected one: orphaned rows nobody notices until a report crashes.

```sql
-- Write this — one service's own tables enforce their own integrity
ALTER TABLE order_items
  ADD CONSTRAINT order_items_product_fk FOREIGN KEY (product_id) REFERENCES products(id);
```

Within a single service's database, keep foreign keys always. Across two services' databases, a foreign key is not possible — replace it with a scheduled reconciliation job that compares both sides and reports drift, rather than assuming application code has already guaranteed consistency.

**Skip when:** the reference crosses a service or database boundary — there is no cross-database foreign key to add; build the reconciliation job instead.

---

## Pattern 6: Route Dashboards and Hot Paths Around the ORM

An ORM earns its place for ordinary single-record CRUD. It stops earning it the moment a page loops over a collection and lazy-loads a relation per row (N+1), or fetches a full wide row to read two columns off it.

```python
# Not this — one query per order, 51 queries for 50 orders
for order in Order.objects.all()[:50]:
    print(order.customer.name)

# Write this — one query, only the columns the page needs
SELECT o.id, o.total, c.name
FROM orders o JOIN customers c ON c.id = o.customer_id
LIMIT 50;
```

Count queries per request while developing; more than roughly twenty for one page is worth a look before it ships. Write reports, dashboards, and any identified hot path as hand-written SQL or a query builder that lets you name the exact columns.

**Skip when:** the endpoint is low-traffic, single-record CRUD where the ORM's convenience outweighs a query count nobody will notice.

---

## Pattern 7: Put Constraints in the Database, Business Rules in the Code

A trigger cannot be skipped by a careless caller, which is exactly why it is dangerous: it also cannot be seen by a careful one. A status change with no matching line in any service's logs, controller, or job usually means a trigger nobody remembered writing. Triggers also fire once per row, so a 200,000-row historical import becomes 200,000 additional writes, and most test suites run against a substitute database that never executes them at all.

```sql
-- Write this — a constraint describing a valid row, always enforced, easy to find
ALTER TABLE order_items ADD CONSTRAINT quantity_non_negative CHECK (quantity >= 0);
ALTER TABLE orders ADD CONSTRAINT orders_customer_fk FOREIGN KEY (customer_id) REFERENCES customers(id);

-- Not this — a business rule with no caller, no stack trace, no test coverage
CREATE TRIGGER recalculate_order_status ...
```

Keep `NOT NULL`, `CHECK`, `UNIQUE`, and foreign keys for structural validity. Keep workflow logic — status transitions, recalculations, side effects — in application code where a reviewer can find it by reading callers.

**Skip when:** the trigger writes an audit or history row and nothing else — that use case has no equivalent business logic to hide, and the write-side cost is usually acceptable for what it buys.

---

## Pattern 8: Move Job Dispatch Off a Polled Table Before It Gets Hot

A jobs table is free to start: no broker to run, and the enqueue happens in the same transaction as the write that created the work. It stops being free once several workers poll it every second — they contend for the same rows, and Postgres's row-versioning means finished jobs pile up as dead tuples faster than autovacuum can reclaim them.

```sql
-- Each worker only claims rows nobody else is holding
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

Keep the transactional outbox pattern — writing the job record atomically with the business data is worth it. Move actual delivery to a queue built for the throughput once polling frequency or job volume grows past what a handful of workers can share, and add a routine cleanup for completed rows.

**Skip when:** volume is a few thousand jobs a day or fewer and a broker is genuinely not worth operating yet — `FOR UPDATE SKIP LOCKED` plus a daily cleanup covers that range fine.

---

## Pattern 9: Promote a JSON Key the Moment Anything Queries It

An open `metadata` JSON column is a reasonable escape hatch for shapes the schema does not control. It becomes a liability the moment two callers write the same concept under different keys (`coupon_code` vs `couponCode`) with nothing enforcing either — a divergence that surfaces only when someone runs a report and gets half the rows back.

```sql
-- Fine: an opaque payload nothing else needs to query
webhook_payload JSONB NOT NULL

-- Once anything filters or reports on a key inside it, promote it:
ALTER TABLE orders ADD COLUMN coupon_code TEXT;
UPDATE orders SET coupon_code = metadata->>'coupon_code' WHERE metadata ? 'coupon_code';
```

Use JSON for data whose shape you genuinely do not control — webhook bodies, third-party API responses. The moment application code filters, joins, or reports on a specific key inside it, migrate that key to a typed column; one migration is cheaper than reconstructing spellings after the fact.

**Skip when:** nothing has ever queried into the column and nothing is expected to — a true passthrough blob (e.g. an immutable copy of a third-party payload kept only for replay) can stay JSON indefinitely.

---

## Pattern 10: Run Migrations as Their Own Deploy Step

Running a migration on application boot works with one instance. With more than one, concurrent pods race to acquire the same migration lock — some give up waiting, one may die partway through — and the schema ends up matching no branch in the repository. Rolling back then requires a fresh code deploy in the middle of an incident.

```text
Not this: pod boots -> runs migrations -> serves traffic   (x3 pods racing)

Write this:
  pipeline: run migrations once (dedicated step) -> deploy pods -> pods serve traffic
```

For changes that cannot be a single atomic step (a column rename, a type change), split into stages that each keep the site up: add the new column, ship code that writes both, backfill, then drop the old column in a later release.

**Skip when:** the deployment topology is genuinely single-instance with no concurrent migration risk — even then, a separate pipeline step is simpler to reason about during rollback, so there is little reason to keep migrations on boot even there.

---

# The Through Line

None of these problems show up in a schema review the day it ships — they show up after the data has grown a hundredfold and the fix now needs a rollback plan.

- Every value that must not change with time is copied onto the record that depends on it, not joined to a source that will drift.
- Every primary key sorts by time so new rows land at the end of the index, not in the middle of it.
- Every index is proven with `EXPLAIN ANALYZE` before it exists and reviewed for `idx_scan = 0` after.
- Every structural invariant is a database constraint; every business decision is application code a reviewer can trace.
- Every soft-deleted table restores rows the business actually asks for back, filtered from exactly one place.
- Every JSON key gets promoted to a column the moment something queries into it.
- Every foreign key stays inside its own service's database; cross-service references get a reconciliation job instead.
- Every dashboard, report, and hot path is checked for query count and column width, not left to the ORM's defaults.
- Every job queue is sized against its actual throughput before it becomes the busiest table in the database.
- Every schema change ships as its own pipeline step, staged so the site stays up through it.

Size every choice against a hundred times today's data — that is the table you will actually live with.
