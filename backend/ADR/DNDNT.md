# Does & Don'ts

An Architectural Decision Record for choice for design and architecture

Our assumptions:

> Build strong **in-process component boundaries first**. Turn a component into an out-of-process service only when independent deployment, scaling, failure isolation, or ownership creates measurable value.

Microservices do not create cohesion or fix coupling; they merely turn poor coupling into network calls, distributed transactions, versioning problems, and operational overhead. A modular-monolith-first strategy lets you discover stable boundaries while changes are still inexpensive. Fowler makes the same argument: service boundaries are difficult to identify early, and the operational premium of microservices is substantial. ([martinfowler.com][1])

The C4 definition is useful here: a component is related functionality encapsulated behind a well-defined interface, and it does not need to be separately deployable. ([C4 model][2])

This roadmap assumes a follow-based social application with profiles, posts, media, comments, reactions, home feeds, search, notifications, blocks, reports, and moderation. Direct messaging, live streaming, and real-time audio should initially be treated as separate future bounded contexts because their consistency and infrastructure requirements are substantially different.

# 1. Recommended day-one architecture

Use a **modular monolith**, PostgreSQL, object storage, and a CDN.

Our of Rust & Go, Move with go:

* **Go for the main application**
* **PostgreSQL as the authoritative data store**
* **Object storage and a CDN for media**
* **The same application repository and release for API and background workers**
* **Rust later for independently deployed CPU-intensive components**, such as video processing, recommendation scoring, or specialized networking

Do not create a polyglot monolith. Pick one language for the initial application.

```text
Web / iOS / Android
          |
          v
 CDN / WAF / Load Balancer
          |
          v
+----------------------------------------+
| Go modular application                 |
|                                        |
|  Identity       Profile                |
|  Social Graph   Content                |
|  Media          Engagement             |
|  Feed           Moderation             |
|  Notifications  Search Projection      |
|                                        |
|  API process + worker process          |
|  same repository, modules and release  |
+-------------------+--------------------+
                    |
          +---------+---------+
          |                   |
          v                   v
     PostgreSQL          Object storage
  schema per module        plus CDN
          |
          v
     Outbox events
          |
          v
 Media processing / notifications /
 feed projection / search indexing
```

Initially, the worker can run in the API process. Asynchronous or CPU-intensive work can later run through a separate `worker` command built from the same source and release. That is operational isolation, not yet an independently evolving microservice architecture.

# 2. Define the bounded contexts

A bounded context is not simply a directory. It is an area in which domain terms have a consistent meaning and a clear owner. Large domains are divided into bounded contexts precisely because maintaining one unified model becomes increasingly difficult. ([martinfowler.com][3])

A reasonable initial context map is:

| Module        | Responsibility                                     | Authoritative data                                              |
| ------------- | -------------------------------------------------- | --------------------------------------------------------------- |
| Identity      | Accounts, authentication, sessions, account status | Accounts, credentials or identity-provider references, sessions |
| Profile       | Public identity and preferences                    | Handle, display name, bio, avatar reference, profile settings   |
| Social Graph  | Relationships between users                        | Follows, follow requests, blocks, mutes                         |
| Content       | Post lifecycle and visibility                      | Posts, post versions, visibility, publication status            |
| Media         | Upload and processing lifecycle                    | Media metadata, storage keys, processing status                 |
| Engagement    | Interactions around content                        | Comments, replies, reactions                                    |
| Feed          | Home-timeline projection                           | Derived feed entries and ranking metadata                       |
| Notifications | User-facing notification delivery                  | Notification inbox and delivery state                           |
| Moderation    | Reports and enforcement decisions                  | Reports, moderation actions, appeal state, audit trail          |
| Search        | Searchable projection                              | Derived search documents, not authoritative content             |
| Analytics     | Product telemetry and aggregates                   | Derived events and analytical models                            |

Important distinctions:

* **Identity is not Profile.** Authentication data should not be mixed with biography, avatar, or follower counts.
* **Content is not Feed.** A post remains valid even if every feed entry is deleted and rebuilt.
* **Feed and Search are projections.** They are not sources of truth.
* **Notifications are effects of activity.** They should not participate in the transaction that creates the post or reaction.
* **Moderation is a business capability**, not merely an admin controller.

# 3. Translate the seven principles into enforceable rules

## High cohesion

Everything required to create, edit, publish, authorize, and delete a post belongs to the Content module.

Do not spread a post operation across:

```text
controllers/
services/
repositories/
models/
validators/
```

That is horizontal organization by technical layer. It makes one feature span the entire codebase.

Instead:

```text
content/
  public API
  use cases
  domain rules
  persistence adapter
  tests
```

## Low coupling

Other modules may depend only on the Content module’s public API. They may not:

* Import its database models
* Query its tables directly
* Mutate its entities
* Depend on its HTTP handlers
* Depend on concrete repository implementations

Public APIs should be small and business-oriented:

