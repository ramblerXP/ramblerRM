# Backend Server Database Design

## 1. Purpose

This document defines a scalable backend for indexing places, historical sites, and related entities from heterogeneous sources such as UNESCO, ASI, government bodies, crawlers, and direct user contributions.

The design uses:

- **Hexagonal architecture (Ports and Adapters)** to isolate domain rules from databases, APIs, brokers, translation vendors, and search engines.
- **Expand-contract migrations** to evolve schemas, adapters, APIs, and indexes without a disruptive cutover.
- **Immutable source provenance** and versioned canonical entity revisions.
- **Asynchronous, idempotent pipelines** for ingestion, translation, review, and indexing.

> Mermaid is used for portability. Flowcharts express UML concepts where Mermaid has no dedicated UML syntax.

## 2. Core Design Rules

1. Preserve every source representation independently.
2. Link multiple source records to one universal `Entity`.
3. Publish immutable canonical revisions derived from source evidence and accepted contributions.
4. Bind every translation to an exact source revision and content hash.
5. Treat search indexes and caches as rebuildable projections.
6. Make ingestion commands and asynchronous consumers idempotent.
7. Keep the application core dependent only on ports, never infrastructure implementations.
8. Apply schema changes through expand, migrate, and contract deployments.

# 3. UML Diagrams

## 3.1 Use Case Diagram

```mermaid
flowchart LR
    classDef actor fill:#fff,stroke:#222;
    classDef usecase fill:#f7f7f7,stroke:#333;

    Visitor["Actor: Visitor"]:::actor
    Contributor["Actor: Contributor"]:::actor
    Reviewer["Actor: Reviewer / Moderator"]:::actor
    Operator["Actor: Data Operator"]:::actor
    Admin["Actor: Administrator"]:::actor
    Source["Actor: External Data Source"]:::actor
    Translator["Actor: Translation Provider"]:::actor

    subgraph Platform["Entity Knowledge Platform"]
        Browse([Browse entity]):::usecase
        Search([Search and filter]):::usecase
        SelectView([Select canonical or source view]):::usecase
        RequestLanguage([Request language]):::usecase
        Fallback([Apply language fallback]):::usecase

        Edit([Propose entity edit]):::usecase
        Comment([Add comment]):::usecase
        Vote([Vote]):::usecase
        Track([Track contribution]):::usecase

        ReviewContribution([Review contribution]):::usecase
        ReviewTranslation([Review translation]):::usecase
        ResolveConflict([Resolve source conflict]):::usecase
        Publish([Publish canonical revision]):::usecase

        Sync([Synchronize source]):::usecase
        MapSchema([Configure schema mapping]):::usecase
        Match([Match or merge entity]):::usecase
        Replay([Replay failed import]):::usecase

        Translate([Generate translation]):::usecase
        Reindex([Reindex entity]):::usecase
        Migrate([Run expand-contract migration]):::usecase
        Audit([Audit provenance]):::usecase
    end

    Visitor --- Browse
    Visitor --- Search
    Visitor --- SelectView
    Visitor --- RequestLanguage

    Contributor --- Edit
    Contributor --- Comment
    Contributor --- Vote
    Contributor --- Track

    Reviewer --- ReviewContribution
    Reviewer --- ReviewTranslation
    Reviewer --- ResolveConflict
    Reviewer --- Publish

    Operator --- Sync
    Operator --- MapSchema
    Operator --- Replay
    Operator --- Audit

    Admin --- Migrate
    Admin --- Reindex
    Admin --- Audit

    Source --- Sync
    Translator --- Translate

    Browse -.->|include| SelectView
    RequestLanguage -.->|include| Fallback
    Sync -.->|include| Match
    Publish -.->|include| Reindex
    Publish -.->|extend| Translate
    ReviewContribution -.->|extend| Publish
    ReviewTranslation -.->|extend| Publish
```

## 3.2 Class Diagram

