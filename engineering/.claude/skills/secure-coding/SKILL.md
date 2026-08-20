---
name: secure-coding
displayName: Secure Coding Habits
description: Habits for shaping code so insecure behavior never becomes the easiest behavior to write. Consult while designing request handlers, services, data models, authorization, and background jobs — before writing the implementation, not after a security test finds the gap.
version: 1.0.0
---

# Secure Coding Habits

Reference for how to shape code so it resists attack by default. Consult before implementing, not after.

Security testing finds vulnerabilities late — by the time a pentest finds broken authorization or unsafe input handling, it's already embedded in controllers, models, and integrations. It rarely started as one dangerous line; it started as an ordinary shortcut, like a service accepting a whole request object because mapping felt repetitive.

Testing cannot compensate for a system that makes insecure behavior the easiest behavior. The goal here is not predicting every attack — it is removing entire categories of dangerous behavior before testing begins.

---

# Defaults

Write these unless there is a specific reason not to:

1. Parse untrusted input into a narrow command before it touches business logic.
2. One command type per operation, not a generic create/update over the full entity.
3. Authorization checked against the actor, the specific resource, the action, and current state — inside the service, not only the route guard.
4. Parameterized queries, typed repositories, and allowlisted dynamic values — never string-built SQL, shell commands, or file paths.
5. A conditional write (constraint, unique index, transactional check) wherever a check-then-write race would matter.
6. A database constraint behind every structural invariant the application already validates.
7. Only the fields a response, log, or event actually needs — never a whole entity "in case it's needed later."
8. Fail closed on missing or uncertain security state — a failed permission lookup is never treated as permission granted.
9. A narrow scope and rotation plan for every secret and credential.
10. Guarantees visible in code — method names, input models, and explicit scope — not held only in a reviewer's memory.

---

## Pattern 1: Define the Trust Boundary Before Writing the Handler

Before implementing, name every value that enters from outside the trusted system: form fields, query params, headers, cookies, uploaded file metadata, webhook payloads, queue events, mobile app requests, imported files, and claims from other internal services. A signed event proves who sent a payload; it does not prove the requested action is still valid.

```python
# Write this — parse into a narrow command at the boundary
def parse_create_invoice_command(req: Request) -> CreateInvoiceCommand:
    return CreateInvoiceCommand(
        customer_id=req.body["customer_id"],
        line_items=parse_line_items(req.body["line_items"]),
    )

# Not this — the raw request body reaches business logic
async def create_invoice(body: dict):
    return await invoice_repository.save(body)
```

Decide at the boundary where shape validation, authentication, authorization, rate limiting, and business rules each belong. Do not let trust accumulate implicitly — a value that passed one schema check is not thereby authorized, and a typed object does not imply its caller was permitted to send it.

---

## Pattern 2: Model Operations, Not Entire Entities

Give each business action its own command instead of spreading a request body into an ORM entity. `UpdateProfile`, `TransferProjectOwnership`, and `ApproveExpense` are different operations even when they touch the same row.

```python
# Write this — the writable surface is explicit
async def update_profile(user_id: str, input: UpdateProfileInput):
    user = await user_repository.find_one_by_or_fail(id=user_id)
    user.display_name = input.display_name
    user.bio = input.bio
    user.avatar_url = input.avatar_url
    return await user_repository.save(user)

# Not this — the request shape grows with the persistence model
async def update_profile(user_id: str, input: dict):
    user = await user_repository.find_one_by_or_fail(id=user_id)
    for key, value in input.items():
        setattr(user, key, value)
    return await user_repository.save(user)
```

Adding a field to the writable set becomes a deliberate decision instead of an accident of the entity's shape. It also lets authorization differ per operation — editing content and transferring ownership should not share one permission.

**Skip when:** an internal admin tool intentionally exposes full-entity editing to a trusted, already-authorized operator — that is the one context where the entity *is* the operation.

---

## Pattern 3: Keep Authorization Attached to the Resource and Action

