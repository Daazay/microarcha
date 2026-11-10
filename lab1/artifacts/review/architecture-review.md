# Architecture review and proposed simplification

**Status:** proposal for discussion. The existing catalog, ADR and Word report describe the earlier eight-service version. This review does not silently replace them.

[Detailed architecture diagram](proposed-architecture.svg) · [PNG](proposed-architecture.png) · [PlantUML source](proposed-architecture.puml)

## Main finding

The first version treated most capabilities as separately deployed services too early. The assignment requires clear ownership and independent deployment of system parts, but does not require a service for every entity, capability or code package. A workload of 10,000 RPS does not by itself determine service count, search technology or server count.

A package/module is code organized around a responsibility. A service is an independently deployed application with an API or event contract and an owner. A database is persistent storage owned by a service. A worker is an application that primarily consumes events or executes background jobs. These are different concepts.

Start with five application services: Schedule, Sales, Payment, Notification and Audit. Notification and Audit primarily run as workers. Internal modules are not separate network applications. This is still microservices: business data and deployments are separated between services.

## Review of every original service

| Original service | Proposed placement | Reason and cost of the choice |
|---|---|---|
| Schedule | Separate Schedule service | Owns the operator's timetable, stations, trips and physical train layout. It changes independently of ticket transactions. |
| Search | journey_search module in Schedule | The current task is structured route/date lookup over Schedule's data. A separate copy and index introduce freshness and recovery problems without measured benefit. Schedule and search initially scale together. |
| Inventory | inventory module in Sales | Seat allocation and issuing a ticket can commit in one local transaction. Inventory still has a strict internal API and owns its tables. It cannot independently deploy or scale until extracted. |
| Booking | Main module in Sales | Owns orders, tickets, fares and the purchase/refund workflow. The application also hosts the inventory and passenger modules. |
| Passenger | passengers module in Sales | Saved traveler profiles mainly support ticket purchases in this scope. Keep a distinct module and access policy. It is not the authentication system. Extraction becomes useful if profiles serve several independent products or require a separate security/team boundary. |
| Payment | Separate Payment service | Isolates provider integration, money-operation idempotency, signed callbacks and financial reconciliation. A distributed purchase/payment workflow is still required. |
| Notification | Separate Notification service/worker | Email/SMS failures and provider limits must not delay issuing a ticket. This could be an internal worker in a smaller system, but separate deployment is useful here. |
| Audit | Separate Audit service/worker and history store | The assignment explicitly requires separate history storage. A separate worker is a design choice, not a literal requirement, and makes archival access and retention independent. |

The combined Sales context describes ticket selling; inventory is a consistency-critical submodule and passenger profiles a supporting submodule. Separate contexts can also live in one deployment, but must retain model ownership. Do not claim that every bounded context necessarily equals one service.

## What Search actually does

Example: a traveler asks for trains from station A to station C on 10 November. Search finds trips that stop at A before C, with departure from A on the requested local date. It returns trip IDs, times and route information. It does not reserve a seat or prove that one is still free.

This can be implemented in Schedule using PostgreSQL tables for trips and ordered stops, suitable indexes, and optionally a derived SQL read model. The exact physical schema and query plan must be measured against the real workload. Full-text search is not necessary merely to filter stations, dates and times.

Seat availability is a different query handled by Sales.inventory. A displayed available seat can become occupied before the user clicks. Only an atomic hold establishes the temporary reservation. Search can later show indicative availability using a derived view, but that is not a booking guarantee.

Extract Search later if evidence shows that its traffic, query complexity, relevance features, data sources or independent release/team needs differ materially from timetable management. OpenSearch is then one candidate, not an inevitable dependency. A read replica or cache could be sufficient without a new microservice.

## Concrete code organization

The following is a proposed layout, not implemented application code:

```text
services/
  schedule/
    api/
    schedule/          # trips, stops, routes, stations
    journey_search/    # route/date queries
    rolling_stock/     # physical trains, carriages and seats
    persistence/       # repositories and migrations
    messaging/         # outbox publisher
  sales/
    api/
    booking/           # orders, tickets, fares and return rules
    inventory/         # segment occupancy, holds and allocations
    passengers/        # traveler profiles and document access
    checkout/          # durable payment saga and timers
    persistence/       # sales_db migrations and transaction boundary
    messaging/         # consumers, inbox and outbox
  payment/
    api/               # payment commands and signed webhook endpoint
    payments/
    refunds/
    provider_adapter/
    reconciliation/
    persistence/
    messaging/
  notification/
    event_consumer/
    templates/
    email_adapter/
    sms_adapter/
    retry_scheduler/
    persistence/       # jobs, delivery attempts, deduplication
    messaging/
  audit/
    event_consumer/
    history_writer/
    history_query/     # restricted staff API
    persistence/
infra/
  gateway/
  databases/
  kafka/
  observability/
  deployment/
contracts/
  api/                 # versioned contracts, not shared domain models
  events/
```

