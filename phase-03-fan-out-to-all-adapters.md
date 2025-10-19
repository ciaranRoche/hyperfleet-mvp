### Fan out to all adapters

#### Overview 
Cloud Pub/Sub receives the published event and **fans it out** to all adapter subscriptions simultaneously. Each adapter (Validation, DNS, Placement) receives a copy of the event independently. This is the **Pub/Sub fan-out pattern** in action.

```mermaid
sequenceDiagram
    autonumber

    participant Broker as Cloud Pub/Sub<br/>Topic
    participant ValSub as Validation<br/>Subscription
    participant DnsSub as DNS<br/>Subscription
    participant PlaceSub as Placement<br/>Subscription
    participant ValAdapter as Validation<br/>Adapter
    participant DnsAdapter as DNS<br/>Adapter
    participant PlaceAdapter as Placement<br/>Adapter

    Note over Broker,PlaceAdapter: Phase 3: Fan-Out to All Adapters

    Note over Broker: Event in topic:<br/>cluster.changed.v1<br/>cls-123, gen 1

    par Fan-Out Delivery (Simultaneous)
        Broker->>ValSub: Push event
        ValSub->>ValAdapter: cluster.changed.v1
        Note right of ValAdapter: Received gen 1 event
    and
        Broker->>DnsSub: Push event
        DnsSub->>DnsAdapter: cluster.changed.v1
        Note right of DnsAdapter: Received gen 1 event
    and
        Broker->>PlaceSub: Push event
        PlaceSub->>PlaceAdapter: cluster.changed.v1
        Note right of PlaceAdapter: Received gen 1 event
    end

    Note over ValAdapter,PlaceAdapter: All adapters received event<br/>Each will now evaluate preconditions
```