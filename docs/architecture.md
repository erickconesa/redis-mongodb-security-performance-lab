# Redis & MongoDB Infrastructure Architecture

**Ubuntu Server | Docker | Redis 7.4 | Percona Server for MongoDB 8.0 | Security | Persistence**

## 1. Overview

This document describes the infrastructure architecture developed during an educational laboratory focused on in-memory storage and NoSQL databases.

The project was completed during my second-year practical training in Network Computer Systems Administration (ASIR).

The main objectives were to:

- Deploy isolated Redis and MongoDB-compatible database services.
- Separate workloads according to their storage and persistence requirements.
- Restrict access through authentication and role-based permissions.
- Validate data persistence and asynchronous task recovery.
- Analyze cache performance and native database auditing.

The architecture below is a simplified, sanitized representation of the laboratory.

## 2. Infrastructure Diagram

```text
                    Internal Service Layer
                              |
               Private Docker Backend Network
                              |
          +-------------------+-------------------+
          |                   |                   |
          v                   v                   v
    +------------+     +-------------+     +-------------+
    | Redis Cache|     |Redis Sessions|    | Redis Queue |
    |------------|     |-------------|     |-------------|
    | allkeys-lru|     | noeviction  |     | noeviction  |
    | TTL: 60 s  |     | AOF + RDB   |     | AOF         |
    +------------+     +-------------+     +------+------+
                                                  |
                                                  v
                                          +---------------+
                                          | Worker        |
                                          | Container     |
                                          |---------------|
                                          | Job Processing|
                                          | Recovery      |
                                          +-------+-------+
                                                  |
                                                  v
                                          +---------------+
                                          | Percona Server|
                                          | for MongoDB   |
                                          |---------------|
                                          | Authentication|
                                          | RBAC          |
                                          | Native Audit  |
                                          +-------+-------+
                                                  |
                                       +----------+----------+
                                       |                     |
                                       v                     v
                                Persistent Data       Audit JSON Logs
```

This diagram represents the main storage and processing relationships. It omits the private application's internal implementation and operational network details.

The worker-to-MongoDB relationship represents the archival of demonstration job results validated during the laboratory.

## 3. Docker Network Isolation

The Redis and MongoDB-compatible database services operated inside a private Docker backend network.

The laboratory verified that:

- Redis port 6379 was not published to the host.
- MongoDB port 27017 was not published to the host.
- Database communication used the internal network.
- Container resource limits were configured.
- Authentication was required for protected services.

Network isolation reduced exposure but did not replace authentication or access-control policies.

## 4. Redis Service Separation

Three Redis instances were used to separate workloads.

| Service | Purpose | Memory policy | Persistence |
|---|---|---|---|
| Cache | Temporary category data | `allkeys-lru` | Not established as a requirement |
| Sessions | Session storage | `noeviction` | AOF and RDB |
| Queue | Background job processing | `noeviction` | AOF |

Each Redis instance had a Docker memory limit of 128 MiB and a CPU limit of 0.5 CPU.

The Redis data-memory limit was configured to 64 MiB per instance.

### Cache

The cache stored a small dataset with a 60-second TTL.

Cache hits, misses, expiration and PostgreSQL fallback were tested.

The isolated LRU experiment demonstrated the eviction of 112 keys after inserting 200 synthetic keys under an 8 MiB Redis memory limit.

### Sessions

The session service used a separate Redis instance to prevent cache eviction policies from affecting session storage.

Session lifetime was configured to 30 minutes through the session-management layer.

AOF, RDB and a persistent Docker volume were used to support session persistence.

### Queue

The queue service stored job metadata and pending/processing lists.

A dedicated worker consumed tasks and updated their results.

A controlled restart test demonstrated recovery and completion of one interrupted synthetic task.

This did not establish exactly-once processing guarantees.

## 5. MongoDB-Compatible Database

The document database used **Percona Server for MongoDB 8.0**, selected for its native audit capabilities.

The deployment included:

- Private Docker networking.
- No published database port.
- Persistent storage.
- Authentication enabled.
- Restricted secret-file access.
- Container memory limit of 1 GiB.
- CPU limit of 1 CPU.
- WiredTiger cache configured to 256 MiB.

A controlled restart verified that two synthetic documents remained accessible after the database became ready.

## 6. Role-Based Security

The MongoDB-compatible database used separate read-only and read/write identities.

The laboratory verified:

- Authorized document reads.
- Authorized document writes.
- Rejection of writes attempted by a read-only user.

The denied operation produced MongoDB authorization error code 13.

These checks demonstrated the enforcement of different privileges.

## 7. Native Auditing

Native auditing was enabled using Percona Server for MongoDB.

The audit configuration recorded selected authorization events affecting the demonstration document collection.

Observed results included:

| Operation | Authorization | Result |
|---|---|---|
| `find` | Allowed | `0` |
| `insert` | Allowed | `0` |
| `insert` | Denied | `13` |

The audit records were checked against the expected user, command, collection and authorization result.

JSON audit logs were maintained on persistent storage with restricted permissions.

## 8. Operational Validation

The laboratory combined several validation techniques:

- Docker network inspection.
- Container resource inspection.
- Redis authentication checks.
- Redis memory and eviction statistics.
- Session TTL verification.
- Controlled queue recovery.
- Document persistence after database restart.
- MongoDB RBAC permission tests.
- Native audit log inspection.
- Redis versus PostgreSQL data-access benchmark.

The final Redis worker validation confirmed seven active containers in its deployment phase.

After completing MongoDB integration and auditing, the laboratory recorded eight containers operating together.

These are observations from different stages of the laboratory, not concurrent measurements of separate environments.

## 9. Performance Findings

The Redis cache was compared against PostgreSQL using 100 data-access measurements per system.

| Metric | PostgreSQL | Redis |
|---|---:|---:|
| Mean | 24.285 ms | 0.271 ms |
| Median | 20.478 ms | 0.193 ms |
| P95 | 38.970 ms | 0.673 ms |

The observed mean reduction was approximately **98.88%** for the measured data-access paths.

The PostgreSQL path included establishing a database connection, whereas the Redis path reused an existing client.

The benchmark did not measure complete HTTP page-loading time or MongoDB performance.

For additional details, see [Redis Memory Management & Performance Analysis](redis-performance.md).

## 10. Security and Reliability Limitations

This architecture was validated within an educational laboratory.

- No production workload was tested.
- No distributed high-availability configuration was implemented.
- No comprehensive disaster-recovery exercise was performed.
- No exactly-once queue-processing guarantee was established.
- Full TLS encryption between all database containers was not verified.
- Redis session persistence does not guarantee zero data loss.
- MongoDB auditing was restricted to selected operations and a specific collection.
- Real infrastructure addresses and operational credentials are intentionally excluded.

## 11. Skills Demonstrated

- Linux server administration.
- Docker networking and service isolation.
- Redis deployment and memory management.
- Cache and session architecture.
- Persistent queue management.
- Background worker supervision.
- MongoDB-compatible document storage.
- Authentication and role-based permissions.
- Native security auditing.
- Performance analysis.
- Incident troubleshooting.
- Technical infrastructure documentation.

---

**Author:** Erick Conesa Pérez  
**Training:** Higher Vocational Training in Network Computer Systems Administration (ASIR)
