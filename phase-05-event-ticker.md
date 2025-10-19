### Event Ticker Decides to Create Reconciliation Event

#### Overview
The **Event Ticker** (K8s operator), lists all clusters from the API, analyzes cluster state, decides a reconciliation event is needed, and calls `POST /api/hyperfleet/v1/events`. The API inserts the event into the outbox table. This is our **three separate concerns** : Ticker (decide) → API (store) → Outbox Reconciler (publish).

```mermaid
sequenceDiagram
    autonumber

    participant Ticker as Event Ticker<br/>(K8s Operator)
    participant API as HyperFleet API
    participant DB as PostgreSQL

    Note over Ticker,DB: Phase 5: Event Ticker Decides to Create Event

    Note right of Ticker: Ticker wakes up<br/>(1s after validation complete)

    loop Ticker Loop (exponential backoff)
        Ticker->>API: GET /api/hyperfleet/v1/clusters?status!=Ready
        API->>DB: SELECT clusters<br/>WHERE status != 'Ready'
        DB-->>API: Pending clusters
        API-->>Ticker: Cluster list
        Note right of Ticker: Response:<br/>[{<br/>  id: "cls-123",<br/>  generation: 1,<br/>  status: "Pending",<br/>  adapterStatuses: {<br/>    validation: {ready: true, observedGen: 1}<br/>  },<br/>  metadata: {<br/>    lastEventAt: "2025-10-19T10:01:30Z"<br/>  }<br/>}]
    end

    Ticker->>Ticker: Analyze cluster
    Note right of Ticker: Decision logic:<br/>• status != "Validated" ✓<br/>• validation.ready = true ✓<br/>• dns.ready = false ✓<br/>• lastEventAt > 1s ago ✓<br/>• DECISION: Event needed

    Ticker->>API: POST /api/hyperfleet/v1/events
    Note right of Ticker: Request Body:<br/>{<br/>  aggregate_type: "cluster",<br/>  aggregate_id: "cls-123",<br/>  event_type: "cluster.changed.v1"<br/>}

    Note over API,DB: API Uses Outbox Pattern

    API->>DB: SELECT cluster<br/>WHERE id = 'cls-123'
    DB-->>API: Cluster (generation: 1)

    API->>DB: INSERT INTO outbox
    Note right of DB: Ticker-Triggered Event:<br/>event_type: "cluster.changed.v1"<br/>aggregate_id: "cls-123"<br/>payload: {id, generation: 1}<br/>published: false

    DB-->>API: Inserted

    API-->>Ticker: 202 Accepted
    Note right of API: Response:<br/>{<br/>  event_id: "evt-def456",<br/>  status: "accepted",<br/>  message: "Event inserted into outbox"<br/>}

    Note right of Ticker: Event creation requested<br/>Ticker's job done<br/>Outbox reconciler will publish
```