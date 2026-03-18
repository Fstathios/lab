# WildFly 30 — Domain Mode Cluster

> A fully configured WildFly 30 domain mode cluster running across 3 virtual machines, using the `full-ha` profile with JGroups TCP clustering, ActiveMQ messaging, and an Oracle datasource.

This project demonstrates hands-on experience setting up an enterprise-grade **WildFly application server cluster** in domain mode — the way WildFly is managed in real production environments. A single domain controller manages all slave hosts and pushes configuration centrally.

---

## What I Built

- A **domain controller (master)** that centrally manages the entire cluster
- **2 slave hosts**, each running an application server instance (`appServer1`, `appServer2`)
- Used the **`full-ha` profile** — the most advanced WildFly profile, enabling high availability, clustering, and messaging
- Configured **JGroups TCP clustering** for inter-node communication
- Set up **ActiveMQ Artemis** messaging with cluster connections using JGroups discovery
- Connected an **Oracle datasource** (`ORCLPDB1`) for persistent storage
- Secured host-to-master communication using **Elytron DIGEST-MD5 authentication**
- Used **SSH key-based authentication** between master and slaves for secure management

---

## Stack

| Component | Version |
|-----------|---------|
| WildFly | 30.0.1.Final |
| Java (OpenJDK) | 17 |
| Mode | Domain mode |
| Profile | full-ha |
| Clustering | JGroups TCP |
| Messaging | ActiveMQ Artemis |
| Database | Oracle DB (ORCLPDB1) |
| Security | Elytron + DIGEST-MD5 |
| Hypervisor | Hyper-V |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Domain Controller                     │
│                  primary (10.35.10.10)                   │
│           /u01/app/dc/wildfly2/wildfly-30.0.1.Final      │
│                                                          │
│   - Hosts domain.xml (profiles, server groups)          │
│   - Management console: http://10.35.10.10:9990         │
│   - Pushes config to all slaves automatically           │
└───────────────────┬─────────────────────────────────────┘
                    │ remote+http (DIGEST-MD5)
          ┌─────────┴──────────┐
          │                    │
┌─────────▼──────────┐  ┌──────▼─────────────┐
│      Slave 1       │  │      Slave 2        │
│  slave1            │  │  slave2             │
│  10.35.10.10       │  │  10.35.10.11        │
│  mgmt: 10090       │  │  mgmt: 9990         │
│                    │  │                     │
│  appServer1        │  │  appServer2         │
│  HTTP: 8230        │  │  HTTP: 8230         │
│  (offset +150)     │  │  (offset +150)      │
└────────────────────┘  └─────────────────────┘
          │                    │
          └────────┬───────────┘
                   │ JGroups TCP clustering
          ┌────────▼───────────┐
          │   Oracle Database  │
          │   10.35.10.12:1521 │
          │   ORCLPDB1         │
          └────────────────────┘
```

---

## Key Configuration Highlights

### Domain controller (`host-primary.xml`)

The master runs `<domain-controller><local/>` — it IS the domain controller. No remote connection needed.

```xml
<host xmlns="urn:jboss:domain:20.0" name="primary">
    <domain-controller>
        <local/>
    </domain-controller>
    <interfaces>
        <interface name="management">
            <inet-address value="${jboss.bind.address.management:127.0.0.1}"/>
        </interface>
    </interfaces>
</host>
```

### Slave hosts (`host-secondary.xml`)

Each slave connects to the master using `remote+http` and authenticates with DIGEST-MD5 via Elytron.

```xml
<domain-controller>
    <remote authentication-context="secondary-hc-auth-context">
        <discovery-options>
            <static-discovery name="primary"
                protocol="remote+http"
                host="10.35.10.10"
                port="9990"/>
        </discovery-options>
    </remote>
</domain-controller>
```

### Server group (`domain.xml`)

All application servers belong to `app-servergroup` which uses the `full-ha` profile.

```xml
<server-group name="app-servergroup" profile="full-ha">
    <jvm name="default">
        <heap size="64m" max-size="512m"/>
    </jvm>
    <socket-binding-group ref="full-ha-sockets"/>
</server-group>
```

### JGroups TCP clustering (`domain.xml`)

Cluster nodes discover each other via TCPING with static host list.

```xml
<protocol type="TCPING">
    <property name="initial_hosts">10.35.10.10[7750],10.35.10.11[7750]</property>
    <property name="port_range">0</property>
</protocol>
```

---

## Key Skills Demonstrated

- **Enterprise middleware** — WildFly domain mode, the standard for managing Java EE clusters in production
- **High availability** — `full-ha` profile with session replication and distributed caching via Infinispan
- **Clustering** — JGroups TCP stack for inter-node communication and failover
- **Messaging** — ActiveMQ Artemis cluster with JGroups-based discovery
- **Database integration** — Oracle JDBC datasource with XA support
- **Security** — Elytron security framework with DIGEST-MD5 authentication
- **Linux administration** — multi-VM setup, SSH key generation and distribution
- **Hypervisor** — Hyper-V VM provisioning and network configuration

---

## Repository Structure

```
wildfly/
├── host-primary.xml            # Domain controller config (master)
├── host-secondary-slave1.xml   # Slave 1 host config (10.35.10.10)
├── host-secondary-slave2.xml   # Slave 2 host config (10.35.10.11)
└── domain.xml                  # Domain-wide config (profiles, server groups)
```

---

## How to Reproduce This Setup

### Prerequisites

- Java 17+
- WildFly 30.0.1.Final
- 3 Linux VMs (RHEL/Rocky/Fedora based)
- Oracle DB (optional — for datasource)

### 1. SSH key setup (master → slaves)

```bash
# On master
ssh-keygen -t rsa

# Copy public key to each slave
ssh-copy-id stathis@10.35.10.11
ssh-copy-id stathis@10.35.10.12
```

### 2. Start the domain controller (master)

```bash
/u01/app/dc/wildfly2/wildfly-30.0.1.Final/bin/domain.sh \
  --host-config=host-primary.xml \
  -b 10.35.10.10 \
  -bmanagement 10.35.10.10
```

### 3. Start each slave

```bash
# On slave 1
/u01/app/wildfly2/wildfly-30.0.1.Final/bin/domain.sh \
  --host-config=host-secondary.xml \
  -b 10.35.10.10 \
  -bmanagement 10.35.10.10

# On slave 2
/u01/app/wildfly2/wildfly-30.0.1.Final/bin/domain.sh \
  --host-config=host-secondary.xml \
  -b 10.35.10.11 \
  -bmanagement 10.35.10.11
```

### 4. Access the management console

Open your browser at: `http://10.35.10.10:9990`

---

## Profiles Available

| Profile | HA | Messaging | Clustering | Used in this lab |
|---------|-----|-----------|------------|-----------------|
| `default` | ✗ | ✗ | ✗ | No |
| `ha` | ✓ | ✗ | ✓ | No |
| `full` | ✗ | ✓ | ✗ | No |
| `full-ha` | ✓ | ✓ | ✓ | **Yes** |
| `load-balancer` | — | — | — | No |

---

## Author

**Stathis** — [github.com/Fstathios](https://github.com/Fstathios)

> Open to roles in Middleware Engineering, DevOps, and Platform Engineering. Feel free to connect!