Within Sales, modules use explicit in-process interfaces rather than HTTP. They own their tables; arbitrary cross-module table access is avoided. Checkout coordinates a local transaction through those module interfaces when confirming an allocation and issuing a ticket. Cross-service access always uses contracts, never another service's tables.

## Databases and broker

| Resource | Owner and contents |
|---|---|
| schedule_db | Schedule: trips, stops, rolling stock, search SQL indexes/read model, outbox. |
| sales_db | Sales: booking, ticket, fare, occupancy, hold, allocation, profile, saga, inbox and outbox tables. |
| payment_db | Payment: monetary operations, attempts, refunds, deduplicated provider callbacks and outbox. |
| notification_db | Notification: jobs, attempts, templates, deduplication and delivery-result outbox. |
| audit_db | Audit: durable business-operation history with unique event IDs and restricted modification rights. |
| Kafka cluster | Shared transport for events; it does not own tickets and does not replace Audit's durable history store. |

For a lab, logical databases may share a PostgreSQL instance with separate credentials and no cross-database queries. This is not evidence of production high availability: critical production databases need replication, failure isolation and verified recovery. Audit remains a separate logical history database; its physical deployment can be isolated as required.

Kafka is not mandated by the assignment. It is retained from the original design to distribute events to several consumers and allow replay. RabbitMQ would also be a reasonable alternative for queue-oriented processing. We should use one broker technology, not introduce both without a need. A durable outbox with polling can support a smaller first implementation, but delivery, fan-out and retries would then be our responsibility.

The outbox is a table in each producer's own database, not a separate database or mandatory standalone microservice. A background publisher in that service sends committed rows to Kafka. Consumers deduplicate EventId. The selected delivery model is at least once.

| Event stream | Producer | Consumers |
|---|---|---|
| schedule.events | Schedule | Sales updates trip-layout/version projection and handles cancellations; Audit records changes. |
| sales.events | Sales | Notification consumes relevant ticket events; Audit consumes ticket, inventory and profile-operation facts. |
| payment.events | Payment | Sales advances the durable workflow; Audit records financial outcomes. |
| notification.events | Notification | Audit records delivery outcomes. |

Consumers have independent consumer groups, so notification delivery does not consume Audit's copy. Persist consumer state before acknowledging progress. Bound retries; send poison events to a consumer-specific failure queue/topic for investigation. Do not assume global ordering across topics.

In this proposal Notification receives the minimal delivery contact in a restricted ticket event, rather than synchronously querying profiles. Audit persists only the approved redacted subset. Passenger documents and card details are excluded from general event streams. Topic access and retention must account for contact data.

## Purchase in the proposed architecture

1. Client searches through Gateway → Schedule.journey_search → schedule_db.
2. Client reads availability through Gateway → Sales.inventory → sales_db.
3. Sales validates the trip, loads the passenger profile, fixes the fare and atomically creates the order plus a temporary hold.
4. Sales requests an idempotent payment from Payment. The user completes hosted checkout with the provider.
5. Payment verifies the provider result, commits the payment outcome plus its outbox event, then publishes to Kafka.
6. Sales consumes the payment event. In one local transaction it checks the still-valid hold, confirms the allocation, issues the ticket and records the outgoing event.
7. Notification delivers confirmation; Audit stores the history independently.

If the hold has expired and the seat is no longer available, Sales initiates a compensating refund. Payment remains external to the local database transaction. Merging Inventory removes one distributed boundary; it does not eliminate all failure handling.

For passenger refunds, Sales locks the ticket into a pending state, Payment returns money, then Sales atomically marks the ticket refunded and releases the allocation after confirmation. Unknown provider outcomes require reconciliation, not a blind retry with a new operation key.

## Other infrastructure and ownership

- Gateway is an edge application/reverse proxy for routing, TLS and token checks. It has no business database and does not coordinate purchases. Authentication is distinct from passenger profiles; services also enforce authorization.
- Identity, payment, and email/SMS providers are external dependencies in this design. Their internal databases are outside our ownership.
- DevOps is a team, not a service. Its responsibilities include CI/CD, deployment, secrets, backups, metrics, logs and tracing. Business teams still own data correctness.
- Logs/metrics/traces are operational telemetry. Audit is business history. They serve different purposes.
- Redis, OpenSearch and Kubernetes are optional choices. None follows automatically from the phrase microservices or 10,000 RPS. Capacity and high-availability claims require measurements and an explicit deployment design.

## References supporting the design approach

The recommendation to begin with coarser boundaries and reconsider size when services interact heavily is consistent with [Microsoft's microservice-boundary guidance](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/microservice-boundaries).

Reliable publication after a local transaction follows the [transactional outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html). The specific five-service grouping above is our design judgment for this lab, not a requirement imposed by either source.

## Impact on the existing submission

If this proposal becomes the chosen architecture, update the Context Map, service catalog, ADR, C4 diagrams and Word report together. In particular, replace the Booking–Inventory saga boundary with a local Sales transaction, move search APIs to Schedule, and document the new profile/contact ownership. Until then, the review and diagrams in this folder are explicitly proposed alternatives to the existing report.