```mermaid
classDiagram
    direction LR

    class Entity {
        +UUID id
        +EntityType type
        +EntityStatus status
        +UUID canonicalRevisionId
        +Instant createdAt
    }

    class EntityRevision {
        +UUID id
        +UUID entityId
        +long revisionNumber
        +LanguageCode defaultLanguage
        +RevisionStatus status
        +JSON structuredData
        +string contentHash
        +Instant publishedAt
        +publish()
        +supersede()
    }

    class DataSource {
        +UUID id
        +string code
        +string name
        +SourceType type
        +TrustLevel trustLevel
        +boolean enabled
    }

    class SourceSchemaMapping {
        +UUID id
        +UUID dataSourceId
        +string sourceSchemaVersion
        +string canonicalSchemaVersion
        +JSON mappingRules
        +Instant effectiveFrom
    }

    class SourceRecord {
        +UUID id
        +UUID dataSourceId
        +string externalId
        +string sourceVersion
        +LanguageCode language
        +JSON normalizedPayload
        +string rawPayloadUri
        +string payloadHash
        +Instant observedAt
    }

    class EntitySourceLink {
        +UUID entityId
        +UUID sourceRecordId
        +MatchType matchType
        +decimal confidence
        +LinkStatus status
    }

    class RevisionEvidence {
        +UUID revisionId
        +UUID sourceRecordId
        +UUID contributionId
        +EvidenceRole role
    }

    class TranslationRevision {
        +UUID id
        +UUID entityRevisionId
        +LanguageCode language
        +TranslationStatus status
        +JSON translatedData
        +string sourceContentHash
        +string provider
        +UUID reviewedBy
        +publish()
        +markStale()
    }

    class Contribution {
        +UUID id
        +UUID entityId
        +UUID authorId
        +ContributionType type
        +LanguageCode language
        +JSON proposedPatch
        +ContributionStatus status
        +submit()
        +accept()
        +reject()
    }

    class Comment {
        +UUID id
        +UUID contributionId
        +UUID authorId
        +string body
        +CommentStatus status
    }

    class Vote {
        +UUID id
        +UUID voterId
        +UUID targetId
        +VoteTargetType targetType
        +short value
    }

    class Review {
        +UUID id
        +UUID reviewerId
        +UUID targetId
        +ReviewTargetType targetType
        +ReviewDecision decision
        +string reason
    }

    class User {
        +UUID id
        +string displayName
        +UserRole role
        +ReputationLevel reputation
    }

    class SearchDocument {
        +UUID entityId
        +UUID entityRevisionId
        +LanguageCode language
        +string sourceView
        +JSON searchableFields
        +long projectionVersion
    }

    class ImportJob {
        +UUID id
        +UUID dataSourceId
        +string cursor
        +JobStatus status
        +int attempt
    }

    class TranslationJob {
        +UUID id
        +UUID entityRevisionId
        +LanguageCode targetLanguage
        +string sourceContentHash
        +JobStatus status
    }

    class OutboxEvent {
        +UUID id
        +string aggregateType
        +UUID aggregateId
        +string eventType
        +JSON payload
        +Instant publishedAt
    }

    Entity "1" *-- "1..*" EntityRevision : revisions
    Entity "1" o-- "0..*" EntitySourceLink : source links
    DataSource "1" *-- "0..*" SourceSchemaMapping : mappings
    DataSource "1" *-- "0..*" SourceRecord : records
    SourceRecord "1" -- "0..*" EntitySourceLink : matched by
    EntityRevision "1" *-- "0..*" TranslationRevision : translations
    EntityRevision "1" o-- "1..*" RevisionEvidence : evidence
    SourceRecord "0..1" -- "0..*" RevisionEvidence : supports
    Contribution "0..1" -- "0..*" RevisionEvidence : supports
    Entity "1" o-- "0..*" Contribution : receives
    User "1" -- "0..*" Contribution : authors
    Contribution "1" *-- "0..*" Comment : discussion
    User "1" -- "0..*" Vote : casts
    Contribution "1" o-- "0..*" Review : reviewed by
    TranslationRevision "1" o-- "0..*" Review : reviewed by
    User "1" -- "0..*" Review : performs
    EntityRevision "1" -- "0..*" SearchDocument : projected as
    DataSource "1" -- "0..*" ImportJob : synchronized by
    EntityRevision "1" -- "0..*" TranslationJob : translated by
    Entity "1" -- "0..*" OutboxEvent : emits
```

### Persistence constraints

- Unique source version: `(data_source_id, external_id, source_version)`.
- Unique canonical revision: `(entity_id, revision_number)`.
- One vote per `(voter_id, target_type, target_id)`.
- Translation identity includes `(entity_revision_id, language, source_content_hash)`.
- Use optimistic concurrency when changing `canonicalRevisionId`.
- Keep raw source payloads in object storage with an immutable URI and checksum.


## 3.3 Sequence Diagram

The sequence covers an upstream update, canonical publication, translation, indexing, and a subsequent user read.

