---
name: map-codebase
displayName: Mapping an Unfamiliar Codebase
description: Technique for building a working mental model of a repository you did not write, instead of reading files at random. Consult when opening an unfamiliar codebase for the first time, before making a change in a system you don't yet understand, or when asked to explain how a feature or bug flows through the system.
version: 1.0.0
---

# Mapping an Unfamiliar Codebase

Use `/map-codebase`. 

Reference for how to build a mental model of a repository. Consult before reading files at random, not after getting lost in them.

Reading more code produces the feeling of progress, not understanding — hours spent opening files top to bottom can still leave you unable to answer "how does an order actually get created?" A codebase is a city, not a book: look at the map first — entry points, major districts, how they connect — before studying individual buildings.

The goal is never full coverage, just a model good enough to answer the question in front of you — usually the 20% of the repository a given task actually touches.

---

# Defaults

Write these unless there is a specific reason not to:

1. Before opening a file, write down the concrete question you're answering — "what happens when a customer places an order," not "what does this repo do."
2. Enter through a real entry point — an HTTP route, CLI command, queue consumer, cron job, webhook handler — never through the file tree.
3. Trace exactly one path start to finish before reading anything else. Happy path first, failure branches second.
4. Read the data model — schema, ORM entities, migrations — before reading the services that operate on it.
5. Read configuration — env files, compose/IaC, feature flags — before assuming you need to read code to find the architecture.
6. Read a test's name before its body, and its body before the implementation it covers.
7. Treat `git log` and `git blame` as a primary source the moment code looks arbitrary, not a last resort.
8. Scope depth to the task. Understand the slice a ticket touches, not the repository end to end.
9. Keep a running written map with an explicit "don't understand yet" section — don't hold the model in your head.
10. Time-box first-pass recon to about 30 minutes. Depth comes from repeated traces, not one long read.

---

## Pattern 1: Ask the Business Question Before Opening a File

Opening `controllers/`, then `services/`, then `repositories/` in file-tree order produces exposure, not a model. Before touching the tree, state what you're trying to answer as a sentence about behavior, not structure.

```
Not this:  "Let me understand this repository."
Write this: "What happens when a customer places an order?"
            "What happens when a payment webhook arrives twice?"
```

A structural question has no end condition — you can always open one more file. A behavioral question ends the moment you can narrate the answer.

---

## Pattern 2: Enter Through a Door, Not the File Tree

Every runtime has a small number of places code starts executing from the outside. Find those first — they are the doors into the system.

```bash
# HTTP routes (Express/Next.js/FastAPI-style — adjust per framework)
rg "app\.(get|post|put|delete)\(|@router\.(get|post)|export (async )?function (GET|POST)"

# CLI entry points
rg "\.command\(|argparse\.ArgumentParser|if __name__ == .__main__."

# Queue / event consumers
rg "@KafkaListener|\.subscribe\(|\.consume\(|on\('message'"

# Scheduled jobs
rg "cron\(|@Scheduled|APScheduler|schedule\.every"
```

Each match is a place a request, message, or timer starts a journey through the system. Pick the one relevant to your question and follow it — don't read the other doors yet.

**Skip when:** the repository is a library with no runtime entry point — start from its public exports (`index.ts`, `__init__.py`, the published API surface) instead.

---

## Pattern 3: Trace One Path

From the entry point, follow function calls one hop at a time. Don't stop to fully understand each function — just note what it calls next, write it down, and move to that.

```
POST /orders
  -> OrderController.create()
       -> OrderService.createOrder()
            -> InventoryService.reserve()
            -> PaymentService.charge()
            -> OrderRepository.save()
            -> eventBus.publish(OrderCreated)
                 -> NotificationConsumer.onOrderCreated()
                 -> AnalyticsConsumer.onOrderCreated()
```

Writing the trace down, literally, is what turns a maze of files into one linear story. Once you have this for one flow, do a second one (e.g. "cancel order") — the same classes start reappearing, and the repository stops feeling arbitrary.

**Skip when:** the flow is trivial (a single-file script, a pure function with no dependencies)

---

## Pattern 4: Read the Data Before the Services

Services read as arbitrary logic until you know what shape they're operating on. The schema or ORM model is the fastest way into a domain — read it before the service files that use it.

