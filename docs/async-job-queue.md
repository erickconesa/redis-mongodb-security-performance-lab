# Redis Asynchronous Job Queue & Recovery

**Redis 7.4 | Docker | Background Workers | Queue Persistence | Fault Recovery**

## 1. Objective

This document describes the implementation and validation of an asynchronous job-processing system using Redis and a dedicated worker container.

The objective was to separate background processing from ordinary data-access operations while maintaining job state, persistence and controlled recovery after a worker interruption.

The work was completed during my second-year ASIR practical training.

## 2. Infrastructure Design

The laboratory used three independent Redis services, each configured for a different purpose.

| Service | Purpose | Memory policy |
|---|---|---|
| Redis cache | Temporary cached data | `allkeys-lru` |
| Redis sessions | Session storage | `noeviction` |
| Redis queue | Asynchronous job processing | `noeviction` |

Separating the services allowed different persistence and memory policies to be applied without mixing their workloads.

The queue service operated on a private Docker network without publishing port 6379 to the host.

## 3. Queue Configuration

The dedicated Redis queue instance used:

| Parameter | Configuration |
|---|---|
| Redis version | 7.4 |
| Data memory limit | 64 MiB |
| Docker memory limit | 128 MiB |
| CPU limit | 0.5 CPU |
| Eviction policy | `noeviction` |
| AOF persistence | Enabled |
| AOF synchronization | `everysec` |
| Persistent Docker volume | Enabled |
| Authentication | Required |

Authentication credentials were provided through protected files rather than being included in source code or public configuration.

The worker used a non-root identity, resource limits and restricted container privileges.

## 4. Job Processing Architecture

The following diagram illustrates the general processing sequence without exposing private application details.

```text
        Job Producer
             |
             v
    +------------------+
    | Redis Job Queue  |
    |------------------|
    | Pending Jobs     |
    | Processing Jobs  |
    | Job Metadata     |
    +------------------+
             |
             v
    +------------------+
    | Worker Container |
    |------------------|
    | Fetch Job        |
    | Process Job      |
    | Update Status    |
    +------------------+
             |
             v
    +------------------+
    | Job Result       |
    |------------------|
    | Status           |
    | Result / Error   |
    | Expiration       |
    +------------------+
```

The worker consumed jobs from the pending queue and maintained information about jobs currently being processed.

## 5. Job Lifecycle

Each job was assigned a unique hexadecimal identifier and associated metadata.

The laboratory implemented the following states:

- **Pending:** Job waiting to be processed.
- **Processing:** Job claimed by the worker.
- **Completed:** Job processed successfully.

The job record also supported result and error information.

### Enqueueing

Job metadata and insertion into the pending list were grouped into a Redis `MULTI/EXEC` transaction.

This reduced the risk of creating a job record without adding it to the queue.

### Processing

The worker used `BRPOPLPUSH` to move a job from the pending list to the processing list.

After successful processing, the worker updated the job metadata, assigned an expiration time and removed the job from the processing list.

Those completion operations were grouped into a Redis transaction.

## 6. Worker Recovery Test

A controlled recovery experiment was performed to validate how the system handled an interrupted job.

### Procedure

1. Stop the worker cleanly.
2. Create a synthetic job marked as processing.
3. Restart the worker.
4. Verify the recovery mechanism.
5. Confirm the final job state and queue contents.

### Observed Results

| Check | Result |
|---|---|
| Interrupted jobs detected | 1 |
| Jobs returned to the queue | 1 |
| Job completed after restart | Yes |
| Result stored | JSON |
| Pending queue after processing | 0 |
| Processing queue after processing | 0 |

The worker recovered the interrupted job and processed it successfully.

The test used a demonstration task that generated a message and a UTC timestamp. It did not process actual incident records.

## 7. Troubleshooting

During the initial implementation, the worker experienced repeated Redis read timeouts while waiting for jobs.

The issue was related to the interaction between the blocking queue operation and the configured socket timeout.

The configuration was adjusted from a 2-second socket timeout to a 10-second socket timeout.

After rebuilding and recreating the worker, the final verification showed:

- Worker running.
- Zero container restarts.
- No new timeout errors in the final check.
- Job-processing functionality maintained.

This troubleshooting process demonstrates the importance of distinguishing an expected blocking wait from a genuine connection failure.

## 8. Reliability and Limitations

The laboratory demonstrated basic job persistence and recovery with one worker.

However, the implementation does not provide exactly-once execution guarantees.

A job may be retried after an interruption, so tasks with external side effects would require idempotency controls.

Additional limitations include:

- No multi-worker concurrency testing.
- No distributed high availability.
- No physical host failure simulation.
- No comprehensive disaster-recovery test.
- No production workload or sustained throughput benchmark.

AOF persistence and a Docker volume improve recovery capabilities but do not eliminate all failure scenarios.

## 9. Skills Demonstrated

- Redis queue administration.
- Docker service isolation.
- Asynchronous processing concepts.
- Persistent job state management.
- Redis transactions.
- Background worker supervision.
- Controlled recovery testing.
- Troubleshooting connection timeouts.
- Technical analysis of reliability limitations.

---

**Author:** Erick Conesa Pérez  
**Training:** Higher Vocational Training in Network Computer Systems Administration (ASIR)
