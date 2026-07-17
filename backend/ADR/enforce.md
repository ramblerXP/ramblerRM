## Enforcing 

Our desired structure:

* High cohesion
* Low coupling
* Business-capability focus
* Bounded contexts and aggregates
* Encapsulated data
* Substitutability
* Composability

Preventing structure from decaying:

1. Architecture Decision Records
2. Architectural fitness functions
3. Bulkheads
4. Two-way-door decisions
5. Delayed decision-making
6. Chaos engineering

A good article that combines these into a decision workflow: determine reversibility, decide whether the choice can be delayed, record it, enforce it, contain its failure, and test its assumptions. ([DEV Community][1])

This is document is based on our architecture and above article, We should update this document with better strategy in future. 

The roadmap should therefore retain its modular-monolith-first architecture, but add an explicit **architecture governance and experimentation layer**.

## Alignment audit

| Article technique | Existing roadmap                                                           | Assessment      | Refinement required                                                    |
| ----------------- | -------------------------------------------------------------------------- | --------------- | ---------------------------------------------------------------------- |
| ADRs              | Initial ADR list, architecture choices, extraction criteria                | Strong          | Add assumptions, reversibility, owner, review trigger and linked tests |
| Fitness functions | Import checks, contract tests, load tests and authorization tests          | Partial         | Create a named catalogue of automated architectural rules              |
| Bulkheads         | Bounded queues, separate workers, rate limits, timeouts                    | Partial         | Define failure domains and resource budgets explicitly from the MVP    |
| Two-way doors     | Ports, adapters, public IDs and reversible projections                     | Partial         | Classify every significant decision by reversal cost                   |
| Delayed decisions | No premature Kafka, Redis, microservices, sharding or active-active writes | Strong          | Add measurable triggers and review dates for delayed decisions         |
| Chaos engineering | Game days and failure injection in later phases                            | Strong but late | Begin with small hypothesis-driven experiments during the MVP          |

For our artitecture the largest missing element is **fitness functions**. The next-largest gaps are formal decision reversibility and explicit failure-domain design.

---

# Important corrections to the linked article

The six-tool framework is useful, but some examples in the article are too absolute to apply literally.

## 1. A repository does not make databases freely interchangeable

The article presents a repository abstraction as making a future database replacement relatively local. A repository can isolate business logic from SQL and make testing easier, but replacing PostgreSQL with Cassandra or another distributed database still changes:

* Data modelling
* Transaction boundaries
* Query capabilities
* Indexing
* Consistency guarantees
* Pagination
* Failure handling
* Operational procedures
* Backfills and migration strategy

Use **domain-specific persistence interfaces**, but do not design to the lowest common denominator or claim that the database has become a plug-in.

Good:

```go
type PostStore interface {
    Create(ctx context.Context, post Post) error

    PagePublishedByAuthors(
        ctx context.Context,
        authors []UserID,
        cursor PostCursor,
        limit int,
    ) ([]Post, error)
}
```

Poor:

```go
type GenericRepository[T any] interface {
    Save(T) error
    Find(any) (T, error)
    Delete(any) error
}
```

The objective is to contain database knowledge, not pretend that database semantics do not matter.

## 2. Separate databases are not the first bulkhead

The article lists separate databases as one possible bulkhead. That can be appropriate later, but physical database separation on day one would make the social application operationally more complicated and would remove useful local transactions.

Begin with:

```text
One PostgreSQL cluster
    ├── identity schema
    ├── profile schema
    ├── graph schema
    ├── content schema
    ├── engagement schema
    ├── feed schema
    ├── notification schema
    └── moderation schema
```

Enforce:

* One owning module per schema or table
* One writer per business entity
* No cross-module mutation
* No cross-module private SQL
* Separate foreground and background connection budgets
* Explicit transaction boundaries

Move to a separate physical database only when independent scaling, security isolation, availability, ownership, or failure containment provides measurable value.

The bulkhead pattern is fundamentally about isolating resources so that exhaustion or failure in one area does not consume the whole system; the isolation can be logical, process-level, deployment-level, or physical. ([Microsoft Learn][2])

