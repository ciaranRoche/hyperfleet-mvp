### Validation Adapater Acts

#### Overview

The Validation Adapter evaluates its preconditions, determines it should **ACT**, creates a Kubernetes Job to validate GCP prerequisites, and reports status back to the API. The other adapters (DNS, Placement) evaluate their preconditions and **IGNORE** the event because their dependencies aren't ready yet.

```mermaid
sequenceDiagram
    autonumber

    participant ValAdapter as Validation Adapter
    participant DnsAdapter as DNS Adapter
    participant PlaceAdapter as Placement Adapter
    participant API as HyperFleet API
    participant K8s as Kubernetes
    participant ValJob as Validation Job
    participant DB as PostgreSQL

    Note over ValAdapter,DB: Phase 4: Validation Adapter Acts

    par All Adapters Fetch Cluster
        ValAdapter->>API: GET /api/hyperfleet/v1/clusters/cls-123
        API->>DB: SELECT cluster
        DB-->>API: Cluster data
        API-->>ValAdapter: Cluster (gen 1, status: Pending)
    and
        DnsAdapter->>API: GET /api/hyperfleet/v1/clusters/cls-123
        API-->>DnsAdapter: Cluster (gen 1, status: Pending)
    and
        PlaceAdapter->>API: GET /api/hyperfleet/v1/clusters/cls-123
        API-->>PlaceAdapter: Cluster (gen 1, status: Pending)
    end

    par All Adapters Evaluate Preconditions
        ValAdapter->>ValAdapter: Evaluate preconditions
        Note right of ValAdapter: ✓ observedGen (0) < gen (1)<br/>DECISION: ACT
    and
        DnsAdapter->>DnsAdapter: Evaluate preconditions
        Note right of DnsAdapter: ✗ validation.ready = false<br/>DECISION: IGNORE
    and
        PlaceAdapter->>PlaceAdapter: Evaluate preconditions
        Note right of PlaceAdapter: ✗ validation.ready = false<br/>✗ dns.ready = false<br/>DECISION: IGNORE
    end

    Note over DnsAdapter,PlaceAdapter: DNS and Placement ACK event<br/>and wait for next event

    Note over ValAdapter,ValJob: Validation Adapter Acts

    ValAdapter->>K8s: Create Job<br/>hyperfleet-validate-cls-123-gen1
    Note right of ValAdapter: Job spec includes:<br/>- Cluster ID<br/>- Generation<br/>- Provider: GCP

    ValAdapter->>API: POST /api/hyperfleet/v1/adapter-status
    Note right of ValAdapter: Body:<br/>{<br/>  status: "InProgress",<br/>  adapterType: "validation"<br/>}
    API->>DB: UPDATE adapter_statuses
    DB-->>API: Updated
    API-->>ValAdapter: 200 OK

    K8s->>ValJob: Start Pod
    Note right of ValJob: Validation Job runs:<br/>- Check Cloud DNS zone<br/>- Check GCS bucket<br/>- Check GCP quotas<br/>- Check credentials

    ValJob->>ValJob: Validate GCP prerequisites
    Note right of ValJob: All checks pass:<br/>✓ Cloud DNS zone exists<br/>✓ GCS bucket configured<br/>✓ Quotas sufficient<br/>✓ Credentials valid

    ValJob-->>K8s: Exit 0 (success)

    ValAdapter->>API: POST /api/hyperfleet/v1/adapter-status
    Note right of ValAdapter: Body:<br/>{<br/>  status: "Ready",<br/>  adapterType: "validation",<br/>  observedGeneration: 1<br/>}

    API->>DB: UPDATE adapter_statuses<br/>SET validation.ready = true,<br/>validation.observedGen = 1
    Note right of DB: Status updated<br/>Generation stays 1<br/>(no spec change)

    DB-->>API: Updated
    API-->>ValAdapter: 200 OK

    Note right of ValAdapter: Validation complete!<br/>ACK Cloud Pub/Sub message
```