Proposed Folder Structure
```
go-scrapper-engine/
├── cmd/
│   └── scrapper/
│       └── main.go                   # Entry point; picks storage backend via config, wires engine
│
├── internal/
│   ├── engine/
│   │   ├── engine.go                  # Core engine: owns job/result channels, worker pool lifecycle
│   │   ├── job.go                     # Job struct (Target URL wrapper)
│   │   └── result.go                  # Result struct (engine-level, not storage-level)
│   │
│   ├── worker/
│   │   ├── pool.go                    # Worker pool manager (spawns Worker 1..N)
│   │   └── worker.go                  # Single worker: reads Job Channel, fetches, pushes to Result Channel
│   │
│   ├── channels/
│   │   └── channels.go                # Internal Job Channel / Internal Result Channel definitions
│   │
│   ├── cache/
│   │   └── local_cache.go             # Thread-Safe Local Cache (sync.RWMutex / sync.Map)
│   │
│   ├── proxy/
│   │   ├── rotator.go                 # Outbound Proxy Rotator logic
│   │   └── pool.go                    # Proxy list/pool management
│   │
│   ├── antibot/
│   │   └── client.go                  # Anti-Bot API client
│   │
│   ├── parser/
│   │   └── parser.go                  # Data Parser: raw response -> structured Result
│   │
│   └── storage/
│       ├── storage.go                 # Storage interface (contract) + shared Result type
│       ├── memory/
│       │   └── memory_store.go        # In-memory implementation — used now, for testing
│       └── postgres/
│           ├── postgres_store.go      # Postgres implementation — added later
│           └── queries.go             # SQL query strings / prepared statements
│
├── pkg/
│   ├── httpclient/
│   │   └── client.go                  # Reusable HTTP client (used by worker, proxy rotator)
│   └── logger/
│       └── logger.go                  # Shared logging utility
│
├── config/
│   └── config.go                      # Config loader: worker count, proxy list, storage_backend, postgres_dsn, cache TTL
│
├── configs/
│   ├── config.yaml                    # Default / dev config -> storage_backend: memory
│   └── config.prod.yaml               # Prod config -> storage_backend: postgres
│
├── migrations/
│   └── 0001_init.sql                  # Postgres schema (applied only when postgres backend is used)
│
├── scripts/
│   └── run.sh
│
├── go.mod
├── go.sum
└── README.md
```

# **Go Scraper Engine: Architectural Specification**

This document provides a formal, detailed breakdown of the go-scrapper-engine codebase architecture. The system is designed to be a highly concurrent, modular, and resilient web scraping platform written in Go. It leverages Go's native concurrency primitives (goroutines and channels) and applies clean architecture principles to decouple core scraping orchestration from storage backends, parsing logic, and networking configurations.

## **1\. System Overview**

The go-scrapper-engine is organized around a centralized worker-pool pattern. The core engine coordinates incoming scraping requests (Jobs) and multiplexes them across a pool of isolated workers. These workers process jobs, execute outbound HTTP requests through rotating proxy networks, bypass defensive mechanisms using anti-bot clients, parse the resulting raw HTML, and persist the structured data into a configurable storage backend.

## **2\. Component Directory Specification**

### **2.1 Entry Point (cmd/)**

* **cmd/scrapper/main.go**  
  * **Responsibility:** The bootstrap environment and application dependency injection root.  
  * **Lifecycle:** It parses configuration settings, initializes shared utility packages (logging, database drivers), instantiates the designated storage engine based on configuration, spins up the scraping engine, feeds initial seed targets into the system, and listens for OS signals (SIGINT, SIGTERM) to trigger a graceful shutdown sequence.

### **2.2 Core Logic (internal/)**

The internal directory contains packages restricted to this project, ensuring encapsulated domain logic.

