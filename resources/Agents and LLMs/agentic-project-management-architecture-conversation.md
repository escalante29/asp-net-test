# Agentic Project Management Architecture Conversation

## User

So I have this collaborative web application based on FastAPI + MAF and CopilotKit/NextJS. It is used as an agentic project management application used by different teams.

We want to be able to receive real-time updates as the teams modify a specific project they are part of.

Thinking of leveraging PostgreSQL Listen/Notify capability and maybe SSE/WS at the browser level.

## Assistant

Yes—`PostgreSQL LISTEN/NOTIFY` plus SSE is a good fit, with one important constraint:

> Use `NOTIFY` as a low-latency wake-up signal, not as the event store.

Notifications are transient, payloads are limited to under 8 KB by default, and disconnected listeners cannot recover missed messages. PostgreSQL itself recommends storing larger data in a table and notifying listeners with the record’s key. Notifications are delivered only after the transaction commits, which is exactly what you want here. [PostgreSQL `NOTIFY` documentation](https://www.postgresql.org/docs/current/sql-notify.html)

## Recommended architecture

```text
Browser
  ├── REST mutations ───────────────► FastAPI
  └── Project SSE connection ◄────── FastAPI instance
                                          ▲
                                          │ LISTEN project_events
                                          │
Mutation transaction:                PostgreSQL
  1. update task/project             ├── domain tables
  2. insert project_event            └── project_events
  3. pg_notify(event_id)
  4. commit
```

Each FastAPI replica maintains one dedicated PostgreSQL listener connection. When it receives a notification, it loads the durable event, checks which locally connected clients are subscribed to that project, and broadcasts it.

Because every PostgreSQL listener receives every notification, this also works across multiple FastAPI replicas: each instance forwards events only to browsers connected to that instance.

## SSE versus WebSockets

I would start with SSE.

Your collaborative traffic is primarily:

- Browser → server: ordinary mutations through REST
- Server → browser: project updates
- Agent → browser: streamed progress/state

That asymmetry fits SSE very well. SSE includes automatic reconnection and `Last-Event-ID` support in the browser standard. [WHATWG Server-Sent Events](https://html.spec.whatwg.org/dev/server-sent-events.html)

Choose WebSockets if you later need:

- Presence and rapidly changing cursor/selection information
- Very frequent bidirectional messaging
- Collaborative text editing with OT or CRDT messages
- A single persistent channel for commands and events

For task/status/comment/project updates, SSE is substantially simpler operationally.

## Durable event table

Something along these lines:

```sql
CREATE TABLE project_events (
    id              bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    project_id      uuid NOT NULL,
    event_type      text NOT NULL,
    entity_type     text NOT NULL,
    entity_id       uuid,
    actor_id        uuid,
    entity_version  bigint,
    payload         jsonb NOT NULL DEFAULT '{}',
    created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX project_events_project_cursor_idx
    ON project_events (project_id, id);
```

Within the same transaction as the business change:

```sql
WITH event AS (
    INSERT INTO project_events (
        project_id,
        event_type,
        entity_type,
        entity_id,
        actor_id,
        entity_version,
        payload
    )
    VALUES (
        :project_id,
        'task.updated',
        'task',
        :task_id,
        :actor_id,
        :version,
        :payload
    )
    RETURNING id, project_id
)
SELECT pg_notify(
    'project_events',
    json_build_object(
        'event_id', id,
        'project_id', project_id
    )::text
)
FROM event;
```

Use one shared channel such as `project_events`, rather than dynamically creating one channel per project. The payload should contain only routing information—not the entire object or sensitive data.

## SSE endpoint

Conceptually:

```http
GET /api/projects/{project_id}/events
Last-Event-ID: 184290
Cookie: session=...
Accept: text/event-stream
```

The endpoint should:

1. Authenticate the user.
2. Verify current project membership.
3. Register the connection in a local project subscription registry.
4. Replay events with `id > Last-Event-ID`.
5. Continue streaming live events.
6. Send a heartbeat every 15–30 seconds.
7. Remove the subscription on disconnect.

Example wire format:

```text
id: 184291
event: task.updated
data: {"task_id":"...","version":7,"changes":{"status":"done"}}

```

Treat delivery as at-least-once. The React client should remember the highest processed event ID and safely ignore duplicates.

There is a subtle initialization race around `LISTEN`. PostgreSQL’s documented sequence is: commit `LISTEN`, read the current database state in a new transaction, then consume notifications. The durable cursor/replay mechanism makes that race harmless. [PostgreSQL `LISTEN` documentation](https://www.postgresql.org/docs/18/sql-listen.html)

## FastAPI implementation considerations

- Use a dedicated long-lived database connection for `LISTEN`; do not return it to the normal request pool.
- Reconnect with backoff if PostgreSQL drops the connection.
- After reconnecting, replay from the last event cursor before resuming live dispatch.
- Give each SSE client a bounded `asyncio.Queue`.
- Disconnect or force a resync when a slow client’s queue fills.
- Disable proxy buffering for the SSE route, for example `X-Accel-Buffering: no` with nginx.
- Configure load-balancer and proxy idle timeouts above the heartbeat interval.
- Periodically check `pg_notification_queue_usage()` operationally.
- Retain events long enough to cover expected client disconnections, then archive or purge them.

## Authorization

Project membership must be checked server-side when the stream opens. Also account for membership being revoked while a stream is active:

- Publish a `project.member_removed` event and disconnect that user, or
- Revalidate authorization periodically and for sensitive event types.

If Next.js and FastAPI are same-origin, an HTTP-only session cookie is convenient. Native `EventSource` does not provide a general custom-header interface, so bearer-token authentication usually requires a fetch-based SSE client or same-origin proxy. Avoid access tokens in query strings.

## CopilotKit and MAF

I would keep two conceptual streams:

- **AG-UI/CopilotKit stream:** agent-run events, tool calls, messages, state deltas
- **Project collaboration stream:** durable domain events such as task updates, comments, assignments, and membership changes

CopilotKit’s AG-UI integration already uses event-based streaming and shared state for agent-to-UI communication. [CopilotKit architecture](https://docs.copilotkit.ai/concepts/architecture), [AG-UI events](https://docs.ag-ui.com/concepts/events)

Do not make CopilotKit shared state your authoritative project state. PostgreSQL remains authoritative. When an MAF agent modifies a task, it should go through the same application command/service and produce the same `project_events` record as a human-originated mutation.

## When to introduce Redis or a broker

This PostgreSQL design should serve a moderate application well. Consider Redis Streams, NATS, or Kafka when:

- Notification volume becomes high enough that every API replica receiving every event is wasteful
- You need independent durable consumers
- You need long retention or event reprocessing
- Multiple services need consumer groups
- PostgreSQL notification handling becomes an observable bottleneck

The browser transport can remain SSE even if the internal event bus changes later. The durable event contract keeps that migration relatively painless.

## User

And can we skip Redis by leveraging native Postgres features like UNLOGGED tables for key-value caching, LISTEN/NOTIFY for pub/sub messaging, FOR UPDATE SKIP LOCKED for task queues, and advisory_locks for distributed locking?

## Assistant

Yes. For this application, I’d choose a PostgreSQL-first architecture and add Redis only after a measured bottleneck or missing guarantee appears.

PostgreSQL can cover all four roles, but they have different durability and operational characteristics:

| Need | PostgreSQL primitive | Recommendation |
|---|---|---|
| Real-time invalidation | `LISTEN/NOTIFY` | Yes, paired with a durable event table |
| Background jobs | `FOR UPDATE SKIP LOCKED` | Yes, using leases and idempotent workers |
| Distributed coordination | Advisory locks | Yes for short database-scoped critical sections |
| Key/value cache | Regular or `UNLOGGED` table | Yes for selected derived data, but it is not Redis-like memory caching |

## 1. Cache tables: use `UNLOGGED` selectively

A PostgreSQL cache table can work well for:

- Computed project summaries
- Agent context snapshots
- Permission calculation results
- Expensive query results
- Short-lived metadata
- Idempotency results

For example:

```sql
CREATE UNLOGGED TABLE app_cache (
    namespace  text NOT NULL,
    key        text NOT NULL,
    value      jsonb NOT NULL,
    expires_at timestamptz NOT NULL,
    updated_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (namespace, key)
);

CREATE INDEX app_cache_expiry_idx
    ON app_cache (expires_at);
```

But `UNLOGGED` has sharp semantics:

- The entire table is automatically truncated after a crash or unclean shutdown.
- It is not replicated to standby servers.
- It has no native TTL or eviction policy.
- It still consumes storage, buffer-cache space, connections, locks, and vacuum resources.

Those properties are documented explicitly by PostgreSQL. [PostgreSQL `UNLOGGED` tables](https://www.postgresql.org/docs/current/sql-createtable.html)

Therefore:

- Use it only when the data is completely reconstructable.
- Expect an empty cache after failover.
- Prevent a post-failover cache stampede.
- Delete expired rows in small batches.
- Tune autovacuum for high-churn cache tables.
- Add a maximum cache size or namespace quotas at the application level.

I would initially use an ordinary logged cache table unless benchmarks show WAL volume is material. Hot rows will normally live in PostgreSQL’s shared buffer and the OS page cache anyway. `UNLOGGED` reduces WAL overhead; it does not transform PostgreSQL into an in-memory store.

Also note that a cache inside the primary database does not reduce connection pressure or move load into a separate failure domain. It can avoid expensive joins or computation, but every hit still reaches PostgreSQL.

## 2. Pub/sub: durable event table plus `NOTIFY`

Use:

```text
project_events table = source of truth and replay
NOTIFY                = wake-up signal
FastAPI memory hub    = local browser fan-out
SSE                    = browser delivery
```

Do not rely on notifications alone. They have no durable history or acknowledgment mechanism, and notification payloads are limited. [PostgreSQL `NOTIFY`](https://www.postgresql.org/docs/current/sql-notify.html)

A notification only needs this:

```json
{
  "event_id": 184291,
  "project_id": "..."
}
```

Each FastAPI instance should have:

- One dedicated PostgreSQL listener connection
- An in-memory map of `project_id → SSE client queues`
- A cursor indicating the most recently dispatched durable event
- Reconnect and replay logic

Do not allocate one PostgreSQL connection per SSE client.

For queue wake-ups, use `NOTIFY` as an optimization too—but workers must periodically poll because notifications can be missed during reconnects.

## 3. Task queue: `SKIP LOCKED` works well

PostgreSQL explicitly identifies `SKIP LOCKED` as useful for multiple consumers accessing a queue-like table. [PostgreSQL locking clause](https://www.postgresql.org/docs/current/sql-select.html)

Use a durable logged table:

```sql
CREATE TABLE jobs (
    id              bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    queue           text NOT NULL,
    payload         jsonb NOT NULL,
    status          text NOT NULL DEFAULT 'pending',
    priority        integer NOT NULL DEFAULT 0,
    available_at    timestamptz NOT NULL DEFAULT now(),
    lease_until     timestamptz,
    worker_id       text,
    attempts        integer NOT NULL DEFAULT 0,
    max_attempts    integer NOT NULL DEFAULT 5,
    idempotency_key text,
    created_at      timestamptz NOT NULL DEFAULT now(),
    completed_at    timestamptz
);

CREATE UNIQUE INDEX jobs_idempotency_idx
    ON jobs (idempotency_key)
    WHERE idempotency_key IS NOT NULL;

CREATE INDEX jobs_claim_idx
    ON jobs (queue, priority DESC, available_at, id)
    WHERE status = 'pending';
```

Claim jobs atomically:

```sql
WITH candidate AS (
    SELECT id
    FROM jobs
    WHERE queue = :queue
      AND status = 'pending'
      AND available_at <= now()
    ORDER BY priority DESC, available_at, id
    FOR UPDATE SKIP LOCKED
    LIMIT :batch_size
)
UPDATE jobs AS j
SET status      = 'running',
    worker_id   = :worker_id,
    lease_until = now() + interval '2 minutes',
    attempts    = attempts + 1
FROM candidate
WHERE j.id = candidate.id
RETURNING j.*;
```

The critical design point is to commit immediately after claiming. Do not keep a database transaction open while the agent or external API call runs.

Instead:

1. Claim and establish a lease.
2. Commit.
3. Execute the job.
4. Renew the lease if necessary.
5. Mark it complete using a conditional update.
6. Requeue jobs whose leases expire.

Assume at-least-once execution. A worker can finish an external side effect and crash before marking the job complete. Use idempotency keys and make job handlers safe to retry.

A PostgreSQL queue is less suitable when you need extremely high enqueue/dequeue rates, sophisticated routing, very long retention, or large independent consumer groups. For typical agent jobs—seconds or minutes of work—the database overhead is usually tiny compared with the work itself.

## 4. Advisory locks: good, but keep them short

Prefer transaction-level locks:

```sql
SELECT pg_try_advisory_xact_lock(:project_lock_key);
```

These automatically release when the transaction commits or rolls back. PostgreSQL supports transaction- and session-level advisory locks, but session locks remain until explicitly released or the connection ends. [PostgreSQL advisory locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)

Good uses include:

- Ensuring only one process schedules recurring jobs
- Serializing a brief project state transition
- Preventing duplicate snapshot generation
- Electing one maintenance worker
- Guarding a migration or initialization step

Avoid advisory locks around long-running agent operations or network calls.

If the database connection disappears, its advisory lock is released, but the process might continue performing external work. A second process can then acquire the lock, producing split-brain behavior. For long operations, use a database lease with a monotonically increasing fencing token:

```text
resource_id | lease_owner | lease_until | fencing_token
```

Every protected write must include the expected fencing token. A stale worker can then be rejected even if it is still alive.

Other advisory-lock cautions:

- All cooperating code must follow the same lock convention; PostgreSQL does not enforce its meaning.
- Use deterministic, documented lock keys.
- Avoid accidentally hashing different resource types into the same key space.
- Advisory locks consume shared lock memory, so they are not appropriate as millions of per-object semaphores.
- Prefer database uniqueness constraints when the invariant can be expressed declaratively.

## Connection pooling caveat

If you use PgBouncer in transaction-pooling mode:

- `LISTEN` does not work.
- Session-level advisory locks do not work.
- Transaction-level advisory locks do work because they live entirely within one transaction.

PgBouncer documents this distinction directly. [PgBouncer feature compatibility](https://www.pgbouncer.org/features.html)

A practical setup is:

```text
Normal API requests
    → PgBouncer transaction pool
    → PostgreSQL

LISTEN connection
    → direct PostgreSQL connection
      or dedicated session-pooled endpoint

Queue workers
    → normal transaction pool

Transaction advisory locks
    → normal transaction pool
```

## Recommended initial stack

I would launch with:

- PostgreSQL domain tables
- Logged `project_events` outbox
- `LISTEN/NOTIFY` for low-latency signaling
- SSE for project collaboration updates
- Logged `jobs` table with `SKIP LOCKED`
- Transaction advisory locks for short coordination
- Regular cache tables initially
- `UNLOGGED` cache tables only for proven high-write, disposable data
- FastAPI in-memory fan-out per replica
- No Redis

Monitor database CPU, WAL generation, connection saturation, table bloat, queue claim latency, notification queue usage, and SSE event latency. Redis becomes justified when it solves an observed problem—particularly independent load isolation, extremely high ephemeral write rates, automatic cache eviction, presence/cursor traffic, or richer message-broker semantics—not merely because the application has “real-time” features.

## User

Now here comes the interesting part, the caveats:

1. We are leveraging Azure infrastructure, so we need to confirm this can be done with Azure Database for PostgreSQL (Flexible Server) and FastAPI running in Azure App Service.
2. We are currently using CosmosDB and the container approach that it follows.

What options do we have?

## Assistant

Yes, all the PostgreSQL primitives are available on Azure Database for PostgreSQL Flexible Server. The Azure caveats are mostly about connection pooling, failover, App Service scaling, and whether Cosmos DB remains the system of record.

The three credible architectures are:

| Option | System of record | Real-time delivery | Best when |
|---|---|---|---|
| PostgreSQL-native | PostgreSQL | `LISTEN/NOTIFY` → FastAPI → SSE | You want relational transactions, reporting, and fewer Azure services |
| Cosmos-native | Cosmos DB | Change Feed → Function → Azure Web PubSub | Your container/partition model already works and global distribution matters |
| Hybrid | One authoritative store plus an asynchronous projection | Change Feed or outbox bridges stores | You need gradual migration or distinct read/write models |

## 1. PostgreSQL Flexible Server compatibility

| Feature | Flexible Server | Important Azure caveat |
|---|---|---|
| `UNLOGGED` tables | Supported | Data can disappear during crash, failover, and near-zero-downtime compute scaling |
| `LISTEN/NOTIFY` | Supported | Listener must use a session-bound connection, normally direct port `5432` |
| `FOR UPDATE SKIP LOCKED` | Supported | Works through normal or transaction-pooled connections |
| Transaction advisory locks | Supported | Work through PgBouncer transaction pooling |
| Session advisory locks | Supported | Require a direct/session-pooled connection and disappear on failover |

Azure Flexible Server’s built-in PgBouncer runs on port `6432` and defaults to transaction pooling. `LISTEN` and session advisory locks are not compatible with transaction pooling, so use two connection paths:

```text
FastAPI request/worker pool
    → PostgreSQL :6432
    → PgBouncer transaction pooling

FastAPI notification listener
    → PostgreSQL :5432
    → dedicated connection
    → LISTEN project_events
```

Azure documents the built-in PgBouncer configuration and its transaction-pooling default. [Flexible Server PgBouncer](https://learn.microsoft.com/en-us/azure/postgresql/connectivity/concepts-pgbouncer)

For private networking:

```text
App Service VNet integration
    → private endpoint/private DNS
    → PostgreSQL Flexible Server
```

App Service VNet integration supports outbound TCP access to private resources. [App Service VNet integration](https://learn.microsoft.com/en-us/azure/app-service/overview-vnet-integration)

### `UNLOGGED` is riskier on Flexible Server

Azure explicitly warns that near-zero-downtime server scaling loses all data in unlogged tables. [Flexible Server scaling considerations](https://learn.microsoft.com/en-us/azure/postgresql/scale/concepts-scaling-resources)

Therefore, unlogged tables are acceptable for disposable caches, but not for:

- Durable project events
- Jobs
- Idempotency records
- Leases
- Client replay cursors

I would use logged tables first and introduce unlogged caching only after measuring WAL pressure.

## 2. FastAPI and SSE/WebSockets on App Service

Both approaches are feasible.

For direct SSE from FastAPI:

- Enable `Always On`.
- Emit a heartbeat every 15–30 seconds.
- Reconnect with `Last-Event-ID`.
- Reauthorize project access on every reconnect.
- Assume instances can restart at any time.
- Disable ARR affinity if the rest of the application is stateless.

App Service has an approximately four-minute idle request timeout, but an SSE stream that periodically sends data is not idle. [App Service request timeout](https://learn.microsoft.com/en-us/troubleshoot/azure/app-service/web-request-times-out-app-service)

With scale-out, every FastAPI process that can own SSE clients must receive notifications:

```text
PostgreSQL NOTIFY
     ├── App Service instance A / worker 1 → its SSE clients
     ├── App Service instance A / worker 2 → its SSE clients
     └── App Service instance B / worker 1 → its SSE clients
```

That means either:

- One Uvicorn worker per App Service instance, with one listener; or
- One listener per worker; or
- An internal process-level fan-out mechanism.

A single listener in one Gunicorn worker cannot reach the in-memory client queues owned by another worker.

WebSockets are also supported on Linux App Service. [App Service Linux WebSocket support](https://learn.microsoft.com/en-us/troubleshoot/azure/app-service/faqs-app-service-linux-new)

## 3. Cosmos-native option

If Cosmos DB is working well as the source of truth, you do not need to introduce PostgreSQL merely to obtain real-time updates.

The Azure-native flow is:

```text
FastAPI
   │ write project item + immutable event
   ▼
Cosmos DB container
   │ Change Feed
   ▼
Azure Function
   │ send to project:{project_id}
   ▼
Azure Web PubSub
   │
   ▼
Authorized browser connections
```

Azure Web PubSub manages the long-running WebSocket connections and routing across scaled App Service instances. It supports server-to-client broadcasting and named groups. [Azure Web PubSub reliability](https://learn.microsoft.com/azure/azure-web-pubsub/concept-availability-zones)

Use one Web PubSub group per project:

```text
project:8ac0e9...
```

FastAPI authenticates the user, verifies project membership, and issues an appropriately scoped Web PubSub token. Clients should not be allowed to join arbitrary project groups.

### Use the Cosmos partition as the consistency boundary

The ideal container design is:

```text
Container: project_data
Partition key: /projectId

Items:
  { type: "project", ... }
  { type: "task", ... }
  { type: "comment", ... }
  { type: "project_event", eventId: "...", ... }
```

Then a single Cosmos transactional batch can:

1. Update a task.
2. Create an immutable event item.
3. Commit both atomically.

Cosmos transactions only span one container and one logical partition key. [Cosmos transactional batches](https://learn.microsoft.com/en-us/azure/cosmos-db/transactional-batch)

If tasks, projects, and events are in separate containers, you cannot atomically write the domain item and outbox event. Your choices become:

- Co-locate immutable events with the changed item.
- Use the source container’s Change Feed directly.
- Accept eventual consistency and deduplicate downstream.
- Redesign container boundaries around the project aggregate.

### Change Feed details

Cosmos Change Feed provides:

- Ordering within a partition key
- Parallel consumption across partition ranges
- Durable continuation/checkpoint positions
- At-least-once processing with the Change Feed Processor

There is no guaranteed global order across partition keys. That is usually acceptable if `projectId` is the partition key because collaboration ordering matters per project. [Cosmos DB Change Feed](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed)

Use immutable event items even when consuming latest-version mode. Otherwise:

- Intermediate document updates can be collapsed.
- Ordinary deletes are not represented in latest-version mode.
- It is difficult to infer business intent from a changed document.

An immutable `project_event` item turns every meaningful action into an insert, so latest-version Change Feed sees every event, including an explicit `task.deleted` event.

### Important Python limitation

The managed Change Feed Processor library currently supports .NET and Java, not Python. Python supports the pull model, but you must manage continuations and consumer coordination yourself. [Change Feed Processor support](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed-processor)

For a FastAPI/Python application, the cleanest choice is usually:

```text
Cosmos Change Feed
    → Azure Functions Cosmos trigger
    → Azure Web PubSub
```

The Functions trigger supports Python in latest-version mode. The immutable event-item approach fits that mode well. [Cosmos Change Feed with Azure Functions](https://learn.microsoft.com/en-us/azure/cosmos-db/change-feed-functions)

## 4. Tasks, locks, and caching if you stay on Cosmos

You can still avoid Redis.

### Background jobs

Options, in order of preference:

1. **Azure Service Bus** for durable agent jobs.
2. Cosmos event/job documents plus Change Feed.
3. A PostgreSQL `SKIP LOCKED` queue if PostgreSQL is already being adopted.

Service Bus gives you durable messages, peek-lock delivery, dead-lettering, duplicate detection, and sessions for per-project ordering. [Azure Service Bus delivery guarantees](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-message-loss-and-duplicates)

Cosmos Change Feed is adequate for “react to this data change,” but Service Bus is better for explicit job semantics such as:

- Retry after a delay
- Maximum delivery count
- Dead-letter handling
- Scheduled execution
- Per-project ordered processing
- Operational queue inspection

### Distributed coordination

Cosmos supports optimistic concurrency through `_etag`. A lease document can contain:

```json
{
  "id": "project-snapshot:123",
  "owner": "worker-7",
  "leaseUntil": "...",
  "fencingToken": 42,
  "ttl": 120
}
```

Acquire or renew it with conditional `_etag` writes. Azure publishes this as a distributed-lock pattern. [Cosmos DB distributed lock pattern](https://learn.microsoft.com/en-us/samples/azure-samples/cosmos-db-design-patterns/distributed-lock/)

As with PostgreSQL advisory locks, use fencing tokens for long external operations.

### Caching

Cosmos TTL can expire cache documents, but a cache stored in Cosmos still consumes RUs and does not isolate load from Cosmos. Prefer:

- Per-process bounded memory cache for very short-lived data
- Cosmos TTL documents for shared, reconstructable values
- Precomputed/project-summary documents
- Azure Managed Redis only if measurement later justifies a dedicated cache

## 5. Hybrid option

A hybrid is appropriate for gradual migration, but define exactly one system of record for each aggregate.

### Cosmos authoritative

```text
Cosmos transaction
    → Change Feed
    → PostgreSQL projection/event table
    → NOTIFY
    → FastAPI SSE
```

This permits PostgreSQL reporting and collaboration replay without rewriting all current storage immediately. It is eventually consistent.

### PostgreSQL authoritative

```text
PostgreSQL transaction/outbox
    → projector
    → Cosmos read/document model
```

This is useful if PostgreSQL becomes authoritative for projects and tasks while Cosmos remains useful for large agent-generated documents or globally distributed read models.

Avoid writing to Cosmos and PostgreSQL independently inside the same request. There is no cross-database atomic transaction. Use an outbox/change-feed projection and make consumers idempotent.

## My recommendation

If your Cosmos container is already partitioned by `projectId`, stay Cosmos-native initially:

```text
Cosmos project container
    + immutable event items
    + transactional batch
    + Change Feed-triggered Azure Function
    + Azure Web PubSub project groups
    + Service Bus for agent jobs
```

That eliminates Redis, avoids a data migration, and makes App Service instances stateless.

Choose PostgreSQL-native if you are already encountering friction from:

- Cross-project or cross-entity queries
- Complex reporting
- Transactions spanning tasks, assignments, dependencies, and memberships
- RU costs from relational-style access patterns
- Container boundaries that do not align with project transaction boundaries

In that case, Flexible Server plus direct `LISTEN` on port `5432`, ordinary traffic through PgBouncer on `6432`, and App Service SSE is a sound architecture. Redis is still optional rather than required.

## User

We are showing the first version of the app to our stakeholders, they don't even know what cosmosDB or postgresql is... They just want the app and the agent to be fast and they can collaborate effectively in their project. I am the lead team and I make the tech decisions, which one do you recommend and why? Stay in cosmos and adopt azure pub/sub? Move to PG and configure the notify listen approach there? We don't care about global capabilities of Cosmos.

## Assistant

My recommendation: stay on Cosmos DB for the stakeholder version and add Azure Web PubSub for collaboration. Do not migrate databases before validating the product.

If we were starting from scratch, I would choose PostgreSQL for a project-management domain. But you are not starting from scratch, and moving now introduces substantial risk without creating a meaningful stakeholder-visible improvement.

## Why this is the right decision now

Switching to PostgreSQL would require you to:

- Redesign or translate the container model
- Migrate existing data
- Rewrite persistence and query code
- Validate transaction and authorization behavior
- Operate dedicated `LISTEN` connections
- Handle listener reconnection and event replay
- Manage SSE connections across App Service workers

None of that makes the agent reason faster or makes the UI visibly better than adding managed real-time delivery to the system you already have.

The latency stakeholders experience will mostly come from:

- Model execution and tool calls
- How much project context you load for the agent
- Browser state updates and refetching
- Slow or cross-partition Cosmos queries

The difference between a correctly partitioned Cosmos point operation and PostgreSQL will usually be irrelevant compared with an agent call.

## Architecture I would ship

```text
                    ┌──────────────────┐
Human mutations ───►│                  │
                    │     FastAPI      │
Agent mutations ───►│ command service  │
                    └────────┬─────────┘
                             │
                    1. Commit mutation
                             ▼
                       Cosmos DB
                             │
                    2. Publish event
                             ▼
                   Azure Web PubSub
                             │
                  project:{project_id}
                             │
               ┌─────────────┴─────────────┐
               ▼                           ▼
          Browser A                   Browser B
```

Use Azure Web PubSub—not Service Bus or Event Grid—for browser collaboration.

Keep the CopilotKit/AG-UI stream responsible for:

- Agent messages
- Tool execution progress
- Generated UI
- Agent-run state

Use Web PubSub for durable application state changes:

- `task.updated`
- `comment.created`
- `member.assigned`
- `project.status_changed`
- `artifact.created`

## Keep every mutation behind one service

Both humans and agents should call the same application command layer:

```python
await project_service.update_task(
    project_id=project_id,
    task_id=task_id,
    changes=changes,
    actor=actor,
)
```

That service should:

1. Authenticate and authorize the actor.
2. Perform an ETag-conditional Cosmos write.
3. Generate an event ID and entity version.
4. Publish an event to `project:{project_id}`.
5. Return the authoritative result.

This prevents agent changes and human changes from behaving differently.

## Treat Web PubSub messages as invalidations

Do not place the entire project state in every message. Publish a small event:

```json
{
  "eventId": "01K...",
  "type": "task.updated",
  "projectId": "project-123",
  "entityId": "task-456",
  "version": 18,
  "actorId": "user-789",
  "changes": {
    "status": "done"
  }
}
```

The browser can:

1. Apply the small patch immediately for responsiveness.
2. Invalidate/refetch the affected query in the background.
3. Ignore events whose `eventId` was already processed.
4. Fetch the current project snapshot after reconnecting.

This keeps Cosmos authoritative and prevents transient messaging issues from corrupting client state.

## Start without Change Feed if all writes pass through FastAPI

For the stakeholder version, I would initially publish directly from FastAPI immediately after a successful Cosmos write.

That gives you:

- The lowest latency
- Fewer Azure components
- Straightforward debugging
- No Change Feed processor complexity
- No Azure Function cold-start concern

The tradeoff is that the Cosmos write could succeed while the Web PubSub send fails. Handle that with:

- Client refetch on reconnect
- Refetch when the browser regains focus
- A lightweight periodic reconciliation
- An entity/project version number

This is acceptable because Web PubSub is announcing state changes; it is not storing the state.

Add Cosmos Change Feed when:

- Some writes bypass FastAPI
- Other services begin updating Cosmos
- You need reliable downstream integrations
- Auditing every committed mutation becomes important

At that point, create immutable event items alongside project changes and bridge Change Feed to Web PubSub. Clients already deduplicate by `eventId`, so the transport can evolve without changing the UI contract.

## Concurrency

Use Cosmos `_etag` optimistic concurrency:

```text
User A reads task version 17
User B reads task version 17

User A writes with ETag 17 → succeeds, version becomes 18
User B writes with ETag 17 → receives conflict
```

FastAPI can then return a `409 Conflict` with the current task. The UI can show:

> This task was updated by another collaborator. Review the latest version.

Real-time delivery alone does not solve concurrent editing; version checks do.

## Azure settings

For Web PubSub:

- One group per project
- FastAPI authorizes group membership
- Short-lived client access tokens
- Reliable Web PubSub client SDK
- Reconnect and state resynchronization
- Never allow clients to select arbitrary project groups without authorization

For App Service:

- Keep FastAPI stateless
- Enable `Always On`
- Disable ARR affinity
- Scale out normally
- Do not hold browser WebSockets inside FastAPI; Web PubSub owns them

That last point removes most of the App Service/SSE scaling complexity.

## When I would reconsider PostgreSQL

After stakeholder validation, review the persistence choice separately. Move to PostgreSQL if the team is experiencing concrete problems such as:

- Frequent cross-container queries
- Complicated denormalization
- Transactions spanning tasks, assignments, dependencies, and membership
- Reporting queries that are difficult or expensive in Cosmos
- RU throttling or unpredictable Cosmos costs
- Developers fighting the container model
- Project data naturally behaving as a relational aggregate

Do not migrate merely to obtain notifications. Web PubSub solves the collaboration transport independently of the database.

So the decision I would document is:

> For the stakeholder release, retain Cosmos DB as the authoritative store, route all human and agent mutations through FastAPI, and use Azure Web PubSub groups for project-scoped collaboration. Reassess PostgreSQL after product validation based on data-model and operational evidence—not real-time transport needs.

## User

But can CosmosDB work for this type of app? Where the users need to work in a single project as a group, modify it, run sharepoint and planner operations, send emails and ms teams messages and more...

## Assistant

Yes—Cosmos DB can work well for this application. In fact, a project is a natural Cosmos partition boundary: multiple users collaborate on one project, while most reads and writes remain scoped to that project.

The external operations do not make PostgreSQL necessary. SharePoint, Planner, Outlook, and Teams introduce distributed-workflow problems that neither database solves by itself.

## Recommended architecture

```text
Browser
   │
   ├── Agent stream ◄──────── CopilotKit / AG-UI
   │
   └── Project events ◄────── Azure Web PubSub
                                  ▲
                                  │
FastAPI ──► Cosmos DB ──► Change Feed ──► Service Bus
   ▲                                      │
   │                                      ▼
   └──────── operation results ◄──── Graph workers
                                          │
                             SharePoint / Planner /
                             Outlook / Teams
```

Use each component for one responsibility:

- **Cosmos DB:** authoritative application and workflow state
- **Web PubSub:** live browser collaboration
- **Service Bus:** durable execution of Microsoft 365 operations
- **Microsoft Graph workers:** execute SharePoint, Planner, email, and Teams actions
- **CopilotKit/AG-UI:** stream the agent’s reasoning progress, tool calls, and UI state

## Model the project correctly

Do not store the entire project in one giant Cosmos document. That would create ETag conflicts whenever two users modify different parts of the project.

Instead, keep separate items under the same project partition:

```json
{
  "id": "task-456",
  "projectKey": "tenant-123:project-789",
  "type": "task",
  "title": "Prepare launch deck",
  "status": "in_progress",
  "version": 12
}
```

```json
{
  "id": "comment-987",
  "projectKey": "tenant-123:project-789",
  "type": "comment",
  "taskId": "task-456",
  "text": "Waiting for legal approval"
}
```

```json
{
  "id": "operation-abc",
  "projectKey": "tenant-123:project-789",
  "type": "integration_operation",
  "operationType": "planner.create_task",
  "status": "pending",
  "requestedBy": "user-123"
}
```

A reasonable shared container is:

```text
Container: project_data
Partition key: /projectKey

projectKey = tenantId + ":" + projectId
```

This gives you:

- Efficient project-scoped reads
- ACID transactional batches within a project
- Ordered Change Feed processing per project partition
- Independent updates to tasks, comments, and operations
- A straightforward tenant-isolation boundary

Cosmos transactions are limited to one container and logical partition, so co-locating project state and its event/outbox items is important. [Cosmos transactional batches](https://learn.microsoft.com/en-us/azure/cosmos-db/transactional-batch)

Use a separate projection for queries such as “show me every project this user belongs to,” because that crosses project partitions.

## External actions should be asynchronous workflows

Suppose the agent wants to create a Planner task.

Do not make the entire operation:

```text
HTTP request → Graph call → wait → update database → respond
```

Graph may throttle, time out, or accept an operation while the response is lost. Instead:

1. FastAPI validates the user and proposed action.
2. A Cosmos transaction updates project state and creates an immutable operation/outbox item.
3. Change Feed relays that operation to Service Bus.
4. A worker calls Microsoft Graph.
5. The worker handles retries and records the external resource ID.
6. Cosmos is updated to `succeeded`, `failed`, or `needs_attention`.
7. Web PubSub broadcasts the result to everyone viewing the project.

Microsoft provides this exact general pattern using Cosmos transactional batches, Change Feed, and Service Bus. [Cosmos transactional outbox architecture](https://learn.microsoft.com/en-us/azure/architecture/databases/guide/transactional-out-box-cosmos)

Example operation:

```json
{
  "id": "op-01K...",
  "projectKey": "tenant-123:project-789",
  "type": "integration_operation",
  "operationType": "planner.task.create",
  "status": "pending",
  "idempotencyKey": "planner-task:internal-task-456",
  "input": {
    "internalTaskId": "task-456",
    "planId": "...",
    "title": "Prepare launch deck"
  },
  "attempts": 0,
  "externalResourceId": null
}
```

The UI can immediately show:

```text
Creating task in Planner…
```

and later receive:

```text
Planner task created successfully
```

That is usually a better user experience than leaving an HTTP request open.

## Microsoft Graph caveats

Graph operations must be designed for throttling. Graph returns `429` with `Retry-After`; workers should honor that rather than retrying immediately. [Microsoft Graph throttling guidance](https://learn.microsoft.com/en-us/graph/throttling)

Use Service Bus because it gives you:

- At-least-once delivery
- Delayed retries
- Dead-lettering
- Per-project ordering through sessions
- Duplicate detection
- Independent scaling of Graph workers

Consumers must still be idempotent. For every external action, store:

- Internal operation ID
- Idempotency key
- Attempt count
- Current status
- External resource ID
- Last error
- Next retry time
- Requesting user
- Permission mode used

Be especially careful with email. If a request times out after Microsoft accepted it, blindly retrying may send the email twice. Mark ambiguous results for reconciliation or human review rather than assuming failure.

## Synchronizing changes made outside your app

Users may modify Planner, SharePoint, or Teams directly. In those cases, Cosmos should hold a projection and mapping, while Microsoft 365 remains authoritative for the external resource.

Use:

- Microsoft Graph change notifications where supported
- Delta queries where supported
- Periodic reconciliation where neither is available

For example:

```text
Graph notification
    → webhook
    → validate notification
    → fetch/reconcile external resource
    → update Cosmos projection
    → broadcast Web PubSub event
```

Teams supports change notifications for messages, membership, teams, and other supported resources. Subscriptions expire and must be renewed. [Teams change notifications](https://learn.microsoft.com/en-us/graph/teams-change-notification-in-microsoft-teams-overview)

## Permissions and agent safety

Separate two authorization questions:

1. Is this user allowed to perform the action in your project?
2. Does the user or application have the required Microsoft Graph permission?

For user-directed actions, prefer delegated/on-behalf-of authorization so Microsoft 365 enforces that user’s access.

For autonomous agent actions, use application permissions only for explicitly approved operations and tenants. High-impact actions should support confirmation policies, for example:

| Action | Suggested policy |
|---|---|
| Read SharePoint document | Execute automatically |
| Create Planner task | Execute automatically if project policy permits |
| Update task assignment | Execute or request confirmation |
| Send Teams message | Preview or confirm depending on audience |
| Send external email | Require confirmation |
| Delete SharePoint content | Require confirmation |

Do not store Graph access tokens or client secrets in ordinary Cosmos project documents. Use managed identity, Microsoft identity libraries, and Key Vault as appropriate.

## When Cosmos would become the wrong choice

Reconsider PostgreSQL when you see actual evidence of:

- Many cross-project or cross-container queries
- Complex relational reporting
- Transactions that cannot fit within one project partition
- Extensive many-to-many querying across projects, users, tasks, and resources
- High RU consumption from relational-style access
- Developers maintaining too many projections and denormalized copies
- A single project becoming a genuinely hot partition

Normal project collaboration, Graph integrations, email, and agent operations do not by themselves require PostgreSQL.

My recommendation remains:

> Keep Cosmos for this version. Partition by tenant plus project, store individual project entities rather than one giant project document, add Web PubSub for collaboration, and use a Cosmos outbox plus Service Bus for durable Microsoft 365 operations.

That architecture can support the stakeholder version and continue into production without forcing an immediate database migration.

## User

Let's expand on several topics you mentioned:

1. We are indeed having a lot of issues with the agent and Graph API operations failing or taking a long time or retries. This degrades the agent experience for our stakeholders. How to fix it?
---
2. Explain in detail how this become part of our current FastAPI + MAF/CopilotKit Agent+ CosmosDB:
  - Service Bus: durable execution of Microsoft 365 operations
  - Microsoft Graph workers: execute SharePoint, Planner, email, and Teams actions
---
3. We are at phase 2 of the project, phases 1 and 2 involved single project operations: create folders and documents in sharepoint, planner tasks and more. The 3rd phase has more visibility for leadership and higher-ups, I call it the portfolio management stage, but officially is called the intelligence stage. 

In this latest stage the agent will need to be able to reply to queries that involve several projects at a time: a C-suite or VP could ask information about multiple projects and would like to have an overview of them and their statuses, and high executive functions common to their roles in a big organization. 

Any advise for this stage?


----

Another considerations:
1. We have only one tenant: the org, with potentially hundreds of users initially.
2. For now the domain is project management in general, but certain domain-specific request could come from different departments like legal, HR, finance, etc...

## Assistant

My recommendation remains to keep Cosmos DB, but change the execution architecture now:

1. Stop letting agent tools call Microsoft Graph synchronously.
2. Represent every external action as a durable operation.
3. Use Cosmos transactional outbox → Service Bus → specialized Graph workers.
4. Stream operation progress through Web PubSub and AG-UI.
5. For phase 3, build a portfolio/intelligence read model rather than querying every project or Microsoft Graph live.

The current Graph reliability problem is primarily an execution-model problem, not a Cosmos-versus-PostgreSQL problem.

# 1. Fixing the agent and Graph experience

## The current anti-pattern

You likely have some variation of:

```text
User asks agent
    → model selects tool
    → tool calls Graph
    → Graph throttles/times out
    → tool retries
    → agent waits
    → AG-UI stream appears stuck
    → entire run eventually fails
```

This couples three systems with very different latency characteristics:

- The interactive agent turn
- Microsoft Graph
- The external Microsoft 365 workload behind Graph

A Planner request, SharePoint request, and Outlook request can all throttle differently. Microsoft Graph returns `429` responses with `Retry-After`, and Microsoft recommends honoring that delay instead of retrying immediately. [Microsoft Graph throttling guidance](https://learn.microsoft.com/en-us/graph/throttling)

## Target experience

Change agent tools from “execute this operation” into “accept and schedule this operation.”

```text
User asks agent
    → agent validates intent
    → tool creates operation
    → tool returns operation ID immediately
    → agent says “I’ve started that”
    → worker performs Graph operation
    → UI displays live progress
    → agent/UI receives completion or failure
```

The operation state machine should be explicit:

```text
accepted
   │
   ▼
queued ───────────────► cancelled
   │
   ▼
running
   ├──► retry_wait ───► queued
   ├──► needs_auth
   ├──► needs_review
   ├──► failed
   └──► succeeded
```

The agent must distinguish:

- “I queued the Planner task”
- “I am creating the Planner task”
- “The Planner task was created”

It must never report success merely because the job was accepted.

## Error classification

Do not retry every Graph failure.

| Response | Handling |
|---|---|
| `429` | Honor `Retry-After`; schedule a retry |
| `502`, `503`, `504` | Exponential backoff with jitter |
| Network timeout | Retry if the operation is safely idempotent; otherwise reconcile |
| `401` | Refresh/reacquire authorization once |
| `403` | Permanent permission failure or approval required |
| `404` | Usually permanent; sometimes retry briefly after resource creation |
| `409`, `412` | Conflict; fetch current resource and reconcile |
| Other `4xx` | Generally permanent validation failure |

Use time budgets:

- One Graph HTTP attempt should have a bounded timeout.
- The worker should not sleep for a long `Retry-After`.
- Schedule a future Service Bus delivery and release the current message.
- Do not stack many Graph SDK retries inside many worker retries.

Graph SDKs already implement `Retry-After`-aware handlers. Keep their per-attempt retry count modest and let the durable workflow control longer retries.

## Isolate different Graph workloads

Because you have one tenant, the organization’s operations share Graph throttling budgets. A burst of SharePoint document creation should not block a critical Teams notification.

Use separate worker pools or Service Bus subscriptions:

```text
graph-sharepoint
graph-planner
graph-mail
graph-teams
```

Each gets its own:

- Concurrency limit
- Retry policy
- Circuit breaker
- Dead-letter monitoring
- Graph permission set
- Operational metrics

For example, you might allow more concurrent SharePoint reads than email sends. The exact values should come from load testing and observed `429` rates.

## Handle the “did it succeed?” crash window

Consider:

```text
Worker sends Teams message successfully
    → worker crashes before updating Cosmos
    → Service Bus redelivers operation
    → worker sends duplicate message
```

No queue or database can universally guarantee exactly-once external side effects.

Classify operations:

### Naturally idempotent or reconcilable

- Create a known SharePoint folder path
- Upload/update a known document
- Update an existing Planner task by ID
- Patch known SharePoint metadata

A retry can first check whether the desired state already exists.

### Non-idempotent or outcome-ambiguous

- Send email
- Send a Teams message
- Create a Planner task without a known external ID
- Invite or notify a large set of users

For these operations:

- Store an internal idempotency key.
- Store any returned external resource ID.
- Use a deterministic external marker where the API allows one.
- If a timeout occurs after sending, use `outcome_unknown`.
- Reconcile before retrying.
- Require human confirmation before retrying high-impact ambiguous operations.

Email should be especially conservative. Duplicated emails are more damaging than a delayed email.

## Measure the right things

Every operation should carry one correlation chain:

```text
agent run ID
thread ID
operation ID
Service Bus message ID
Graph client-request-id
Graph request-id
project ID
actor ID
```

Track:

- Operation acceptance latency
- Time waiting in queue
- Graph execution latency
- Retry count
- Success rate by Graph workload and endpoint
- `429` rate
- `403` rate
- `outcome_unknown` count
- Dead-letter count
- End-to-end time from user request to completion

That will tell you whether the bottleneck is throttling, authorization, worker capacity, or a particular Microsoft 365 workload.

# 2. Integrating Service Bus and Graph workers

## Recommended component architecture

```text
┌─────────────────────────────────────────────────────────┐
│ FastAPI App Service                                     │
│                                                         │
│ REST commands     MAF agent       AG-UI endpoint        │
│       │               │                  │              │
│       └──────► shared OperationService ◄─┘              │
└──────────────────────────┬──────────────────────────────┘
                           │ transactional batch
                           ▼
                 ┌───────────────────┐
                 │ Cosmos project_data│
                 │                   │
                 │ domain item       │
                 │ operation item    │
                 │ outbox event      │
                 └─────────┬─────────┘
                           │ Change Feed
                           ▼
                    Outbox relay Function
                           │
                           ▼
                  Service Bus project-events
                    │       │       │
             ┌──────┘       │       └───────────┐
             ▼              ▼                   ▼
       Graph workers   Realtime worker   Portfolio projector
             │              │                   │
             ▼              ▼                   ▼
      Microsoft Graph   Web PubSub       Portfolio read model
             │
             ▼
      Cosmos operation result
             │
             └── new outbox event ──► Service Bus
```

Microsoft has published this Cosmos transactional-outbox pattern using transactional batches, Change Feed, and Service Bus. [Cosmos transactional outbox](https://learn.microsoft.com/en-us/azure/architecture/databases/guide/transactional-out-box-cosmos)

## Cosmos records

A user or agent command should atomically write:

1. The intended internal state change, if applicable.
2. The operation record.
3. An immutable outbox event.

All must share the same project partition.

### Operation item

```json
{
  "id": "op-01KABC",
  "projectId": "project-123",
  "type": "integration_operation",
  "operationType": "planner.task.create",
  "status": "queued",
  "requestedBy": "user-456",
  "requestedByAgent": true,
  "agentRunId": "run-789",
  "threadId": "thread-321",
  "authMode": "delegated",
  "authContextRef": "session-abc",
  "idempotencyKey": "planner-task:internal-task-987",
  "input": {
    "internalTaskId": "task-987",
    "planId": "plan-...",
    "title": "Review contract"
  },
  "attempt": 0,
  "createdAt": "...",
  "updatedAt": "..."
}
```

### Outbox event

```json
{
  "id": "event-01KDEF",
  "projectId": "project-123",
  "type": "domain_event",
  "eventType": "integration.operation.requested",
  "operationId": "op-01KABC",
  "sequence": 143,
  "createdAt": "...",
  "ttl": 604800
}
```

The event is immutable. TTL can remove old outbox events after an appropriate retention period.

## FastAPI endpoint

Expose the operation concept independently of the agent:

```http
POST /api/projects/{project_id}/operations
```

Response:

```http
HTTP/1.1 202 Accepted
Location: /api/operations/op-01KABC
```

```json
{
  "operationId": "op-01KABC",
  "status": "queued",
  "message": "Planner task creation has been queued."
}
```

Also provide:

```http
GET /api/operations/{operation_id}
POST /api/operations/{operation_id}/cancel
POST /api/operations/{operation_id}/retry
```

That lets the regular UI and the agent use exactly the same operation service.

## MAF tool design

The agent tool should schedule work:

```python
@tool
async def create_planner_task(
    project_id: str,
    internal_task_id: str,
    plan_id: str,
    title: str,
    context: ToolContext,
) -> dict:
    """Schedule creation of a Planner task.

    Returns an operation identifier. A queued operation is not yet complete.
    """
    operation = await operation_service.submit(
        project_id=project_id,
        operation_type="planner.task.create",
        actor=context.user,
        agent_run_id=context.run_id,
        thread_id=context.thread_id,
        payload={
            "internalTaskId": internal_task_id,
            "planId": plan_id,
            "title": title,
        },
    )

    return {
        "operationId": operation.id,
        "status": "queued",
        "message": "Planner task creation was accepted for processing.",
    }
```

The tool description matters: it must tell the model that `queued` does not mean `completed`.

For dependent multi-step actions, use a MAF workflow:

```text
Create SharePoint folder
    → wait for completion
    → upload document
    → wait for completion
    → create Planner task with document link
```

Persist workflow state between steps. Do not keep one Service Bus message or Graph HTTP request open for the whole workflow.

MAF workflows exposed through AG-UI support structured steps, custom events, and interrupt/resume behavior, which fits approvals and external-operation waiting. [MAF workflows with AG-UI](https://learn.microsoft.com/en-us/agent-framework/integrations/ag-ui/workflows)

## Service Bus topology

Use a topic such as:

```text
project-events
```

Create filtered subscriptions:

```text
graph-sharepoint
graph-planner
graph-mail
graph-teams
realtime
portfolio-projector
audit
```

Message properties:

```json
{
  "messageId": "event-01KDEF",
  "sessionId": "project-123",
  "subject": "integration.operation.requested",
  "correlationId": "op-01KABC"
}
```

Recommendations:

- `MessageId = eventId` for broker duplicate detection.
- Use `SessionId = projectId` only where per-project ordering is necessary.
- Do not serialize unrelated operations unnecessarily.
- Configure dead-lettering after a bounded number of deliveries.
- Do not put Graph tokens, secrets, or large document contents in messages.
- Messages should contain IDs; workers load authoritative input from Cosmos.

Service Bus supports peek-lock delivery, sessions, dead-lettering, and automatic lock renewal in the Python SDK. [Azure Service Bus Python client](https://learn.microsoft.com/en-us/python/api/overview/azure/servicebus-readme)

## Graph worker algorithm

Each worker should:

1. Receive the Service Bus message.
2. Load the operation from Cosmos.
3. Ignore it if already completed.
4. Claim it with an ETag-conditional update.
5. Acquire the appropriate Graph authorization.
6. Execute one bounded Graph step.
7. Classify the result.
8. Persist success, failure, or retry state.
9. Emit a new immutable event.
10. Complete the Service Bus message.

Pseudo-code:

```python
async def process(message):
    operation = await operations.get(message.correlation_id)

    if operation.status == "succeeded":
        await receiver.complete_message(message)
        return

    operation = await operations.claim(
        operation,
        worker_id=worker_id,
        expected_etag=operation.etag,
    )

    try:
        token = await graph_auth.acquire_for(operation)
        result = await graph_executor.execute(operation, token)

        await operations.mark_succeeded(
            operation,
            external_resource_id=result.resource_id,
            result=result.safe_metadata,
        )

        await receiver.complete_message(message)

    except GraphThrottled as exc:
        await operations.mark_retry_wait(
            operation,
            retry_at=utcnow() + exc.retry_after,
        )
        await schedule_retry(operation, exc.retry_after)
        await receiver.complete_message(message)

    except PermanentGraphFailure as exc:
        await operations.mark_failed(operation, exc.safe_details)
        await receiver.dead_letter_message(
            message,
            reason=exc.code,
        )
```

If a Graph call may run longer than the message-lock period, use lock renewal. Better still, split long workflows into smaller persisted steps.

## Where to run workers

I recommend separate Python Azure Functions using Service Bus triggers:

```text
FastAPI App Service
    → handles users and agent streams

Azure Functions
    → handles Service Bus messages and Graph operations
```

Advantages:

- Web traffic cannot starve workers.
- Worker failures do not crash the agent API.
- Separate scaling per Graph workload.
- Built-in Service Bus trigger integration.
- Easier dead-letter and retry operations.
- Deployable from the same repository using shared Python packages.

Microsoft provides Python Service Bus-triggered Function examples using managed identity and private networking. [Python Service Bus Function sample](https://learn.microsoft.com/en-us/samples/azure-samples/functions-quickstart-python-azd-service-bus/service-bus-trigger-virtual-network/)

Do not run an unmanaged background `asyncio` task inside the FastAPI App Service process for durable Graph work. App Service can restart or scale down that process at any point.

## Authentication

There are two distinct modes.

### User-directed actions

Use delegated authorization/OBO when the action should be limited by the requesting user’s own Microsoft 365 permissions.

The worker message should contain an `authContextRef`, not an access token. Use a secure, distributed MSAL token cache and attempt silent acquisition in the worker. If interaction is required:

```text
status = needs_auth
```

Notify the user and resume after they authenticate.

Microsoft documents using MSAL’s token cache so background services can continue acting on behalf of users after the foreground app has exited. [MSAL background token-cache use](https://learn.microsoft.com/en-us/entra/identity-platform/msal-acquire-cache-tokens)

### Autonomous organizational actions

Use app-only permissions for explicitly authorized automation:

- Scheduled portfolio synchronization
- Approved SharePoint provisioning
- Controlled organization-wide data reads
- Agent automation that does not represent one particular user

Use separate identities/app registrations for different worker classes where practical. Grant the least privilege supported by each Graph endpoint. Application permissions are broader and require administrator consent. [Microsoft Graph permission modes](https://learn.microsoft.com/en-us/graph/permissions-overview)

Use managed identities for Cosmos, Service Bus, Key Vault, and other Azure resources. [Service Bus managed identity](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-managed-service-identity)

# 3. Designing the intelligence/portfolio stage

The key rule is:

> An executive question must not cause the agent to open 50 projects, read hundreds of Cosmos documents, and call Microsoft Graph repeatedly.

That would recreate your current latency and reliability problem at a much larger scale.

## Introduce an intelligence read model

Keep your current operational data:

```text
project_data
partitioned by projectId
```

Add a materialized project summary:

```json
{
  "id": "project-123",
  "projectId": "project-123",
  "type": "portfolio_project_summary",
  "name": "Contract Modernization",
  "department": "Legal",
  "sponsorId": "user-...",
  "ownerIds": ["..."],
  "status": "at_risk",
  "healthScore": 62,
  "scheduleVarianceDays": 18,
  "budgetStatus": "on_budget",
  "openRiskCount": 7,
  "criticalRiskCount": 2,
  "blockedMilestoneCount": 1,
  "nextMilestone": {
    "name": "Vendor approval",
    "dueDate": "..."
  },
  "topBlockers": [
    "External counsel review",
    "Data retention approval"
  ],
  "allowedGroupIds": ["entra-group-legal", "entra-group-executives"],
  "lastOperationalUpdateAt": "...",
  "lastExternalSyncAt": "...",
  "summaryVersion": 83
}
```

The portfolio projector subscribes to project events and updates these summaries incrementally.

Cosmos Change Feed materialized views are specifically intended to precompute alternative query representations for efficient access. [Cosmos materialized-view pattern](https://learn.microsoft.com/en-us/samples/azure-samples/cosmos-db-design-patterns/materialized-views/)

## Three intelligence data paths

### 1. Structured operational intelligence

For questions such as:

- How many projects are at risk?
- Which departments have overdue milestones?
- Show schedule variance by business unit.
- Which projects have not been updated in 30 days?

Query the materialized portfolio summaries.

Do not ask the LLM to calculate counts from raw documents. Your code calculates the metrics; the model explains them.

### 2. Semantic evidence retrieval

For questions such as:

- Why are the legal projects at risk?
- What themes are appearing across status reports?
- Which projects mention regulatory dependencies?
- Summarize the common causes of delay.

Index authorized summaries, risks, decisions, status reports, and selected SharePoint content in Azure AI Search.

Use:

- Hybrid keyword and vector retrieval
- `projectId`
- `department`
- `status`
- `date`
- `documentType`
- Entra user/group IDs for security trimming

Azure AI Search supports document-level security filtering based on user and group identities. [Azure AI Search document-level access](https://learn.microsoft.com/en-us/azure/search/search-document-level-access-overview)

### 3. Leadership analytics and trends

For:

- Quarterly trends
- Portfolio performance over time
- Power BI dashboards
- Historical schedule variance
- Department comparisons
- Executive reporting

Consider Microsoft Fabric Mirroring for Cosmos DB. It continuously copies Cosmos changes into OneLake without consuming transactional RUs and exposes T-SQL, Spark, and Power BI access. [Fabric Mirroring for Cosmos DB](https://learn.microsoft.com/en-us/fabric/mirroring/azure-cosmos-db)

Fabric is appropriate for analytical and historical queries. It should not be the synchronous dependency for an immediate task update.

## Portfolio-agent tools

Give the intelligence agent controlled tools rather than raw database access:

```text
query_portfolio_metrics(
    scope,
    filters,
    group_by,
    time_range
)

search_project_evidence(
    query,
    authorized_project_ids,
    departments,
    time_range
)

get_project_details(
    project_ids,
    fields
)

create_executive_brief(
    project_ids,
    audience,
    format
)
```

Example execution:

```text
VP: “Which projects in my organization are at risk, and why?”

1. Resolve VP’s authorized portfolio scope.
2. Query structured summaries for at-risk projects.
3. Select the most material projects.
4. Retrieve supporting risks/status evidence.
5. Generate a cited summary.
6. Include data freshness and missing-data warnings.
```

The response should distinguish:

- Observed facts
- Deterministic metrics
- Agent interpretations
- Missing or stale data

## Do not partition by tenant ID

You only have one tenant, so `tenantId` has a cardinality of one. It is a poor partition key.

For operational data, use:

```text
/projectId
```

or an equivalent synthetic project key.

Hundreds of users are not a meaningful Cosmos scaling challenge. The important dimensions are:

- Number of projects
- Size of each project
- Write concentration on a project
- Number of physical partitions
- Cross-project query frequency

Cross-partition queries are acceptable at modest data sizes, but cost and latency grow with physical-partition count. Materialized views avoid making every executive query a fan-out operation. [Cosmos cross-partition query guidance](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-query-container)

## Authorization is more important in phase 3

Leadership visibility does not necessarily mean unrestricted visibility.

Examples:

- A VP may see all projects in their business unit.
- Legal projects may have matter-specific confidentiality.
- HR projects may contain employee-sensitive information.
- Finance projects may include restricted budgets or forecasts.
- Executives may see health summaries but not underlying documents.

Enforce this in every layer:

```text
Cosmos query scope
Portfolio summaries
Azure AI Search filters
Fabric semantic model
Agent tool authorization
Web PubSub group membership
```

Never rely on the system prompt to prevent data disclosure.

## Supporting Legal, HR, Finance, and other departments

Use one project platform with a stable core model:

```text
Project
Task
Milestone
Risk
Issue
Decision
Dependency
Document
Operation
Status update
Membership
```

Add versioned department extensions:

```json
{
  "department": "legal",
  "domainSchema": "legal-project-v2",
  "domainData": {
    "matterType": "contract_review",
    "privilegeLevel": "restricted",
    "jurisdiction": ["CR", "US"],
    "counterparty": "..."
  }
}
```

Then introduce domain-specific capabilities:

- Vocabulary and retrieval sources
- Risk rules
- Approval policies
- Tool allowlists
- Output templates
- Data-retention constraints
- Department-specific Graph integrations

Avoid one enormous prompt that tries to make a single agent an expert in every department. Use MAF routing/workflows to select the appropriate domain policy and tool set.

High-impact executive actions should use plan/preview/approve:

```text
Executive asks for cross-project update
    → agent gathers data
    → agent creates draft
    → human reviews recipients and content
    → approval interrupt
    → Service Bus operation sends messages/emails
    → results are audited
```

MAF workflow interrupts fit this human-in-the-loop model.

# Recommended sequence

I would implement the next architecture work in this order:

1. Add the Cosmos `integration_operation` model and operation-status API.
2. Convert the worst Graph tool—probably SharePoint document or Planner creation—into an asynchronous tool.
3. Add the Cosmos outbox relay and Service Bus topic.
4. Deploy one isolated Graph worker.
5. Add Web PubSub operation-status cards to CopilotKit.
6. Add retry classification, DLQ management, and end-to-end correlation.
7. Convert the remaining Graph tools by workload.
8. Build the project-summary event projector.
9. Give the agent structured portfolio-query tools.
10. Add Azure AI Search for evidence retrieval.
11. Add Fabric Mirroring when leadership needs historical dashboards and broader analytics.

This solves the stakeholder-facing reliability problem first, while building the event stream and read models needed for the intelligence stage. PostgreSQL can remain a future option if the operational domain becomes strongly relational, but it is not required to solve either the Graph failures or the initial portfolio-intelligence requirements.

## User

Export this conversation as is into a .md file in my Downloads folder.