## 3. Do not freeze a physical sharding key on day one

The article says data-partitioning decisions should not be delayed. That is too broad.

Once a system is physically sharded, the partition key becomes a difficult, potentially one-way decision. But before access patterns and skew are known, choosing a shard key may lock the application into the wrong model.

On day one, preserve **shardability**:

* Use stable resource identifiers.
* Give every entity an unambiguous owner.
* Avoid transactions spanning unbounded aggregates.
* Keep derived state rebuildable.
* Do not use database-local sequence values as external identities.
* Record likely locality dimensions such as `user_id`, `author_id` and `tenant_id`.
* Measure hot-user and celebrity-account behavior.

Delay actual physical sharding and the final partition-key commitment until the workload justifies it.

## 4. API compatibility matters more than a particular versioning syntax

The article specifically mentions a version header. The important day-one decision is a **compatibility and deprecation policy**, not necessarily one particular URL or header scheme.

For a social application with mobile clients, decide:

* Whether breaking changes are prohibited within a major version
* How fields are added and removed
* How long older clients remain supported
* How clients discover deprecation
* Whether versions use paths, headers or media types
* How event-schema compatibility is handled
* How incompatible authentication changes are rolled out

`/v1/...` is reasonable, but the real one-way door is publishing an API without an evolution policy.

## 5. Chaos engineering is not random production breakage

Chaos engineering should be an experiment:

1. Define a measurable steady state.
2. State a hypothesis.
3. Introduce a realistic failure.
4. Limit the blast radius.
5. Define abort conditions.
6. Observe the outcome.
7. Restore the system.
8. record and fix weaknesses.

The formal principles emphasize measurable steady state, realistic events, automation and minimizing blast radius. ([principlesofchaos.org][3])

Begin with deterministic fault injection in tests and staging. Controlled production experiments come later, after observability, rollback, ownership and safety controls exist.

---

# How the seven modular qualities map to the six tools

| Desired quality              | Mechanism that preserves it                                   |
| ---------------------------- | ------------------------------------------------------------- |
| High cohesion                | Module charters, ADRs and change-coupling reviews             |
| Low coupling                 | Dependency fitness functions and contract tests               |
| Business-capability focus    | Context map and ownership ADRs                                |
| Bounded context or aggregate | Explicit consistency boundaries and one-writer rules          |
| Encapsulated data            | Schema ownership fitness functions                            |
| Substitutability             | Two-way-door analysis and adapter contract tests              |
| Composability                | Public module APIs, versioned events and acyclic dependencies |
| Scalability and resilience   | Bulkheads, delayed decisions and chaos experiments            |

A modular structure can be excellent on its first day and still degrade within six months. Fitness functions are what turn “please respect the boundary” into an enforceable system property.

Thoughtworks defines architectural fitness functions as objective assessments of architectural characteristics and recommends placing them into delivery pipelines and operational monitoring. ([Thoughtworks][4])

---

# Refined architecture decision workflow

Use this process for every architecturally significant choice.

## Step 1: Classify the door

Use three categories rather than only two:

| Category                 | Meaning                                                                  |
| ------------------------ | ------------------------------------------------------------------------ |
| Reversible               | Can be changed cheaply and safely                                        |
| Reversible but expensive | Can be changed, but requires migration or coordinated rollout            |
| One-way-ish              | Published data, contracts or semantics make reversal extremely difficult |

Amazon’s two-way-door model similarly distinguishes decisions that can be reversed locally from decisions requiring more careful treatment. ([Amazon News][5])

## Step 2: Decide whether it is needed now

Ask:

```text
What evidence do we currently have?
What information will waiting provide?
What is the cost of waiting?
What is the cost of deciding incorrectly now?
```

## Step 3: Write an ADR

Michael Nygard’s original ADR proposal describes concise records for architecturally significant decisions affecting structure, quality attributes, dependencies, interfaces or construction techniques. ([Cognitect.com][6])

Use this expanded template:

