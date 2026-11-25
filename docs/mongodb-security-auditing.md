# MongoDB Security, RBAC & Native Auditing

**Percona Server for MongoDB 8.0 | Docker | Authentication | RBAC | Security Auditing**

## 1. Objective

This document describes the deployment, security hardening and auditing of a MongoDB-compatible document database during my second-year ASIR practical training.

The laboratory used **Percona Server for MongoDB 8.0**, selected for its native auditing capabilities.

The work covered three technical controls:

- **CT-07:** Deployment, isolation and data persistence.
- **CT-08:** Authentication and role-based access control.
- **CT-09:** Native database auditing and access traceability.

The objective was to verify not only that database permissions were configured, but also that authorized and unauthorized operations produced the expected results.

## 2. Database Deployment

The database was deployed as a Docker container using Percona Server for MongoDB 8.0.

The deployment included:

- Internal Docker network.
- No publication of port 27017 to the host.
- Persistent Docker volume.
- Database authentication enabled.
- Protected configuration and secret files.
- Container resource limits.
- Restricted container privileges.

### Resource Configuration

| Parameter | Laboratory configuration |
|---|---|
| Database distribution | Percona Server for MongoDB 8.0 |
| Docker memory limit | 1 GiB |
| CPU limit | 1 CPU |
| Process limit | 160 PIDs |
| WiredTiger cache | 256 MiB |
| Persistent data directory | `/data/db` |
| Authentication | Enabled |

These values were selected for the educational laboratory and are not universal production recommendations.

## 3. Data Persistence

A document collection was created using synthetic records.

Two demonstration documents were inserted and retrieved successfully.

The database container was then restarted.

After the database completed its startup, both documents were retrieved again.

**Observed result:** The documents remained available after the controlled container restart.

This test verified basic persistence through a restart. It did not simulate physical host failure or the loss of the persistent volume.

## 4. Authentication

Database authentication was enabled.

The laboratory verified:

- Successful connection using authorized credentials.
- Rejection of an unauthorized administrative operation.
- Credentials loaded from protected files.
- No database credentials stored directly in public source code.

An anonymous administrative query returned an `Unauthorized` response with error code 13.

## 5. Role-Based Access Control

The database used separate identities for read-only and read/write operations.

| Role | Authorized operations |
|---|---|
| `read` | Query documents |
| `readWrite` | Read and modify documents |

### Permission Validation

Three tests were performed against a demonstration collection.

**Test 1 — Unauthorized write**

A user with the `read` role attempted an `insertOne` operation.

MongoDB rejected the operation:

```text
Unauthorized (code 13)
```

**Test 2 — Authorized write**

A user with the `readWrite` role inserted and retrieved a demonstration document.

The operation completed successfully.

**Test 3 — Authorized read**

The read-only user queried the same document with `findOne`.

The query succeeded without granting write privileges.

### Technical Finding

The tests demonstrated effective separation between read-only access and document modification.

This reduces the permissions required by components that only need to consult information.

## 6. Native Security Auditing

Native auditing was enabled using Percona Server for MongoDB.

Auditing was configured for a selected demonstration collection.

The configuration included:

- JSON audit output.
- Persistent audit log storage.
- Restricted log-file permissions.
- Authorization-check events (`authCheck`).
- Collection-specific filtering.
- Recording of successful authorization checks.

The audit configuration was validated before activation.

### Audit Validation Results

| Operation | User permission | Audit result |
|---|---|---|
| `find` | Read/write | `result: 0` |
| `insert` | Read/write | `result: 0` |
| `insert` | Read-only | `result: 13` |

Result `0` represented an authorized operation.

Result `13` represented an operation rejected by the database authorization system.

### Audit Traceability

The recorded events were checked to confirm that they identified the expected:

- Database user.
- Operation.
- Target collection.
- Authorization result.

The unauthorized write attempt was both rejected by RBAC and recorded in the native audit log.

This demonstrates the difference between **access control**, which determines whether an operation is permitted, and **auditing**, which records relevant authorization activity.

## 7. Backup and Validation

Before enabling native auditing, a compressed backup of the demonstration document collection was created.

The backup was checked using MongoDB restoration tooling in dry-run mode.

The original database configuration was also backed up.

**Limitation:** A dry-run verification does not prove that a complete restoration would succeed in a separate recovery environment.

After auditing was enabled, database access, authorized operations and service availability were checked again.

The temporary audit-test document was removed, leaving the original demonstration records intact.

## 8. Security Architecture

```text
           Authorized Database Clients
                       |
                       v
             Private Docker Network
                       |
                       v
          +--------------------------+
          | Percona Server for       |
          | MongoDB 8.0              |
          |--------------------------|
          | Authentication           |
          | RBAC: read / readWrite   |
          | Native Audit Logging     |
          +--------------------------+
                  |           |
                  v           v
          Persistent      Persistent
          Data Volume     Audit Logs
```

The database did not publish its port to the host.

Network isolation, authentication, role permissions and audit logging provided separate layers of protection.

The architecture does not demonstrate TLS encryption between containers or distributed high availability.

## 9. Technical Limitations

- Single-instance database deployment.
- No replica set or automatic failover.
- No production workload testing.
- No complete disaster-recovery exercise.
- No validated TLS encryption between database containers.
- Audit logs were stored locally rather than forwarded to a centralized logging platform.
- The tests demonstrate the selected operations and permissions, not every possible security scenario.

## 10. Skills Demonstrated

- MongoDB-compatible database administration.
- Docker deployment and private networking.
- Database persistence.
- Authentication configuration.
- Role-based access control.
- Least-privilege security.
- Native audit logging.
- Investigation of authorized and denied operations.
- Backup verification.
- Security validation and technical documentation.

---

**Author:** Erick Conesa Pérez  
**Training:** Higher Vocational Training in Network Computer Systems Administration (ASIR)