```go
type Content interface {
    CreatePost(
        ctx context.Context,
        cmd CreatePost,
    ) (PostID, error)

    GetVisiblePosts(
        ctx context.Context,
        viewer UserID,
        ids []PostID,
    ) ([]PostView, error)

    DeletePost(
        ctx context.Context,
        actor UserID,
        post PostID,
    ) error
}
```

Avoid generic interfaces such as:

```go
type Repository[T any] interface {
    Save(T) error
    Find(any) (T, error)
    Delete(any) error
}
```

These provide superficial substitutability but hide domain semantics, transaction behavior, locking, pagination, and query requirements.

## Focused on a business capability

Use package-by-component rather than package-by-layer.

Good:

```text
identity
profile
graph
content
media
engagement
feed
moderation
```

Poor:

```text
controllers
services
repositories
entities
dto
utils
```

## Bounded context or aggregate

An aggregate is a **transactional consistency boundary**, not a synonym for module.

Good initial aggregates include:

* `Account`
* `Profile`
* `FollowEdge`
* `BlockEdge`
* `Post`
* `MediaAsset`
* `Comment`
* `Reaction`

Do not create a `User` aggregate containing:

* Profile
* All followers
* All followed users
* All posts
* All reactions
* All notifications

Those collections are unbounded and cannot be loaded, locked, or updated as one consistency boundary.

A Post aggregate may contain:

```text
post identity
author identity
body
visibility
publication state
media references
edit version
```

It should not contain every comment and reaction. Comments and reactions grow independently and should be separate aggregates.

## Encapsulated data

Give every table exactly one writer module.

Recommended PostgreSQL schemas:

```text
identity.*
profile.*
graph.*
content.*
media.*
engagement.*
feed.*
notification.*
moderation.*
platform.*
```

Rules:

1. Only the owner module writes its tables.
2. Other modules use its public API.
3. Cross-module references use stable identifiers.
4. Invariants inside one module are enforced with database constraints and transactions.
5. Cross-module screen composition uses batch APIs or derived read models.
6. Production business logic must not contain ad hoc joins across private schemas.

Database schemas alone do not enforce modularity. Combine them with code boundaries, import checks, ownership documentation, and tests.

## Substitutable

Apply substitutability at **volatile external boundaries**:

* Object storage provider
* Email provider
* Push notification provider
* Authentication provider
* Search engine
* Clock
* ID generator

Do not attempt to make every database interchangeable. A generic “works with PostgreSQL, Cassandra, MongoDB, and files” storage interface usually becomes a lowest-common-denominator abstraction.

Use domain-specific ports:

```go
type PostStore interface {
    Insert(ctx context.Context, post Post) error

    PageByAuthors(
        ctx context.Context,
        authorIDs []UserID,
        cursor Cursor,
        limit int,
    ) ([]Post, error)
}
```

## Composable

Composition should happen through:

* Public module facades
* Application workflows
* Batch query APIs
* Versioned domain events
* Derived read models

For example, publishing a post should look conceptually like:

```text
1. Verify author is active
2. Verify referenced media is ready
3. Create post in Content transaction
4. Insert PostPublished event into outbox
5. Commit
6. Feed, Search and Notification react asynchronously
```

Do not send notifications or update a search engine within the database transaction.

# 4. Suggested Go repository layout

Use nested `internal` packages to make implementation details inaccessible outside their owning component.

```text
social/
├── cmd/
│   ├── api/
│   │   └── main.go
│   └── worker/
│       └── main.go
│
├── internal/
│   ├── bootstrap/
│   │   ├── config.go
│   │   ├── modules.go
│   │   └── server.go
│   │
│   ├── modules/
│   │   ├── identity/
│   │   │   ├── api.go
│   │   │   ├── module.go
│   │   │   └── internal/
│   │   │       ├── domain/
│   │   │       ├── application/
│   │   │       └── postgres/
│   │   │
│   │   ├── profile/
│   │   ├── graph/
│   │   ├── content/
│   │   ├── media/
│   │   ├── engagement/
│   │   ├── feed/
│   │   ├── notification/
│   │   └── moderation/
│   │
│   └── platform/
│       ├── clock/
│       ├── database/
│       ├── id/
│       ├── telemetry/
│       └── transaction/
│
├── migrations/
├── api/
│   └── openapi.yaml
├── architecture/
│   ├── context.dsl
│   ├── containers.dsl
│   ├── components.dsl
│   └── adr/
└── go.mod
```

Go’s `internal` mechanism prevents packages outside the permitted parent tree from importing protected implementation packages. The official Go layout guidance also recommends `internal` for server implementation packages and `cmd` for multiple commands. ([Go][4])

The `platform` area may contain technical infrastructure, but must not become a dumping ground for business concepts. Avoid:

```text
platform/models
platform/common
platform/business
platform/utils
```

Add a build-time architecture test that rejects forbidden imports. For example:

```text
feed may import content's public API
feed may not import content/internal/*
content may not import feed
notification may not be imported by content
graph may not query content tables
```

For an all-Rust implementation, use one Cargo workspace with a crate per component and explicit dependencies between crates. Keep implementation modules private and expose only each component’s public facade.

