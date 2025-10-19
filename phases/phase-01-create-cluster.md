### Phase 1 : User create cluster

#### Overview 

The user initiates cluster creation by sending a POST request to the HyperFleet API. The API uses a database transaction to atomically create both the cluster record and an event in the outbox table. This is the first use of the outbox pattern in the workflow


```mermaid
sequenceDiagram
    autonumber

    participant User
    participant API as HyperFleet API
    participant DB as PostgreSQL

    Note over User,DB: Phase 1: User Creates Cluster (Generation 1)

    User->>API: POST /api/hyperfleet/v1/clusters
    Note right of User: Request Body:<br/>{<br/>  name: "my-cluster",<br/>  provider: "gcp",<br/>  region: "us-east1"<br/>}

    API->>DB: BEGIN TRANSACTION

    API->>DB: INSERT INTO clusters
    Note right of DB: Cluster Record:<br/>id: "cls-123"<br/>generation: 1 (user intent)<br/>status: "Pending"<br/>adapterStatuses: {}

    API->>DB: INSERT INTO outbox
    Note right of DB: User-Triggered Event:<br/>event_type: "cluster.changed.v1"<br/>aggregate_id: "cls-123"<br/>payload: {id, generation: 1}<br/>published: false

    API->>DB: COMMIT TRANSACTION

    API-->>User: 201 Created
    Note right of User: Response:<br/>{<br/>  id: "cls-123",<br/>  status: "Pending",<br/>  generation: 1<br/>}
```