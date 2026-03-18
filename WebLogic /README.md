# Oracle WebLogic 12c — Domain Mode Cluster

> A fully configured WebLogic 12.2.1.4.0 domain with an Admin Server, a 2-node managed server cluster, a standalone load balancer, JMS, and an Oracle datasource — built and configured manually on Linux.

This project demonstrates hands-on experience setting up an enterprise **Oracle WebLogic** domain in domain mode — the standard way WebLogic is deployed in production Oracle environments. The Admin Server centrally manages all servers and pushes configuration across the domain.

---

## What I Built

- An **Admin Server** acting as the domain controller
- A **2-node cluster** (`Cluster-0`) with `ms1` and `ms2` using unicast messaging
- A **standalone managed server** (`ms3`) acting as a load balancer
- Configured **Node Manager** for remote server lifecycle management
- Set up **JMS** with a JDBC-backed persistent store
- Connected an **Oracle datasource** (`MyOracleDS`) targeted to `Cluster-0`
- Configured **JTA migratable targets** for transaction recovery across cluster members
- Secured the domain with **XACML role mapping** and **AES256 encrypted credentials**

---

## Stack

| Component | Version |
|-----------|---------|
| Oracle WebLogic | 12.2.1.4.0 |
| Java (JDK) | 8+ |
| Domain name | wl_server |
| Clustering | Unicast |
| JMS | WebLogic JMS + JDBC store |
| Database | Oracle |
| Security | XACML + DefaultAuthenticator |
| OS | Linux |
| VM | 10.35.10.10 |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    WebLogic Domain                       │
│                      wl_server                          │
│                   10.35.10.10                           │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Admin Server (port 7001)            │   │
│  │         Console: http://10.35.10.10:7001/console │   │
│  │         Node Manager: port 5556                  │   │
│  └──────────────────┬───────────────────────────────┘   │
│                     │ manages                           │
│         ┌───────────┼───────────┐                       │
│         │           │           │                       │
│  ┌──────▼──────┐    │    ┌──────▼──────┐               │
│  │     ms1     │    │    │     ms2     │               │
│  │  port 8001  │    │    │  port 8002  │               │
│  │  Cluster-0  │    │    │  Cluster-0  │               │
│  └─────────────┘    │    └─────────────┘               │
│                     │                                   │
│              ┌──────▼──────┐                           │
│              │     ms3     │                           │
│              │  port 8003  │                           │
│              │Load Balancer│                           │
│              └─────────────┘                           │
└─────────────────────────────────────────────────────────┘
                        │
              ┌─────────▼──────────┐
              │   Oracle Database  │
              │   MyOracleDS       │
              │   (→ Cluster-0)    │
              └────────────────────┘
```

---

## Key Configuration Highlights

### Admin Server

The Admin Server is the domain controller — it hosts the management console and controls all managed servers.

```xml
<server>
  <name>AdminServer</name>
  <machine>Machine-0</machine>
  <!-- HTTP: 7001 | HTTPS: 7002 (disabled in this lab) -->
  <ssl>
    <enabled>false</enabled>
    <listen-port>7002</listen-port>
  </ssl>
</server>
```

### Cluster (unicast)

`Cluster-0` uses unicast messaging — no multicast needed. `ms1` and `ms2` are members with JTA migratable targets for transaction failover.

```xml
<cluster>
  <name>Cluster-0</name>
  <cluster-messaging-mode>unicast</cluster-messaging-mode>
</cluster>
```

### Oracle datasource targeted to cluster

`MyOracleDS` is targeted directly to `Cluster-0` — meaning both `ms1` and `ms2` automatically get access to the Oracle database.

```xml
<jdbc-system-resource>
  <name>MyOracleDS</name>
  <target>Cluster-0</target>
</jdbc-system-resource>
```

### Node Manager

Node Manager is configured on `Machine-0` and allows the Admin Server to start, stop, and monitor managed servers remotely.

```xml
<machine>
  <name>Machine-0</name>
  <node-manager>
    <nm-type>Plain</nm-type>
    <listen-address>10.35.10.10</listen-address>
    <listen-port>5556</listen-port>
  </node-manager>
</machine>
```

> **Note:** Node Manager was configured in this lab but not fully activated.

---

## Key Skills Demonstrated

- **Enterprise middleware** — WebLogic 12c domain mode, the standard for Oracle-stack production deployments
- **Clustering** — unicast cluster with JTA migratable targets for high availability and transaction recovery
- **JMS** — WebLogic JMS server with JDBC-backed persistent message store
- **Database integration** — Oracle datasource targeted at cluster level
- **Security** — XACML role mapper, DefaultAuthenticator, AES256 encrypted credentials
- **Node Manager** — remote server lifecycle management configuration
- **Linux administration** — WebLogic installation, domain creation, and server management on Linux

---

## Repository Structure

```
weblogic/
├── config.xml       # Full domain configuration (trimmed, key parts)
└── README.md        # This file
```

---

## Server Summary

| Server | Role | Cluster | Port |
|--------|------|---------|------|
| AdminServer | Domain controller | — | 7001 |
| ms1 | Managed server | Cluster-0 | 8001 |
| ms2 | Managed server | Cluster-0 | 8002 |
| ms3 | Load balancer | — | 8003 |

---

## Accessing the Admin Console

```
http://10.35.10.10:7001/console
```

Login with your WebLogic admin credentials.

---

## Author

**Stathis** — [github.com/Fstathios](https://github.com/Fstathios)

> Open to roles in Middleware Engineering, DevOps, and Platform Engineering. Feel free to connect!