# 5. Day-one data and API choices

## Identifiers

Use stable IDs that do not encode mutable business information.

A practical schema is:

```text
Internal database key: BIGINT
External public ID:    UUIDv7
```

For example:

```sql
CREATE TABLE content.posts (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    public_id   uuid NOT NULL UNIQUE,
    author_id   bigint NOT NULL,
    body        text NOT NULL,
    visibility  text NOT NULL,
    state       text NOT NULL,
    created_at  timestamptz NOT NULL,
    updated_at  timestamptz NOT NULL
);
```

UUIDv7 is standardized as a time-ordered UUID based on a Unix-epoch timestamp plus random bits. ([RFC Editor][5])

Benefits of the two-ID model:

* Compact internal foreign keys
* Opaque external identifiers
* Stable external contracts
* Easier extraction into another service later
* No dependence on database sequence values in public events

Opaque IDs do not replace authorization. An attacker who learns another user’s post ID must still be denied access. OWASP identifies object-level authorization as a primary API risk. ([OWASP][6])

## API style

Start with REST and an OpenAPI contract.

Examples:

```text
POST   /v1/posts
GET    /v1/posts/{post_id}
DELETE /v1/posts/{post_id}

PUT    /v1/users/{user_id}/follow
DELETE /v1/users/{user_id}/follow

PUT    /v1/posts/{post_id}/reactions/like
DELETE /v1/posts/{post_id}/reactions/like

GET    /v1/feed?cursor=...
GET    /v1/users/{user_id}/posts?cursor=...
```

Use HTTP operations that are naturally idempotent for follow, unfollow, like, and unlike.

For operations such as post creation, accept an idempotency key:

```text
Idempotency-Key: 018f...
```

Store:

```text
actor ID
operation scope
idempotency key
request hash
result resource ID
response status
expiration
```

Mobile networks retry requests. Without idempotency, a successful request whose response was lost can create duplicate posts or comments.

## Cursor pagination

Use cursor pagination from the beginning.

Order chronological data by:

```text
created_at DESC, id DESC
```

The cursor contains both values:

```json
{
  "created_at": "2026-07-17T09:30:00Z",
  "id": 8675309
}
```

Do not use high-offset pagination for:

* Feeds
* Posts
* Comments
* Notifications
* Followers
* Reactions

Always enforce a maximum page size.

## Handle design

Do not use the username or email address as the user’s identity.

Store separately:

```text
user_id
public_id
handle_display
handle_normalized
```

Define on day one:

* Case-sensitivity policy
* Unicode normalization policy
* Reserved handles
* Rename policy
* Deleted-handle reuse policy
* Impersonation protection for privileged names

Put a unique constraint on the normalized value.

# 6. Consistency choices

Do not attempt strong consistency everywhere. Decide which behavior must be immediate and which can lag.

| Behavior                         | Consistency                |
| -------------------------------- | -------------------------- |
| Account status and credentials   | Strong                     |
| Handle uniqueness                | Strong                     |
| Follow, follow request and block | Strong                     |
| Post creation and visibility     | Strong inside Content      |
| Comment creation                 | Strong inside Engagement   |
| Feed appearance                  | Eventual                   |
| Reaction count                   | Eventual                   |
| Comment count                    | Eventual                   |
| Search indexing                  | Eventual                   |
| Notification delivery            | Eventual                   |
| Analytics                        | Eventual                   |
| Privacy and authorization check  | Authoritative at read time |

The last row is crucial.

A feed entry may be stale. Therefore:

```text
Feed returns candidate post IDs
       |
       v
Content batch-loads posts and performs current visibility checks
       |
       v
Only authorized posts are returned
```

A delayed `PostDeleted`, `UserBlocked`, or `ProfileMadePrivate` event must never leak content. Feed projections accelerate discovery; they must not be the final authority for access.

# 7. Initial data model

## Social graph

```sql
CREATE TABLE graph.follows (
    follower_id bigint NOT NULL,
    followee_id bigint NOT NULL,
    state       text NOT NULL,
    created_at  timestamptz NOT NULL,

    PRIMARY KEY (follower_id, followee_id),
    CHECK (follower_id <> followee_id)
);

CREATE INDEX follows_by_followee
    ON graph.follows (followee_id, follower_id);
```

You generally need both access directions:

```text
Who does Alice follow?
Who follows Alice?
```

Blocks and mutes should be separate relationships because their semantics differ.

## Posts

A useful index for a user’s post timeline is:

```sql
CREATE INDEX posts_by_author_recent
    ON content.posts (author_id, created_at DESC, id DESC)
    WHERE state = 'published';
```

Partial indexes can index only rows satisfying a predicate, and every index adds write and storage overhead. Indexes should therefore be driven by real query patterns and inspected through query plans. ([PostgreSQL][7])

## Comments

Do not store comments as a JSON array inside a post.

```text
comments
  id
  public_id
  post_id
  author_id
  parent_comment_id
  body
  state
  created_at
```

Use bounded reply depth initially. Unlimited recursive threading complicates querying, moderation, deletion, ranking, and user interfaces.

