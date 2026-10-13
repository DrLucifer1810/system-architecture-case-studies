# 🖥️ Local Server: Edge Bare-Metal Kubernetes Micro-Datacenter

[![Domain](https://img.shields.io/badge/Domain-Edge%20Computing%20%7C%20Bare--Metal%20Kubernetes-0284c7?style=flat-square)](https://github.com/DrLucifer1810)
[![Orchestration](https://img.shields.io/badge/Orchestration-K3d%20%7C%20K3s%20%7C%20Docker%20Registry-10b981?style=flat-square)](https://github.com/DrLucifer1810)
[![Networking & Security](https://img.shields.io/badge/Networking-Cloudflare%20Tunnel%20%7C%20Dual--WAN%204G%20Failover-8b5cf6?style=flat-square)](https://github.com/DrLucifer1810)
[![Power & Hardware](https://img.shields.io/badge/Hardware-Intel%2035W%20TDP%20%7C%20LiFePO4%20EPS%2014h-f59e0b?style=flat-square)](https://github.com/DrLucifer1810)

> **Case Study Overview**: The `local-server` project represents a complete, self-sovereign edge micro-datacenter architecture. It orchestrates the entire Tadagram/Biztada backend infrastructure—comprising **10 containerized microservices**, **4 polyglot data engines**, and an **edge reverse-proxy ingress**—running bare-metal on an energy-efficient on-premises node with 24/7 power and network resilience.

---

## 📊 Core Operational Metrics & Highlights

- **Workload Density:** Runs **10 distinct production microservices** + **4 database engines** (MySQL 8.0, MongoDB 6.0, Redis 7.0, Mosquitto MQTT) seamlessly on a single **16GB RAM / 35W TDP** bare-metal Mini PC node.
- **Cost Reduction:** Slashed cloud infrastructure expenses from **$420/month** (managed AWS/DigitalOcean Kubernetes cluster) down to **~$1.80/month** in local electricity costs (a **99.5% cost reduction**).
- **Network Resilience (Dual-WAN 4G Failover):** Router-level load balancer automatically fails over from Fiber Optic WAN to a 4G LTE backup connection within **3–5 seconds**; Cloudflare Tunnels re-establish connections instantly without IP reconfiguration.
- **Power High-Availability (14-Hour EPS):** 512Wh LiFePO4 portable power station operates in EPS mode (< **30ms transfer time**), sustaining continuous server and networking operation through prolonged power outages.
- **Zero Inbound Attack Surface:** Operates with **zero open incoming firewall ports**; all traffic is routed securely via outbound Cloudflare Zero Trust reverse tunnels.

---

## 🏛️ Master System Architecture Topology

```mermaid
graph TD
    %% Internet & Security Layer
    subgraph External_Network ["🌐 Public Internet & Edge Security Layer"]
        User["📱 Client Web / Mobile / Telegram Mini App"]
        CF_Pages["Cloudflare Pages (Frontend Host)"]
        CF_Tunnel["🔒 Cloudflare Zero Trust Tunnel (cloudflared daemon)"]
        User -->|HTTPS Frontend| CF_Pages
        User -->|HTTPS API Ingress| CF_Tunnel
    end

    %% Edge Machine Layer
    subgraph BareMetal_Node ["⚡ On-Premises Edge Node (Ubuntu Server 22.04 LTS / K3d Cluster)"]
        
        %% Ingress Layer
        subgraph Ingress_Layer ["🚪 Ingress & Reverse Proxy Routing"]
            Traefik["Traefik / NGINX Ingress Controller (Port 80/443)"]
            CF_Tunnel -->|Forward HTTP Loopback:80| Traefik
        end

        %% Core Backend Microservices
        subgraph Core_Services ["🧠 Core Platform Services"]
            Orch["1. orchestrator (Go/Python)\nTask Leases & Worker Registry"]
            CoreAPI["2. svc-core-api (Go)\nIdentity, Quotas & Auth"]
            Sidecar["3. telegram-sidecar (Rust)\nMTProto Session Engine"]
            Obs["4. svc-observability (Go)\nCentralized Prometheus Logs"]
        end

        %% Business Microservices
        subgraph Business_Services ["💼 Domain Microservices"]
            Chatbot["5. svc-business-chatbot (Go)\nMulti-Channel Webhooks"]
            BizCRM["6. svc-business-crm (Go)\nEnterprise CRM Sync"]
            BiztadaCRM["7. svc-biztada-crm (Node/Go)\nCustomer Data Platform"]
            Mktg["8. svc-business-marketing (Go)\nCampaign Workflow Blueprints"]
            AI["9. svc-ai-controller (Python)\nLocal Model & Prompt Broker"]
            Credit["10. svc-tada-credit (Go)\nBilling Ledger & Wallets"]
        end

        %% Routing
        Traefik -->|orchestrator.tadagram.com| Orch
        Traefik -->|core.tadagram.com| CoreAPI
        Traefik -->|business-chatbot.tadagram.com| Chatbot
        Traefik -->|business-crm.tadagram.com| BizCRM
        Traefik -->|biztada-crm.tadagram.com| BiztadaCRM
        Traefik -->|business-marketing.tadagram.com| Mktg
        Traefik -->|ai-controller.tadagram.com| AI
        Traefik -->|observability.tadagram.com| Obs
        Traefik -->|tada-credit.tadagram.com| Credit

        %% Data Persistence Layer
        subgraph Data_Layer ["💾 Polyglot Persistence Layer (Namespaces: database & tadagram)"]
            MySQL[("🐬 MySQL 8.0\nPersistent Volume (20GB)\nUsers, Billing, Sessions")]
            Mongo[("🍃 MongoDB 6.0\nPersistent Volume (20GB)\nWorkflows, Leads, Logs")]
            Redis[("⚡ Redis 7.0\nIn-Memory State\nTask Streams & Leases")]
            MQTT[("📡 Mosquitto MQTT\nPersistent Volume (5GB)\nWorker Control Plane")]
        end

        CoreAPI --> MySQL
        CoreAPI --> Redis
        BizCRM --> MySQL
        Chatbot --> MySQL
        Chatbot --> MQTT
        Mktg --> Mongo
        Orch --> Mongo
        Orch --> Redis
    end

    %% Physical Infrastructure Layer
    subgraph Physical_Hardware ["🔋 Physical Resilience Hardware"]
        MINI_PC["HP / Dell Mini PC (i5-6500T, 16GB RAM, 256GB NVMe)"]
        ROUTER["Dual-WAN Router (Primary Fiber + 4G LTE Backup)"]
        POWER["512Wh LiFePO4 Power Station (EPS Mode < 30ms Transfer)"]
        
        POWER -.->|220V Clean Sine Power (14h Backup)| MINI_PC
        POWER -.->|Power Backup| ROUTER
        ROUTER -.->|Gigabit LAN (Zero Wi-Fi)| MINI_PC
    end
```

---

## 📑 In-Depth Architectural Modules

1. [**System Overview & Microservices Topology (`architecture/01-system-overview.md`)**](./architecture/01-system-overview.md)  
   Complete breakdown of the 10 containerized microservices, polyglot database layout, and K3d Kubernetes cluster deployment manifests.

2. [**Zero-Trust Edge Ingress via Cloudflare Tunnels (`architecture/02-zero-trust-edge-ingress.md`)**](./architecture/02-zero-trust-edge-ingress.md)  
   How the on-premise cluster delivers public services over HTTPS with zero inbound router port forwarding or static IP requirements.

3. [**Deterministic 6-Stage Cluster Bootstrap (`architecture/03-deterministic-bootstrap.md`)**](./architecture/03-deterministic-bootstrap.md)  
   The automated, clean-slate provisioning workflow: dependency ordering, database health probes, and database seed ingestion (45,000+ MongoDB records).

4. [**Secure Remote Kubernetes Cluster Management (`architecture/04-remote-k8s-management.md`)**](./architecture/04-remote-k8s-management.md)  
   Managing edge Kubernetes nodes securely via private overlay networks (Tailscale/WireGuard) and Cloudflare Access.

---

## 🔬 Engineering Bottlenecks & Deep Dives

- [**Hardware, Dual-WAN & 14-Hour EPS Power Resilience (`deep-dives/01-hardware-power-failover.md`)**](./deep-dives/01-hardware-power-failover.md)  
  Hardware architecture: 35W TDP computing, automatic 4G LTE router failover, and battery chemistry (LiFePO4) selection for 24/7 reliability.
- [**Cost & Resource Profiling: Cloud VPC vs. Edge Bare-Metal (`deep-dives/02-cost-and-resource-profiling.md`)**](./deep-dives/02-cost-and-resource-profiling.md)  
  Financial and operational audit comparing a $420/month cloud cluster against a $1.80/month local edge server with equivalent performance.
- [**Automated Stateful Database Restoration (`deep-dives/03-database-restoration-automation.md`)**](./deep-dives/03-database-restoration-automation.md)  
  Orchestrating health loops (`mysqladmin ping`, `mongosh` readiness) to safely restore multi-schema SQL and BSON dumps during bootstrap.

---

## 💻 Sanitized Reference Implementations

Standalone scripts demonstrating the core automation of this edge infrastructure:

- [`../sanitized-code/k3d-bootstrap-orchestrator/`](../sanitized-code/k3d-bootstrap-orchestrator/): Automated shell/Python script deploying the 6-stage K3d cluster and private local registry.
- [`../sanitized-code/cloudflared-tunnel-ingress/`](../sanitized-code/cloudflared-tunnel-ingress/): Production Kubernetes manifests for running `cloudflared` daemon alongside Traefik Ingress.
- [`../sanitized-code/db-health-probe-watchdog/`](../sanitized-code/db-health-probe-watchdog/): Resilient polling scripts verifying socket connectivity before initiating database seeding.
