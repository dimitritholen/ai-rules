# Qdrant 1.15 Production Rules

## 1. Architecture & Deployment

### Storage Infrastructure
- **MUST** use block-level storage with POSIX-compatible filesystems (local disk, iSCSI); Qdrant requires direct block access for write-ahead logs and memory-mapped files.
- **NEVER** use NFS or object storage (S3) for Qdrant data; these lack the required POSIX semantics and cause WAL file lock errors in distributed deployments.
- **MUST** use SSD or NVMe drives when offloading vectors to disk with `on_disk: true`; rotational drives create unacceptable query latency.
- **MUST** run runtime filesystem compatibility checks on startup (v1.15+); warnings about unknown filesystems indicate potential data corruption risks.

### Cluster Topology
- **MUST** deploy 3+ nodes with 2+ shard replicas for production; this ensures zero-downtime operations during single-node failures and enables load balancing.
- **MUST** plan for 12 shards in collections with anticipated growth; allows scaling from 1→2→3→6→12 nodes without costly resharding operations.
- **NEVER** exceed 12 shards in small clusters; the indexing and coordination overhead degrades performance without scale benefits.
- **MUST** configure Pod affinity rules and topology spread constraints in Kubernetes; ensures replicas distribute across availability zones for fault tolerance.

### Deployment Mode Selection
- **MUST** use Qdrant Cloud or Kubernetes for production workloads; single-node Docker deployments lack HA, automated failover, and operational tooling.
- **MUST** use Private Cloud deployment for regulated industries (banking, healthcare); keeps both control plane and data plane within customer network boundaries.

---

## 2. Collection & Index Configuration

### HNSW Index Parameters
- **MUST** set `m: 16` (default) or `m: 32` for production HNSW graphs; lower values sacrifice recall, higher values waste memory without significant accuracy gains.
- **MUST** tune `ef_construct: 100+` based on accuracy requirements; higher values improve search quality but increase indexing time proportionally.
- **MUST** leverage HNSW healing (v1.15+) during optimization; reuses existing graph structure to accelerate re-indexing by orders of magnitude.
- **NEVER** rebuild HNSW from scratch during updates with v1.15+; healing preserves connectivity information and dramatically reduces resource consumption.

### Indexing Strategy for Bulk Uploads
- **MUST** set `on_disk: true` from collection creation for bulk ingestion; prevents RAM exhaustion before optimization by storing raw vectors on disk immediately.
- **MUST** set `m: 0` to disable HNSW during bulk uploads; eliminates real-time indexing overhead, then re-enable post-ingestion with production `m` value.
- **MUST** set `indexing_threshold: 0` to defer HNSW construction during ingestion; accumulates vectors first, builds index in single optimized pass after upload completes.
- **MUST** use 2 large segments (200MB+ each) rather than many small segments; reduces vector comparisons and improves parallel request handling.

### Payload Indexing
- **MUST** create payload indexes only on high-cardinality fields used in filters (IDs, user_ids); indexing low-cardinality fields (colors, categories) wastes memory.
- **MUST** use single-collection multitenancy with payload-based partitioning (`group_id` filtering); multiple collections degrade performance cluster-wide.
- **NEVER** create separate collections per tenant; Qdrant's architecture optimizes for single large collections with filtered queries, not collection proliferation.

---

## 3. Query Optimization & Performance (v1.15)

### Quantization Selection (CRITICAL for v1.15)
- **MUST** evaluate 1.5-bit and 2-bit quantization (v1.15+) for memory-constrained deployments; offers 24X and 16X compression respectively with tunable accuracy tradeoffs.
- **MUST** use 2-bit quantization over binary (1-bit) for vectors <128 dimensions; improves precision significantly for smaller vectors at 2X memory cost vs. binary.
- **MUST** use asymmetric quantization (v1.15+) when query vectors can remain unquantized; maintains full query precision while compressing stored vectors, improving accuracy without memory increase.
- **MUST** test quantization accuracy against unquantized baseline on representative queries; compression-accuracy tradeoffs vary significantly by embedding model and data characteristics.

### Storage Backend (v1.15+)
- **MUST** use Gridstore storage backend for new v1.15+ deployments; eliminates RocksDB garbage collection pauses and accelerates ingestion throughput.
- **MUST** plan migration path from RocksDB to Gridstore for existing clusters; Gridstore provides superior write performance and storage management.

### Search Performance
- **MUST** use `on_disk: true` with memory-mapped caching; Qdrant automatically caches hot vectors in RAM while mapping cold data from disk with minimal latency impact.
- **MUST** monitor P99 latency, not average latency; worst-case performance (slowest 1% of queries) reveals tail latency issues masked by averages.
- **NEVER** rely solely on average latency metrics; production systems must optimize for consistent experience, not just typical-case performance.

---

## 4. Security & Access Control

### Authentication (CRITICAL)
- **NEVER** deploy Qdrant without authentication enabled; default configuration is completely unsecured and exposes data to anyone with network access.
- **MUST** enable TLS encryption when using API key authentication; unencrypted connections expose credentials to MITM and sniffing attacks.
- **MUST** use JWT with RBAC (v1.9+) for multi-tenant or enterprise deployments; provides granular collection-level and operation-level access control.
- **MUST** use database API keys with collection-level restrictions (v1.11+); limits blast radius of compromised credentials to specific collections.
- **MUST** use separate read-only API keys for analytics/reporting workloads; prevents accidental or malicious data modification from read-heavy services.