```mermaid
sequenceDiagram
    autonumber

    participant DS as External Data Source
    participant SCH as Sync Scheduler
    participant ADP as Source Adapter
    participant ING as Ingestion Service
    participant MATCH as Matching Service
    participant DB as Transactional Database
    participant BUS as Event Bus
    participant TRW as Translation Worker
    participant TP as Translation Provider
    participant REV as Review Service
    participant IDX as Indexer Worker
    participant SEARCH as Search Cluster
    participant API as Query API
    participant CACHE as Cache
    actor USER as End User

    SCH->>ADP: Pull changes using cursor or ETag
    ADP->>DS: Fetch source records
    DS-->>ADP: Raw records and source version
    ADP->>ADP: Validate, normalize, and hash
    ADP->>ING: UpsertSourceRecord(command, idempotencyKey)

    ING->>DB: Check source ID, version, and hash
    alt Duplicate payload
        DB-->>ING: Existing source record
        ING-->>ADP: Idempotent no-op
    else New or changed payload
        ING->>DB: Begin transaction
        ING->>DB: Append source record
        ING->>MATCH: Find candidate entities
        MATCH-->>ING: Candidates and confidence

        alt High-confidence match
            ING->>DB: Link record to existing entity
        else Ambiguous match
            ING->>DB: Create match review case
        else No match
            ING->>DB: Create entity shell and link
        end

        ING->>DB: Create draft canonical revision
        ING->>DB: Insert outbox event
        ING->>DB: Commit
        ING-->>ADP: Accepted
    end

    DB-->>BUS: Outbox relay publishes revision proposal
    REV->>DB: Load proposal and provenance
    REV->>DB: Record approval and publish revision
    REV->>DB: Insert publication event
    DB-->>BUS: Outbox relay publishes EntityRevisionPublished

    par Translation
        BUS-->>TRW: EntityRevisionPublished
        TRW->>DB: Check language and source hash
        alt Missing or stale translation
            TRW->>TP: Translate structured blocks
            TP-->>TRW: Translation draft
            TRW->>DB: Store translation revision
            TRW->>DB: Create review task or auto-publish
        else Current translation exists
            TRW->>TRW: Idempotent no-op
        end
    and Search projection
        BUS-->>IDX: EntityRevisionPublished
        IDX->>DB: Read canonical and source views
        IDX->>SEARCH: Upsert versioned documents
        SEARCH-->>IDX: Acknowledged
    and Cache invalidation
        BUS-->>CACHE: Invalidate entity keys
    end

    USER->>API: GET entity with language and source preference
    API->>CACHE: Lookup versioned response
    alt Cache hit
        CACHE-->>API: Cached response
    else Cache miss
        API->>DB: Load selected view and approved translation
        alt Approved translation exists
            DB-->>API: Localized entity
        else Translation unavailable or stale
            DB-->>API: Default language and fallback metadata
        end
        API->>CACHE: Store response
    end
    API-->>USER: Entity, provenance, and language status
```

### Sequence guarantees

- Derive ingestion idempotency from source, external ID, source version, and payload hash.
- Commit publication state and its outbox event in one transaction.
- Deduplicate consumers by event ID and monotonic projection version.
- Search and cache are eventually consistent; the transactional database is authoritative.
- Responses identify approved, machine-generated, stale, and fallback translations.

## 3.4 Statechart Diagram

This models the lifecycle of a canonical entity revision and localized publication.

```mermaid
stateDiagram-v2
    [*] --> Received

    Received --> Normalizing
    Normalizing --> Rejected : invalid or unsupported
    Normalizing --> Matching : valid

    state MatchDecision <<choice>>
    Matching --> MatchDecision
    MatchDecision --> Linked : confident match
    MatchDecision --> AwaitingMatchReview : ambiguous
    MatchDecision --> Linked : create new entity
    AwaitingMatchReview --> Linked : resolved
    AwaitingMatchReview --> Rejected : invalid record

    Linked --> DraftCanonical
    DraftCanonical --> AwaitingContentReview
    AwaitingContentReview --> DraftCanonical : changes requested
    AwaitingContentReview --> Rejected : rejected
    AwaitingContentReview --> Published : approved

    state Published {
        [*] --> TranslationEvaluation
        state TranslationDecision <<choice>>
        TranslationEvaluation --> TranslationDecision
        TranslationDecision --> TranslationQueued : language missing
        TranslationDecision --> SearchIndexQueued : no translation required

        TranslationQueued --> MachineTranslated
        MachineTranslated --> AwaitingTranslationReview : review required
        MachineTranslated --> TranslationPublished : trusted policy
        AwaitingTranslationReview --> TranslationQueued : revision requested
        AwaitingTranslationReview --> TranslationPublished : approved
        AwaitingTranslationReview --> TranslationRejected : rejected
        TranslationPublished --> SearchIndexQueued
        TranslationRejected --> SearchIndexQueued

        SearchIndexQueued --> Searchable
        Searchable --> [*]
    }

    Published --> Superseded : newer revision published
    Superseded --> Archived : retention policy
    Published --> Withdrawn : moderation or legal action
    Withdrawn --> Published : reinstated
    Published --> TranslationStale : source content hash changes
    TranslationStale --> TranslationQueued : retranslate
    TranslationStale --> Superseded : newer revision selected

    Rejected --> [*]
    Archived --> [*]
```