```python
# SQLAlchemy / Django-style model — read this before PaymentService
class Payment(Base):
    id: str
    order_id: str
    amount: Decimal
    status: PaymentStatus       # tells you a state machine exists
    provider_transaction_id: str  # tells you an external system is involved
    created_at: datetime
```

Once you've seen `status: PaymentStatus`, a line like `payment.status = PaymentStatus.SUCCEEDED` stops being unexplained mutation — it's one transition in a state machine you already know exists. `provider_transaction_id` tells you to expect a client to an external payment gateway before you've even found it.

---

## Pattern 5: Let Configuration Name the Architecture

Config files describe integrations and constraints faster than the code that implements them, because they're forced to be explicit about what's tunable.

```yaml
# docker-compose.yml / .env — read this before tracing PaymentService
payment:
  timeout_ms: 3000
kafka:
  brokers: broker:9092
  retries: 3
redis:
  url: redis://cache:6379
  default_ttl_seconds: 3600
```

This one file tells you: the system talks to a payment provider with a hard timeout, uses Kafka with bounded retries, and caches in Redis with a one-hour expiry — all before reading a single service class. Treat `docker-compose.yml`, `.env.example`, Terraform/CDK, and feature-flag config as documentation, not boilerplate to skip past.

---

## Pattern 6: Map External Systems, and Ask Why Each One Exists

Large systems rarely live alone. Find what they talk to, then ask why — the "why" is what you'll actually remember.

```bash
rg "axios\.create\(|new (Stripe|S3Client|KafkaProducer)\(|createClient\(|boto3\.client\("
```

```
Not this:            "This system uses Kafka."
Write this instead:  "Order processing publishes to Kafka so notification
                       delivery doesn't block the checkout response."
```

A fact about a dependency is forgettable. A fact about *why* a dependency exists attaches to the business flow you already understand, which makes it stick.

---

## Pattern 7: Use Git History

Code that looks arbitrary — a retry count, a sleep, a strange conditional — usually has a reason that isn't visible in the diff you're looking at. Check history before guessing.

```bash
git log --oneline -- src/services/paymentService.ts
git blame -L 40,55 src/services/paymentService.ts
```

```python
# Looks arbitrary in isolation:
if attempt > 2:
    raise PaymentException("giving up")

# git log on this line: "cap retries at 3 — provider starts rate-limiting
# after the third attempt (incident #482)"
```

The commit message turns a mysterious constant into a documented constraint. Messy-looking code is frequently the fossil record of a production incident, not carelessness — read the history before assuming otherwise.

---

## Pattern 8: Read Tests as Executable Business Rules

Test names are often the closest thing a codebase has to a business-rules document, and they cost far less to read than the implementation.

```ts
describe('createOrder', () => {
  it('rejects the order when inventory is unavailable', () => {})
  it('refunds the payment if order creation fails after charge', () => {})
  it('does not duplicate an order for the same idempotency key', () => {})
})
```

Before opening `orderService.ts`, skim `orderService.test.ts`. The `it(...)` names alone usually tell you the edge cases the implementation exists to handle — read the implementation afterward to see how, not to discover what.

**Skip when:** the test suite is thin or absent — fall back to Pattern 7 (git history) and Pattern 5 (config) for the same context instead.

---

## Pattern 9: Chase the Weird Parts

Once the happy path makes sense, deliberately look for anything that makes you ask "why would they do this?" — those spots are usually where the real architectural decisions live.

```python
time.sleep(0.5)                 # why this specific delay?
cache.set(key, value, ttl=300)  # why 300, not 60 or 3600?
if retry_count < 3:             # why exactly three?
with db.transaction():
    event_bus.publish(...)      # why is a publish inside a DB transaction?
```

Each of these is a question worth answering with Pattern 7 (git blame) or Pattern 8 (tests) rather than skimming past. They're disproportionately likely to encode a past incident or a constraint that isn't written down anywhere else.

---

## Pattern 10: Scope Depth to the Task, Not the Repository

You do not need to understand every service to fix one bug. Given "customers are sometimes charged twice on double-click," the relevant slice is `POST /payments -> PaymentService -> PaymentGateway -> PaymentRepository` — not the recommendation engine, the reporting service, or user profile management.

```
Task-driven scope:                    Not repository-driven scope:
  Payment API                           Every service in the monorepo
    -> PaymentService                   read in file-tree order,
    -> PaymentGateway                   whether or not it's relevant
    -> PaymentRepository                to the task at hand
```