A route guard checking `projects:update` proves the actor holds that capability somewhere. It does not prove the specific project belongs to them. Authorization must include the actor, the specific resource, the action, and the current business state — checked near the mutation, not only in the controller.

```python
# Write this — scope is part of the query, not assumed from the route
await project_repository.update(
    {"id": project_id, "organization_id": actor.organization_id},
    changes,
)

# Not this — cross-tenant id can reach the mutation
await project_repository.update({"id": project_id}, changes)
```

A controller-only check disappears the moment a background worker, internal endpoint, or second transport calls the same service method. Ask "does this user have permission to perform this exact action on this exact resource under its current conditions" — not just "does this user have permission."

---

## Pattern 4: Make the Safe Data-Access Pattern the Default Path

Unsafe shortcuts survive when they are shorter than the safe alternative. Default to parameterized queries, query builders, typed repositories, and server-controlled allowlists for anything dynamic.

```python
# Write this — client selects among known options
SORT_COLUMNS = {
    "newest": "orders.created_at",
    "amount": "orders.total_amount",
    "status": "orders.status",
}

sort_column = SORT_COLUMNS.get(input.sort_by, SORT_COLUMNS["newest"])

# Not this — client input becomes part of query structure
sort_column = input.sort_by  # used directly in ORDER BY
```

The same applies to file uploads (server-generated storage names, not client-provided paths), shell commands (fixed executables and controlled arguments, or avoided), and outbound URLs (validated protocol and destination before connecting). The paved road should already separate data from executable structure so nobody has to remember injection defenses per feature.

---

## Pattern 5: Design Every State-Changing Operation for Concurrency

A check followed by a later write is not secure under overlapping requests, even when each step is logically correct alone. Coupon redemption, inventory reservation, one-time token consumption, and balance updates all need the state to remain true at the moment the write becomes durable — not just at the moment it was checked.

```python
# Write this — the write itself enforces the condition
result = await db.execute(
    """UPDATE tokens SET used_at = now()
       WHERE id = %s AND used_at IS NULL
       RETURNING id""",
    [token_id],
)
if result.rowcount == 0:
    raise ValidationError("Token already used")

# Not this — the gap between read and write is exploitable
token = await get_token(token_id)
if token.used_at:
    raise ValidationError("Token already used")
await mark_token_used(token_id)
```

Design for retries as the normal case, not the exception: networks fail after success, responses disappear, queues redeliver. Give any operation that must happen once a stable identifier (an idempotency key, a unique operation id) so a repeated instruction converges on one result instead of repeating its effect. See **Pattern 10** in `coding-patterns` for the identity mechanism itself.

---

## Pattern 6: Make Invalid States Difficult to Store

Application validation improves error messages; it should not be the only thing preventing a structurally invalid row. Put the smaller set of facts that must always be true behind database constraints: uniqueness, foreign keys, check constraints, and unique event identifiers for exactly-once processing.

```sql
ALTER TABLE users ADD CONSTRAINT users_email_unique UNIQUE (email);
ALTER TABLE orders ADD CONSTRAINT orders_customer_fk FOREIGN KEY (customer_id) REFERENCES customers(id);
ALTER TABLE order_items ADD CONSTRAINT quantity_non_negative CHECK (quantity >= 0);
```

Without these, every writer — API handlers, imports, migrations, queue consumers, admin scripts, future services — must independently reproduce the same assumptions, and eventually one will not. Complex authorization and workflow intent still belong in application code; the database's job is refusing the smaller set of structural facts that must hold regardless of the caller.

---

## Pattern 7: Minimize Sensitive Data Before It Starts Moving

Every copy of sensitive data — API response, log line, analytics event, cache entry, export, frontend bundle, backup — needs its own access control and retention story. The safest copy is the one never created.

