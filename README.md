# Hive-Compute: A Low-Budget Massive Parallel Computing Framework

Hive-Compute is a distributed computing framework designed to achieve High-Performance Computing (HPC) levels of computation by harnessing the power of a large swarm of commodity, non-specialized computing nodes. The project's description states it is a "Low-Budget Massive Parallel Computing framework - distributed computing for everyone (wealthy users are welcome, too)". The framework is designed to distribute computational jobs, such as matrix multiplication and sorting algorithms, across a large number of nodes to achieve massive parallelism.

## Core Concepts

### Decentralized and Resilient Architecture

The swarm operates without fixed leader nodes, creating a highly resilient and decentralized topology. This is achieved through several key mechanisms:

*   **Gossip Protocol**: Instead of a central server, nodes discover each other and share state using a **gossip protocol**. This ensures that the system is resilient to individual node failures, as there is no single point of failure.
*   **Dynamic Discovery (Pheromones)**: The process of discovery is analogous to ants following pheromone trails. Each node in the swarm broadcasts its specific capabilities, or **"traits"**, and available **"resources"** as part of its gossip messages. These traits act as pheromones, allowing other nodes to discover them and understand their capabilities. For example, a node might advertise: `Trait::CanExecute` and a `ResourceSnapshot` detailing its available CPU, memory, and disk space, effectively saying, "I can run computations and here is how much capacity I have." Another node might advertise its large memory and storage, broadcasting a pheromone like, "I'm a 64 gigabyte machine with MongoDB, maybe I can aggregate your data."
*   **Failure Detection**: The framework is designed to handle failures at multiple levels. It includes logic for `task_failure`, `worker_failure`, and provides `continue_on_failure` options for workflows. The gossip protocol itself helps in quickly identifying nodes that have gone offline.

### Inter-Node Communication

The communication between nodes in the swarm is built on a foundation of security and resilience. The `tokio-rustls` library is used to ensure that all inter-node communication is secured using TLS.

*   **Post-Quantum Encryption**: To protect against future threats, the framework has integrated post-quantum cryptographic algorithms as part of its "Cambrian Protocol". Specifically, it uses **Kyber** (from the `ml-kem` crate) for Key Encapsulation Mechanisms (KEM) and **Dilithium** (from the `fips204` crate) for digital signatures. This ensures that even if an attacker records encrypted traffic today, they will not be able to decrypt it in the future with a quantum computer.

*   **Sealed Computing in Communication**: For sensitive computations, Hive-Compute employs a sealed-computing mechanism. When a sealed computation is requested, the job is encapsulated in a `SealedEnvelope` which is a post-quantum encrypted object that is passed as part of a `SwarmMessage`.

*   **Predator Ecosystem**: The swarm is protected by a sophisticated, autonomous "predator" ecosystem that identifies and neutralizes malicious or misbehaving nodes to ensure the integrity of the swarm. This system is named **Neuromancer** and it orchestrates a "kill chain" of specialized micro-services:
    *   **Crocodile** and **Viper**: These components are part of the initial stages of identifying and tracking suspicious nodes.
    *   **Elektra**: This component is responsible for the final action of the kill chain, which is to blacklist and effectively eject a malicious node from the swarm.
    *   **African Wild Dogs**: This component is also part of the predator ecosystem, working to maintain swarm health.

### Sealed Computing

Sealed computing provides a verifiable, high-assurance environment for executing sensitive tasks on untrusted nodes. 

*   **Execution Environment**: Computations are wrapped in a **"Sealed Envelope"**, which is a post-quantum encrypted package that runs in a **Wasm sandbox** powered by the **Wasmtime** engine. On Linux, this sandbox is further hardened using **seccomp-BPF** to restrict the system calls the process can make, providing an extremely strong isolation boundary.

*   **Verification and Attestation**: After a sealed computation is complete, it produces a `SealedResult`. The integrity and authenticity of this result can be verified offline. The framework provides the `hive-sealed` and `cambrian-cli` command-line tools for this purpose. These tools allow a user to verify the **Dilithium attestation** on a `SealedResult`, inspect the audit trail, and decrypt the results.

### Distributed PostgreSQL

Hive-Compute includes a managed, distributed PostgreSQL layer designed to run on the swarm.

*   **Cluster Management**: The `PgManager` is responsible for managing a cluster of PostgreSQL nodes with **primary** and **replica** roles. It tracks the health and status of each node and manages the cluster topology.

*   **Failover and Replication**: The system includes a `PgFailoverHandler` which contains the logic to promote a replica to a primary in case of a failure, ensuring high availability of the database. The `ReplicationManager` oversees the process of keeping replicas in sync with the primary.

*   **Custom SQL Extensions**: The framework includes the `sqlparser` crate, which is used to parse and analyze SQL queries. This capability is leveraged to introduce a custom SQL dialect for distributed queries. This allows for adding extra keywords to SQL queries and providing custom, detailed `EXPLAIN PLAN` outputs that describe how a query will be parallelized and executed across the swarm.

### Restricted Binaries

The framework is designed to be deployed in high-security environments, and to that end, it supports the creation of restricted, hardened binaries with non-relaxable security and compliance features.

*   **Build-Time Configuration**: This is achieved through a combination of Cargo features and a build script (`build.rs`). The build script reads configuration from `.toml` files in the `compliance/` directory (e.g., `fedramp.toml`, `hipaa.toml`, `military.toml`) at compile time.

*   **Baked-in Compliance**: This mechanism allows for the creation of `ComplianceOverride` values that are baked into the binary. These overrides can enforce specific settings, and their UI visibility can be set to `VisibleReadonly`, which means an end-user can see the setting but cannot change it. This is how, for example, a binary can be produced where sealed computation is permanently enabled and cannot be relaxed.

*   **Hardened Features**: By enabling Cargo features like `military-sandbox`, the binary is compiled with support for the Wasmtime sandbox and seccompiler, providing the necessary components for the highest level of sealed computation security.
