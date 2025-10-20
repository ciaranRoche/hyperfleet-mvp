
### Event Ticker Flow

**Purpose**: This diagram shows how the Event Ticker (Kubernetes operator) decides when to create reconciliation events for clusters.

**Pattern**: Continuous reconciliation loop triggered by CRD configuration

```mermaid
flowchart TD
    Start([Reconcile Starts]) --> FetchClusters["Fetch All Clusters<br>GET /clusters"]

    FetchClusters --> |Success| HasClusters{"Any<br>Clusters?"}
    FetchClusters --> |API Error| LogFetchError["Log Error"]
    LogFetchError --> Sleep

    HasClusters --> |No| Sleep["Sleep for Reconcile Interval<br>(defined in CRD spec)"]
    HasClusters --> |Yes| NextCluster["Get Next Cluster from List"]

    NextCluster --> CheckPhase{"Phase ==<br>Ready?"}

    CheckPhase --> |Yes| CheckReadyTime{"lastUpdated ><br>1 hour?"}
    CheckPhase --> |No| CheckPendingTime{"lastUpdated ><br>1 minute?"}

    CheckReadyTime --> |Yes| VerifyEvent["Verify Event Needed<br>GET /events?aggregateId=X&published=false"]
    CheckReadyTime --> |No| MoreClusters{"More<br>Clusters?"}

    CheckPendingTime --> |Yes| VerifyEvent
    CheckPendingTime --> |No| MoreClusters

    VerifyEvent --> |Success| HasUnpublished{"Unpublished<br>Event Exists?"}
    VerifyEvent --> |API Error| LogVerifyError["Log Error"]
    LogVerifyError --> MoreClusters

    HasUnpublished --> |Yes| MoreClusters
    HasUnpublished --> |No| CreateEvent["Create Event<br>POST /events"]

    CreateEvent --> |Success| LogSuccess["Log: Event Created"]
    CreateEvent --> |API Error| LogCreateError["Log: Event Creation Failed"]

    LogSuccess --> MoreClusters
    LogCreateError --> MoreClusters

    MoreClusters --> |Yes| NextCluster
    MoreClusters --> |No| Sleep

    Sleep --> Start

    %% Styling
    classDef startEnd fill:#4caf50,stroke:#2e7d32,stroke-width:2px,color:#fff
    classDef process fill:#2196f3,stroke:#1565c0,stroke-width:2px,color:#fff
    classDef decision fill:#ff9800,stroke:#ef6c00,stroke-width:2px,color:#fff
    classDef error fill:#f44336,stroke:#c62828,stroke-width:2px,color:#fff
    classDef success fill:#8bc34a,stroke:#558b2f,stroke-width:2px,color:#fff

    class Start,Sleep startEnd
    class FetchClusters,NextCluster,VerifyEvent,CreateEvent process
    class HasClusters,CheckPhase,CheckReadyTime,CheckPendingTime,HasUnpublished,MoreClusters decision
    class LogFetchError,LogVerifyError,LogCreateError error
    class LogSuccess success
```

## Key Patterns

### CRD-Driven Configuration

**Operator Config (CRD)**:
```yaml
apiVersion: hyperfleet.io/v1
kind: EventTickerConfig
metadata:
  name: cluster-event-ticker
spec:
  reconcileInterval: 30s  # How often to reconcile
  objectType: cluster      # What to reconcile
```

**Benefits**:
- Configuration separate from code
- Can update reconcile interval without redeploying
- Standard Kubernetes operator pattern

---

### Two-Tier Time Windows

**Non-Ready Clusters**:
- Check every reconcile cycle (e.g., 30s)
- Create event if `lastUpdated > 1 minute`
- **Purpose**: Drive active processing, prevent spam

**Ready Clusters**:
- Check every reconcile cycle
- Create event if `lastUpdated > 1 hour`
- **Purpose**: Health monitoring, detect drift

---

### Continuous Reconciliation Loop

```
Reconcile → Fetch → Evaluate → Create Events → Sleep → Reconcile → ...
```

**Benefits**:
- Self-healing (retries failures automatically)
- Stateless (no memory of previous cycles)
- Simple error recovery (just skip and retry)

---

### Event Spam Prevention

**Three Layers**:
1. **Time-based**: Don't create if recently updated (1 min for pending, 1 hour for ready)
2. **Outbox check**: Don't create if unpublished event already exists
3. **Buffer window**: Adapters have time to process and update status before next event

**Result**: Efficient event creation without overwhelming the system

---

### MVP Simplicity

**MVP Characteristics**:
- Fetch all clusters (no filtering)
- Sequential processing (one at a time)
- Single operator instance
- Simple error handling (log and continue)

**Post-MVP Enhancements**:
- Sharding: Multiple operators handling different cluster subsets
- Parallel processing: Process multiple clusters concurrently
- State-based filtering: Only fetch clusters in specific phases
- Advanced backoff: Dynamic intervals based on cluster age/state