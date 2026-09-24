# Stratum-Core — Layered Infrastructure & Island Isolation Architecture

> *"Isolation by design, not by discipline."*

[![Architecture](https://img.shields.io/badge/Architecture-Stratum%20Core%20v1.0-4f46e5?style=for-the-badge&logo=docker&logoColor=white)](https://github.com/sandrapuerto/stratum-core)
[![Author](https://img.shields.io/badge/Architect-Sandra%20Puerto-f59e0b?style=for-the-badge&logo=linkedin&logoColor=white)](https://sandrapuerto.com)
[![Security](https://img.shields.io/badge/Security-Strict%20Cap--Drop%20%7C%20DMZ%20Isolated-emerald?style=for-the-badge)](SECURITY.md)
[![Status](https://img.shields.io/badge/Environment-Production%20Verified-009639?style=for-the-badge)](https://github.com)
[![License](https://img.shields.io/badge/License-MIT%20Custom-yellow.svg?style=for-the-badge)](LICENSE)

**Stratum-Core** is an enterprise-grade, cloud-agnostic infrastructure orchestration pattern designed for deploying multi-tenant, zero-trust containerized workloads on single or hybrid virtual private servers (VPS). 

Engineered by **Sandra Gabriela Puerto Torres**, Stratum solves the friction between high operational cloud costs (OPEX) and security isolation. It enables independent enterprise applications (legacy and modern) to coexist securely on a unified physical/virtual host without port collisions, public surface exposure, or operational cross-contamination.

---

## 1. Executive Summary & Engineering Motivation

Traditional single-server Docker deployments suffer from three structural failure modes:
1. **Network Sprawl & Port Collisions:** Multiple applications competing for host ports (`80`, `443`, `3306`), leading to brittle port mapping workarounds.
2. **Excessive Cloud OPEX:** Over-reliance on managed cloud services (e.g., dedicated load balancers, multi-VPC gateways, managed database instances) for modest enterprise workloads.
3. **Implicit Lateral Movement:** Containers sharing default bridge networks without segmentation, allowing a compromised web container to directly query internal database ports.

**Stratum-Core** eliminates these failure modes through **Topological Island Isolation**:
* **-60% OPEX Reduction:** Consolidates isolated workloads onto self-managed infrastructure with zero cloud-vendor lock-in.
* **100% Ingress Cloaking:** Zero inbound open ports on the host. All ingress traffic is terminated exclusively via authenticated outbound Cloudflare Tunnels.
* **Boundary Client Pattern:** Persistence engines exist in air-gapped internal networks (`internal: true`) and can only be reached via hardened TCP stream boundary clients.

---

## 2. High-Level Architecture

### 2.1 Complete Platform Topology

```mermaid
graph TB
    subgraph WAN ["🌐 Public Internet & Cloudflare Edge"]
        Client["Incoming Client HTTPS"] --> Cloudflare["Cloudflare Zero-Trust Edge"]
    end

    subgraph Host ["🖥️ Virtual Private Server (Host Level - No Open Ingress Ports)"]
        subgraph GatewayIsland ["🚪 Layer 1: Access Layer (gateway/)"]
            CF_Tunnel["cloudflared<br/>(Outbound Tunnel Engine)"]
            NPM["Nginx Proxy Manager<br/>(Internal Routing & SSL)"]
            CF_Tunnel -->|TCP Forward| NPM
        end

        subgraph SharedDMZ ["🛡️ Layer 2: Shared Demilitarized Zone (dmz/)"]
            DMZ_Net(("stratum_dmz<br/>(External Bridge Network)"))
        end

        subgraph ConsumerApps ["📦 Layer 3: Application Consumers (Isolated Workloads)"]
            App1["landing-sgpt-web<br/>(Static Content Nginx)"]
            App2["erp-legacy-core<br/>(PHP / Laravel App)"]
            App3["nextcloud-enterprise<br/>(Private Cloud Suite)"]
        end

        subgraph DatabaseIsland ["🗄️ Layer 4: Data Layer Island (database/)"]
            DB_Proxy["nginx-database<br/>(Boundary TCP Proxy)"]
            DB_Net(("stratum_database_internal<br/>(Air-Gapped: internal=true)"))
            
            MariaDB[("MariaDB Engine<br/>Relational Data")]
            MongoDB[("MongoDB Engine<br/>Document Store")]
            Redis[("Redis Engine<br/>Cache & Streams")]

            DB_Proxy --> DB_Net
            DB_Net --> MariaDB
            DB_Net --> MongoDB
            DB_Net --> Redis
        end
    end

    Cloudflare -.->|Encrypted Outbound Tunnel| CF_Tunnel
    NPM --> DMZ_Net
    DMZ_Net --> App1
    DMZ_Net --> App2
    DMZ_Net --> App3
    DMZ_Net --> DB_Proxy

    classDef edge fill:#f59e0b,stroke:#b45309,stroke-width:2px,color:#000;
    classDef gateway fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#fff;
    classDef dmz fill:#0f172a,stroke:#f59e0b,stroke-width:2px,color:#fff;
    classDef consumer fill:#1e293b,stroke:#10b981,stroke-width:2px,color:#fff;
    classDef database fill:#1e293b,stroke:#ec4899,stroke-width:2px,color:#fff;

    class Client,Cloudflare edge;
    class CF_Tunnel,NPM gateway;
    class DMZ_Net dmz;
    class App1,App2,App3 consumer;
    class DB_Proxy,MariaDB,MongoDB,Redis database;
```

---

## 3. Core Component Matrix

Stratum is modularized into three autonomous components, each maintaining an independent Compose project, lifecycle, configuration, and security perimeter:

| Component | Responsibility | Provisioned Network | Isolation Level |
| :--- | :--- | :--- | :--- |
| [`dmz/`](dmz/) | Shared network provisioning & persistence anchor | `stratum_dmz` | Inter-island convergence bridge |
| [`gateway/`](gateway/) | Ingress tunnel termination & reverse proxying | `stratum_gateway_internal` | Egress-only access layer |
| [`database/`](database/) | Multi-engine persistence & boundary stream proxy | `stratum_database_internal` | Air-gapped internal data tier |

---

## 4. Ingress & Traffic Flow Model

```mermaid
sequenceDiagram
    autonumber
    actor User as Client Browser
    participant CF as Cloudflare Edge (WAF/DDoS)
    participant Tunnel as cloudflared (Tunnel Engine)
    participant NPM as Nginx Proxy Manager
    participant DMZ as stratum_dmz Network
    participant App as Consumer Web Service
    participant DBProxy as nginx-database Boundary
    participant MariaDB as MariaDB Internal Engine

    User->>CF: HTTPS Request (e.g. app.domain.com)
    CF->>Tunnel: Encrypted Tunnel Frame (Outbound-Initiated)
    Tunnel->>NPM: HTTP Forward via stratum_gateway_internal
    NPM->>DMZ: Route by Hostname via stratum_dmz
    DMZ->>App: Process Application Request
    opt Backend Data Query
        App->>DMZ: Query db-boundary:3306
        DMZ->>DBProxy: Terminate at Boundary Client
        DBProxy->>MariaDB: Stream TCP via stratum_database_internal (Air-Gapped)
        MariaDB-->>DBProxy: Return Query Result
        DBProxy-->>App: Stream Response to App
    end
    App-->>NPM: HTTP 200 OK Response
    NPM-->>Tunnel: Pipe Output
    Tunnel-->>CF: Encrypted Response
    CF-->>User: Rendered Secure Web Response
```

---

## 5. Security & Hardening Baseline

Every container deployed across Stratum-Core adheres strictly to the **Stratum Hardening Specification**:

```
                         ┌────────────────────────────────────┐
                         │   STRATUM HARDENING BASELINE       │
                         ├────────────────────────────────────┤
                         │ • read_only: true (Immutable root) │
                         │ • cap_drop: [ALL]                  │
                         │ • no-new-privileges: true          │
                         │ • tmpfs for volatile paths         │
                         │ • internal: true for data bridges  │
                         │ • Memory & PID constraints         │
                         │ • Structured log-rotation (10MBx3) │
                         └────────────────────────────────────┘
```

### 5.1 Defense-in-Depth Attributes

* **No Inbound Public Ports:** The VPS does not listen on ports `80`, `443`, or database ports. Cloudflare handles edge TLS termination and forwards requests through `cloudflared`.
* **Zero Trust Lateral Movement:** Compromising a web application gives an attacker access only to `stratum_dmz`. The database storage engines (`mariadb`, `mongodb`, `redis`) reside on `stratum_database_internal` (`internal: true`), completely unreachable without valid credentials through the TCP proxy.
* **Static Host Integrity:** Containers cannot write to their own root filesystem (`read_only: true`). Temporary execution files reside in capped RAM mounts (`tmpfs`).

---

## 6. Deployment & Orchestration Runbook

The components must be initialized sequentially to satisfy Docker network dependency graphs.

```mermaid
graph LR
    Step1["1. dmz/<br/>(Provision DMZ Network)"] --> Step2["2. gateway/<br/>(Launch Access Tunnel & NPM)"]
    Step2 --> Step3["3. database/<br/>(Deploy Persistence & Proxy)"]
    Step3 --> Step4["4. Consumer Apps<br/>(Attach to stratum_dmz)"]

    style Step1 fill:#0f172a,stroke:#f59e0b,stroke-width:2px,color:#fff
    style Step2 fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#fff
    style Step3 fill:#0f172a,stroke:#ec4899,stroke-width:2px,color:#fff
    style Step4 fill:#0f172a,stroke:#10b981,stroke-width:2px,color:#fff
```

### 6.1 Provisioning Commands

```bash
# 1. Provision the Shared DMZ Network
cd dmz/
cp .env.example .env && chmod 600 .env
docker compose up -d

# 2. Deploy the Ingress Access Layer
cd ../gateway/
cp .env.example .env && chmod 600 .env
docker compose up -d

# 3. Deploy the Data Layer & Boundary Proxy
cd ../database/
cp .env.example .env && chmod 600 .env
docker compose up -d
```

### 6.2 State Verification

```bash
# Verify container health across all Stratum tiers
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Networks}}"

# Verify network segmentation
docker network ls | grep stratum
```

Expected Network Inventory:
* `stratum_dmz` (Bridge / External)
* `stratum_gateway_internal` (Bridge / Isolated Egress)
* `stratum_database_internal` (Bridge / Air-Gapped `internal: true`)

---

## 7. Repository Structure

```
stratum-core/
├── LICENSE                 # MIT License with Security Reporting Condition
├── README.md               # Master Architecture & Technical Specification
├── SECURITY.md             # Responsible Vulnerability Disclosure Protocol
├── .gitignore              # Multi-tier exclusion rules
├── dmz/                    # Shared DMZ Network Anchor & Topology
│   ├── docker-compose.yml  # Network lifecycle container
│   ├── .env.example        # Network parameter definitions
│   └── README.md           # DMZ component manual & subnet specs
├── gateway/                # Ingress & Access Tier
│   ├── docker-compose.yml  # cloudflared + Nginx Proxy Manager
│   ├── .env.example        # Cloudflare credentials & port settings
│   └── README.md           # Gateway routing manual & TLS administration
└── database/               # Data Persistence & Boundary Tier
    ├── docker-compose.yml  # MariaDB + MongoDB + Redis + TCP Proxy
    ├── .env.example        # Root credentials & memory configurations
    ├── README.md           # Data operations, backup & rotation manual
    └── nginx/
        └── nginx.conf      # Layer 4 TCP Stream Boundary configuration
```

---

## 8. Author & Architectural Provenance

**Stratum-Core** was conceived, architected, and implemented by **Sandra Gabriela Puerto Torres**.

* **Role:** Backend Developer & Technical Infrastructure Director
* **Core Philosophy:** *Systems must be financially viable, structurally isolated, and resilient to operational turbulence without requiring oversized cloud budgets.*
* **Specialization:** PHP/Laravel, Linux Systems, Network Topology, Proxmox VE, Cloudflare Edge Architecture, Container Hardening.
* **Website:** [https://sandrapuerto.com](https://sandrapuerto.com)
* **Contact:** [contacto@sandrapuerto.com](mailto:contacto@sandrapuerto.com)

---

## 9. License

This repository is distributed under an enhanced MIT License with a mandatory private security vulnerability disclosure requirement. See [LICENSE](LICENSE) for full legal text.