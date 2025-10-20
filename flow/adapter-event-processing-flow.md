### Adapter Event Processing Flow

Purpose: This diagram shows the complete flow for how a HyperFleet adapter processes an event, from receiving the message through final status reporting.

Pattern: Event-driven, precondition-based job orchestration

```mermaid
flowchart TD
    Start([Adapter Receives Event]) --> Extract["Extract Event Data<br>clusterId, generation"]

    Extract --> FetchAPI["Fetch Cluster Object<br>GET /api/clusters/id"]

    FetchAPI --> |Success| EvalPrecond{"Evaluate<br>Preconditions"}
    FetchAPI --> |API Error| LogAPIError["Log Error and Exit"]
    LogAPIError --> End1([Exit - Retry via Next Event])

    EvalPrecond --> |Fail| ReportPrecondFail["Report Conditions:<br>Applied=False reason=PreconditionsNotMet<br>Available=False<br>Health=True"]
    ReportPrecondFail --> ACKSkip[ACK Message]
    ACKSkip --> End2([Exit - Preconditions Not Met])

    EvalPrecond --> |Pass| CheckJobExists{"Does Job<br>Exist?"}

    CheckJobExists --> |Yes| CheckJobStatus{"Job<br>Status?"}

    CheckJobStatus --> |Succeeded| ReportSuccess["Report Conditions:<br>Applied=True reason=JobLaunched<br>Available=True reason=JobSucceeded<br>Health=True reason=AllChecksPassed"]

    CheckJobStatus --> |Failed| ReportJobFailed["Report Conditions:<br>Applied=True reason=JobLaunched<br>Available=False reason=JobFailed<br>Health=True"]

    CheckJobStatus --> |Running| ReportRunning["Report Conditions:<br>Applied=True reason=JobLaunched<br>Available=Unknown reason=JobInProgress<br>Health=True"]

    CheckJobExists --> |No| CreateJob["Create Kubernetes Job<br>Name: adapter-clusterId-genN"]

    CreateJob --> |Success| ReportCreated["Report Conditions:<br>Applied=True reason=JobLaunched<br>Available=Unknown reason=JobInProgress<br>Health=True"]

    CreateJob --> |Failure| ReportCreateFailed["Report Conditions:<br>Applied=False reason=JobCreationFailed<br>Available=False<br>Health=False reason=CreateError"]

    ReportSuccess --> ACKSuccess[ACK Message]
    ReportJobFailed --> ACKFailed[ACK Message]
    ReportRunning --> ACKRunning[ACK Message]
    ReportCreated --> ACKCreated[ACK Message]
    ReportCreateFailed --> ACKCreateError[ACK Message]

    ACKSuccess --> End3([Exit - Success])
    ACKFailed --> End4([Exit - Job Failed])
    ACKRunning --> End5([Exit - Job Running])
    ACKCreated --> End6([Exit - Job Created])
    ACKCreateError --> End7([Exit - Create Failed])

    %% Styling
    classDef startEnd fill:#4caf50,stroke:#2e7d32,stroke-width:2px,color:#fff
    classDef process fill:#2196f3,stroke:#1565c0,stroke-width:2px,color:#fff
    classDef decision fill:#ff9800,stroke:#ef6c00,stroke-width:2px,color:#fff
    classDef error fill:#f44336,stroke:#c62828,stroke-width:2px,color:#fff
    classDef success fill:#8bc34a,stroke:#558b2f,stroke-width:2px,color:#fff
    classDef skip fill:#9e9e9e,stroke:#616161,stroke-width:2px,color:#fff

    class Start,End1,End2,End3,End4,End5,End6,End7 startEnd
    class Extract,FetchAPI process
    class EvalPrecond,CheckJobExists,CheckJobStatus decision
    class LogAPIError,ReportCreateFailed error
    class ReportSuccess,ReportCreated,ReportJobFailed,ReportRunning,ReportPrecondFail,ACKSuccess,ACKFailed,ACKRunning,ACKCreated,ACKCreateError,ACKSkip success
```

### Key Patterns

#### Stateless Reconciliation
**Purpose**: Each event is a snapshot check, not a long-running process

**Pattern**:
```
1. Receive event
2. Fetch cluster
3. Evaluate preconditions
4. Check job exists
   - Yes → Report current job status
   - No → Create job and report
5. ACK message
6. Exit

Next event (from ticker):
7. Receive event again
8. Check job status again
9. Report updated status
```

**Benefits**:
- Simple, predictable code
- Fast event processing (~100ms)
- No resource leaks (no long-running watches)
- Easy to reason about and test

---

#### Idempotency
**Purpose**: Safe to process same event multiple times

**Mechanisms**:
1. **observedGeneration check**: Skip if already processed this generation
2. **Job name with generation**: Kubernetes prevents duplicate job creation
3. **Check job first**: If job exists, just report its status

**Example**:
```go
// Event 1: Create job, report Applied=True/Available=Unknown
// Event 2 (30s later): Job still running, report Applied=True/Available=Unknown
// Event 3 (60s later): Job succeeded, report Applied=True/Available=True
// Event 4 (90s later): observedGen=1, cluster.gen=1, SKIP (already processed)
```

---

#### Conditions vs Ready


**Pattern** (current):
```json
{
  "conditions": [
    {"type": "Applied", "status": "True"},
    {"type": "Available", "status": "Unknown"},
    {"type": "Health", "status": "True"}
  ]
}
```

**Why conditions?**
- **Granular**: Know exactly what succeeded/failed
- **Kubernetes-native**: Matches K8s condition pattern
- **Machine-readable**: Easy to query and alert on

---

#### Message Acknowledgment
**ACK**: Removes message from queue, won't be redelivered

**When to ACK (Always!)**:
- ✅ Preconditions not met (report status)
- ✅ Job checked and status reported
- ✅ Job created and status reported
- ✅ Job creation failed and status reported
- ✅ API fetch fails (log error, exit)

**Rule**: Always ACK the message! No NACKs - the ticker will create new events for retries.