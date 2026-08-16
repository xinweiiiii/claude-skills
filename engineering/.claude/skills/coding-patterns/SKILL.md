---
name: coding-patterns
displayName: Production Coding Patterns
description: Patterns to follow when writing application code that must be maintained and must survive growth in traffic, clients, and contributors. Consult while designing services, APIs, domain models, background jobs, and integrations — before writing the implementation.
version: 2.0.0
---

# Production Coding Patterns

Reference for how to shape application code. Consult before implementing, not after.

Clean code is boring code. It makes bugs easier to find and changes easier to make. Generating code is easier than maintaining it, so default to the boring shape.

Scalability follows the same rule. Infrastructure cannot rescue code that creates work without limits. If one request loads every record into memory, ten servers do that ten times. If a payment is not idempotent, retries across instances create duplicates faster than one instance could.

- **Patterns 1–7** — shapes that survive maintenance.
- **Patterns 8–17** — shapes that survive growth.

Each pattern states the rule, shows the shape to write, and says when it does not apply. Applying a pattern where it does not belong is its own defect — the "Skip when" lines are part of the guidance, not caveats.

---

# Defaults

Write these unless there is a specific reason not to:

1. Guard clauses before the happy path.
2. Business names, not container names.
3. External data mapped to internal models at the boundary.
4. Types that cannot represent an invalid state.
5. Pure decisions, separate from the actions that carry them out.
6. Structured errors with stable codes.
7. One behavioral change per commit.
8. An explicit bound on anything that can grow.
9. A deliberate concurrency limit, never `Promise.all` over unbounded input.
10. A stable identifier on any command that must not run twice.
11. Durable jobs for work that cannot finish inside a request.
12. Additive contract changes.

---

# Part I — Shapes That Survive Maintenance

## Pattern 1: Return Early

Handle every negative path first, then let the happy path run unindented at the end.

```ts
async function updateUserProfile(userId: string, input: ProfileInput) {
  const user = await getUser(userId);
  if (!user) throw new NotFoundError("User not found");
  if (!input.email) throw new ValidationError("Email is required");
  if (!user.canEditProfile) throw new ForbiddenError("User cannot edit profile");

  return saveProfile(user.id, input);
}
```

Never nest the success case inside conditionals:

```ts
// Do not write this
if (user) {
  if (input.email) {
    if (user.canEditProfile) return saveProfile();
  }
}
```

Each guard names one reason to stop, so an incident trace points at a specific line.

---

## Pattern 2: Name Business Meaning

Name a variable for the business concept it holds, not the container it arrived in.

```ts
// Write this
const subscription = await getSubscription(subscriptionId);
if (subscription.isBillable) await chargeSubscription(subscription);

// Not this
const result = await getData(id);
if (result.status === "active") await process(result);
```

Do not introduce: `data`, `result`, `item`, `payload`, `response`, `value`, `temp`, `obj`, `list`.

Reach for: `customer`, `invoice`, `subscription`, `paymentAuthorization`, `refundEligibility`, `accountBalance`.

Long names are fine when they carry intent. `refundEligibility` beats `check`.

**Skip when:** a genuinely generic utility operates on any type — `items` is correct inside `mapWithConcurrency`.

---

## Pattern 3: Map External Data at the Boundary

Convert third-party shapes into internal models in one place. Everything downstream depends only on the internal model.

```ts
function mapBillingCustomer(response: BillingCustomerResponse): Customer {
  return {
    id: response.id,
    name: response.user_name,
    isBillable: response.status === "ACTIVE",
    planName: response.subscription?.plan_name ?? "Free"
  };
}
```

Never let `response.data.subscription.plan_name` spread through business logic.

Boundaries belong around third-party APIs, webhooks, databases, message queues, framework request objects, and environment variables. Keep raw DB rows out of UI logic, HTTP shapes out of domain logic, framework objects out of business services, and env parsing in one module.