* **internal/engine/**  
  * engine.go: The central orchestrator. It manages the lifecycle of the worker pool, instantiates the bidirectional communication channels, collects final results, and handles system-wide context cancellations.  
  * job.go: Defines the Job struct. It acts as a transfer object enclosing metadata such as target URLs, retry counters, custom headers, and parsing rules.  
  * result.go: Defines the engine-level Result struct, wrapping metadata regarding scraping execution status, timing metrics, and raw payloads before storage execution.  
* **internal/worker/**  
  * pool.go: Manages worker allocation and scale. It spawns ![][image1] independent worker goroutines and manages their synchronization lifecycle using sync.WaitGroup.  
  * worker.go: Represents a single execution unit. It continuously polls the incoming job channel, performs the network requests, coordinates with the anti-bot and proxy layers, and pushes the output to the engine-level result channel.  
* **internal/channels/**  
  * channels.go: Holds the structural declarations for the system's pipeline. By centralizing channel definitions, it prevents tight coupling between the worker pool and the engine orchestrator.  
* **internal/cache/**  
  * local\_cache.go: A thread-safe, in-memory caching mechanism. Utilizing sync.RWMutex or sync.Map, it tracks recently requested URLs and raw contents to enforce Rate Limiting rules and prevent duplicate network calls within a configurable Time-To-Live (TTL) window.  
* **internal/proxy/**  
  * pool.go: Manages a list of available outbound IP addresses or proxy servers, supporting validation and health checks.  
  * rotator.go: Implements selection algorithms (e.g., Round-Robin, Random, or Latency-based) to cycle through the proxy pool, mitigating IP ban risks.  
* **internal/antibot/**  
  * client.go: Handles anti-bot mitigation strategies. This includes injecting user-agent footprints, managing cookies, handling javascript challenges, or routing through external bypass API endpoints.  
* **internal/parser/**  
  * parser.go: Decouples DOM traversal from network operations. It receives raw HTTP response payloads and extracts structured data fields according to predefined schemas.  
* **internal/storage/**  
  * storage.go: Defines the generic Storage interface (contract) detailing data persistence operations (Save, Get, Exists).  
  * memory/memory\_store.go: An ephemeral, thread-safe memory mock implementing Storage, used for local development, debugging, and unit testing.  
  * postgres/: Persistent production-grade storage.  
    * postgres\_store.go: Concrete implementation of the Storage interface using PostgreSQL driver pools (pgx or sqlx).  
    * queries.go: Centralized SQL prepared statements and raw queries to isolate database mutations.

### **2.3 Shared Utilities (pkg/)**

The pkg directory holds reusable utilities that could theoretically be exported to other projects.

* **pkg/httpclient/**  
  * client.go: A wrapper around the standard http.Client. It configures safe connection pooling, custom timeouts, TLS settings, and exposes hooks for integrating custom proxy configurations.  
* **pkg/logger/**  
  * logger.go: A structured logger (e.g., using uber-go/zap or standard slog) providing distinct trace, debug, info, and error logs with context support.

### **2.4 Configuration (config/ and configs/)**

* **config/config.go**  
  * **Responsibility:** The configuration parser. It reads environmental variables or YAML files and maps them to a Go configuration structure. It sets defaults for worker counts, HTTP timeouts, proxy lists, active storage backends, and cache behaviors.  
* **configs/config.yaml / config.prod.yaml**  
  * **Responsibility:** Declarative environment files. The development configuration defaults to the memory store to reduce database dependencies, while the production configuration dictates a clustered PostgreSQL backend and stricter rate limiting.

### **2.5 Supporting Assets (migrations/, scripts/, and Meta)**

* **migrations/0001\_init.sql**: Relational schemas, indexes, and tables matching the expected data model for the PostgreSQL backend.  
* **scripts/run.sh**: Automation scripts to streamline environment setups, dependency downloads, and local database container spawning.

## **3\. Data and Concurrency Flow**

The life cycle of a scraping transaction proceeds through the following sequential stages:
```
[Main Thread] ---> Reads Config ---> Instantiates Storage (Memory/Postgres)  
     |  
     +---> Spawns Engine ---> Initializes Channels (Job Ch, Result Ch)  
                                 |  
                                 +---> Spawns Worker Pool (N Goroutines)  
                                            |  
[Job Seeded] ------------------------------>+ (Worker Polls Job)  
                                            |  
                                    [Cache Lookup] (Hits? Return Cached)  
                                            | (Miss)  
                                     [Proxy Rotator] (Assign Outbound IP)  
                                            |  
                                    [Anti-Bot Client] (Configure Headers/Session)  
                                            |  
                                    [HTTP Client] (Executes Outbound Call)  
                                            |  
                                     [Parser Engine] (HTML -> Structured Format)  
                                            |  
                                     [Result Channel] ---> [Storage Worker] ---> [PostgreSQL/Memory]
```
1. **Orchestration and Fan-Out:** Engine initializes a buffered Job Channel. The Pool manager registers ![][image1] workers. Each worker runs on its own goroutine, continuously listening to the Job Channel.  
2. **Scraping Phase:** Once a job is claimed, the worker checks local\_cache to verify freshness. If cache validation fails, the worker retrieves a proxy IP from the rotator, updates the HTTP client parameters, applies anti-bot headers, and dispatches the request.  
3. **Parsing and Fan-In:** The raw response is handled by Parser. The resulting structured Result structure is fed into the central Result Channel.  
4. **Persistence:** The Engine processes the incoming results off the Result Channel and writes them into the selected active Storage backend concurrently.

## **4\. Concurrency Safety and Graceful Shutdowns**

To ensure data integrity and avoid memory leaks:

* **Context Propagation:** The context.Context API is utilized across all structural levels. If a timeout occurs or an OS interrupt is captured, contexts are cancelled, signaling workers to stop acquiring new jobs and exit cleanly.  
* **Thread Safety:** The local\_cache relies on strict read/write locking patterns (sync.RWMutex) to guarantee that parallel write operations do not result in race conditions.  
* **Database Connections:** The PostgreSQL backend manages connection limits internally via thread-safe connection pooling, ensuring workers do not exhaust the target database's file descriptors.
