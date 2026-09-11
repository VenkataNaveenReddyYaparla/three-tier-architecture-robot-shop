# DevOps Practice Roadmap — Stan's Robot Shop

A single, self-contained curriculum for using this repo as an end-to-end DevOps training ground: what the app is made of, how to run it locally, and 13 disciplines (Docker → Compose → shell → CI/CD → Kubernetes → Istio → Terraform → AWS EKS → Ansible → AKS/GKE → Observability → alternate orchestrators → GitOps), each with a **plan** (what you're building toward and why) and a **step-by-step implementation** (actual commands/config to run in this repo).

This replaces `PRACTICE_GUIDE.md`, which now just points here.

> **Standing rule for this repo:** `dispatch.yaml`, `payment.yaml`, `shipping.yaml`, and `user.yaml` under `.github/workflows/` are broken **on purpose** (leftover copy-paste stubs). They are a debugging exercise (Discipline 4) — don't fix them without being asked. `cart.yaml` and `catalogue.yaml` are the already-fixed reference versions.

---

## 0. What's in the app

### 0.1 Services

| Service | Tech | Path | What it's good for practicing |
|---|---|---|---|
| web | Nginx + AngularJS (1.x) static frontend | [web/](web/) | Reverse-proxy config, static asset serving, CI for a non-compiled service |
| cart | Node.js / Express | [cart/server.js](cart/server.js) | REST API, Redis session/state, fixed CI reference |
| catalogue | Node.js / Express | [catalogue/server.js](catalogue/server.js) | MongoDB queries, REST API, fixed CI reference (immutable tags) |
| user | Node.js / Express | [user/server.js](user/server.js) | Auth, MongoDB, sessions, **broken CI stub** |
| shipping | Java / Spring Boot (Maven) | [shipping/pom.xml](shipping/pom.xml) | JPA, MySQL, multi-stage JVM builds, **broken CI stub** |
| ratings | PHP 7.4 / Apache (Composer) | [ratings/Dockerfile](ratings/Dockerfile) | PHP-Apache container, MySQL, **no CI workflow at all** |
| payment | Python / Flask | [payment/payment.py](payment/payment.py) | RabbitMQ producer/consumer, Prometheus metrics, **broken CI stub** |
| dispatch | Go | [dispatch/main.go](dispatch/main.go) | RabbitMQ consumer, Go builds, **broken CI stub** |
| load-gen | Python / Locust | [load-gen/robot-shop.py](load-gen/robot-shop.py) | Load testing / traffic generation (used across almost every discipline below) |
| fluentd | Log forwarding | [fluentd/](fluentd/) | Centralized logging pipelines |

### 0.2 Data stores / messaging

| Component | Purpose | Path |
|---|---|---|
| MongoDB | catalogue + user data | [mongo/](mongo/) |
| MySQL (+ MaxMind data) | shipping + ratings data | [mysql/](mysql/) |
| Redis | cart session storage | pulled as base image, no dedicated dir |
| RabbitMQ | payment ↔ dispatch messaging | pulled as base image, no dedicated dir |

### 0.3 Deployment targets

| Platform | Path | Status |
|---|---|---|
| Docker Compose | [docker-compose.yaml](docker-compose.yaml), [docker-compose-load.yaml](docker-compose-load.yaml) | Ready today |
| Kubernetes (generic) | [K8s/](K8s/) | Ready today (Helm chart + Istio manifests) |
| AWS EKS | [EKS/](EKS/) | Numbered docs only, no live cluster |
| Azure AKS | [AKS/](AKS/) | Helm chart only, no setup docs |
| Google GKE | [GKE/](GKE/) | Helm chart only, no setup docs |
| Docker Swarm | [Swarm/](Swarm/) | Scripts only, no docs |
| Mesos / DC/OS | [DCOS/](DCOS/) | Scripts + JSON manifests, no docs |
| OpenShift | [OpenShift/](OpenShift/) | Script + README |
| CI | [.github/workflows/](.github/workflows/) | 6/8 services covered, 4 broken by design |
| Terraform | *not present* | Build it yourself (Discipline 7) |
| Ansible | *not present* | Build it yourself (Discipline 9) |
| Observability stack | *not present (endpoints exist)* | Build it yourself (Discipline 11) |
| GitOps | *not present* | Build it yourself (Discipline 13, stretch) |

---

## Suggested order

| # | Discipline | Status |
|---|---|---|
| 1 | [Docker fundamentals](#1-docker-fundamentals) | Ready today |
| 2 | [Docker Compose](#2-docker-compose) | Ready today |
| 3 | [Shell scripting](#3-shell-scripting) | Ready today (extend existing scripts) |
| 4 | [CI/CD pipelines](#4-cicd-pipelines) | Partially broken by design |
| 5 | [Kubernetes (generic)](#5-kubernetes-generic) | Ready today |
| 6 | [Service mesh (Istio)](#6-service-mesh-istio) | Ready today |
| 7 | [Infrastructure as Code — Terraform](#7-infrastructure-as-code--terraform) | Build it yourself |
| 8 | [Cloud Kubernetes — AWS EKS](#8-cloud-kubernetes--aws-eks) | Docs/templates only |
| 9 | [Configuration management — Ansible](#9-configuration-management--ansible) | Build it yourself |
| 10 | [Cross-cloud port — AKS / GKE](#10-cross-cloud-port--aks--gke) | Templates only |
| 11 | [Observability](#11-observability) | Partial |
| 12 | [Alternate orchestrators](#12-alternate-orchestrators--comparison-pass) | Scripts only |
| 13 | [GitOps (stretch)](#13-stretch--gitops) | Not present |

---

## 1. Docker fundamentals

**Plan:** before touching Compose or CI, understand what each service's `Dockerfile` actually does, and build muscle memory for the base commands everything else is built on.

**Prerequisites:** Docker Desktop installed, WSL2 backend enabled (Settings → General → "Use the WSL 2 based engine"), virtualization on in BIOS.
```powershell
docker --version
docker compose version
```

**Implementation:**
1. Build each service image by hand and time it:
   ```powershell
   cd shipping; docker build -t shipping-local .     # Java/Maven — multi-stage
   cd ../dispatch; docker build -t dispatch-local .   # Go — multi-stage
   cd ../payment; docker build -t payment-local .     # Python — single-stage
   cd ../ratings; docker build -t ratings-local .     # PHP/Apache + Composer
   cd ../cart; docker build -t cart-local .           # Node
   cd ../catalogue; docker build -t catalogue-local . # Node
   cd ../user; docker build -t user-local .           # Node
   cd ../web; docker build -t web-local .             # Nginx static
   ```
2. Compare results:
   ```powershell
   docker images | Select-String "local"
   docker history shipping-local
   docker history payment-local
   ```
   Note which Dockerfiles use `FROM ... AS build` (multi-stage) vs. a single `FROM` — Java/Go should be multi-stage (compile then copy the artifact into a slim runtime image); Python/PHP typically aren't since there's nothing to compile.
3. Rebuild one image with `--no-cache` and compare build time against the cached rebuild, to feel what Docker's layer cache actually buys you:
   ```powershell
   docker build --no-cache -t shipping-local2 shipping
   ```
4. Tag and push one image to your own Docker Hub repo to practice registry auth:
   ```powershell
   docker login
   docker tag cart-local <your-dockerhub-username>/cart-practice:v1
   docker push <your-dockerhub-username>/cart-practice:v1
   ```

**Verification:** `docker images` shows all 8 local images; `docker history` output clearly differs (layer count/size) between multi-stage and single-stage builds; the pushed tag is visible on hub.docker.com.

**What you get:** image layering and caching, multi-stage builds for compiled languages vs. interpreted ones, registry login/push — the foundation every later track sits on top of.

---

## 2. Docker Compose

**Plan:** run the full 10-container stack (8 services + Mongo + MySQL + Redis + RabbitMQ) on one host, then intentionally break a dependency to see the graph in action.

**Prerequisites:** free port 8080, 4–6 GB RAM allocated to Docker Desktop (Settings → Resources).

**Implementation:**
1. Go to the compose project folder:
   ```powershell
   cd D:\robot-shop-project\three-tier-architecture-robot-shop
   ```
2. Pull the pre-built images (`.env` already sets `REPO=robotshop`, `TAG=2.1.0`, so nothing needs building):
   ```powershell
   docker compose pull
   ```
   Only build from source if you're modifying a service's code: `docker compose build` (the `web` build references an unused `INSTANA_AGENT_KEY` build arg — safe to leave unset).
3. Start everything in the background and watch it come up:
   ```powershell
   docker compose up -d
   docker compose ps
   docker compose logs -f web
   ```
   Wait until every service shows `healthy` (Mongo/MySQL take 30–60s longer on first run).
4. Open the store at **http://localhost:8080**.
5. Generate load against it:
   ```powershell
   docker compose -f docker-compose.yaml -f docker-compose-load.yaml up -d
   ```
6. Break the dependency graph on purpose:
   ```powershell
   docker compose stop redis
   ```
   Refresh the site, add something to the cart, and watch which service fails — then `docker compose start redis` and confirm recovery.
7. Scale a stateless service and see Compose's default networking load-balance across replicas:
   ```powershell
   docker compose up -d --scale catalogue=3
   ```
8. Tear down:
   ```powershell
   docker compose down          # stop + remove containers
   docker compose down -v       # also wipe Mongo/MySQL volumes for a clean slate
   ```

**Verification / quick commands:**
```shell
curl http://localhost:8080/api/cart/metrics
curl http://localhost:8080/api/payment/metrics
```

**Troubleshooting:**

| Symptom | Fix |
|---|---|
| Port 8080 already in use | Stop whatever's using it, or remap `web`'s `ports:` in [docker-compose.yaml](docker-compose.yaml) (e.g. `8081:8080`) |
| Containers keep restarting / OOM | Increase Docker Desktop memory limit |
| `mysql`/`mongodb` unhealthy for a long time | First-run DB init takes 30–60s — check `docker compose logs mysql` / `mongodb` |
| `docker compose` not recognized | Use `docker-compose` (older standalone binary), same syntax |

**What you get:** multi-container orchestration on one host — service discovery via Compose's internal DNS, startup ordering/healthchecks, named volumes for stateful services, scaling replicas, and how a load-gen overlay composes with a base file.

---

## 3. Shell scripting

**Plan:** treat Bash as its own discipline, not an incidental skill — it's the glue every CI step, Terraform provisioner, and Ansible `shell`/`command` module quietly relies on.

**Implementation:**
1. Read the existing scripts end to end and explain each line before changing anything: [pullbaseimages.sh](pullbaseimages.sh) (loops every `Dockerfile`, extracts its `FROM` image, pulls it), [Swarm/create-swarm.sh](Swarm/create-swarm.sh), [Swarm/deploy.sh](Swarm/deploy.sh), [DCOS/deploy.sh](DCOS/deploy.sh) / [DCOS/destroy.sh](DCOS/destroy.sh), [OpenShift/setup.sh](OpenShift/setup.sh).
2. Write a healthcheck-polling script that doesn't exist yet, e.g. `scripts/wait-for-healthy.sh`:
   ```bash
   #!/usr/bin/env bash
   set -euo pipefail
   TIMEOUT=120
   ELAPSED=0
   while true; do
     UNHEALTHY=$(docker compose ps --format json | grep -c '"Health":"unhealthy"' || true)
     STARTING=$(docker compose ps --format json | grep -c '"Health":"starting"' || true)
     if [ "$UNHEALTHY" -eq 0 ] && [ "$STARTING" -eq 0 ]; then
       echo "All services healthy after ${ELAPSED}s"; exit 0
     fi
     if [ "$ELAPSED" -ge "$TIMEOUT" ]; then
       echo "Timed out waiting for healthy services"; docker compose logs --tail=50; exit 1
     fi
     sleep 5; ELAPSED=$((ELAPSED + 5))
   done
   ```
3. Write a "reset everything" script, e.g. `scripts/reset.sh`:
   ```bash
   #!/usr/bin/env bash
   set -euo pipefail
   docker compose down -v
   docker compose pull
   docker compose up -d
   ./scripts/wait-for-healthy.sh
   ```
4. Run both against the stack from Discipline 2 and confirm they behave correctly when you break something first (stop a service, then run the reset script).

**What you get:** argument handling, `set -euo pipefail` discipline, exit codes, polling loops with timeouts — the automation glue every other discipline in this list depends on.

---

## 4. CI/CD pipelines

**Plan, in two stages:**

**Stage 1 — debug by reading, don't run.** [cart.yaml](.github/workflows/cart.yaml) and [catalogue.yaml](.github/workflows/catalogue.yaml) are the fixed reference versions (real commit history shows the fixes applied: `actions/checkout@v7`, `--password-stdin` login instead of `-p`, `${{ github.run_number }}` image tags). Compare them line-by-line against the 4 broken stubs and write down *every* reason each would fail if it ran:
- [dispatch.yaml](.github/workflows/dispatch.yaml) — `runs-on: [self-hosted]` with no runner attached; the "Go install" step pipes `curl` into nothing (dangling `|`); job name still says `payment` (copy-paste leftover); comment says "Only trigger on changes to Python files" for a `.go` path filter.
- [payment.yaml](.github/workflows/payment.yaml) — same `self-hosted`/no-runner problem; installs `python3` via `apt-get` even though the Docker build already handles this; `${name}` is never defined anywhere in the workflow, so the tag/push step fails.
- [shipping.yaml](.github/workflows/shipping.yaml) — `sudo apt-get install openjdk 17` is invalid apt syntax (package is `openjdk-17-jdk`, not `openjdk` + a version argument); same `${name}` and `self-hosted` issues.
- [user.yaml](.github/workflows/user.yaml) — same `${name}`/`self-hosted` issues, plus a literal typo: `docker user ${name}/user:latest` instead of `docker push`.

Do this stage entirely on paper/in your head — the point is reading a pipeline like a compiler would, not running it.

**Stage 2 — extend.** Once the debugging pass is written down, create the two workflows that don't exist at all — `ratings` and `web` — using the fixed `cart.yaml` as your template:
```yaml
# .github/workflows/ratings.yaml (starting point — adapt paths/build steps for PHP)
name: Ratings
on:
  push:
    branches: [master]
    paths: ['ratings/**', '.github/workflows/ratings.yaml']
  pull_request:
    branches: [master]
    paths: ['ratings/**', '.github/workflows/ratings.yaml']
jobs:
  ratings:
    name: Build and push ratings microservice
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ratings
    steps:
      - uses: actions/checkout@v7
      - name: Build Docker image
        run: docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/ratings:${{ github.run_number }} .
      - name: Docker login
        run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u ${{ secrets.DOCKERHUB_USERNAME }} --password-stdin
      - name: Docker push
        run: docker push ${{ secrets.DOCKERHUB_USERNAME }}/ratings:${{ github.run_number }}
```
Repeat for `web` (build context is the static Nginx site — no compiled-language build step needed). Add `DOCKERHUB_USERNAME`/`DOCKERHUB_TOKEN` as repo secrets under Settings → Secrets and variables → Actions if not already present, then push a trivial change under `ratings/` or `web/` to confirm the new workflow actually triggers and goes green.

**What you get:** pipeline triggers (`paths:` filters scoped per service), GitHub Actions secrets, image-tagging strategy (`latest`/mutable vs. `${{ github.run_number }}`/immutable), and the difference between `self-hosted` and `ubuntu-latest` runners.

---

## 5. Kubernetes (generic)

**Plan:** get the whole app running on a local cluster via the existing Helm chart, then exercise quotas and autoscaling under real load.

**Prerequisites:** minikube or kind, `kubectl`, `helm` v3.

**Implementation:**
1. Start a local cluster and install the chart ([K8s/helm/README.md](K8s/helm/README.md)):
   ```shell
   minikube start --cpus=4 --memory=6g
   kubectl create ns robot-shop
   helm install robot-shop --namespace robot-shop K8s/helm --set nodeport=true
   kubectl -n robot-shop get pods -w
   ```
2. Reach the site:
   ```shell
   minikube ip
   kubectl -n robot-shop get svc web
   # browse http://<minikube-ip>:<web-nodeport>
   ```
3. Apply the resource quota so the namespace can't over-allocate:
   ```shell
   kubectl -n robot-shop apply -f K8s/resource-quota.yaml
   kubectl -n robot-shop describe resourcequota
   ```
4. Read and run the autoscale helper, then drive load at it with `load-gen/` (build/run per [load-gen/README.md](load-gen/README.md)) and watch the HPA react:
   ```shell
   K8s/autoscale.sh
   kubectl -n robot-shop get hpa -w
   ```
5. Try `K8s/load-deployment.yaml` as an alternate one-off load source and compare its effect on the HPA to the Locust-based `load-gen`.

**What you get:** Helm chart structure and value overrides (`image.repo`, `image.version`, `nodeport`, `payment.gateway`, per-workload `affinity`/`tolerations`/`nodeSelector` — see [K8s/helm/README.md](K8s/helm/README.md)), namespace resource quotas, and horizontal pod autoscaling under self-generated load.

---

## 6. Service mesh (Istio)

**Plan:** layer a service mesh on top of the cluster from Discipline 5 and use it to do a canary release instead of a plain rolling update.

**Implementation:**
1. Install Istio on the same cluster and label the namespace for sidecar injection:
   ```shell
   istioctl install --set profile=demo -y
   kubectl label namespace robot-shop istio-injection=enabled
   kubectl -n robot-shop rollout restart deployment
   ```
2. Replace the plain NodePort/ingress with the mesh gateway:
   ```shell
   kubectl -n robot-shop apply -f K8s/Istio/gateway.yaml
   ```
3. Apply the payment fix manifest and the canary split, then confirm both `payment` versions are actually receiving traffic:
   ```shell
   kubectl -n robot-shop apply -f K8s/Istio/payment-deployment-fix.yaml
   kubectl -n robot-shop apply -f K8s/Istio/canary.yaml
   for i in $(seq 1 20); do curl -s http://<gateway-ip>/api/payment/health; done
   ```
4. Shift the traffic weight in `canary.yaml` (e.g. 90/10 → 50/50 → 0/100) and re-run the loop each time to watch the split move.

**What you get:** ingress gateways, weighted traffic-splitting/canary releases, and the basics of a sidecar-injected mesh — the layer that sits on top of "Kubernetes already works."

---

## 7. Infrastructure as Code — Terraform

**Plan:** stop clicking/CLI-ing through the manual steps in [EKS/](EKS/) and provision the same cluster declaratively and reproducibly.

**Implementation:**
1. Create a `terraform/` directory at the repo root with a standard module layout:
   ```
   terraform/
     main.tf
     variables.tf
     outputs.tf
     modules/
       vpc/
       eks/
   ```
2. `main.tf` — provision what [EKS/01-prerequisites.md](EKS/01-prerequisites.md) through [EKS/05-ebs-csi-driver.md](EKS/05-ebs-csi-driver.md) currently describe by hand, using the well-known community modules as your starting point rather than writing raw resources from scratch:
   ```hcl
   terraform {
     required_providers {
       aws = { source = "hashicorp/aws", version = "~> 5.0" }
     }
   }
   provider "aws" { region = var.aws_region }

   module "vpc" {
     source  = "terraform-aws-modules/vpc/aws"
     version = "~> 5.0"
     name    = "${var.cluster_name}-vpc"
     cidr    = "10.0.0.0/16"
     azs             = ["${var.aws_region}a", "${var.aws_region}b"]
     private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
     public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
     enable_nat_gateway = true
   }

   module "eks" {
     source  = "terraform-aws-modules/eks/aws"
     version = "~> 20.0"
     cluster_name    = var.cluster_name
     cluster_version = "1.29"
     vpc_id     = module.vpc.vpc_id
     subnet_ids = module.vpc.private_subnets

     enable_irsa = true   # this is the OIDC/IAM piece from EKS/03-oidc-IAM.md

     eks_managed_node_groups = {
       default = {
         instance_types = ["t3.medium"]
         min_size = 1, max_size = 3, desired_size = 2
       }
     }
   }
   ```
3. `variables.tf` / `outputs.tf` — parameterize `cluster_name`/`aws_region`, and output `cluster_endpoint`, `cluster_name`, and the OIDC provider ARN so later steps (ALB controller IRSA role, EBS CSI IRSA role) can reference them.
4. Provision and prove it's reproducible:
   ```shell
   cd terraform
   terraform init
   terraform plan -out=tfplan
   terraform apply tfplan
   aws eks update-kubeconfig --name <cluster_name> --region <aws_region>
   kubectl get nodes
   terraform destroy   # confirm it tears down cleanly too
   ```

**What you get:** declarative infrastructure, remote/local state, and the single skill that turns the existing EKS docs from "a runbook you follow by hand" into "code you run." Usually the single highest-leverage track on this list for job relevance.

---

## 8. Cloud Kubernetes — AWS EKS

**Plan:** run the actual numbered docs end-to-end against a real cluster — ideally the one Terraform (Discipline 7) just created — and deploy the app with cloud-native ingress and storage instead of NodePort/local volumes.

**Implementation:**
1. Work through the docs in order, cross-checking each against [eks-robot-shop-runbook.pdf](eks-robot-shop-runbook.pdf) at the repo root: [EKS/01-prerequisites.md](EKS/01-prerequisites.md) → [EKS/02-eks-cluster-setup.md](EKS/02-eks-cluster-setup.md) → [EKS/03-oidc-IAM.md](EKS/03-oidc-IAM.md) → [EKS/04-alb-configuration.md](EKS/04-alb-configuration.md) → [EKS/05-ebs-csi-driver.md](EKS/05-ebs-csi-driver.md). They're thin (13–59 lines) — treat them as a checklist, filling gaps from AWS's own docs where a step is under-specified.
2. Deploy the app with the EKS-specific Helm values (ALB ingress + `gp2` storage class):
   ```shell
   kubectl create ns robot-shop
   helm install robot-shop --namespace robot-shop EKS/helm \
     --set redis.storageClassName=gp2
   kubectl -n robot-shop get ingress
   ```
3. Confirm the AWS Load Balancer Controller actually provisioned an ALB, and that Redis's PVC is bound via the EBS CSI driver:
   ```shell
   kubectl -n robot-shop get ingress web -o wide   # ALB hostname
   kubectl -n robot-shop get pvc
   ```
4. Tear down in reverse order when done (`helm uninstall`, then `terraform destroy` from Discipline 7) so nothing keeps billing.

**What you get:** real cloud Kubernetes: IAM-roles-for-service-accounts via OIDC, the AWS Load Balancer Controller driving an ALB from Ingress objects, and the EBS CSI driver for persistent volumes — the pieces that don't exist on minikube.

---

## 9. Configuration management — Ansible

**Plan:** use Ansible for the layer Terraform shouldn't own — configuring what already exists, idempotently, instead of one-shot shell scripts.

**Implementation:**
1. Create an `ansible/` directory:
   ```
   ansible/
     inventory.ini
     bootstrap-docker-host.yml
     roles/
   ```
2. `inventory.ini`:
   ```ini
   [docker_hosts]
   node1 ansible_host=<ec2-public-ip> ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
   ```
3. `bootstrap-docker-host.yml` — take a bare EC2/VM instance to a working Docker host, idempotently (the config-management equivalent of what [pullbaseimages.sh](pullbaseimages.sh) does with plain shell):
   ```yaml
   - hosts: docker_hosts
     become: true
     tasks:
       - name: Install Docker
         apt:
           name: docker.io
           state: present
           update_cache: true

       - name: Ensure docker service is running
         systemd:
           name: docker
           state: started
           enabled: true

       - name: Add user to docker group
         user:
           name: "{{ ansible_user }}"
           groups: docker
           append: true

       - name: Pull base images used by every Dockerfile in the repo
         community.docker.docker_image:
           name: "{{ item }}"
           source: pull
         loop:
           - node:14-alpine
           - openjdk:17-jdk-slim
           - python:3.9-slim
           - golang:1.18
           - php:7.4-apache
   ```
   Run it, then run it again unchanged — confirm the second run reports no changes (that's idempotency working).
   ```shell
   ansible-playbook -i ansible/inventory.ini ansible/bootstrap-docker-host.yml
   ansible-playbook -i ansible/inventory.ini ansible/bootstrap-docker-host.yml   # should show 0 changed
   ```
4. Second exercise: rewrite [OpenShift/setup.sh](OpenShift/setup.sh) (or [DCOS/deploy.sh](DCOS/deploy.sh)) as an idempotent playbook instead of an imperative script, and verify it produces the same end state no matter how many times it runs.

**What you get:** the actual Terraform-vs-Ansible boundary (provisioning infrastructure vs. configuring what runs on it), idempotency, and inventory-based targeting of multiple hosts.

---

## 10. Cross-cloud port — AKS / GKE

**Plan:** [AKS/helm/](AKS/helm/) and [GKE/helm/](GKE/helm/) are Helm-only with no setup docs, and their `values.yaml` is nearly identical to EKS's (mainly differing by `storageClassName`). Use the EKS docs and Terraform module from Disciplines 7–8 as your template and write the missing equivalents yourself.

**Implementation (AKS):**
```hcl
# terraform/modules/aks/main.tf — same shape as the EKS module, different provider
resource "azurerm_kubernetes_cluster" "this" {
  name                = var.cluster_name
  location            = var.location
  resource_group_name = azurerm_resource_group.this.name
  dns_prefix          = var.cluster_name
  default_node_pool {
    name       = "default"
    node_count = 2
    vm_size    = "Standard_DS2_v2"
  }
  identity { type = "SystemAssigned" }
}
```
```shell
az aks get-credentials --resource-group <rg> --name <cluster_name>
helm install robot-shop --namespace robot-shop AKS/helm --set redis.storageClassName=default
```

**Implementation (GKE):**
```hcl
resource "google_container_cluster" "this" {
  name     = var.cluster_name
  location = var.region
  initial_node_count = 2
}
```
```shell
gcloud container clusters get-credentials <cluster_name> --region <region>
helm install robot-shop --namespace robot-shop GKE/helm --set redis.storageClassName=standard
```

For both, write a short setup doc mirroring the EKS numbered-docs style (prereqs → cluster → identity/IAM equivalent → ingress controller → storage class) so the three cloud directories end up documented consistently.

**What you get:** this is where "cloud-agnostic" stops being a buzzword — you're translating one cloud's IaC/K8s pattern to another provider's primitives and feeling exactly where the abstraction leaks (storage classes, ingress controllers, IAM equivalents).

---

## 11. Observability

**Plan:** the app already exposes Prometheus metrics and forwards logs via fluentd — build the missing dashboard/log-store layer on top and drive real traffic through it.

**Implementation:**
1. Confirm the existing endpoints work (see [README.md](README.md#prometheus)):
   ```shell
   curl http://localhost:8080/api/cart/metrics
   curl http://localhost:8080/api/payment/metrics
   ```
2. Stand up Prometheus + Grafana as a sidecar Compose file, e.g. `observability/docker-compose.yaml`:
   ```yaml
   services:
     prometheus:
       image: prom/prometheus:latest
       volumes:
         - ./prometheus.yml:/etc/prometheus/prometheus.yml
       ports: ["9090:9090"]
     grafana:
       image: grafana/grafana:latest
       ports: ["3000:3000"]
   ```
   `observability/prometheus.yml`:
   ```yaml
   scrape_configs:
     - job_name: cart
       metrics_path: /api/cart/metrics
       static_configs: [{ targets: ["host.docker.internal:8080"] }]
     - job_name: payment
       metrics_path: /api/payment/metrics
       static_configs: [{ targets: ["host.docker.internal:8080"] }]
   ```
   ```shell
   docker compose -f observability/docker-compose.yaml up -d
   # Prometheus UI: http://localhost:9090  |  Grafana: http://localhost:3000 (admin/admin)
   ```
3. Add Prometheus as a Grafana data source, build one dashboard panel per metric (cart items-added counter, payment purchase counter, payment cart-value histogram).
4. Point `fluentd/`'s output at something queryable instead of the default sink — even a minimal Loki container — so logs are searchable, not just forwarded into the void.
5. Drive real traffic through [load-gen/](load-gen/) (Locust) while watching the dashboard update live.

**What you get:** a full metrics-and-logs pipeline, dashboard-building, and the habit of watching a system under generated load instead of guessing at its behavior.

---

## 12. Alternate orchestrators — comparison pass

**Plan:** deploy the same app to Swarm, DC/OS, and OpenShift and note what each does differently from the Kubernetes concepts you already know from Disciplines 5–6.

**Implementation:**
1. **Swarm:**
   ```shell
   Swarm/create-swarm.sh
   Swarm/deploy.sh
   docker service ls
   ```
2. **DC/OS:** review the per-service JSON manifests under [DCOS/manifest/](DCOS/manifest/) (one per service — this is Marathon's app-definition style, contrasted with a single Helm chart), then:
   ```shell
   DCOS/deploy.sh
   # ... explore, then:
   DCOS/destroy.sh
   ```
3. **OpenShift:** follow [OpenShift/README.md](OpenShift/README.md) — `oc login`, `oc new-project`, then install the same K8s Helm chart with the OpenShift flag on:
   ```shell
   helm install robot-shop --set openshift=true K8s/helm
   ```
   Note the Route/Project model layered on top of the same underlying Kubernetes primitives.

**What you get:** enough orchestrator breadth to recognize the same underlying concepts (service discovery, scaling, rolling updates) across platforms, instead of only knowing them through Kubernetes' vocabulary.

---

## 13. (Stretch) GitOps

**Plan:** close the loop CI (Discipline 4) doesn't — instead of CI directly running `kubectl apply`/`helm upgrade`, let a GitOps controller reconcile the cluster to match what's in git.

**Implementation:**
1. Install Argo CD on the cluster from Discipline 5:
   ```shell
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   kubectl -n argocd port-forward svc/argocd-server 8081:443
   ```
2. Point an Argo CD `Application` at this repo's Helm chart:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: robot-shop
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: <this-repo-url>
       path: K8s/helm
       targetRevision: master
     destination:
       server: https://kubernetes.default.svc
       namespace: robot-shop
     syncPolicy:
       automated: { prune: true, selfHeal: true }
   ```
3. Change a value in `K8s/helm/values.yaml`, push it, and watch Argo CD pick up the diff and reconcile automatically — no CI deploy step required.

**What you get:** the push-CI vs. pull-GitOps distinction, and exposure to the pattern most production Kubernetes shops actually use for delivery, one step beyond "CI also does the deploy."