## Reactions

Prefer a uniqueness constraint:

```text
UNIQUE(actor_id, post_id, reaction_kind)
```

A repeated like then becomes naturally idempotent.

At high scale, do not update one `posts.like_count` row synchronously for every reaction. The reaction edge is authoritative; the displayed count is a projection that can be recomputed or reconciled.

## Post edits

Keep post-version or moderation history:

```text
post_versions
  post_id
  version
  body
  edited_by
  edited_at
```

This helps with:

* Moderation review
* Abuse investigations
* Restoring accidental edits
* Auditing
* Cache and search invalidation

## Deletion

Do not apply one generic `deleted_at` column to every table.

Use explicit lifecycles:

```text
draft
published
hidden
removed_by_author
removed_by_moderator
expired
purged
```

Deletion of a post is a workflow:

```text
1. Make post inaccessible
2. Emit PostRemoved
3. Remove feed projections
4. Remove search document
5. Invalidate caches
6. Schedule media deletion if unreferenced
7. Preserve required moderation audit
8. Eventually purge eligible data
```

# 8. Media architecture

Do not send large media through the main API or store image/video bytes in PostgreSQL.

Use:

```text
Client
  |
  | request upload session
  v
Media module
  |
  | short-lived signed upload authorization
  v
Object storage
  |
  | upload-completed event
  v
Media worker
  |
  +-- validate actual content type
  +-- malware/security scan
  +-- remove unwanted metadata
  +-- generate image variants
  +-- transcode video
  +-- record dimensions and duration
  +-- mark asset ready
```

A media asset state machine might be:

```text
requested
   -> uploaded
   -> scanning
   -> processing
   -> ready

uploaded/scanning/processing
   -> rejected
   -> failed

ready
   -> deleting
   -> deleted
```

Use immutable object keys and create new objects for new versions. This avoids ambiguous CDN cache invalidation.

File uploads create risks including malicious files, oversized payloads, parser exploits, storage exhaustion, and active content. OWASP recommends size limits, generated storage names, validation, and malicious-content analysis. ([OWASP Cheat Sheet Series][8])

Do not let the server download arbitrary user-supplied avatar or media URLs. That creates an SSRF surface. ([OWASP Cheat Sheet Series][9])

# 9. Feed evolution

The feed is usually the first social-media subsystem requiring specialized scaling. Evolve it gradually.

## Stage 1: query on read

For the MVP:

1. Graph returns followed author IDs.
2. Content returns recent visible posts by those authors.
3. Results are ordered by `created_at, id`.
4. Profiles are batch-loaded for the returned authors.

Public read APIs should be batch-oriented:

```go
Profiles.GetMany(ctx, userIDs)
Content.GetVisibleMany(ctx, viewerID, postIDs)
```

This avoids one call per post or author.

This approach is sufficient while follow counts and traffic remain modest. Measure it rather than guessing.

## Stage 2: feed projection

When query-on-read becomes expensive, introduce:

```text
feed.entries
  owner_user_id
  post_id
  source_author_id
  rank_key
  reason
  inserted_at
```

`feed.entries` is derived and rebuildable.

On `PostPublished`:

```text
outbox event
    |
    v
feed projector
    |
    v
insert feed entries for eligible followers
```

On `PostDeleted` or `UserBlocked`, remove or suppress corresponding entries.

## Stage 3: hybrid fan-out

Pure fan-out-on-write creates the celebrity problem: publishing one post may require millions of feed writes.

Use a hybrid strategy:

* Ordinary accounts: fan out on write.
* High-fan-out accounts: keep posts in an author timeline and merge on read.
* New followers: backfill only a limited recent window.
* Feed reads: merge normal feed entries with recent high-fan-out-author posts.
* Always hydrate and re-authorize content before returning it.

Do not fan out to every follower synchronously in the `CreatePost` request.

## Stage 4: ranking

Separate feed delivery from ranking.

```text
Candidate generation
       |
       v
Eligibility and policy filtering
       |
       v
Scoring
       |
       v
Diversity / deduplication
       |
       v
Final page
```

Begin with a chronological feed. Add ranking only after you have:

* Reliable impression events
* Click/open events
* Follow and unfollow events
* Hide/report signals
* Experiment assignment
* Reproducible offline datasets
* A chronological fallback

Store a ranking-model or algorithm version in the cursor so pagination remains coherent across ranking changes.

# 10. Asynchronous work and the outbox

Avoid this dual write:

```text
BEGIN
  INSERT post
COMMIT

publish event to broker
```

The application can crash between the commit and event publication.

Instead:

```text
BEGIN
  INSERT post
  INSERT outbox_event
COMMIT
```

A background publisher reads committed outbox events and sends them to consumers. The outbox pattern avoids inconsistency between internal database state and emitted events. ([Debezium][10])

Initial outbox schema:

```text
event_id
aggregate_type
aggregate_id
event_type
schema_version
payload
occurred_at
published_at
attempt_count
next_attempt_at
```