Name what's out of scope as explicitly as what's in scope — it stops you from re-deriving the same boundary later, and tells a reviewer what you deliberately did not check.

---

# Running the Map

## Mode 1: First-Pass Recon (new repository, no specific task yet)

Time-box to ~30 minutes. Work in this order, and stop as soon as you can answer the questions below — don't keep reading past that point:

1. README, then top-level project structure.
2. Application entry point(s) — Pattern 2.
3. **Checkpoint.** From the entry points found, name the 2–4 business flows they most likely represent (e.g. "place an order," "cancel a subscription," "ingest a webhook") and ask the user which one to trace first, rather than picking unilaterally. If the user has no preference, trace whichever flow touches the most other entry points — it usually teaches the most about the system.
4. Configuration — Pattern 5.
5. Data model / schema — Pattern 4.
6. The chosen business flow, traced end to end — Pattern 3.
7. External integrations — Pattern 6.

You're done with first-pass recon once you can answer: What does this system do? Where does a request enter? What are the major components? Where does data live? What does it talk to? What part is relevant to my actual task?

## Mode 2: Task-Scoped Trace (a specific ticket, bug, or feature)

1. Restate the ticket as a behavioral question (Pattern 1). If the ticket is vague enough to support more than one reading (e.g. "fix the checkout bug" with no repro), state the reading you're about to trace and confirm it with the user before spending time on it — a wrong guess here wastes the whole trace.
2. Find the one entry point it touches (Pattern 2) — resist opening unrelated doors.
3. Trace only that path, including its failure branches, not just the happy path (Pattern 3).
4. Check tests and git history around the exact lines involved (Patterns 7–8).
5. Report the trace and the fix inline. Only write a persistent map file if the task is large enough that Mode 1's output would help, or the user asks for one.

## Mode 3: Update an Existing Map

If a map file from a previous session already exists, extend it — don't regenerate it from scratch:

- Append the newly traced flow rather than rewriting existing ones.
- Move resolved items out of "Don't Understand Yet" into the relevant section, with the answer.
- Add new unresolved questions as you hit them, instead of letting them stay only in conversation.

---

# Output: The Map Itself

When Mode 1 or a large Mode 2 task warrants a persistent artifact, write it as a single markdown file (e.g. `CODEBASE_MAP.md`) with this shape:

```markdown
# System Purpose
One paragraph: what this system does, in business terms.

# Entry Points
- POST /orders -> OrderController.create
- OrderCreated (Kafka) -> NotificationConsumer, AnalyticsConsumer
- nightly cron -> ReconciliationJob

# Data Entities
- Order (id, customer_id, status, total) -> owns OrderItem[]
- Payment (id, order_id, status, provider_transaction_id)

# Major Flows
## Create Order (happy path)
POST /orders -> OrderController -> OrderService.createOrder
  -> InventoryService.reserve -> PaymentService.charge
  -> OrderRepository.save -> publish(OrderCreated)

## Create Order (payment fails)
... same entry point, divergent branch ...

# External Dependencies
- Stripe (PaymentService) — charges and refunds. Timeout 3s (see config).
- Kafka — decouples order write from notification delivery.
- Redis — caches product catalog, 1h TTL.

# Configuration Notes
- payment.timeout_ms = 3000 — payment gateway calls are capped, see incident #482.

# Don't Understand Yet
- Why does OrderCreated get published inside the DB transaction in OrderService? (TODO: git blame)
```

The "Don't Understand Yet" section is not optional filler — it converts vague confusion into a specific, investigable list, which is strictly more useful than either pretending to understand or staying quiet about the gap.

---

# The Through Line

None of these techniques are about reading faster — they're about reading less of what doesn't matter yet.

- Every exploration starts from a behavioral question, not "understand the repo."
- Every trace starts at a real entry point and follows one path before branching to another.
- Every service is read after its data model, not before.
- Every config file is read as documentation of constraints, not skipped as boilerplate.
- Every unexplained constant or delay gets a `git blame` before a guess.
- Every test name is read as a business rule before its implementation is read as code.
- Every task scopes the map to the slice it touches, with the rest named as out of scope.
- Every map is written down, with a live list of what's still unknown — never held only in memory.

A codebase stops looking like a monster the moment you stop trying to memorize it and start reconstructing the story it's telling — one traced path at a time.