Attach the operational policy for that dependency here too — see **Pattern 15**.

---

## Pattern 4: Make Invalid States Unrepresentable

Model each lifecycle stage as its own type instead of one type with optional fields.

```ts
type DraftUser = { email: string; role: "admin" | "member" };

type SavedUser = {
  id: string;
  email: string;
  role: "admin" | "member";
  status: "active" | "disabled";
};
```

An all-optional type forces defensive checks at every use and lets impossible combinations compile:

```ts
// Do not write this
type User = { id?: string; email?: string; role?: string; status?: string };
```

Mark a field optional only when its absence is a real business state.

---

## Pattern 5: Separate Decisions From Actions

Put the rule in a pure function that returns a verdict. Let the caller perform the side effects.

```ts
function getRefundEligibility(invoice: Invoice): RefundEligibility {
  if (invoice.status !== "paid") {
    return { allowed: false, reason: "Invoice is not paid" };
  }
  if (invoice.refundedAt) {
    return { allowed: false, reason: "Invoice is already refunded" };
  }
  return { allowed: true };
}

async function refundInvoice(invoiceId: string) {
  const invoice = await getInvoice(invoiceId);

  const eligibility = getRefundEligibility(invoice);
  if (!eligibility.allowed) throw new ValidationError(eligibility.reason);

  await paymentProvider.refund(invoice.paymentId);
  await markInvoiceRefunded(invoice.id);
  await sendRefundEmail(invoice.customerId);
}
```

The rule is then testable with no mocks, and reusable wherever the same question is asked.

**Skip when:** the decision is a single comparison with no branching — extracting it adds indirection without buying a test.

---

## Pattern 6: Throw Structured Errors

Give every error a stable machine code, a human message, and operational context.

```json
{
  "code": "USER_EMAIL_ALREADY_EXISTS",
  "message": "A user with this email already exists.",
  "details": { "field": "email" },
  "requestId": "req_8f91a2"
}
```

Codes are the contract — `PAYMENT_DECLINED`, `REFUND_NOT_ALLOWED` — and must not change once shipped. Messages are for humans and may be rewritten freely, which is exactly why nothing may branch on them:

```ts
// Never do this
if (error.message.includes("already exists")) showEmailTakenError();
```

Log with context, as fields rather than interpolated strings:

```ts
logger.warn("Refund rejected", { invoiceId, customerId, reason, requestId });
```

Never log passwords, tokens, payment details, secrets, or sensitive personal data.

---

## Pattern 7: Commit One Behavioral Change at a Time

Split work so each commit has a single reason to be reverted, and keep mechanical changes away from logic changes.

```txt
refactor: extract refund eligibility into a pure function
chore: rename payment fields to match domain model
feat: reject refunds for already-refunded invoices
```

A single "update billing flow" commit carrying a refactor, a rename, a rule change, and a UI tweak hides which one changed behavior, and forces a rollback to discard all four.

If the description needs a bulleted list of unrelated items, it is more than one change.

---

# Part II — Shapes That Survive Growth

Infrastructure adds capacity only after the software defines how that capacity is used. It cannot set limits, preserve invariants, or decide which repeated request is the same business action.

---

## Pattern 8: Bound Every Growing Operation

Give anything that can grow an explicit limit, written into the operation rather than added to config after an incident.

- List endpoints paginate.
- Uploads cap request body size.
- Batch jobs process a fixed number of records per pass.
- Every external call carries a deadline.
- In-memory caches have capacity and an eviction rule.

Assume every input grows. A query returning ten rows in development returns a million in production; a 50 MB upload is fine once and destructive when twenty arrive together. Scalability failures are multiplication failures.

The limit is also a product contract: pagination tells clients the dataset is too large to treat as one object, and a timeout states when the result stops being useful. Decide those numbers while writing the endpoint, not during the incident.

---

## Pattern 9: Bound Concurrency

