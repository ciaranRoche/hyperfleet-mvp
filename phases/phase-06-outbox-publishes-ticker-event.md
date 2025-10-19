### Outbox Publishes Ticker Event

#### Overview

The Outbox Reconciler polls the database, finds the ticker-created event from Phase 5, publishes it to Cloud Pub/Sub, and marks it as published. This completes the second cycle of the outbox pattern.

This is the secret sauce, events are not driven by adapters themselves, they are drive my either : 
- 1 Create/Update on objects (If the intent of the object changes)
- 2 Time based decided by the ticker operator

```mermaid
sequenceDiagram
    autonumber

    participant DB as PostgreSQL
    participant Outbox as Outbox Reconciler
    participant Publisher as Message Publisher
    participant Broker as Cloud Pub/Sub

    Note over DB,Broker: Phase 6: Outbox Publishes Ticker Event

    loop Poll Outbox (every 100ms)
        Outbox->>DB: SELECT * FROM outbox<br/>WHERE published = false
        DB-->>Outbox: Events (including evt-def456)
    end

    Outbox->>Publisher: Publish(event)
    Note right of Outbox: Event:<br/>cls-123, gen 1<br/>(ticker-triggered)

    Publisher->>Publisher: Construct CloudEvents
    Publisher->>Broker: Publish to topic
    Broker-->>Publisher: ACK

    Outbox->>DB: UPDATE outbox<br/>SET published = true<br/>WHERE event_id = 'evt-def456'
    DB-->>Outbox: Updated
```