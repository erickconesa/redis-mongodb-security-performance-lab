# Redis Memory Management & Performance Analysis

**Redis 7.4 | PostgreSQL 17 | LRU Eviction | Caching | Benchmarking**

## 1. Objective

This document describes two laboratory experiments performed during my second-year ASIR practical training:

- Validation of Redis memory management and automatic key eviction.
- Comparison of Redis and PostgreSQL data access times using a controlled benchmark.

The objective was to understand how Redis behaves under memory pressure and evaluate the benefit of caching frequently accessed information.

## 2. Test Environment

The laboratory used Redis 7.4 and PostgreSQL 17 running in Docker containers on Ubuntu Server.

The Redis caching service was configured with:

| Parameter | Configuration |
|---|---|
| Redis version | 7.4 |
| Maximum Redis memory | 64 MiB |
| Eviction policy | `allkeys-lru` |
| Docker memory limit | 128 MiB |
| Docker CPU limit | 0.5 CPU |
| Network | Private Docker network |
| Authentication | Required |

The cache and session storage used separate Redis instances with different memory policies.

## 3. Experiment 1 — LRU Eviction

### Objective

Verify that Redis automatically evicts keys when its configured memory limit is reached.

### Methodology

A disposable Redis container was launched independently from the operational cache and session services.

The test environment used:

- Redis 7.4 Alpine.
- `maxmemory 8mb`.
- `maxmemory-policy allkeys-lru`.
- Docker memory limit of 128 MiB.
- CPU limit of 0.5.
- No network connectivity.
- Persistence disabled.

A total of 200 synthetic keys, each containing approximately 64 KiB of data, were inserted.

Redis statistics were then inspected to verify the eviction counter and the number of remaining keys.

### Observed Results

| Metric | Result |
|---|---:|
| Keys inserted | 200 |
| Keys evicted | 112 |
| Keys remaining | 88 |
| Redis memory limit | 8 MiB |

The observed results confirmed that Redis automatically removed keys when the configured memory threshold was reached.

The disposable container was removed after the experiment. The operational Redis services remained running.

### Technical Interpretation

The `allkeys-lru` policy allows Redis to evict less recently used keys when memory pressure requires it.

This approach is appropriate for disposable cached information that can be retrieved again from its original data source.

It should not be applied indiscriminately to persistent sessions or asynchronous job queues.

The experiment validated eviction behavior under artificial memory pressure. It did not measure production throughput or latency.

## 4. Experiment 2 — Data Caching

### Objective

Reduce repeated PostgreSQL access for a small, frequently requested category dataset.

### Cache Design

The laboratory used a Redis cache containing five category records.

The caching strategy followed this logic:

1. Check whether the requested data exists in Redis.
2. On a cache hit, return the cached data.
3. On a cache miss, retrieve the data from PostgreSQL.
4. Store the result in Redis with a TTL of 60 seconds.
5. If Redis is unavailable, attempt to retrieve the data from PostgreSQL.

### Validation

The laboratory verified:

- Initial cache miss.
- Successful subsequent cache hit.
- TTL configured to 60 seconds.
- Expiration of a temporary test key.
- Fallback to PostgreSQL when the cache was simulated as unavailable.
- Equivalent category data returned by both storage systems.

The cache was limited to category identifiers and names. Customer records, incident details and user sessions were not included in this cache.

## 5. Experiment 3 — Redis vs PostgreSQL Benchmark

### Objective

Compare the time required to retrieve the same category dataset through PostgreSQL and Redis.

### Methodology

The benchmark performed:

- 100 measurements per data source.
- Five warm-up operations.
- Alternating measurement order.
- Retrieval of five equivalent category records.
- Validation of result equivalence.

The PostgreSQL path opened a new database connection and executed a SELECT query for each measurement.

The Redis path used an existing client and performed a GET operation followed by JSON deserialization.

**Important:** These paths have different connection overheads. The results measure the complete data-access operations used in this experiment, not an isolated comparison of database engines.

### Observed Results

| Metric | PostgreSQL | Redis |
|---|---:|---:|
| Mean | 24.285 ms | 0.271 ms |
| Median | 20.478 ms | 0.193 ms |
| 95th percentile | 38.970 ms | 0.673 ms |
| Measurements | 100 | 100 |

**Observed reduction in mean data-access time: 98.88%.**

This percentage refers only to the measured data-access paths.

It does not mean that the complete web application or every PostgreSQL query became 98.88% faster.

## 6. Technical Findings

### Redis is suitable for repeated access to small datasets

Caching can avoid repeated database connections and queries for information that changes infrequently.

### Cache expiration requires a trade-off

A 60-second TTL limits how long a cached category list is retained, but modifications may take up to 60 seconds to become visible.

Explicit cache invalidation was not implemented in this experiment.

### Memory policies depend on the workload

- **Disposable cache:** `allkeys-lru`.
- **Persistent sessions:** `noeviction`, with AOF/RDB persistence.
- **Asynchronous queue:** separate persistent Redis service with `noeviction`.

### Resilience requires fallback handling

The laboratory verified a PostgreSQL fallback for simulated Redis connection errors.

This does not guarantee recovery from every possible Redis or infrastructure failure.

## 7. Limitations

- The dataset contained only five categories.
- The eviction experiment used synthetic data.
- The benchmark was performed in a laboratory environment.
- No sustained production load was tested.
- PostgreSQL connection creation and Redis client reuse introduce different overheads.
- HTTP response times were not measured in this benchmark.
- Results represent the recorded test execution, not guaranteed performance under other conditions.
- No complete high-availability or disaster-recovery scenario was validated.

## 8. Skills Demonstrated

- Redis memory configuration.
- LRU eviction policies.
- Docker resource management.
- Cache-aside design.
- TTL management.
- Database fallback strategies.
- Benchmark methodology.
- Performance measurement and interpretation.
- Technical documentation of experimental limitations.

---

**Author:** Erick Conesa Pérez  
**Training:** Higher Vocational Training in Network Computer Systems Administration (ASIR)
