# ⚙️ Architecture: Deterministic 6-Stage Cluster Bootstrap

## 1. The Challenge of Bare-Metal Orchestration

In cloud platforms, managed databases and networking are abstracted by cloud APIs. On bare-metal edge hardware, deploying 10 interdependent microservices and 4 stateful databases from scratch often causes **race conditions**:
- Microservices start before MySQL has finished initial socket binding, causing crash-loops.
- Database restore scripts run while the storage daemon is still initializing tables, causing data corruption.
- Container registries external to the machine cause network timeouts during high-density image pulls.

To guarantee 100% reproducibility, the infrastructure utilizes a **Deterministic 6-Stage Bootstrap Workflow**:

```mermaid
flowchart TD
    S0["Stage 0: Clean-Slate Teardown\n• Prune old K3d clusters & registries\n• Verify >= 25GB free NVMe storage"] --> S1
    S1["Stage 1: K3d & Local Registry Init\n• Spawn k3d-myregistry.localhost:5000\n• Create cluster with port 80/443 mapped"] --> S2
    S2["Stage 2: Compile & Push Microservices\n• Build 10 Docker images\n• Push directly to local port 5000"] --> S3
    S3["Stage 3: Provision Storage & Databases\n• Apply PVCs, MySQL, Mongo, Redis, MQTT"] --> C1
    
    C1{"Health Probe Watchdog:\nmysqladmin ping & mongosh ready?"}
    C1 -- Not ready --> W1["Wait 2s & Retry Loop"]
    W1 --> C1
    
    C1 -- Ready (Exit Code 0) --> S4["Stage 4: Automated Database Seeding\n• Restore 45,371 Mongo BSON records\n• Execute MySQL table migrations & users"] --> S5
    S5["Stage 5: Deploy 10 Microservices\n• Apply 04-microservices.yaml\n• Pods connect to fully prepared DBs"] --> S6
    S6["Stage 6: Ingress Routing & Verification\n• Map Traefik Ingress rules\n• Launch cloudflared tunnel daemon"] --> END["Cluster 100% Operational!"]
```

---

## 2. Stage-by-Stage Implementation Details

### Stage 0: Teardown & Clean-Slate Reset
Guarantees zero configuration drift:
```bash
k3d cluster delete mycluster
k3d registry delete myregistry.localhost
docker system prune -af --volumes
```

### Stage 1: Local Registry & Cluster Binding
Creates an isolated Kubernetes control plane bound to a local registry:
```bash
k3d registry create myregistry.localhost --port 5000
k3d cluster create mycluster \
  --registry-use k3d-myregistry.localhost:5000 \
  -p "80:80@loadbalancer" \
  -p "443:443@loadbalancer"
```

### Stage 4: Active Health Probes Before Seeding
Microservices must never be scheduled until database schemas are guaranteed to be populated:
```bash
# MySQL Health Verification Loop
until docker exec -i $(docker ps -qf "name=mysql") mysqladmin ping -u root -proot --silent; do
    echo "Waiting for MySQL socket binding..."
    sleep 2
done

# MongoDB Seed Restoration (Over 45,000 records)
docker exec -i $(docker ps -qf "name=mongo") mongorestore --drop /data/db-dumps/mongo-dump/
```

### Stage 5 & 6: Zero-Restart Microservice Scheduling
Because downstream databases are completely healthy and pre-seeded before microservice manifests are applied, all 10 application pods reach `1/1 Running` status **on the very first attempt without a single restart event**.