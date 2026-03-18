# Apache Kafka KRaft Cluster — Built from Scratch

> A fully functional, ZooKeeper-free Kafka 4.2.0 cluster built and configured manually on Linux, with a web UI for real-time cluster management.

This project demonstrates hands-on experience setting up a production-style **Apache Kafka** cluster using the modern **KRaft consensus protocol** — no ZooKeeper required. Every node was configured manually from scratch, including separate controller and broker roles, dedicated log directories, and a live monitoring UI.

---

## What I Built

- A **2-broker, 2-controller Kafka cluster** running in KRaft mode with clean role separation
- Used **`controller.quorum.bootstrap.servers`** — the modern Kafka 4.x dynamic quorum method, replacing the older static `controller.quorum.voters`
- Configured each node with **dedicated listeners, advertised listeners, and isolated log directories**
- Deployed **Kafbat UI** (v1.4.2) as a web-based monitoring and management dashboard
- Tracked all configuration files in **Git** for version control

---

## Why KRaft?

KRaft (Kafka Raft) is Kafka's built-in consensus mechanism that **replaces ZooKeeper** entirely. As of Kafka 4.x, ZooKeeper is fully removed. Setting this up manually — rather than using Docker Compose or a managed service — required a deep understanding of how controllers elect leaders, how brokers register with the quorum, and how the metadata log works.

---

## Stack

| Component | Version |
|-----------|---------|
| Apache Kafka | 4.2.0 (Scala 2.13) |
| Java (OpenJDK) | 21 |
| Mode | KRaft (ZooKeeper-free) |
| Kafbat UI | v1.4.2 |
| OS | Linux (RHEL/Fedora-based) |
| Version control | Git |

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│                  KRaft Cluster                   │
│                                                  │
│  ┌─────────────────┐   ┌──────────────────────┐  │
│  │  Controller 1   │   │    Controller 2      │  │
│  │  node.id=1      │   │    node.id=2         │  │
│  │  port 9093      │   │    port 9095         │  │
│  └─────────────────┘   └──────────────────────┘  │
│         process.roles=controller                 │
│                                                  │
│  ┌─────────────────┐   ┌──────────────────────┐  │
│  │    Broker 1     │   │      Broker 2        │  │
│  │  node.id=3      │   │    node.id=4         │  │
│  │  port 9092      │   │    port 9094         │  │
│  └─────────────────┘   └──────────────────────┘  │
│         process.roles=broker                     │
└──────────────────────────────────────────────────┘
                        │
              ┌─────────▼──────────┐
              │    Kafbat UI       │
              │   (port 8080)      │
              └────────────────────┘
```

---

## Controller Configuration (Key Settings)

Each controller node runs with `process.roles=controller` — a pure controller with no broker responsibilities. This is the recommended production pattern.

```properties
# Pure controller role — no broker responsibilities
process.roles=controller
node.id=1

# Modern Kafka 4.x dynamic quorum discovery
# (replaces the older static controller.quorum.voters)
controller.quorum.bootstrap.servers=localhost:9093,localhost:9095

# Dedicated controller listener
listeners=CONTROLLER://:9093
advertised.listeners=CONTROLLER://localhost:9093
controller.listener.names=CONTROLLER

# Isolated log directory per node
log.dirs=/u01/kafka_2.13-4.2.0/kraft-controller-logs-1
```

---

## Key Skills Demonstrated

- **Distributed systems** — designing a multi-node Kafka cluster with clean separation of controller and broker roles
- **KRaft internals** — storage formatting, cluster ID generation, dynamic quorum bootstrap (Kafka 4.x)
- **Linux administration** — file permissions, process management, firewall configuration, service startup
- **Networking** — configuring listeners, advertised listeners, and inter-node communication across ports
- **Monitoring** — deploying and connecting Kafbat UI to a live running cluster
- **Git & version control** — tracking all infrastructure config files in a remote repository

---

## Repository Structure

```
/u01/kafka/
├── cluster-configs/
│   ├── controller.properties      # Controller node 1 (port 9093)
│   ├── controller-2.properties    # Controller node 2 (port 9095)
│   ├── server.properties          # Broker 1 (port 9092)
│   └── server-2.properties        # Broker 2 (port 9094)
└── ui-config/
    └── application-local.yml      # Kafbat UI connection config
```

---

## How to Reproduce This Setup

### Prerequisites

- Java 21+
- Linux (RHEL / Fedora / Ubuntu)
- Kafka 4.2.0 binary extracted to `/u01/`

```bash
sudo dnf install java-21-openjdk-devel -y
java -version
```

### 1. Format storage for each node

```bash
# Generate a cluster-wide unique ID
KAFKA_CLUSTER_ID="$(kafka-storage.sh random-uuid)"

# Format all nodes with the same cluster ID
kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c /u01/kafka_2.13-4.2.0/config/controller.properties
kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c /u01/kafka_2.13-4.2.0/config/server.properties
kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c /u01/kafka_2.13-4.2.0/config/server-2.properties
```

### 2. Start the cluster

```bash
# Start controllers first
kafka-server-start.sh /u01/kafka_2.13-4.2.0/config/controller.properties &

# Then start brokers
kafka-server-start.sh /u01/kafka_2.13-4.2.0/config/server.properties &
kafka-server-start.sh /u01/kafka_2.13-4.2.0/config/server-2.properties &
```

### 3. Verify cluster health

```bash
kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status
```

---

## Kafbat UI — Web Dashboard

```bash
nohup java \
  -Dspring.config.additional-location=/u01/kafbat-ui/application-local.yml \
  --add-opens java.rmi/javax.rmi.ssl=ALL-UNNAMED \
  -jar /u01/kafbat-ui/api-v1.4.2.jar \
  > /u01/kafbat-ui/kafbat.log 2>&1 &
```

Access at: `http://<your-server-ip>:8080`

```bash
# Open firewall port if needed
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

---

## Stopping the Cluster

```bash
pkill -f kafka
pkill -f kafbat
```

### Full reset (clears all data)

```bash
pkill -f kafka && pkill -f kafbat
rm -rf /u01/kafka_2.13-4.2.0/kraft-controller-logs*
rm -rf /u01/kafka_2.13-4.2.0/kraft-broker-logs*
```

---

## Author

**Stathis** — [github.com/Fstathios](https://github.com/Fstathios)

> Open to roles in Data Engineering, DevOps, and Platform Engineering. Feel free to connect!