### State rules

- Published revisions are immutable; corrections create a new revision.
- A translation can be published only when its source hash matches the relevant revision.
- Withdrawn revisions remain auditable but are excluded from normal reads and indexes.
- Publishing a newer revision supersedes the previous one and invalidates dependent projections.

## 3.5 Activity Diagram

```mermaid
flowchart TD
    Start([Start]) --> Trigger{Trigger type?}

    subgraph Inputs["External Source or Contributor"]
        SourceChange[Source exposes changed record]
        UserAction[User submits edit, comment, or vote]
    end

    subgraph Adapters["Inbound Adapters"]
        LoadMapping[Load versioned source mapping]
        Parse[Parse and validate]
        Normalize[Normalize universal command]
        Authenticate[Authenticate and authorize]
        ValidateContribution[Validate contribution]
    end

    subgraph Core["Application and Domain Core"]
        Deduplicate{Already processed?}
        ResolveEntity[Resolve or create universal entity]
        PersistSource[Append source record and provenance]
        CreateProposal[Create canonical proposal]
        CreateContribution[Create contribution and review case]
        NeedsReview{Review required?}
        Review[Evaluate evidence and policy]
        Approved{Approved?}
        Publish[Publish immutable canonical revision]
        Reject[Reject or request changes]
        Emit[Write events to outbox]
    end

    subgraph Pipelines["Asynchronous Pipelines"]
        Languages[Determine required languages]
        Translate[Translate missing or stale blocks]
        TranslationReview{Translation review required?}
        ReviewTranslation[Review translation]
        BuildProjection[Build localized search documents]
        Invalidate[Invalidate cache keys]
    end

    subgraph ReadPath["Serving"]
        Searchable[Entity becomes searchable]
        Serve[Serve canonical or selected source]
        HasLanguage{Approved requested language?}
        Fallback[Apply explicit fallback]
    end

    Trigger -->|Source update| SourceChange
    Trigger -->|User action| UserAction

    SourceChange --> LoadMapping --> Parse
    Parse -->|Invalid| Reject
    Parse -->|Valid| Normalize --> Deduplicate

    UserAction --> Authenticate
    Authenticate -->|Unauthorized| Reject
    Authenticate -->|Authorized| ValidateContribution
    ValidateContribution -->|Invalid| Reject
    ValidateContribution -->|Valid| CreateContribution --> NeedsReview

    Deduplicate -->|Yes| NoOp([Stop: idempotent no-op])
    Deduplicate -->|No| ResolveEntity --> PersistSource --> CreateProposal --> NeedsReview

    NeedsReview -->|Yes| Review --> Approved
    NeedsReview -->|No, trusted policy| Publish
    Approved -->|No| Reject
    Approved -->|Yes| Publish

    Reject --> Notify([Record decision and notify actor])
    Publish --> Emit
    Emit --> Languages --> Translate --> TranslationReview
    TranslationReview -->|Yes| ReviewTranslation
    ReviewTranslation -->|Rejected| Translate
    ReviewTranslation -->|Approved| BuildProjection
    TranslationReview -->|No| BuildProjection
    Emit --> BuildProjection
    Emit --> Invalidate

    BuildProjection --> Searchable --> Serve --> HasLanguage
    HasLanguage -->|Yes| Complete([Return localized response])
    HasLanguage -->|No| Fallback --> Complete
```


## 3.6 Component Diagram

