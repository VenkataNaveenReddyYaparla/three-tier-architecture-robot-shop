# Practice Guide — Robot Shop

Reference notes for practicing with this repo: what tech is used, where it lives, and what to try.

## 1. Application Services

| Service | Tech | Path | Practice ideas |
|---|---|---|---|
| web | Nginx + AngularJS (1.x) static frontend | [web/](web/) | Reverse proxy config, static asset serving |
| cart | Node.js / Express | [cart/server.js](cart/server.js) | REST API, Redis session/state |
| catalogue | Node.js / Express | [catalogue/server.js](catalogue/server.js) | MongoDB queries, REST API |
| user | Node.js / Express | [user/server.js](user/server.js) | Auth, MongoDB, sessions |
| shipping | Java / Spring Boot (Maven) | [shipping/pom.xml](shipping/pom.xml) | JPA, MySQL, Spring Boot builds |
| ratings | PHP 7.4 / Apache (Composer) | [ratings/Dockerfile](ratings/Dockerfile) | PHP-Apache container, MySQL |
| payment | Python / Flask | [payment/payment.py](payment/payment.py) | RabbitMQ producer/consumer, Prometheus metrics |
| dispatch | Go | [dispatch/main.go](dispatch/main.go) | RabbitMQ consumer, Go builds |
| load-gen | Python / Locust | [load-gen/robot-shop.py](load-gen/robot-shop.py) | Load testing, traffic generation |
| fluentd | Log forwarding | [fluentd/](fluentd/) | Centralized logging pipelines |

## 2. Data Stores / Messaging

| Component | Purpose | Path |
|---|---|---|
| MongoDB | catalogue + user data | [mongo/](mongo/) |
| MySQL (+ MaxMind data) | shipping + ratings data | [mysql/](mysql/) |
| Redis | cart session storage | pulled as base image, no dedicated dir |
| RabbitMQ | payment ↔ dispatch messaging | pulled as base image, no dedicated dir |

## 3. Deployment Targets (practice one at a time)

| Platform | Path | Notes |
|---|---|---|
| Docker Compose | [docker-compose.yaml](docker-compose.yaml), [docker-compose-load.yaml](docker-compose-load.yaml) | Easiest starting point, run everything locally |
| Kubernetes (generic) | [K8s/](K8s/) | Includes Helm chart, Istio manifests |
| AWS EKS | [EKS/](EKS/) | Step-by-step docs: prerequisites → cluster → OIDC/IAM → ALB → EBS CSI |
| Azure AKS | [AKS/](AKS/) | Helm-based |
| Google GKE | [GKE/](GKE/) | Helm-based |
| Docker Swarm | [Swarm/](Swarm/) | `create-swarm.sh`, `deploy.sh` |
| Mesos / DC/OS | [DCOS/](DCOS/) | `deploy.sh`, `destroy.sh` |
| OpenShift | [OpenShift/](OpenShift/) | `setup.sh` |
| CI | [.github/workflows/](.github/workflows/) | GitHub Actions |

## 4. Run Completely Locally (Windows laptop)

### Prerequisites

- **Docker Desktop** installed, with the **WSL2** backend enabled (Docker Desktop → Settings → General → "Use the WSL 2 based engine").
- Virtualization enabled in BIOS (required for WSL2/Hyper-V).
- At least **4-6 GB RAM** allocated to Docker Desktop (Settings → Resources) — 10 containers (app services + Mongo + MySQL + Redis + RabbitMQ) run at once.
- Free port **8080** on your laptop (that's what the `web` service publishes).

Verify Docker works:

```powershell
docker --version
docker compose version
```

### Step 1 — go to the compose project folder

```powershell
cd d:\robot-shop-project\three-tier-architecture-robot-shop
```

### Step 2 — get the images

The [.env](.env) file already points at the official pre-built images (`REPO=robotshop`, `TAG=2.1.0`), so you don't have to build anything. Fastest path:

```powershell
docker compose pull
```

Only build from source instead if you want to modify a service's code:

```powershell
docker compose build
```
(The `web` build references `INSTANA_AGENT_KEY` as a build arg, but it's unused by [web/Dockerfile](web/Dockerfile) — safe to leave unset for a plain local run.)

### Step 3 — start everything

```powershell
docker compose up -d
```

`-d` runs it in the background. Watch it come up:

```powershell
docker compose ps
docker compose logs -f web
```

Wait until all services show `healthy` in `docker compose ps` (mysql/mongo take a bit longer to initialize on first run).

### Step 4 — open the store

Browse to **http://localhost:8080**

### Step 5 (optional) — generate load

```powershell
docker compose -f docker-compose.yaml -f docker-compose-load.yaml up -d
```

### Stopping / cleaning up

```powershell
docker compose down          # stop and remove containers
docker compose down -v       # also wipe Mongo/MySQL data volumes (fresh start next time)
```

### Troubleshooting

| Symptom | Fix |
|---|---|
| Port 8080 already in use | Stop whatever else is using it, or edit the `ports:` mapping for `web` in [docker-compose.yaml](docker-compose.yaml) (e.g. `8081:8080`) |
| Containers keep restarting / OOM | Increase Docker Desktop memory limit (Settings → Resources → Memory) |
| `mysql`/`mongodb` unhealthy for a long time | First-run DB init can take 30-60s — check `docker compose logs mysql` / `docker compose logs mongodb` |
| `docker compose` not recognized | Use `docker-compose` (older standalone binary) instead, syntax is the same here |

## 5. Suggested Practice Path (broader, beyond local)

1. **Docker Compose** — `docker-compose build`, `docker-compose up`, hit `http://localhost:8080`.
2. **Load generation** — run [load-gen/](load-gen/) against the compose stack.
3. **Kubernetes basics** — deploy via [K8s/helm/](K8s/helm/) on minikube/kind.
4. **Cloud Kubernetes** — pick one of EKS/AKS/GKE and follow its docs end-to-end.
5. **Service mesh** — try [K8s/Istio/](K8s/Istio/) on top of the K8s deployment.
6. **Observability** — Prometheus metrics on cart/payment (`/metrics`), fluentd log forwarding.
7. **Alternate orchestrators** — Swarm, DC/OS, OpenShift for comparison.

## 6. Quick Commands

```shell
# Metrics check
curl http://localhost:8080/api/cart/metrics
curl http://localhost:8080/api/payment/metrics

# Kubernetes (minikube)
minikube ip
kubectl get svc web
```