Choose a concurrency number from the capacity of the dependency. Never let array length decide how much work starts at once.

```ts
async function mapWithConcurrency<T, R>(
  items: T[],
  limit: number,
  worker: (item: T) => Promise<R>
): Promise<R[]> {
  const results = new Array<R>(items.length);
  let nextIndex = 0;

  async function runWorker(): Promise<void> {
    while (true) {
      const index = nextIndex++;
      if (index >= items.length) return;
      results[index] = await worker(items[index]);
    }
  }

  const workerCount = Math.min(limit, items.length);
  await Promise.all(Array.from({ length: workerCount }, () => runWorker()));
  return results;
}
```

Use any equivalent helper or library. What matters is that the limit is a deliberate number derived from the database pool, API quota, CPU cost, or memory footprint — measured, not copied from an example that happened to use ten.

Unlimited parallelism speeds up one request while destabilizing the service. Bounded concurrency trades a little latency for predictable behavior under load.

**Skip when:** the collection has a small fixed size known at the call site — awaiting three independent lookups with `Promise.all` is correct.

---

## Pattern 10: Give Commands a Stable Identity

Any operation that must not happen twice gets an identifier supplied by the caller or derived from the intent, stored under a uniqueness constraint alongside the durable result.

Apply to checkouts, refunds, transfers, imports, and webhook deliveries. On a repeat, the handler returns the existing outcome, resumes incomplete work, or rejects an identifier reused with different input.

Do not infer duplicates from business data. Two legitimate orders may share customer and amount; one retried order may arrive minutes later. **Similarity is not identity.**

Write for retries as the normal case: clients retry, proxies resend, brokers redeliver, workers restart after doing the work but before recording it, and users double-click. The same identifier should also flow into logs, traces, and queue messages (**Pattern 17**).

---

## Pattern 11: Move Long Work Out of the Request

When duration is unpredictable or several systems are involved, have the request validate, persist an accepted operation, and return its identifier. A worker does the rest with explicit retries, progress states, and failure handling.

The operation must be **durable** — a database row, queue message, or job record that outlives the request process. Then expose its state so the client can poll: pending, running, completed, or failed.

An unawaited promise is not a background job:

```ts
// Do not write this — the work vanishes if the process restarts
app.post("/reports", async (req, res) => {
  generateReport(req.body);
  res.status(202).send();
});
```

**Skip when:** the work is bounded and fast. A queue adds real complexity — use it when duration, retries, or external dependencies make synchronous completion an unreliable promise, not for every slow function.

---

## Pattern 12: Build Reads Separately From Writes

Query for the exact fields a response needs instead of loading an entity and serializing it.

A write model protects ownership, status transitions, and invariants. A read model serves one screen or report. Sharing one broad entity means list endpoints load complete objects, serializers traverse unrequested relationships, and schema changes leak into public responses.

So: a dashboard query selects dashboard fields; a search result is smaller than a detail view; an export runs its own query rather than loading entities. Writes stay narrow and never accept a whole entity object from the client.

**Skip when:** the project is small and one model genuinely serves both. This is not a mandate for full CQRS — it is a refusal to assume one model must represent storage, behavior, and every response.

---

## Pattern 13: Batch at Expensive Boundaries

Collapse repeated round trips into one bounded call: one query for a set of identifiers, inserts grouped into safe transaction sizes, bulk endpoints where the dependency offers them, queue consumers taking a limited batch.

Watch for the N+1 shape — a loop that queries, inserts, or calls an API once per item. Each operation is fast; the round trips dominate.

Give every batch a maximum size and defined failure semantics before writing it:

- Does one failure roll back the batch, or do successful items commit?
- Can a retry skip what already succeeded?
- Does the dependency return per-item results?

**Skip when:** the batch would be unbounded. Batching groups work, but it also groups consequences — one enormous transaction means lock contention, long retries, and painful partial recovery.

---