```text
# ADR-012: Begin with query-on-read home feeds

Status:
Accepted

Owner:
Feed team / application team

Date:
2026-07-17

Decision class:
Reversible but expensive

Context:
The MVP requires chronological feeds.
Expected follow counts and traffic are still uncertain.

Quality attributes:
- Feed freshness
- Read latency
- Database load
- Privacy correctness
- Operational simplicity

Options considered:
1. Query on read
2. Fan-out on write
3. Hybrid fan-out
4. Dedicated feed service

Decision:
Use query on read with batched author queries and cursor pagination.

Consequences:
Positive:
- Minimal derived state
- Simple deletion and privacy behavior
- Easy iteration

Negative:
- Read cost grows with followed-author count
- Celebrity and high-follow-count users may need another path

Assumptions:
- Normal users follow a manageable number of accounts
- Indexed PostgreSQL queries meet the feed SLO
- Chronological ranking is acceptable initially

Reversibility:
Introduce a feed projection populated from PostPublished events.
Keep Content authoritative so projections remain rebuildable.

Review triggers:
- Feed p95 misses its SLO after query optimization
- Feed queries exceed their assigned database budget
- Follow-count distribution makes query-on-read unbounded
- Product requires nonchronological ranking

Fitness functions:
- Feed cannot query private Content tables
- Feed responses must be reauthorized through Content
- Cursor pagination must remain stable

Bulkhead:
Feed queries have a bounded concurrency limit and deadline.

Chaos experiment:
Disable the feed projection or delay feed queries and verify that
direct post access and post creation remain available.
```

## Step 4: Add a fitness function

Every enforceable ADR should link to at least one automated check.

## Step 5: Define the failure boundary

Document:

```text
What resources can this component consume?
What happens when it becomes slow?
Which user journeys must remain available?
What must not share its queue, pool or failure path?
```

## Step 6: Test the assumptions

Create a failure or load experiment tied to the ADR.

## Step 7: Revisit by trigger, not memory

Each delayed decision needs:

* Owner
* Review date
* Evidence required
* Metric or event that triggers review
* Consequence of continuing to delay

The linked article proposes substantially the same sequence: classify reversibility, delay where appropriate, document, enforce, isolate and test. ([DEV Community][1])

---

# Refined day-one decision register

## Decisions to make immediately

| Decision                            | Door class                | Day-one treatment                                                      |
| ----------------------------------- | ------------------------- | ---------------------------------------------------------------------- |
| Module and data ownership           | One-way-ish               | Define bounded contexts and exactly one writer per table               |
| Stable external resource identity   | One-way-ish               | Do not expose mutable usernames or local database IDs as identity      |
| Authentication and account recovery | One-way-ish               | Choose the identity model and security boundaries                      |
| Privacy, block and mute semantics   | One-way-ish               | Define authoritative access rules before feeds and caches              |
| API compatibility policy            | One-way-ish               | Define additive changes, deprecation and client support                |
| Deletion and moderation lifecycle   | One-way-ish               | Define hide, delete, purge, appeal and audit states                    |
| Event envelope                      | Reversible but expensive  | Include event ID, type, schema version, actor, aggregate and timestamp |
| Source-of-truth rules               | One-way-ish               | Content is authoritative; feeds and search are projections             |
| Observability context               | Reversible but expensive  | Standardize request, trace, user, resource and deployment identifiers  |
| Backup and restore model            | One-way-ish operationally | Define RPO, RTO and restore verification                               |

## Decisions to make now but preserve reversibility

| Decision            | Initial choice                 | Reversibility mechanism                                              |
| ------------------- | ------------------------------ | -------------------------------------------------------------------- |
| Application shape   | Modular monolith               | Public module facades and isolated data ownership                    |
| Main language       | Go                             | Network and event boundaries only where specialization later matters |
| Transactional store | PostgreSQL                     | Domain-specific persistence ports and controlled migrations          |
| Media storage       | Object storage and CDN         | Thin media-storage capability adapter                                |
| API style           | REST with an explicit contract | Compatibility tests and deprecation policy                           |
| Initial feed        | Query on read                  | Rebuildable feed projection from content events                      |
| Initial search      | PostgreSQL-backed search       | Search remains a derived projection                                  |
| Async publication   | Transactional outbox           | Publisher abstraction and idempotent consumers                       |