```mermaid
flowchart LR
    classDef adapter fill:#eef6ff,stroke:#276fbf;
    classDef port fill:#fff7e6,stroke:#b36b00;
    classDef core fill:#eefbea,stroke:#2c7a2c;
    classDef infra fill:#f7eefc,stroke:#7c3aa5;
    classDef external fill:#fff,stroke:#555,stroke-dasharray:4 3;

    subgraph External["External actors and systems"]
        Clients["Web, Mobile, and Partner Clients"]:::external
        Sources["UNESCO, ASI, Government APIs"]:::external
        Crawlers["Crawler Fleet"]:::external
        Providers["Translation Providers"]:::external
        Moderators["Reviewer Console"]:::external
    end

    subgraph PrimaryAdapters["Primary / Driving Adapters"]
        API["REST / GraphQL Adapter"]:::adapter
        AdminAPI["Admin and Review Adapter"]:::adapter
        SourceAdapter["Versioned Source Connectors"]:::adapter
        CrawlerAdapter["Crawler Ingestion Adapter"]:::adapter
        Scheduler["Synchronization Scheduler"]:::adapter
    end

    subgraph ApplicationCore["Hexagonal Application Core"]
        QueryPort(("Entity Query Port")):::port
        IngestPort(("Ingestion Port")):::port
        ContributionPort(("Contribution Port")):::port
        ReviewPort(("Review Port")):::port
        MigrationPort(("Migration Port")):::port

        EntityApp["Entity Application Service"]:::core
        IngestionApp["Ingestion and Matching Service"]:::core
        ContributionApp["Contribution Service"]:::core
        ReviewApp["Review and Publication Service"]:::core
        TranslationApp["Translation Orchestrator"]:::core
        IndexApp["Index Projection Service"]:::core
        MigrationApp["Expand-Contract Coordinator"]:::core
        Domain["Domain Model\nEntities, revisions, provenance,\ntranslations, contributions, policies"]:::core

        QueryPort --> EntityApp
        IngestPort --> IngestionApp
        ContributionPort --> ContributionApp
        ReviewPort --> ReviewApp
        MigrationPort --> MigrationApp

        EntityApp --> Domain
        IngestionApp --> Domain
        ContributionApp --> Domain
        ReviewApp --> Domain
        TranslationApp --> Domain
        IndexApp --> Domain
        MigrationApp --> Domain
    end

    subgraph SecondaryPorts["Secondary / Driven Ports"]
        RepoPort(("Repository Port")):::port
        RawPort(("Raw Payload Port")):::port
        EventPort(("Event Publisher Port")):::port
        SearchPort(("Search Port")):::port
        CachePort(("Cache Port")):::port
        TranslatePort(("Translation Port")):::port
        IdentityPort(("Identity and Policy Port")):::port
        TelemetryPort(("Telemetry Port")):::port
    end

    subgraph SecondaryAdapters["Secondary / Driven Adapters"]
        PostgresAdapter["PostgreSQL Adapter"]:::adapter
        ObjectAdapter["Object Storage Adapter"]:::adapter
        BrokerAdapter["Message Broker Adapter"]:::adapter
        SearchAdapter["Search Engine Adapter"]:::adapter
        RedisAdapter["Redis Adapter"]:::adapter
        TranslationAdapter["Translation Vendor Adapter"]:::adapter
        AuthAdapter["OIDC / RBAC Adapter"]:::adapter
        TelemetryAdapter["Logs, Metrics, Traces Adapter"]:::adapter
    end

    subgraph Infrastructure["Infrastructure"]
        DB[("Transactional Database")]:::infra
        Blob[("Raw Payload and Media Storage")]:::infra
        Bus[("Event Bus and DLQ")]:::infra
        Search[("Search Cluster")]:::infra
        Cache[("Distributed Cache")]:::infra
        IAM["Identity Provider"]:::infra
        Obs["Observability Platform"]:::infra
    end

    Clients --> API
    Moderators --> AdminAPI
    Sources --> SourceAdapter
    Crawlers --> CrawlerAdapter
    Scheduler --> SourceAdapter

    API --> QueryPort
    API --> ContributionPort
    AdminAPI --> ReviewPort
    AdminAPI --> MigrationPort
    SourceAdapter --> IngestPort
    CrawlerAdapter --> IngestPort

    Domain --> RepoPort
    IngestionApp --> RawPort
    EntityApp --> EventPort
    IngestionApp --> EventPort
    ReviewApp --> EventPort
    TranslationApp --> TranslatePort
    TranslationApp --> EventPort
    IndexApp --> SearchPort
    EntityApp --> CachePort
    ContributionApp --> IdentityPort
    ReviewApp --> IdentityPort
    Domain --> TelemetryPort

    RepoPort --> PostgresAdapter --> DB
    RawPort --> ObjectAdapter --> Blob
    EventPort --> BrokerAdapter --> Bus
    SearchPort --> SearchAdapter --> Search
    CachePort --> RedisAdapter --> Cache
    TranslatePort --> TranslationAdapter --> Providers
    IdentityPort --> AuthAdapter --> IAM
    TelemetryPort --> TelemetryAdapter --> Obs

    Bus -. entity events .-> TranslationApp
    Bus -. entity events .-> IndexApp
    Bus -. cache invalidation .-> RedisAdapter
```

### Component responsibilities

| Component | Responsibility |
|---|---|
| Source connectors | Convert source-specific contracts into versioned universal ingestion commands |
| Ingestion and matching | Deduplicate, version, match, preserve provenance, and create proposals |
| Entity service | Compose canonical/source views and language fallback |
| Contribution service | Accept edits, comments, and votes and initiate workflows |
| Review service | Resolve conflicts, record decisions, and publish revisions |
| Translation orchestrator | Track demand, translate blocks, detect staleness, and route review |
| Index projection service | Build source- and language-specific search documents |
| Migration coordinator | Control expand, backfill, read switch, validation, and contract |
| Repository adapter | Persist authoritative transactional state |
| Broker adapter | Decouple publication from translation, indexing, and invalidation |

