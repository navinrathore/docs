# 🏗️ System Architecture & Distributed Systems: Master Revision Index

A clean, hierarchical quick-revision checklist of core system architecture, distributed systems, and backend engineering topics and subtopics (max 1–2 levels). Designed for rapid review, technical interviews, and systems design audits.

---

## 🌐 1. Networking, Protocols & Inter-Service Communication

### Synchronous Communication
- **REST**: HTTP/1.1 vs HTTP/2, Idempotency, Status codes, Resource modeling, OpenAPI
- **gRPC**: Protocol Buffers (Protobuf), HTTP/2 multiplexing, Unary vs Bidirectional streaming
- **GraphQL**: Schema Definition Language (SDL), Queries, Mutations, Resolvers, N+1 problem

### Asynchronous & Real-Time Protocols
- **WebSockets**: Full-duplex persistent connection, Frame format, Handshake upgrade
- **Server-Sent Events (SSE)**: Unidirectional push over HTTP, Auto-reconnection, Event streams
- **WebRTC**: Peer-to-peer audio/video/data, SDP negotiation, STUN/TURN servers

---

## 📨 2. Messaging, Event-Driven Architecture (EDA) & Streaming

### Message Queues & Task Brokers
- **RabbitMQ**: AMQP protocol, Exchanges (Direct, Fanout, Topic), Routing keys, Dead-letter queues (DLQ)
- **Celery / Redis Streams**: Background task distribution, Worker prefetches, Priority queues

### Event Streaming & Log Storage
- **Apache Kafka**: Topics, Partitions, Consumer Groups, Offsets, Exactly-once semantics (EOS)
- **Modern Streaming**: Redpanda (C++ Kafka API), Apache Pulsar (tiered storage, compute/storage separation)

### Distributed Messaging Patterns
- **Saga Pattern**: Orchestration vs Choreography, Compensating transactions
- **Reliable Publishing**: Transactional Outbox Pattern, Change Data Capture (CDC / Debezium)
- **CQRS & Event Sourcing**: Command-Query Responsibility Segregation, Append-only event logs

---

## 🛡️ 3. Traffic Management, API Gateways & Edge

### API Gateways & Reverse Proxies
- **Gateways**: Envoy, NGINX, Traefik, Kong
- **Core Functions**: TLS termination, Auth validation, Request transformation, Dynamic upstream routing

### Traffic Control & Resiliency
- **Rate Limiting**: Token Bucket, Leaky Bucket, Sliding Window Counter, Distributed Redis counters
- **Circuit Breakers**: States (Closed, Open, Half-Open), Failure rate thresholds, Fallback responses
- **Load Balancing**: Layer 4 (TCP) vs Layer 7 (HTTP), Round-robin, Least connections, Consistent hashing

---

## ⚡ 4. Caching Strategies & Memory Management

### Caching Layers
- **In-Memory Caches**: Redis (Data types: Hashes, Sets, Sorted Sets, HyperLogLog), Memcached
- **Edge / CDN Caching**: Cloudflare, Fastly, Cache-Control headers, Stale-while-revalidate

### Patterns & Cache Invalidation
- **Caching Patterns**: Cache-Aside, Read-Through, Write-Through, Write-Behind (Write-Back)
- **Eviction Policies**: LRU (Least Recently Used), LFU (Least Frequently Used), TTL expiry
- **Cache Failures**: Cache Penetration (Bloom filters), Cache Breakdown / Stampede (Mutex locks), Cache Avalanche (Jittered TTLs)

---

## 🗄️ 5. Databases, Storage & Data Systems

### Relational Databases (RDBMS)
- **Core Engine**: ACID guarantees, Write-Ahead Logging (WAL), MVCC (Multi-Version Concurrency Control)
- **Indexing & Optimization**: B-Tree, Hash, GIN/GiST (PostgreSQL), Composite indexes, Query plan (EXPLAIN ANALYZE)
- **Scaling**: Read replicas, Connection pooling (PgBouncer), Table partitioning (Range, List, Hash)

### NoSQL & Distributed Key-Value
- **Document & Wide-Column**: MongoDB, Cassandra (LSM-trees, SSTables, Gossip protocol), DynamoDB
- **Distributed Theory**: CAP Theorem, PACELC Theorem, Eventual consistency vs Linearizability

### Analytical & Columnar (OLAP)
- **Columnar Storage**: ClickHouse, DuckDB, Parquet format, Vectorized execution
- **Lakehouse**: Apache Iceberg, Delta Lake

---

## ⚙️ 6. Concurrency, Async & Runtime Architecture

### Concurrency Primitives
- **Synchronization**: Mutexes, Read-Write Locks, Semaphores, Condition variables
- **Pitfalls**: Race conditions, Deadlocks (Coffman conditions), Livelocks, Priority inversion

### Async Runtimes (Python / Systems Context)
- **Python AsyncIO**: Single-threaded cooperative multitasking, Event Loop, Coroutines, Tasks
- **Execution Parallelism**: Python GIL (Global Interpreter Lock), Multiprocessing, Thread pools
- **Actor Model / CSP**: Erlang/Elixir OTP actors, Go channels and goroutines

---

## ☸️ 7. Compute, Virtualization & Orchestration

### Containers & Sandboxing
- **Container Runtimes**: Docker, OCI specs, `containerd`, Namespaces, Cgroups
- **MicroVMs & Sandboxing**: AWS Firecracker, gVisor, WebAssembly (WASM/WASI)

### Orchestration & Service Mesh
- **Kubernetes (K8s)**: Pods, Deployments, ReplicaSets, Services (ClusterIP, NodePort, LoadBalancer), Ingress
- **K8s Storage & State**: PersistentVolumes (PV), PersistentVolumeClaims (PVC), StatefulSets
- **Service Mesh**: Istio, Linkerd (mTLS mutual authentication, Traffic splitting, Service-to-service telemetry)

---

## 📈 8. Observability, Reliability & SRE

### The Three Pillars of Observability
- **Metrics**: Prometheus (Counters, Gauges, Histograms), Grafana, Pull vs Push model
- **Distributed Tracing**: OpenTelemetry (OTel), Trace context propagation (W3C traceparent), Spans, Jaeger
- **Structured Logging**: JSON logging, Log correlation IDs, Vector, ELK Stack, Grafana Loki

### System Reliability & Health
- **Probes**: Liveness probe, Readiness probe, Startup probe
- **Service Level Objectives**: SLI (Indicators), SLO (Objectives), SLA (Agreements), Error Budgets
- **Resilience Engineering**: Chaos engineering, Graceful degradation, Exponential backoff with jitter

---

## 🚀 9. Next Candidates & Frontier Architecture Topics (Backlog Pool)

A quick comma-separated list of advanced architecture topics to explore next:

> **Consensus & Coordination**: Raft, Paxos, Zookeeper, etcd, Distributed Locks (Redlock)  
> **Kernel & Networking**: eBPF (Cilium), DPDK, QUIC / HTTP/3, Zero-copy I/O (io_uring)  
> **Zero Trust Security**: OAuth2 / OIDC, JWT revocation patterns, SPIFFE / SPIRE, mTLS  
> **Data Infrastructure**: Vector database architecture (HNSW vs IVF-PQ), Vectorized execution engines  
> **Storage Primitives**: LSM-Trees vs B+ Trees, Write-Ahead Logs, Raft-based storage  