A modular monolith is particularly suitable while boundaries are still being learned. Fowler notes that even experienced teams have difficulty identifying correct service boundaries at the beginning, while microservices introduce an additional operational premium. ([martinfowler.com][7])

## Decisions to delay

| Decision                           | Do not introduce until                                                                     |
| ---------------------------------- | ------------------------------------------------------------------------------------------ |
| Redis or another distributed cache | A measured hot-read path exceeds its database budget and staleness behavior is defined     |
| Kafka or another event log         | Outbox publication cannot meet lag, replay, retention or independent-consumer requirements |
| Dedicated search engine            | PostgreSQL search cannot meet required relevance, language, scale or latency               |
| Feed fan-out                       | Query-on-read misses the SLO after indexing, batching and query optimization               |
| Recommendation model               | Reliable impression and interaction data exists                                            |
| Graph database                     | Concrete graph traversals cannot be served economically from the existing model            |
| Microservices                      | A stable module needs independent scale, deployment, failure isolation or ownership        |
| Physical database separation       | A module has an independent operational or security requirement                            |
| Sharding                           | A single optimized database cannot meet capacity, maintenance or availability targets      |
| Active-active regional writes      | Product requirements justify explicit conflict-resolution complexity                       |
| Kubernetes                         | Deployment count and operational requirements justify an orchestrator                      |

“Delayed” does not mean ignored. Every row belongs in the decision register with an owner and measurable trigger.

---

# Day-one architectural fitness functions

Create an `architecture/fitness/` directory and give each rule an identifier.

```text
architecture/
├── adr/
├── fitness/
├── experiments/
├── diagrams/
├── decision-register.yaml
└── module-ownership.yaml
```

## Structural fitness functions

| ID      | Rule                                                                   | Enforcement                                            |
| ------- | ---------------------------------------------------------------------- | ------------------------------------------------------ |
| MOD-001 | Module dependencies must be acyclic                                    | Analyze the Go or Cargo dependency graph in CI         |
| MOD-002 | Callers may use only another module’s public facade                    | Reject imports of private implementation packages      |
| MOD-003 | Business modules cannot import HTTP, SQL-driver or vendor SDK packages | Static dependency rule                                 |
| MOD-004 | Platform packages cannot contain business concepts                     | Package-ownership review and forbidden dependency list |
| MOD-005 | Every module has a named owner and public contract                     | Metadata validation                                    |
| MOD-006 | No `common`, `utils` or generic business-model package                 | CI path and package-name rule                          |

For Go, combine public facade packages with nested `internal` implementation packages. For Rust, use workspace crates with private modules and a deliberately small `pub` surface.

## Data fitness functions

| ID       | Rule                                                                   | Enforcement                                               |
| -------- | ---------------------------------------------------------------------- | --------------------------------------------------------- |
| DATA-001 | Every table has exactly one writer module                              | Ownership manifest checked against migrations and queries |
| DATA-002 | Modules cannot query another module’s private tables                   | SQL-location and integration checks                       |
| DATA-003 | Cross-module references use stable identifiers                         | Schema review test                                        |
| DATA-004 | Feed and Search data is marked derived and rebuildable                 | Projection contract test                                  |
| DATA-005 | Database migrations use expand-and-contract                            | Migration compatibility pipeline                          |
| DATA-006 | Every uniqueness or lifecycle invariant has a database or domain check | Schema and test review                                    |

## API and event fitness functions

| ID      | Rule                                                             | Enforcement                      |
| ------- | ---------------------------------------------------------------- | -------------------------------- |
| API-001 | Published API changes are backward compatible                    | Contract diff in CI              |
| API-002 | List endpoints require cursor pagination and a maximum page size | API schema validation            |
| API-003 | Mutating retryable endpoints have defined idempotency            | Contract and integration tests   |
| EVT-001 | Events contain ID, type, aggregate, timestamp and schema version | Event-schema validation          |
| EVT-002 | Event changes are backward compatible                            | Schema compatibility test        |
| EVT-003 | Every consumer handles duplicate delivery                        | Duplicate-event integration test |

## Security fitness functions

These are especially important for a social application:

| ID      | Rule                                                                                |
| ------- | ----------------------------------------------------------------------------------- |
| SEC-001 | A blocked user cannot access content through direct read, feed, search or media URL |
| SEC-002 | Private-account content cannot be returned from a stale feed projection             |
| SEC-003 | Deleted or moderator-hidden content is rejected during authoritative hydration      |
| SEC-004 | Object IDs never bypass authorization                                               |
| SEC-005 | Uploads enforce size, type and processing-state restrictions                        |
| SEC-006 | Administrative actions create immutable audit entries                               |

## Reliability and performance fitness functions

| ID       | Rule                                                         | Cadence                          |
| -------- | ------------------------------------------------------------ | -------------------------------- |
| REL-001  | Every queue has a finite capacity                            | Every pull request               |
| REL-002  | Every external call has a deadline                           | Every pull request               |
| REL-003  | Retries have an attempt limit and backoff                    | Every pull request               |
| REL-004  | Duplicate work produces an idempotent outcome                | Integration pipeline             |
| REL-005  | Graceful shutdown stops intake before waiting for work       | Integration pipeline             |
| PERF-001 | Feed and post APIs remain inside agreed budgets              | Nightly representative load test |
| PERF-002 | Database connection use stays inside configured budgets      | Load and staging                 |
| PERF-003 | Worker queue age stays inside its freshness objective        | Runtime monitor                  |
| OPS-001  | Every async component publishes lag, error and retry metrics | Deployment validation            |

Do not run a large, noisy load test on every small pull request. Use:

```text
Every PR:
    static architecture rules
    contracts
    unit tests
    small regression benchmarks

Nightly:
    representative load
    concurrency tests
    migration compatibility
    failure injection

Before release:
    capacity test
    soak test
    rollback rehearsal

Periodically:
    game day
    restore exercise
    projection rebuild
```

---

# Explicit bulkhead design

A bounded context is not automatically a bulkhead. In-process modules share:

* Memory
* CPU
* File descriptors
* Process lifetime
* Database pools
* Runtime scheduler
* Deployment fate

Therefore use several levels of isolation.

## Level 1: Logical bulkheads

Introduce these in the MVP:

```text
Per-operation concurrency limits
Bounded queues
Request deadlines
External-call timeouts
Retry budgets
Per-user and per-tenant quotas
Maximum upload sizes
Maximum page sizes
Maximum fan-out batches
```

## Level 2: Workload bulkheads

Do not put all background work into one queue.

```text
media-processing
feed-fanout
notifications
search-indexing
moderation-enrichment
analytics-export
```

Each should have:

* Independent queue capacity
* Independent concurrency
* Independent retry policy
* Independent dead-letter behavior
* Independent lag metric

A ten-minute notification backlog should not prevent moderation events or media processing.

## Level 3: Process bulkheads

Build two commands from the same repository:

```text
cmd/api
cmd/worker
```

They may use the same release and PostgreSQL cluster, but should have different:

* CPU and memory limits
* Database connection budgets
* Autoscaling behavior
* Health checks
* Shutdown policy
* Alerting

This gives useful failure isolation without introducing microservice ownership, network APIs or distributed data.

CPU-heavy or untrusted media parsing can later receive its own worker process or sandbox.

## Level 4: Deployment and data bulkheads

Use only when evidence requires them:

```text
Independent deployment
Independent service
Independent database
Separate cluster
Cell-based architecture
Regional isolation
```

---

# Social-application chaos experiment catalogue

Every experiment should have a steady state, hypothesis, blast radius, abort condition and recovery step.

## Experiment 1: Feed projector unavailable

**Steady state**

```text
Posts can be created.
Direct post reads succeed.
Ordinary posts appear in feeds inside the freshness objective.
```

**Fault**

Stop the feed projector for 15–30 minutes.

**Expected behavior**

* Post creation remains successful.
* Direct post reads remain available.
* Feed freshness degrades.
* Feed lag alert fires.
* Events remain durable.
* Restarting the projector catches up.
* No post is permanently omitted.

This verifies that Content is authoritative and Feed is a projection.

## Experiment 2: Notification provider is slow