**Dependency rule:** dependencies point inward. The domain and application components depend on port interfaces; infrastructure adapters implement those ports.

## 3.7 Deployment Diagram

```mermaid
flowchart TB
    classDef device fill:#fff,stroke:#333;
    classDef runtime fill:#eef6ff,stroke:#276fbf;
    classDef datastore fill:#f7eefc,stroke:#7c3aa5;
    classDef external fill:#fff7e6,stroke:#b36b00;

    Users["Device: Web and Mobile Clients"]:::device
    Partners["Device: Partner and Admin Clients"]:::device
    Sources["External Nodes: Data Sources and Crawlers"]:::external
    TranslationProviders["External Translation APIs"]:::external

    subgraph Edge["Global Edge"]
        DNS["Managed DNS"]:::runtime
        CDN["CDN and Static Asset Cache"]:::runtime
        WAF["WAF, DDoS Protection, Rate Limiting"]:::runtime
        LB["Global / Regional Load Balancer"]:::runtime
    end

    subgraph PrimaryRegion["Primary Cloud Region"]
        subgraph Cluster["Multi-AZ Container or Kubernetes Cluster"]
            Gateway["API Gateway / Ingress\nN replicas"]:::runtime

            subgraph AppPool["Stateless Application Pool"]
                QueryAPI["Query API\nHPA replicas"]:::runtime
                CommandAPI["Contribution and Admin API\nHPA replicas"]:::runtime
                IngestionSvc["Ingestion Service\nHPA replicas"]:::runtime
                ReviewSvc["Review Service\nHPA replicas"]:::runtime
            end

            subgraph WorkerPool["Asynchronous Worker Pool"]
                ConnectorWorkers["Source Connector Workers"]:::runtime
                TranslationWorkers["Translation Workers"]:::runtime
                IndexWorkers["Indexer Workers"]:::runtime
                OutboxRelay["Outbox Relay"]:::runtime
                BackfillWorkers["Migration and Backfill Workers"]:::runtime
            end

            ServiceMesh["Service Discovery, mTLS, Retry Policies"]:::runtime
            TelemetryAgent["Metrics, Logs, Trace Collectors"]:::runtime
        end

        subgraph DataPlane["Regional Data Plane"]
            PgPrimary[("PostgreSQL Primary\nMulti-AZ")]:::datastore
            PgReplica[("PostgreSQL Read Replicas")]:::datastore
            PgPool["Connection Poolers"]:::runtime
            Redis[("Redis Cluster")]:::datastore
            Broker[("Partitioned Event Broker")]:::datastore
            DLQ[("Dead-Letter Topics")]:::datastore
            SearchCluster[("Search Cluster\nMulti-node / Multi-AZ")]:::datastore
            ObjectStore[("Versioned Object Storage")]:::datastore
        end

        subgraph Operations["Operations Plane"]
            IAM["OIDC Identity and Policy"]:::runtime
            Secrets["Secrets and Key Management"]:::runtime
            Observability["Logs, Metrics, Traces, Alerts"]:::runtime
            Config["Feature Flags and Dynamic Configuration"]:::runtime
        end
    end

    subgraph RecoveryRegion["Secondary / Disaster Recovery Region"]
        WarmApp["Warm API and Worker Capacity"]:::runtime
        PgDR[("Cross-region Database Replica")]:::datastore
        SearchDR[("Rebuildable Search Capacity")]:::datastore
        ObjectReplica[("Replicated Object Storage")]:::datastore
    end

    Users --> DNS --> CDN --> WAF --> LB
    Partners --> DNS
    LB --> Gateway
    Gateway --> QueryAPI
    Gateway --> CommandAPI

    Sources --> ConnectorWorkers --> IngestionSvc
    TranslationWorkers --> TranslationProviders

    QueryAPI --> PgPool
    CommandAPI --> PgPool
    IngestionSvc --> PgPool
    ReviewSvc --> PgPool
    OutboxRelay --> PgPool
    BackfillWorkers --> PgPool
    PgPool --> PgPrimary
    QueryAPI --> PgReplica
    PgPrimary --> PgReplica

    QueryAPI --> Redis
    CommandAPI --> Redis

    PgPrimary --> OutboxRelay --> Broker
    Broker --> TranslationWorkers
    Broker --> IndexWorkers
    Broker --> DLQ

    ConnectorWorkers --> ObjectStore
    IngestionSvc --> ObjectStore
    IndexWorkers --> SearchCluster
    QueryAPI --> SearchCluster

    Gateway --> IAM
    CommandAPI --> IAM
    ReviewSvc --> IAM
    AppPool --> Secrets
    WorkerPool --> Secrets
    AppPool --> Config
    WorkerPool --> Config
    AppPool --> TelemetryAgent
    WorkerPool --> TelemetryAgent
    TelemetryAgent --> Observability

    PgPrimary -. asynchronous replication .-> PgDR
    ObjectStore -. cross-region replication .-> ObjectReplica
    SearchCluster -. snapshot or event replay .-> SearchDR
    LB -. regional failover .-> WarmApp
```