## Pattern 14: Apply Backpressure

Whenever one component produces faster than another consumes, build in a way for the producer to slow down. Otherwise the difference accumulates in memory, pending promises, queue depth, socket buffers, or temp files.

- Streams: pause the writer when the destination is full; use pipelines that propagate this rather than reading everything into memory.
- Queues: set consumer concurrency and prefetch limits.
- Imports: await the current database batch before parsing further.
- APIs: return rate limits and capacity responses instead of accepting unprocessable work.

Scaling raises producer speed before consumer capacity — more instances accept requests faster than one database can write. Do not treat a queue as an infinite shock absorber; a growing backlog is delayed customer outcomes plus a recovery problem.

Emit **queue age**, not only queue length. Ten thousand messages moving quickly is healthy; one hundred delayed for hours is not.

---

## Pattern 15: Give Each Dependency an Explicit Failure Policy

Wrap each important dependency in a client that defines its timeouts, retryable conditions, backoff, request identifiers, error translation, and logging — in one place, at the boundary from **Pattern 3**.

Set connection and response timeouts explicitly; a default of "wait forever" is the most common cause of pool exhaustion. Before adding a retry, classify the operation: reads and idempotent commands may retry, and anything else needs an identity from **Pattern 10** first.

Add isolation deliberately, never by reflex:

- **Bulkhead** — cap how much concurrency one dependency may consume. Use when a shared pool is at risk.
- **Circuit breaker** — stop calling after repeated failures. Around a fast-recovering dependency, this manufactures outages.
- **Fallback** — degrade gracefully where the product tolerates it. Do not use one that silently serves stale or wrong data.

Remember that retries amplify load exactly when a dependency is struggling. One slow dependency without a policy can exhaust request workers, connection pools, and memory across the whole application.

---

## Pattern 16: Change Contracts Additively

Add before removing. A new field ships alongside the old one; a new column coexists with the previous representation; an event gets a new version rather than a silently changed meaning.

Schema migrations run as **expand and contract**:

1. Expand the schema.
2. Backfill safely.
3. Move writers, then readers.
4. Drop the old structure once nothing depends on it.

This spans multiple deployments and keeps every step reversible. It is what makes rolling deployments safe when old and new instances run simultaneously, and it protects mobile clients, integrations, and workers that update on their own schedule.

Treat database schemas, public APIs, and event formats as shared contracts, not private implementation details. A direct breaking change feels faster until one missed consumer leaves the system unable to move forward or back.

---

## Pattern 17: Record State Transitions

Persist meaningful states, not just "started" and "failed", and carry the operation identifier (**Pattern 10**) through logs, traces, queue messages, and database rows.

A report moves through requested → processing → uploaded → completed. A payment distinguishes provider success from local order completion. An import records the last committed batch, so a restart resumes instead of repeating.

Write these states as data that can be queried, because they drive recovery: a stopped worker leaves findable intermediate rows, a retry links to existing work, and an incident can separate external success from internal failure.

Expose metrics over them — operations pending, age of the oldest, retry counts, latency by dependency, anything stuck past its expected duration.

Record the evidence needed to recover work, not every value in scope.

---

# The Through Line

Unbounded work creates overload. Unlimited concurrency multiplies it. Missing command identity turns retries into duplicates. Long request paths hold resources. Broad models inflate query and serialization cost. Repeated boundary crossings waste capacity. Missing backpressure lets queues and memory grow. Uncontrolled dependencies spread failure. Breaking contracts makes deployment fragile. Missing observability makes every incident slower.

None of these is fixed by adding another application instance.

- Every collection has a boundary.
- Every expensive dependency has a policy.
- Every important command has an identity.
- Every long workflow has a durable owner.
- Every shared contract has a safe evolution path.
- Every critical operation leaves evidence.

Code is ready to scale when more traffic creates more of the same controlled work, rather than exposing an assumption that was only safe while the system was small.