**Fault**

Make the provider delay every call and return intermittent errors.

**Expected behavior**

* Likes, comments and posts still succeed.
* Notification work remains queued.
* Retries are bounded and use backoff.
* Notification backlog does not consume the feed or media worker pool.
* Queue-age alerts fire.
* Recovery drains the backlog at a controlled rate.

This tests the notification bulkhead.

## Experiment 3: Duplicate outbox delivery

Deliver the same `PostPublished` or `CommentCreated` event multiple times.

**Expected behavior**

* One feed projection entry
* One search document version
* One user-visible notification
* No duplicated counter effects
* Consumer acknowledges the duplicate safely

## Experiment 4: Worker crash after performing a side effect

Crash a worker after writing a media variant or calling an external provider but before acknowledging the task.

**Expected behavior**

* Retry does not create an inconsistent duplicate.
* Work is resumed or reconciled.
* Attempt and idempotency identifiers make the outcome traceable.

## Experiment 5: Database connection-pool exhaustion

Saturate the pool using controlled concurrent load.

**Expected behavior**

* Requests time out within their deadlines.
* Load shedding begins before memory or goroutine counts become unbounded.
* Background work cannot consume the entire API connection budget.
* Recovery occurs without restarting every process.

## Experiment 6: Stale feed entry after a block

1. Alice follows Bob.
2. Bob’s post enters Alice’s feed.
3. Alice blocks Bob.
4. The feed projection remains temporarily stale.

**Expected behavior**

The authoritative hydration and authorization check removes Bob’s post before returning the response.

This is a correctness experiment, not just an availability experiment.

## Experiment 7: High-fan-out account publishes

Simulate an account with a very large follower set.

**Expected behavior**

* `CreatePost` latency does not depend directly on follower count.
* Fan-out is asynchronous.
* The queue remains bounded.
* High-fan-out traffic cannot starve ordinary posts.
* The system can switch the account to a read-merge path.

---

# Refined phase-wise roadmap

## Phase 0 — Architecture and decision foundation

**Duration:** one to two weeks.

### Architecture work

Define:

* Product scope
* Bounded contexts
* Module responsibilities
* Authoritative data ownership
* Aggregate boundaries
* Privacy and authorization rules
* Strong versus eventual consistency
* Initial SLOs
* Failure domains
* Data-retention and deletion lifecycle

### Six-tool work

Create:

```text
Context map
C4 system and container diagrams
Module ownership manifest
Decision register
Initial ADR collection
Fitness-function catalogue
Failure-domain map
Chaos experiment backlog
```

Initial ADRs should include:

```text
ADR-001 Modular monolith first
ADR-002 Module and table ownership
ADR-003 PostgreSQL as authoritative store
ADR-004 Object storage for media
ADR-005 Strong and eventual consistency boundaries
ADR-006 Stable public identifiers
ADR-007 API compatibility and pagination
ADR-008 Authentication and account recovery
ADR-009 Privacy, blocks and authorization
ADR-010 Asynchronous-work strategy
```

### Exit gate

Every module must have:

* A business responsibility
* A public interface
* Owned data
* Allowed dependencies
* Forbidden dependencies
* Consistency model
* Failure behavior
* Named owner

---

## Phase 1 — Modular-monolith MVP

### Architecture

Implement:

```text
Identity
Profile
Social Graph
Content
Media metadata
Engagement
Chronological feed
Blocks and mutes
Basic moderation
```

Use:

```text
Go modular monolith
PostgreSQL
Object storage
CDN
REST contract
Cursor pagination
One deployment region
```

Do not add:

```text
Kafka
Redis
Microservices
Dedicated search
Database sharding
Recommendation models
Active-active regional writes
```

### Fitness functions

Activate all structural, ownership, API and authorization fitness functions in CI.

### Bulkheads

Introduce:

* Separate `api` and `worker` commands
* Bounded worker queues
* Per-operation concurrency limits
* API and worker database budgets
* Upload size limits
* Per-account request quotas

### Experiments

Run:

* Duplicate post submission
* Worker killed mid-task
* Object storage temporarily unavailable
* Queue filled to capacity
* Block-after-feed-entry security test