### Deployment notes

- APIs are stateless and horizontally scalable.
- Query and command workloads scale and deploy independently.
- Read replicas absorb entity and provenance reads; the primary handles publication transactions.
- Partition broker events by `entity_id` to preserve per-entity order while retaining parallelism.
- Raw imports and replay artifacts live in versioned object storage.
- Search and cache are reconstructable; disaster recovery prioritizes the transactional database and immutable objects.
- Use workload identities and short-lived credentials rather than secrets embedded in containers.


# 4. Database and Storage Design

## 4.1 Logical schemas

```text
identity
  users
  roles
  user_reputation

catalog
  entities
  entity_revisions
  entity_source_links
  revision_evidence

sources
  data_sources
  source_schema_mappings
  source_records
  import_jobs
  import_failures

localization
  translation_revisions
  translation_jobs
  language_demand

community
  contributions
  comments
  votes
  reviews

operations
  outbox_events
  idempotency_keys
  migration_runs
  audit_log
```

These can initially share one PostgreSQL cluster while preserving ownership boundaries. Split them only when measured load or team ownership justifies the operational cost.

## 4.2 Important indexes

| Table | Suggested index |
|---|---|
| `source_records` | Unique `(data_source_id, external_id, source_version)` |
| `source_records` | `(data_source_id, payload_hash)` |
| `entity_source_links` | `(source_record_id, status)` and `(entity_id, status)` |
| `entity_revisions` | Unique `(entity_id, revision_number)` |
| `entity_revisions` | Partial `(entity_id, published_at DESC)` for published rows |
| `translation_revisions` | `(entity_revision_id, language, status)` |
| `contributions` | `(entity_id, status, created_at)` |
| `reviews` | `(target_type, target_id, created_at)` |
| `outbox_events` | Partial `(occurred_at)` where `published_at IS NULL` |
| `idempotency_keys` | Unique `(scope, key)` with expiration |
| `import_jobs` | `(data_source_id, status, started_at)` |

Use geospatial database indexes for exact point/polygon operations. Keep broad multilingual full-text relevance in the search engine rather than adding an unbounded set of transactional indexes.

# 5. Translation Pipeline

Represent content as stable semantic blocks:

```json
{
  "block_id": "history.summary",
  "source_language": "hi",
  "source_text": "...",
  "source_hash": "sha256:...",
  "do_not_translate": ["ASI", "Qutub Minar"],
  "format": "markdown"
}
```

Translate and review per block, then assemble a localized revision. This avoids retranslating unchanged paragraphs.

Suggested language resolution:

1. Approved translation of the selected canonical or source revision.
2. Approved configured fallback locale.
3. Selected representation's default language.
4. Canonical default language.
5. Explicit `translation_pending` metadata.

A translation is current only when:

```text
translation.source_content_hash == source_revision.content_hash
```

When the source changes, mark dependent translations stale, preserve them for audit and translation-memory reuse, and enqueue only changed blocks.

# 6. Contribution and Moderation

Contributions are proposed changes, not direct mutations of published content.

Recommended controls:

- authenticated authorship and immutable audit records;
- reputation-based review routing;
- spam and abuse screening;
- one active vote per user and target;
- optimistic locking when multiple reviewers act;
- structured patches instead of arbitrary full-document replacement;
- reason codes for acceptance, rejection, withdrawal, and rollback;
- provenance links from accepted contributions to resulting revisions.

Comments and votes influence review and ranking policy but should not automatically become canonical facts without a recorded decision.

# 7. Indexer Pipeline

A search document should contain:

- entity identity and type;
- canonical and source-specific names;
- approved localized text;
- geospatial fields;
- source identifiers and attribution;
- normalized tags and categories;
- publication and moderation status;
- source trust and freshness signals;
- canonical revision and projection version.

Use alias-based index deployment:

1. Build a new versioned index.
2. Backfill from authoritative revisions or replay events.
3. Validate counts, checksums, and representative queries.
4. Atomically switch the read alias.
5. Retain the previous index for rollback.
6. Delete it after the rollback window.

# 8. Expand-Contract Migration Strategy

Use separate deployments for each phase.

## Expand