Every consumer must be idempotent because delivery will normally be at least once.

An event envelope should include:

```json
{
  "event_id": "uuid",
  "event_type": "content.post_published",
  "schema_version": 1,
  "aggregate_id": "post-public-id",
  "occurred_at": "2026-07-17T10:00:00Z",
  "trace_id": "...",
  "payload": {}
}
```

Initially, a PostgreSQL polling worker is sufficient. Introduce a managed queue or log only when you need one or more of:

* High asynchronous throughput
* Multiple independent consumers
* Long retention
* Replay
* Independent consumer scaling
* Cross-service event distribution

Do not add Kafka merely because the architecture contains events.

# 11. Observability and reliability from day one

Instrument the initial application with OpenTelemetry so traces, metrics, and logs share context. OpenTelemetry provides vendor-neutral APIs and SDKs for these signals. ([OpenTelemetry][11])

Record at least:

```text
request ID
trace ID
authenticated actor ID, when safe
operation
module
resource ID
latency
result category
error code
deployment version
```

Never log passwords, tokens, raw session cookies, or unnecessary personal content.

Core metrics:

```text
API traffic, errors and latency
database pool utilization
database query duration
outbox age and backlog
worker queue age and depth
media processing latency
feed projection lag
feed hydration rejection count
notification delivery lag
rate-limit rejections
cache hit ratio, once caching exists
```

Queue age is often more meaningful than queue depth: ten old jobs can indicate a worse condition than a thousand rapidly draining jobs.

Define user-facing SLOs before large-scale optimization. Google’s SRE guidance recommends starting from behavior users care about rather than from whichever metrics are easiest to collect. ([Google SRE][12])

Example initial SLOs, to be adjusted using product requirements:

```text
99.9% of valid post-creation requests complete successfully.
99% of feed requests complete within the agreed latency target.
99% of published posts appear in ordinary followers' feeds
within an agreed freshness target.
99% of ready image assets become available within the media target.
```

# 12. Security and abuse prevention

Security and abuse controls are part of the social-media domain, not something to add after launch.

Perform a threat-modeling exercise during design and update it as the system evolves. OWASP recommends threat modeling early in the development lifecycle and maintaining it as the application changes. ([OWASP Cheat Sheet Series][13])

Day-one controls should cover:

| Area                   | Minimum control                                                       |
| ---------------------- | --------------------------------------------------------------------- |
| Object authorization   | Check viewer access for every post, profile, comment and media object |
| Function authorization | Separate user, moderator and administrator operations                 |
| Registration           | Email or identity verification, signup rate limits                    |
| Login                  | Rate limits, secure recovery flow, session revocation                 |
| Passwords              | Managed identity provider or modern password hashing                  |
| Posting                | Per-account quotas and burst limits                                   |
| Following              | Rate limits and anti-enumeration controls                             |
| Comments/reactions     | Actor and target validation, quotas                                   |
| Media                  | Size/type limits, scanning and processing isolation                   |
| Moderation             | Reports, blocks, mutes, audit trail                                   |
| APIs                   | Request-size limits, page-size limits and timeouts                    |
| Administration         | Strong authentication and immutable audit events                      |

OWASP’s API guidance calls out broken object authorization, broken authentication, unrestricted resource consumption, sensitive-business-flow abuse, and unsafe third-party API consumption as major risks. ([OWASP][6])

When storing passwords yourself, use an established password-hashing library with a memory-hard algorithm such as Argon2id rather than fast general-purpose hashing. ([OWASP Cheat Sheet Series][14])

# 13. Database evolution rules

## Add indexes only for demonstrated access patterns

For every important query, save:

```text
query
parameters and cardinality
EXPLAIN ANALYZE result
rows examined
index used
p50/p95/p99 duration
expected call frequency
```

Indexes speed reads but impose storage and write overhead, so they should be chosen deliberately. ([PostgreSQL][15])

## Do not partition tables on day one

Partitioning is useful when it solves a specific problem such as:

* Partition pruning for large queries
* Dropping old data efficiently
* Managing maintenance on very large tables
* Isolating hot and cold data

It is not a substitute for indexes or query design. PostgreSQL supports declarative partitioning, but the partition key and query patterns must align for it to help. ([PostgreSQL][16])

Likely future partition candidates include:

```text
notification deliveries by time
analytics events by time
feed entries by user hash
very large posts/comments tables by time or author range
```

Do not partition every table by `created_at`.

## Use PostgreSQL search first

For basic user, post, and hashtag search, begin with PostgreSQL full-text search and appropriate indexes. PostgreSQL includes parsing, ranking, highlighting, text-search indexes, and query configuration. ([PostgreSQL][17])

Move to a dedicated search engine when you demonstrably need:

* More sophisticated language analysis
* Fuzzy matching at large scale
* Complex relevance ranking
* Faceting
* Very high independent search traffic
* Independent index scaling
* Search-specific operational control

## Use expand-and-contract migrations

Never require new code and a destructive migration to become active simultaneously.

Use:

```text
1. Expand: add nullable column/table/index
2. Deploy code that writes old and new representation when necessary
3. Backfill
4. Validate
5. Switch reads
6. Stop old writes
7. Contract: remove obsolete data later
```

Large backfills must be restartable, rate-limited, observable, and safe under concurrent writes.

# 14. Phase-by-phase roadmap

## Phase 0 — Product boundaries and architecture contract

**Duration:** approximately one to two weeks.

Define:

* Core user journeys
* Privacy modes
* Follow versus friend semantics
* Post and comment lifecycle
* Block and mute semantics
* Moderation workflow
* Expected traffic envelope
* Data-retention requirements
* Consistency matrix
* SLO candidates
* Threat model
* Context map
* System-context and container diagrams
* Initial architecture decisions

Create ADRs for:

```text
ADR-001 Modular monolith first
ADR-002 Module and table ownership
ADR-003 PostgreSQL as source of truth
ADR-004 Object storage for media
ADR-005 Strong versus eventual consistency
ADR-006 Public identifiers
ADR-007 API and pagination style
ADR-008 Asynchronous outbox
ADR-009 Authentication strategy
```

C4 provides system-context, container, component, and code-level abstractions for documenting architecture at different zoom levels. ([C4 model][18])

**Move to Phase 1 when:** the team can name every initial module, its owned data, its public operations, and the dependencies it is allowed to have.

---

## Phase 1 — Functional modular-monolith MVP

Implement:

```text
Identity
Profile
Graph
Content
Media upload
Engagement
Simple chronological feed
Block and mute
Basic reports
```

Infrastructure:

```text
one application
one PostgreSQL primary
object storage
CDN
single region
structured logs
metrics and traces
automated migrations
automated backups
```

Feed:

```text
query on read
cursor pagination
batched profile/content hydration
current authorization check
```

Do not introduce:

```text
microservices
Kafka
Kubernetes
distributed databases
database sharding
multiple writable regions
dedicated search cluster
complex recommendation models
```

Tests:

```text
domain unit tests
module integration tests against PostgreSQL
HTTP API tests
authorization tests
architecture dependency tests
migration tests
property tests for state transitions
```

**Move to Phase 2 when:** the primary use cases work end to end, forbidden dependencies fail CI, backups can be restored, and the application survives a basic load test above expected beta traffic.

---

## Phase 2 — Production readiness

Add:

* Outbox and durable jobs
* Separate worker process from the same artifact
* Media validation and variants
* Notifications
* Moderation queue and actions
* Rate limits and quotas
* Idempotency
* Audit trail
* SLO dashboards
* Alerting
* Feature flags
* Rolling or canary deployment
* Expand-and-contract migrations
* Database connection limits
* Timeouts and cancellation
* Bounded worker queues
* Retry policy with jitter
* Dead-letter state
* Restore testing

Run failure exercises:

```text
kill API replica
kill worker during media processing
restart database
exhaust database pool
lose an external notification provider
duplicate an outbox event
delay feed projection
fill worker queue
submit a malicious or oversized upload
```

Google’s SRE guidance notes that overload and long queues can trigger cascading failures; services should use bounded queues, load shedding, deadlines, limited retries, exponential backoff, and retry budgets. ([Google SRE][19])

**Move to Phase 3 when:** failure behavior is understood, overload causes controlled rejection rather than process collapse, and restore and rollback procedures have been exercised.

---

## Phase 3 — Optimize measured bottlenecks

Before introducing new infrastructure:

1. Capture production query statistics.
2. Fix missing or poorly ordered indexes.
3. Remove N+1 access patterns.
4. Batch module reads.
5. Tune connection pools.
6. Reduce unnecessarily large payloads.
7. Move expensive side effects out of request paths.
8. Add CDN caching for public immutable media.
9. Add application caching only for proven hot reads.
10. Introduce a read replica only after identifying read load and acceptable staleness.

Possible additions:

```text
feed projection table
denormalized reaction/comment counters
profile summary projection
read-through or cache-aside caching
PostgreSQL full-text search
```

Cache rules:

* Cache is not authoritative.
* Every cache has an explicit key model.
* Every cache has TTL and invalidation behavior.
* The application still works when cache is unavailable.
* Authorization-sensitive values are not blindly reused.
* Do not cache errors indefinitely.
* Protect against cache stampedes.

**Move to Phase 4 when:** the application has a capacity model, handles several times expected peak load, and remaining limitations are architectural rather than simple query or implementation problems.

---

## Phase 4 — Event-driven projections and feed scale

Introduce a durable queue or stream when outbox polling and direct database projections become limiting.

Consumers may include:

```text
feed builder
search indexer
notification builder
analytics pipeline
moderation enrichment
media orchestration
```

Requirements:

* Event schema versioning
* Idempotent consumers
* Per-consumer checkpoints
* Lag metrics
* Dead-letter handling
* Replay procedure
* Online projection rebuild
* Backfill throttling
* Event ordering rules
* Data reconciliation jobs

Feed changes:

```text
hybrid fan-out
high-fan-out account path
feed backfill limits
candidate deduplication
ranking version
privacy revalidation
```