### Exit gate

* Architecture violations fail CI.
* No cross-module private SQL exists.
* Authorization works through every read path.
* Backups can be restored.
* Overload causes controlled rejection rather than unbounded resource growth.

---

## Phase 2 — Production readiness

### Architecture

Add:

* Transactional outbox
* Durable background processing
* Media scanning and variants
* Notifications
* Audit trail
* Rate limiting
* Idempotency keys
* Structured logs, metrics and traces
* Feature flags
* Expand-and-contract migrations
* Dead-letter state
* Reconciliation jobs

### Fitness functions

Add:

* Event compatibility
* Duplicate-delivery tests
* Migration compatibility
* SLO checks
* Queue-bound checks
* Required telemetry checks
* Graceful shutdown tests

### Bulkheads

Separate concurrency and queues for:

```text
media
feed
notifications
search projections
moderation
analytics
```

### Chaos experiments

Run:

* Notification provider outage
* Database pool exhaustion
* Duplicate outbox delivery
* Worker crash after a side effect
* Delayed feed processing
* Corrupted or missing cache entry, if caching exists

### Exit gate

* Foreground user actions survive background dependency failures.
* Every queue has lag monitoring.
* Every retry is bounded.
* Projection rebuilds and replay have been tested.
* Restore and rollback procedures have been exercised.

---

## Phase 3 — Evidence-driven optimization

Do not select new infrastructure because traffic is “growing.” Use a measured trigger.

Optimization order:

```text
1. Inspect query plans
2. Add or correct indexes
3. Eliminate N+1 queries
4. Batch module reads
5. Reduce payload size
6. Move side effects out of request paths
7. Tune connection budgets
8. Add CDN caching
9. Add application caching to proven hot paths
10. Add read replicas where staleness is acceptable
```

For each new component, execute the six-step workflow.

### Example: introducing Redis

Require:

* A documented hot-read path
* Measured database cost
* Defined acceptable staleness
* Cache key and ownership model
* TTL and invalidation policy
* Cache-unavailable behavior
* Stampede protection
* An ADR
* A cache-bypass fitness test
* A cache-failure chaos experiment

### Example: introducing a message broker

Require:

* Outbox lag cannot meet the freshness objective, or
* Multiple consumers require independent retention and replay, or
* Throughput exceeds the safe polling design

Then define:

* Ordering guarantees
* Retention
* Replay
* Consumer idempotency
* Poison-message behavior
* Per-workload queues or topics
* Broker-unavailable behavior
* Reversal or migration plan

### Exit gate

* A capacity model exists.
* The first saturated resource is known.
* The system handles an agreed multiple of expected peak traffic.
* New infrastructure solves an observed problem.

---

## Phase 4 — Feed, search and projection scale

### Architecture

Introduce where justified:

* Feed projection
* Hybrid fan-out
* High-fan-out-account handling
* Search projection
* Derived counters
* Event replay
* Online projection rebuild
* Reconciliation jobs

All derived systems must remain disposable:

```text
Delete feed projection -> rebuild from authoritative content/events
Delete search index -> rebuild
Lose reaction counts -> recompute from reaction edges
Lose profile summary cache -> reconstruct
```

### Fitness functions

Add:

* Projection rebuild correctness
* Projection lag objectives
* Event replay compatibility
* Current authorization during feed hydration
* No synchronous fan-out in post creation
* Celebrity-account workload test

### Chaos experiments

* Stop every projector
* Replay a large event range
* Delay one partition
* Deliver events out of order where ordering is not guaranteed
* Delete a projection and rebuild it
* Create a privacy change while projections are stale

### Exit gate

Derived systems can be stopped, recreated and caught up without losing authoritative business state.

---

## Phase 5 — Selective service extraction

A component becomes a service only when it has:

* Stable business boundaries
* Stable public operations
* Exclusive data ownership
* No external reads of private tables
* Independent telemetry
* Independent operational owner
* Versioned contracts
* Idempotent operations
* A measurable reason for extraction

Valid reasons include:

```text
Independent resource scaling
Independent release cadence
Failure isolation
Security isolation
Different availability objective
Different technology requirements
Stable team ownership
```