- Add new nullable columns, tables, event fields, or API fields.
- Deploy code that understands old and new representations.
- Start dual-writing where required.
- Add metrics for both paths.
- Never remove or rename a live field in place.

## Migrate

- Backfill in resumable, idempotent batches.
- Throttle using database load and replication lag.
- Compare row counts, checksums, invariants, and business results.
- Move reads gradually behind a feature flag.
- Preserve rollback capability.

## Contract

- Stop old writes.
- Verify that no consumers use the old representation.
- Remove compatibility code.
- Remove old schema and indexes in a later deployment.
- Record migration evidence in `migration_runs`.

Example:

| Phase | Replacing one description column with localized blocks |
|---|---|
| Expand | Add content-block tables and continue writing the old column |
| Migrate | Dual-write, backfill, validate hashes, and switch reads |
| Contract | Stop old writes, remove the old reader, then drop the column |

# 9. Reliability, Scalability, Security, and Observability

## Reliability

- Transactional outbox for reliable event publication.
- Idempotency keys on ingestion and contribution commands.
- Consumer deduplication and monotonic projection versions.
- Exponential backoff, jitter, retry limits, and dead-letter queues.
- Per-source circuit breakers and rate limits.
- Replay tools for raw payloads, event ranges, and failed jobs.
- Point-in-time database recovery and versioned object storage.

## Scalability

- Stateless APIs and horizontally scalable workers.
- Independent scaling for reads, ingestion, translation, and indexing.
- Broker partitioning by entity or source.
- Revision-aware cache keys.
- Read replicas for high-volume detail queries.
- Cursor-based source synchronization.
- Batch translation and index writes where supported.

## Security

- OIDC authentication and role- or attribute-based authorization.
- Source credentials in a secrets manager.
- Authenticated service-to-service communication.
- Encryption in transit and at rest.
- Parser hardening and request-size limits.
- HTML and Markdown sanitization.
- Immutable moderation and migration audit trails.
- Licensing and attribution enforced as domain policy.

## Observability

Track:

- ingestion lag, rejection rate, and deduplication rate;
- records awaiting entity-match review;
- canonical publication latency;
- translation queue depth, cost, status, and staleness;
- index projection lag and failed documents;
- cache hit ratio;
- database latency, lock time, saturation, and replica lag;
- outbox age and broker consumer lag;
- migration progress and invariant failures;
- moderation turnaround time.

# 10. Architecture Decision Summary

| Decision | Benefit | Cost |
|---|---|---|
| Immutable source and canonical revisions | Auditability, rollback, provenance | Increased storage |
| Universal schema behind adapters | Isolates source schema churn | Mapping governance |
| Hexagonal architecture | Testable domain and replaceable infrastructure | More interfaces |
| Async translation and indexing | Independent scaling and low write latency | Eventual consistency |
| Transactional outbox | Reliable downstream publication | Relay and cleanup |
| Block-level translation | Lower cost and focused review | Content decomposition |
| Search as projection | Fast multilingual/geospatial queries | Rebuild and lag handling |
| Expand-contract | Backward-compatible evolution | Multiple releases |
| Raw payload archive | Replay after mapping changes | Retention cost |
| Canonical plus source views | Curation without hiding provenance | More complex reads |

# 11. Requirements Traceability

| Requirement | Design elements |
|---|---|
| Millions of users | CDN, cache, stateless APIs, read replicas, search cluster |
| Multiple changing sources | Versioned adapters, schema mappings, raw payload archive |
| Universal database | Canonical entities separate from source records |
| Future migrations | Ports and adapters plus expand-contract coordinator |
| Multiple entries per entity | Entity-source links, matching, and revision evidence |
| Platform-curated version | Immutable canonical revisions and review workflow |
| Selectable source | Query service composes canonical or source-specific view |
| Translation pipeline | Block hashes, jobs, provider port, review, and staleness |
| Contribution pipeline | Contributions, comments, votes, reviews, and audit |
| Frequent queries | Search documents, cache, and read replicas |
| Robust processing | Idempotency, outbox, retries, DLQ, and replay |
| Scalable deployment | Multi-AZ services, workers, broker, storage, and DR |

# 12. References

- IBM Developer, *An introduction to the Unified Modeling Language*:  
  https://developer.ibm.com/articles/an-introduction-to-uml/
- IBM Developer, *The component diagram*:  
  https://developer.ibm.com/articles/the-component-diagram/
- Sparx Systems, *Database Modeling in UML*:  
  https://sparxsystems.com/resources/tutorials/uml/datamodel.html
- Alistair Cockburn, *Hexagonal Architecture*:  
  https://alistair.cockburn.us/hexagonal-architecture
- Martin Fowler, *Parallel Change*:  
  https://martinfowler.com/bliki/ParallelChange.html