Search may become an independent projection backed by a search engine. Analytics should move away from the transactional primary database.

**Move to Phase 5 when:** one component has a clearly different scaling, availability, deployment, or ownership requirement from the rest of the application.

---

## Phase 5 — Selective service extraction

Likely first service candidates:

1. Media processing
2. Notification delivery
3. Search indexing and query
4. Feed generation
5. Recommendation scoring

These are good candidates because they commonly:

* Use asynchronous workflows
* Have different resource requirements
* Can tolerate eventual consistency
* Own derived state
* Benefit from independent scaling
* Have relatively clear failure boundaries

Avoid extracting the core Content, Graph, or Identity modules first unless a concrete requirement demands it.

Before extracting a module, require:

```text
stable public module API
no external reads of private tables
no cyclic dependencies
documented data ownership
versioned events
idempotent operations
module-specific telemetry
module-specific SLO
operational owner
data migration strategy
fallback and rollback strategy
```

Extraction sequence:

```text
1. Enforce the in-process boundary
2. Measure the module independently
3. Introduce an internal transport adapter
4. Copy data using snapshot plus change capture
5. Shadow requests or reads
6. Compare results
7. Switch traffic incrementally
8. Remove old direct access
9. Give the service independent ownership
```

Do not have the old and new systems independently write the same business state. Use a single writer and outbox/change capture.

Extract around a vertical business capability, not a technical layer such as “database service” or “controller service.” Fowler’s monolith-decomposition guidance similarly emphasizes extracting vertical capabilities and deciding what to decouple based on actual needs. ([martinfowler.com][20])

Use Rust here when it supplies an actual advantage, for example:

```text
media transcoding coordinator
high-throughput feed scorer
specialized recommendation runtime
real-time WebSocket gateway
content-processing sandbox
```

---

## Phase 6 — Very large scale

Only at this stage consider:

* Data sharding
* Cell-based architecture
* Regional read replicas
* Regional write ownership
* Multi-region failover
* Dedicated graph storage where justified
* Distributed feed stores
* Separate analytical lake or warehouse
* Automated capacity management
* Chaos testing
* Recommendation feature stores
* Online model serving
* Globally distributed media processing

A likely sharding dimension is user or author identity, but do not select it without analyzing:

```text
feed reads
post writes
follower lookups
celebrity accounts
cross-user transactions
moderation queries
data locality
rebalancing requirements
```

Active-active multi-region writes require explicit conflict semantics. “Last write wins” is not a universal solution for:

* Follows
* Blocks
* Reactions
* Usernames
* Moderation actions
* Post edits

Use single-region writes for as long as they satisfy product requirements.

# 15. Initial pitfalls to avoid

| Pitfall                             | Why it causes trouble                                    | Better initial choice                     |
| ----------------------------------- | -------------------------------------------------------- | ----------------------------------------- |
| Microservices from day one          | Unstable boundaries become expensive network contracts   | Modular monolith                          |
| Global technical layers             | Every feature spans the codebase                         | Package by business component             |
| Shared ORM/domain models            | Modules become coupled through representation            | Public DTOs and stable IDs                |
| Direct cross-module SQL             | Data cannot later be separated safely                    | Module APIs and read projections          |
| Generic `common` library            | Creates hidden coupling and unclear ownership            | Small technical platform packages         |
| Giant User aggregate                | Unbounded data and transaction contention                | Small independent aggregates              |
| Generic CRUD services               | Business invariants leak into controllers                | Explicit commands and use cases           |
| Dual database-and-broker writes     | Lost or phantom events                                   | Transactional outbox                      |
| Synchronous feed fan-out            | Post latency depends on follower count                   | Asynchronous projection                   |
| Exact synchronous counters          | Hot-row contention                                       | Authoritative edges plus projected counts |
| Offset pagination                   | Slow and unstable under concurrent inserts               | Cursor pagination                         |
| Media stored in PostgreSQL          | Database and backup growth, bandwidth pressure           | Object storage and CDN                    |
| Media proxied through API           | API bandwidth and memory bottleneck                      | Direct controlled upload                  |
| Authorization only at routing layer | Internal paths and stale projections can bypass checks   | Domain/application authorization          |
| Feed as source of truth             | Stale data can leak deleted or private content           | Hydrate through authoritative Content     |
| Redis added automatically           | Cache invalidation and new failure modes without benefit | Add after profiling                       |
| Kafka added automatically           | Operational complexity without replay/throughput need    | PostgreSQL outbox first                   |
| Search cluster added automatically  | Another datastore and consistency pipeline               | PostgreSQL search first                   |
| Unbounded goroutines/tasks          | Resource exhaustion under load                           | Semaphores and bounded queues             |
| Unlimited retries                   | Retry storms amplify outages                             | Budgets, backoff, jitter and deadlines    |
| Soft delete everywhere              | Bloat and unclear lifecycle semantics                    | Explicit state machines                   |
| No abuse controls                   | Spam and automation become architectural incidents       | Quotas, moderation and audit from MVP     |
| Analytics on primary DB             | Product queries compete with serving traffic             | Event pipeline or replica later           |
| Premature sharding                  | Permanent routing and transaction complexity             | Scale one relational primary first        |
| Premature active-active             | Conflict and correctness complexity                      | Single write region first                 |
| GraphQL without limits              | Expensive nested queries and authorization complexity    | REST initially, or strict query budgets   |

