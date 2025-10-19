### Outbox Publishes Events

#### Overview

The Outbox Reconciler polls the database for unpublished events, finds the event created in phase 1, publishes it to the Pub/Sub via the Message Publisher, and marks it as published. This completes the outbox pattern, Note the outbox reconciler is always reconciling on the event table looking for unpublished events

```mermaid
sequenceDiagram
    autonumber

    participant DB as PostgreSQL
    participant Outbox as Outbox Reconciler
    participant Publisher as Message Publisher
    participant Broker as Cloud Pub/Sub

    Note over DB,Broker: Phase 2: Outbox Publishes User Event

    loop Poll Outbox (every 100ms)
        Outbox->>DB: SELECT * FROM outbox<br/>WHERE published = false<br/>ORDER BY created_at ASC<br/>LIMIT 100
        DB-->>Outbox: Events (including cls-123 event)
    end

    Note right of Outbox: Found 1 unpublished event

    Outbox->>Publisher: Publish(event)
    Note right of Outbox: Event:<br/>type: cluster.changed.v1<br/>id: cls-123<br/>generation: 1

    Publisher->>Publisher: Construct CloudEvents envelope
    Note right of Publisher: Add metadata:<br/>- ce-id<br/>- ce-source<br/>- ce-type<br/>- ce-specversion

    Publisher->>Broker: Publish to topic<br/>hyperfleet.clusters.changed.v1
    Note right of Publisher: Message:<br/>CloudEvents JSON with<br/>anemic payload

    Broker-->>Publisher: ACK (message accepted)

    Publisher-->>Outbox: Success

    Outbox->>DB: UPDATE outbox<br/>SET published = true,<br/>published_at = NOW()<br/>WHERE event_id = 'evt-abc123'

    DB-->>Outbox: Updated

    Note right of Outbox: Event successfully published
```