### Network Security
- **MUST** expose only ports 6333 (HTTP/gRPC) and 6334 (metrics) to clients; port 6335 (p2p) is for internal cluster communication only.
- **MUST** block outbound network access from single-node Qdrant instances; mitigates SSRF attacks as Qdrant requires no external connectivity.
- **MUST** restrict port 6335 (p2p) to cluster nodes only in multi-node deployments; this internal gossip protocol must not be exposed externally.
- **MUST** run Qdrant with read-only root filesystem in containers; prevents exploitation of vulnerabilities requiring system file modification.

---

## 5. Scaling & High Availability

### Replication & Fault Tolerance
- **MUST** configure 2+ replicas per shard for production; single-replica clusters lose data permanently on node failure without backups.
- **MUST** verify automatic shard rebalancing during horizontal scaling; cloud platforms redistribute shards evenly across new nodes to maintain balance.
- **MUST** use zero-downtime resharding when changing shard count; Qdrant keeps collections available during resharding without requiring maintenance windows.
- **NEVER** skip minor versions during upgrades (e.g., 1.11→1.13); incremental upgrades prevent compatibility issues and state corruption.
- **MUST** upgrade versions sequentially: 1.13→1.14→1.15; skipping versions risks encountering unhandled migration paths.

### High Availability Guarantees
- **MUST** maintain 3+ node clusters for full HA; supports complete operation during single-node outages when all shards have 2+ replicas.
- **MUST** configure health checks and readiness probes in Kubernetes; enables automatic traffic shifting away from unhealthy nodes before failures cascade.

---

## 6. Monitoring & Error Handling

### Observability (CRITICAL)
- **MUST** expose Prometheus metrics on port 6333 at `/metrics`; provides real-time visibility into query latency, throughput, and system health.
- **MUST** integrate with monitoring platforms (Datadog, New Relic, Dynatrace, or Grafana); automated alerting prevents performance degradation from becoming outages.
- **MUST** monitor memory usage during index building; consistent 90%+ usage indicates undersized nodes requiring vertical or horizontal scaling.
- **MUST** track P99 latency, throughput, CPU usage, disk I/O, and pending operations; comprehensive metrics reveal bottlenecks before user impact.

### Error Prevention & Recovery
- **MUST** validate collections created in v1.15.0 for point shard routing bugs; multi-shard collections from 1.15.0 may exhibit update anomalies, fixed in 1.15.1+.
- **NEVER** reuse storage directories across multiple Qdrant instances; causes WAL file lock conflicts and data corruption in distributed clusters.
- **MUST** test cold-start performance after restarts; initial queries may take 50-100X longer than steady-state as caches rebuild.
- **MUST** perform load testing with production-like data and traffic patterns; synthetic data fails to reveal real-world indexing and query characteristics.

---

## 7. Client SDK Usage (Python)

### Connection & Deployment Modes
- **MUST** use async client methods (v1.6.1+) for high-concurrency applications; synchronous clients block threads during I/O operations.
- **MUST** use local mode (`:memory:` or disk path) for unit tests and prototyping; eliminates server dependency for development workflows.
- **NEVER** use local mode for production; lacks clustering, replication, and operational features required for reliability.

### API Patterns
- **MUST** use FastEmbed integration for simplified embedding workflows; bundles embedding models with client, eliminating external model management.
- **MUST** use `add()` method with FastEmbed for convenience; wraps Point creation and insertion into single API call for cleaner code.
- **MUST** choose distance metric (cosine, dot, L2) matching embedding model training; mismatched metrics produce nonsensical similarity rankings.

### Hybrid Search
- **MUST** normalize scores or use weighted sum when combining dense + sparse search (v1.10+); raw score scales differ dramatically between modalities.
- **MUST** use Qdrant's query DSL with multiple "should" clauses (v1.10+) for hybrid search; provides robust, declarative score fusion.

---

## 8. Production Readiness Checklist

### Pre-Deployment Validation
- **MUST** enable authentication (API key minimum, JWT/RBAC preferred) and TLS encryption; unsecured Qdrant is unsuitable for any production use case.
- **MUST** configure automated backups with tested restore procedures; replication protects against node failures but not logical corruption or accidental deletion.
- **MUST** provision monitoring dashboards and alerting before launch; observability cannot be retrofitted during incidents.
- **MUST** conduct load testing at 2X expected peak traffic; provides headroom for traffic spikes and gradual performance degradation.

### Operational Practices
- **MUST** implement blue-green or canary deployments for version upgrades; incremental rollouts contain blast radius of unexpected compatibility issues.
- **MUST** maintain runbooks for common failure scenarios (node loss, network partition, OOM conditions); ad-hoc incident response causes extended outages.
- **MUST** track storage growth rates and plan capacity 6+ months ahead; emergency scaling during exhaustion risks data loss and downtime.

### Version-Specific Considerations (v1.15)
- **MUST** audit collections created in v1.15.0 for point routing integrity; upgrade to v1.15.1+ if multi-shard collections exhibit update anomalies.
- **MUST** evaluate Gridstore migration for write-heavy workloads; v1.15 defaults new deployments to Gridstore for superior ingestion performance.
- **MUST** benchmark 1.5-bit/2-bit quantization against production query workloads; v1.15's granular quantization options enable unprecedented memory-accuracy optimization.
