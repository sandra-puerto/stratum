# ⚓ CORE-Islands: Sovereign Cloud Deployment

*On-Premise Framework with Micro-Segmentation and Zero-Trust Perimeter*

CORE-Islands implements **highly secure private clouds** on dedicated hardware with:

- 🌐 **Persistent Network**  
- 🔒 **Strict Service Isolation**  
- ⚓🌉 **"Anchors and Bridges" Architecture**

---

## 🚀 Deployment Sequence

The deployment must follow a **strict hierarchical order**, as each layer depends on the availability of the previous layer.

### 1️⃣ Core Layer: Global
Contains the **Orchestrator (NPM)** and defines the **global transit network**.

```bash
cd Global/
cp .env.example .env   # Configure global network names
docker-compose up -d
```

> ⚠️ Without this, no other island can communicate.

---

### 2️⃣ Bridge Layer: Gateway
Houses the **Cloudflare Tunnel**, connected to the previously created transit network.

```bash
cd ../Gateway/
cp .env.example .env   # Insert your CLOUDFLARE_TUNNEL_TOKEN
docker-compose up -d
```

---

### 3️⃣ Persistence Layer: Database
Deploys the **Storage Island**. Requires both the Gateway and Global layers to be active.

```bash
cd ../Database/
cp .env.example .env   # Define DB credentials and Ambassador rules
docker-compose up -d
```

---

## 🛠️ Verification Checklist

- **Networks:**  
  ```bash
  docker network ls
  ```  
  The defined networks should appear (`global_network`, `database_network`)

- **Connectivity:**  
  `Cloudflared` container should report status **Connected**  

- **Isolation:**  
  `ping google.com` from MariaDB should fail (internal network)

---

## 📁 Repository Structure

```plaintext
.
├── Global/            # Step 1: Orchestration & Global Network
├── Gateway/           # Step 2: Zero Trust Tunneling
├── Database/          # Step 3: Isolated Storage & DB Ambassadors
├── Applications/      # (Optional) Specific Application Islands
└── README.md          # Project Documentation
```

---

## 📐 CORE-Islands Network Diagram

```mermaid
graph TD
    %% Global Nodes Definition
    classDef orchestrator stroke-width:4px, stroke:#000, fill:none;
    classDef ambassador stroke-width:2px, stroke:#000, stroke-dasharray: 5 5, fill:none;
    classDef storage stroke-width:1px, stroke:#000, fill:none;
    classDef boundary stroke-width:3px, stroke:#000, fill:none;

    subgraph WAN [External World]
        Internet((Internet)) <--> CF{Cloudflare Edge}
    end

    subgraph Host [Host Machine - OS Agnostic]
        
        subgraph Global_Network [Global Transit Network]
            NPM[[Nginx Proxy Manager - Orchestrator]]
        end

        subgraph Gateway_Island [Gateway Island]
            Cloudflared[Cloudflared Tunnel]
        end

        subgraph App_Island [Application Island]
            Amb_App([Nginx Ambassador]) --- App_Node[Application Node]
        end

        subgraph Data_Island [Storage Island]
            Amb_DB([Nginx Ambassador]) --- DB_Cluster
            subgraph DB_Cluster [Storage Services]
                MariaDB((MariaDB))
                MongoDB((MongoDB))
                Redis((Redis))
            end
        end

    end

    %% Optimized Traffic Flow
    CF <==> Cloudflared
    Cloudflared <==> NPM
    NPM == Routing ==> Amb_App
    NPM == Routing ==> Amb_DB

    %% Apply Classes
    class NPM orchestrator;
    class Amb_App,Amb_DB ambassador;
    class MariaDB,MongoDB,Redis storage;
    class Host boundary;

    %% Global Style Overrides
    style WAN fill:none,stroke:#000,stroke-width:1px
    style Global_Network fill:none,stroke:#000,stroke-width:2px,stroke-dasharray: 2 2
    style Gateway_Island fill:none,stroke:#000,stroke-width:1px
    style App_Island fill:none,stroke:#000,stroke-width:1px
    style Data_Island fill:none,stroke:#000,stroke-width:1px
    style DB_Cluster fill:none,stroke:#000,stroke-width:1px
```

---

## 🏷️ Credits & Authorship

- **Lead Architect & Maintainer:** @gabypuertor964 ([Github](https://github.com/gabypuertor964) / [LinkedIn](https://www.linkedin.com/in/gabypuertor964/)) 
- **Technical Copilot:** Powered by **Google Gemini** (AI Orchestration)  
- **License:** Apache License 2.0