Likely early candidates:

1. Media processing
2. Notification delivery
3. Search
4. Feed generation
5. Recommendation scoring

### Required extraction ADR

The ADR must include:

* Why in-process isolation is insufficient
* Expected benefit
* Added network and operational failure modes
* Data migration plan
* Single-writer cutover
* Rollback plan
* Shadow-testing plan
* Contract compatibility
* Expected cost
* Post-extraction success metrics

### Extraction sequence

```text
1. Strengthen in-process boundary
2. Measure component independently
3. Add transport behind the same public interface
4. Copy historical data
5. Stream subsequent changes
6. Shadow reads or requests
7. Compare results
8. Shift traffic gradually
9. Remove old writes
10. Remove direct table access
```

### Fitness functions

* No shared private database access
* Contract backward compatibility
* No synchronous request cycles
* Every remote call has a deadline
* Every retry is safe
* Service can be deployed independently
* Old and new implementations return equivalent results during shadowing

### Chaos experiments

* Service unavailable
* Network latency
* Partial response loss
* Old service version communicating with new client
* New service version communicating with old client
* Event backlog during deployment
* Rollback after partial data migration

### Exit gate

Measurements demonstrate that extraction improved at least one intended characteristic without creating unacceptable latency, reliability or operating cost.

---

## Phase 6 — Sharding, cells and multiple regions

These are one-way-ish decisions and should receive the strongest review.

Before sharding:

* Confirm a single optimized primary is insufficient.
* Model key distribution.
* Identify celebrity and hot-key behavior.
* Analyze cross-partition operations.
* Test rebalancing.
* Build shard-aware observability.
* Rehearse migration with production-like data.
* Define rollback before cutting over.

Cell-based architecture can provide a stronger bulkhead:

```text
Global routing
    |
    +-- Cell A: users 0–N
    +-- Cell B: users N–M
    +-- Cell C: users M–Z
```

Each cell should limit the blast radius of:

* Bad deployment
* Database overload
* Queue backlog
* Hot users
* Corrupt projection
* Regional dependency failure

Multi-region writes should wait until the product has explicit semantics for conflicting:

* Profile edits
* Username claims
* Follows
* Blocks
* Reactions
* Post edits
* Moderation decisions

---

# Final architecture definition of done

An architectural change is not complete until it has:

1. A decision classification
2. An ADR
3. Explicit assumptions
4. A reversal or migration plan
5. A review trigger
6. At least one fitness function
7. A defined failure boundary
8. Resource budgets
9. Required telemetry
10. A failure or load experiment

That is the most important refinement from the article.

The previous roadmap gives the social application a good evolutionary structure. The article adds the control system that prevents the structure from becoming accidental: **ADRs make choices visible, fitness functions keep boundaries intact, bulkheads contain bad outcomes, reversible decisions preserve options, delayed decisions avoid premature lock-in, and chaos experiments test whether the assumptions are actually true.**

[1]: https://dev.to/manojsatna31/6-tools-that-will-save-you-from-architecture-hell-no-buzzwords-1bi1 " 6 Tools That Will Save You From Architecture Hell (No Buzzwords) - DEV Community"
[2]: https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead?utm_source=chatgpt.com "Bulkhead Pattern - Azure Architecture Center | Microsoft Learn"
[3]: https://principlesofchaos.org/?utm_source=chatgpt.com "PRINCIPLES OF CHAOS ENGINEERING"
[4]: https://www.thoughtworks.com/radar/techniques/architectural-fitness-function?utm_source=chatgpt.com "Architectural fitness function | Technology Radar | Thoughtworks"
[5]: https://www.aboutamazon.com/news/company-news/amazon-ceo-andy-jassy-2024-letter-to-shareholders?utm_source=chatgpt.com "Amazon CEO Andy Jassy’s 2024 Letter to Shareholders"
[6]: https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions?utm_source=chatgpt.com "Documenting Architecture Decisions - Cognitect.com"
[7]: https://martinfowler.com/bliki/MonolithFirst.html?utm_source=chatgpt.com "Monolith First - Martin Fowler"