```python
# Write this — only what the operation needs
logger.warning("Refund rejected", extra={
    "invoice_id": invoice_id, "customer_id": customer_id,
    "reason": reason, "request_id": request_id,
})

# Not this — full payload becomes a new place to protect
logger.warning("Refund rejected", extra={
    "invoice": invoice, "customer": customer, "request": full_request_body,
})
```

List endpoints return response-specific shapes, not complete entities (**Pattern 12**, `coding-patterns`). Background events carry the minimum a consumer needs, or a reference to the protected record instead of a duplicate. Privileged credentials stay server-side; browser code gets only publishable identifiers. A field returned "in case the frontend needs it later" is already exposed today.

---

## Pattern 8: Treat Errors, Fallbacks, and Missing Data as Security Decisions

Uncertainty must never increase authority. If a permission check times out, if a tenant id is missing, if token verification throws — fail closed, not open.

```python
# Write this — uncertainty narrows access
try:
    actor = await verify_token(token)
    return await handle_request(actor, req)
except VerificationError:
    return JSONResponse({"error": "UNAUTHENTICATED"}, status_code=401)

# Not this — a failure silently becomes a permissive default
try:
    actor = await verify_token(token)
    return await handle_request(actor, req)
except VerificationError:
    return await handle_request(ANONYMOUS_ACTOR, req)  # now trusted by accident
```

Failing closed does not require taking the whole product offline — degrade one feature, go read-only, or serve only public data. Never let a missing tenant id cause a query to silently become global.

Keep internal errors internal: stack traces, SQL errors, file paths, and dependency details stay in logs with a safe diagnostic context; clients get a stable, public error contract (**Pattern 6**, `coding-patterns`).

---

## Pattern 9: Give Secrets a Lifecycle, Not Just a Storage Location

Storage location (env vars, secret manager, encrypted config) answers where a secret lives. Design who can access it, which service needs it, how it rotates, and what happens when it leaks — before it leaks.

- Scope each credential narrowly: a service that reads a few tables does not get an admin database account.
- Never let one secret double as two unrelated credentials (a webhook signing secret is not an API key).
- Separate secrets per environment — production never shares a value with staging or development.
- Design rotation before compromise: support accepting an old and new key during a transition window, and know how to revoke tokens signed with a compromised key.

The same lifecycle applies to user-facing secrets — reset tokens, API keys, refresh tokens, remembered-device tokens: hash where recovery is unnecessary, show only once where appropriate, and wire them to revocation and audit.

---

## Pattern 10: Make Security Assumptions Visible During Ordinary Review

Hidden assumptions survive only until the surrounding code changes. A reviewer should be able to tell, from the code itself, whether tenant scope is guaranteed, which fields a request can write, and whether a retry is safe — without asking the person who wrote it.

Make this visible through the shapes above rather than comments: method names that distinguish authorized operations from internal low-level mutations (**Pattern 2**), input models that show the writable surface (**Pattern 2**), queries with explicit tenant scope (**Pattern 3**), and idempotency keys threaded explicitly instead of inferred (**Pattern 5**).

During review, ask about misuse, not just the happy path: What if this id belongs to another tenant? What if the request has an extra field? What if two copies arrive together? What if the dependency succeeds but the response is lost? What if the security service is unavailable? These are ordinary engineering questions, not a separate formal security pass.

---

# The Through Line

None of these gaps are closed by a later security test — they are closed by removing the shortcut that made the insecure version the easiest one to write.

- Every untrusted value is parsed into a narrow shape before it reaches business logic.
- Every operation exposes only the fields it needs to change.
- Every authorization check is scoped to the actor, resource, action, and state.
- Every dynamic query, path, or command is built from known values, not untrusted strings.
- Every check-then-write is a single conditional write.
- Every structural invariant has a database constraint behind it.
- Every sensitive value is minimized before it is copied anywhere.
- Every failure and fallback fails closed.
- Every secret has a scope and a rotation plan.
- Every important guarantee is visible in the code, not held in memory.

Security testing should reveal the weaknesses nobody anticipated — not the first place the application discovers that ordinary code was never designed to protect itself.