# 16. Microservice decision gate

A module should become a service only when at least one strong reason exists:

| Reason                    | Evidence                                                    |
| ------------------------- | ----------------------------------------------------------- |
| Independent scaling       | Its resource curve differs substantially from the rest      |
| Independent deployment    | Releases are blocked by application-wide coordination       |
| Failure isolation         | Its failures consume too much of the product’s error budget |
| Team ownership            | A stable team can own development and operations end to end |
| Security isolation        | It needs a materially different trust or data boundary      |
| Technology specialization | A different runtime provides measurable value               |
| Availability requirements | It needs a different SLO or deployment topology             |

These are not valid reasons:

```text
the codebase is messy
the module has many files
microservices are considered modern
another company uses them
Kubernetes is already available
we may need scale someday
```

A messy monolith should first become a clean modular monolith. Otherwise, service extraction produces a distributed mess.

# 17. Practical first 12 weeks

| Weeks | Deliverable                                                 |
| ----- | ----------------------------------------------------------- |
| 1–2   | Context map, C4 diagrams, threat model, ADRs, module APIs   |
| 3–4   | Identity, Profile and Graph modules                         |
| 5–6   | Content and Media modules, direct object-storage uploads    |
| 7–8   | Engagement and chronological feed                           |
| 9     | Blocks, mutes, reports and moderation workflow              |
| 10    | Outbox, worker, notifications and media processing          |
| 11    | Observability, SLOs, rate limits, idempotency and audit     |
| 12    | Load tests, restore test, failure game day and beta release |

At the end of week 12, the architecture should still be relatively simple:

```text
one codebase
one primary application release
one PostgreSQL database
one object store
one CDN
one clear set of module boundaries
```

That is not an architecture that has failed to scale. It is an architecture that has preserved the ability to evolve.

The most important day-one outcome is not choosing the final database or service topology. It is ensuring that **every business capability has a clear interface, owner, data boundary, consistency model, and observable behavior**. Once those exist, moving a component out of process becomes an engineering decision rather than a rewrite.

[1]: https://martinfowler.com/bliki/MonolithFirst.html?utm_source=chatgpt.com "Monolith First - Martin Fowler"
[2]: https://c4model.com/abstractions/component?utm_source=chatgpt.com "Component | C4 model"
[3]: https://www.martinfowler.com/bliki/BoundedContext.html?utm_source=chatgpt.com "Bounded Context"
[4]: https://go.dev/doc/modules/layout?utm_source=chatgpt.com "Organizing a Go module - The Go Programming Language"
[5]: https://www.rfc-editor.org/info/rfc9562/?utm_source=chatgpt.com "RFC 9562: Universally Unique IDentifiers (UUIDs) | RFC Editor"
[6]: https://owasp.org/API-Security/editions/2023/en/0x11-t10/?utm_source=chatgpt.com "OWASP Top 10 API Security Risks – 2023 - OWASP API Security Top 10"
[7]: https://www.postgresql.org/docs/current/indexes-partial.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 11.8. Partial Indexes"
[8]: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html?utm_source=chatgpt.com "File Upload - OWASP Cheat Sheet Series"
[9]: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html?utm_source=chatgpt.com "Server Side Request Forgery Prevention - OWASP Cheat Sheet Series"
[10]: https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html?utm_source=chatgpt.com "Outbox Event Router :: Debezium Documentation"
[11]: https://opentelemetry.io/?utm_source=chatgpt.com "OpenTelemetry"
[12]: https://sre.google/sre-book/service-level-objectives/?utm_source=chatgpt.com "Google SRE - Defining slo: service level objective meaning"
[13]: https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html?utm_source=chatgpt.com "Threat Modeling - OWASP Cheat Sheet Series"
[14]: https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html?utm_source=chatgpt.com "Password Storage - OWASP Cheat Sheet Series"
[15]: https://www.postgresql.org/docs/current/indexes.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 11. Indexes"
[16]: https://www.postgresql.org/docs/current/ddl-partitioning.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: 5.12. Table Partitioning"
[17]: https://www.postgresql.org/docs/current/textsearch.html?utm_source=chatgpt.com "PostgreSQL: Documentation: 18: Chapter 12. Full Text Search"
[18]: https://c4model.com/?utm_source=chatgpt.com "Home | C4 model"
[19]: https://sre.google/sre-book/addressing-cascading-failures/?utm_source=chatgpt.com "Google SRE - Cascading Failures: Reducing System Outage"
[20]: https://martinfowler.com/articles/break-monolith-into-microservices.html?utm_source=chatgpt.com "How to break a Monolith into Microservices - Martin Fowler"
