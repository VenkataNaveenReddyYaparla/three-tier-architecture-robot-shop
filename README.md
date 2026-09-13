# Stan's Robot Shop — Three-Tier Microservices on Docker, Kubernetes & AWS EKS

A polyglot microservices e-commerce application used as a hands-on sandbox for container orchestration, CI/CD and observability. Eight services in six languages, four data stores, one nginx front door — deployable to plain Docker, Compose, Kubernetes, EKS, AKS, GKE, OpenShift, Swarm and DC/OS from this single repo.

> Originally [Stan's Robot Shop](https://www.instana.com/blog/stans-robot-shop-sample-microservice-application/) by Instana. This fork adds CI pipelines that publish to a personal registry, a local-run path, a refactored EKS chart, and a full learning path in **[DEVOPS_ROADMAP.md](DEVOPS_ROADMAP.md)**.
>
> Error handling is patchy and there is no security built in. That is deliberate — the defects are the teaching material.

---

## Table of contents

- [Architecture](#architecture)
- [Technologies](#technologies)
- [Repository structure](#repository-structure)
- [Where the images come from](#where-the-images-come-from)
- [Running it](#running-it)
- [CI/CD](#cicd)
- [Observability](#observability)
- [Load generation](#load-generation)
- [Learning path](#learning-path)

---

## Architecture

A browser talks to **one** entry point. `web` (nginx) serves the AngularJS UI and reverse-proxies every `/api/*` path to the service that owns it. Each service owns its own data store — no service reads another's database.

```
                    browser  (http://localhost:8080)
                        |
                  +-----------+
                  |    web    |   nginx, serves the UI
                  +-----------+   and reverse-proxies /api/*
                        |
   +--------+-----------+-----------+----------+----------+
   |        |           |           |          |          |
catalogue  user        cart      shipping   ratings    payment
 (Node)   (Node)      (Node)     (Java)      (PHP)     (Python)
   |        |  \        |           |          |          |
   |        |   \       |           |          |          v
 MongoDB  MongoDB \   Redis       MySQL      MySQL     RabbitMQ
                   \                                      |
                    Redis                                 v
                                                       dispatch (Go)
```

### Routing table

[web/default.conf.template](web/default.conf.template) is the real routing table, rendered at container start by `envsubst` in [web/entrypoint.sh](web/entrypoint.sh):

| Path | Backend |
|---|---|
| `/api/catalogue/` | `catalogue:8080` |
| `/api/user/` | `user:8080` |
| `/api/cart/` | `cart:8080` |
| `/api/shipping/` | `shipping:8080` |
| `/api/payment/` | `payment:8080` |
| `/api/ratings/` | `ratings:80` ← **port 80, the only one** |
| `/nginx_status` | nginx stub status |

> **Service names are load-bearing.** [web/Dockerfile](web/Dockerfile) bakes in `CATALOGUE_HOST=catalogue`, `CART_HOST=cart` and so on as env defaults. Docker and Kubernetes both resolve a service's name as its hostname, so renaming a service in any manifest returns 502 for that route until you also change the matching `*_HOST` variable.

---

## Technologies

| Service | Language / runtime | Framework | Base image | Data store | Port |
|---|---|---|---|---|---|
| **web** | nginx + AngularJS 1.x | — | `nginx:1.21.6` | — | 8080 |
| **catalogue** | Node.js 20 | [Express](http://expressjs.com/) | `node:20-alpine` | MongoDB | 8080 |
| **user** | Node.js 20 | Express | `node:20-alpine` | MongoDB + Redis | 8080 |
| **cart** | Node.js 20 | Express | `node:20-alpine` | Redis | 8080 |
| **shipping** | Java / JRE 25 | [Spring Boot](https://spring.io/) | `maven:3.6.3-jdk-8` → `eclipse-temurin:25-jre-alpine` | MySQL | 8080 |
| **ratings** | PHP 7.4 | Symfony + Apache | `php:7.4-apache` | MySQL | 80 |
| **payment** | Python 3.9 | [Flask](http://flask.pocoo.org) + uWSGI | `python:3.9` | RabbitMQ (publisher) | 8080 |
| **dispatch** | Go 1.24 | — | `golang:1.24` → `alpine` | RabbitMQ (consumer) | — |

**Data stores:** MongoDB 5 (catalogue + users), MySQL 5.7 ([Maxmind](http://www.maxmind.com) geo data for shipping), Redis 6.2 (cart + sessions), RabbitMQ 3.8 (order queue).

**Infrastructure:** Docker, Docker Compose, Kubernetes, Helm 3, GitHub Actions, AWS EKS + ALB Ingress Controller + EBS CSI, Istio (optional), Prometheus, Fluentd.

---

## Repository structure

```
.
├── .github/workflows/       8 CI pipelines, one per service
│
├── catalogue/ user/ cart/   Node.js services
├── shipping/                Java / Spring Boot
├── ratings/                 PHP / Symfony
├── payment/                 Python / Flask
├── dispatch/                Go
├── web/                     nginx + AngularJS UI
│   ├── default.conf.template    routing, rendered by envsubst at start
│   └── entrypoint.sh
│
├── mongo/                   MongoDB image + SEED DATA (catalogue.js, users.js)
├── mysql/                   MySQL image + SEED DATA (scripts/)
│
├── docker-compose.yaml          builds from source, upstream image names
├── docker-compose.local.yaml    pulls only, YOUR registry  ← day-to-day
├── docker-compose-load.yaml     load generator overlay
├── .env                         REPO/TAG for the build path
├── .env.local                   DH_USER + one tag per service
│
├── K8s/                     vanilla Kubernetes — Helm chart, Istio, autoscale, quotas
├── EKS/                     AWS EKS — 5 setup docs + Helm chart (per-service images)
├── AKS/  GKE/               Azure / Google — Helm charts + cloud ingress
├── OpenShift/               setup.sh for SCC permissions
├── Swarm/                   docker stack deploy scripts
├── DCOS/                    Marathon JSON manifests
│
├── fluentd/                 log shipping for Compose and Kubernetes
├── load-gen/                Locust load generator
│
├── DEVOPS_ROADMAP.md        12-part learning path  ← start here
└── PRACTICE_GUIDE.md        index into the roadmap
```

---

## Where the images come from

Twelve containers run. They come from **four genuinely different places**, and conflating them is the most common way to end up with an empty product catalogue and no error message.

### 1. Built by CI, pushed to your registry

Each service has its own pipeline pushing to `naveenreddy9`, tagged with both `:latest` and the `github.run_number` of the run that built it:

| Image | Tags |
|---|---|
| `naveenreddy9/catalogue` | `latest`, `12`, `11`, `10`… |
| `naveenreddy9/cart` | `latest`, `16`, `15`, `14`, `13` |
| `naveenreddy9/user` | `latest`, `5`, `4`, `3` |
| `naveenreddy9/dispatch` | `latest`, `8`, `7` |
| `naveenreddy9/shipping` | `latest`, `7`, `6`, `5` |
| `naveenreddy9/rating` | `1` ← repo is `rating`, service is `ratings`; no `latest` yet, build is failing |
| `naveenreddy9/payment` | `latest`, `2`, `1` |
| `naveenreddy9/web` | `latest`, `2`, `1` |

> `github.run_number` is a **per-workflow** counter. Cart's pipeline has run 15 times, payment's once — they will never converge. There is no single number identifying "the current version of the app", which is why [docker-compose.local.yaml](docker-compose.local.yaml) carries one tag variable per service and the EKS chart carries one image per workload.

### 2. Upstream `robotshop/*` — images with data baked in

`robotshop/rs-mongodb:2.1.0` and `robotshop/rs-mysql-db:2.1.0`. These have Dockerfiles here but **no pipeline builds them**, so they come from upstream.

> **They are not interchangeable with plain `mongo:5` / `mysql:5.7`.** [mongo/Dockerfile](mongo/Dockerfile) copies in `catalogue.js` and `users.js`; [mysql/Dockerfile](mysql/Dockerfile) copies in `scripts/`. That's the product catalogue, the demo users and the shipping tables. Swap in the official images and every container starts, nothing errors, and the shop is simply **empty**.

### 3. Open source, used unmodified

`redis:6.2-alpine` and `rabbitmq:3.8-management-alpine`. No Dockerfile here, no customisation.

### 4. Base images — build time only

The `FROM` lines. Consumed on the CI runner, never run as containers in the stack. [pullbaseimages.sh](pullbaseimages.sh) scrapes exactly this set.

---

## Running it

### Local — your CI images (fastest)

Pulls only, nothing builds. This is the day-to-day path.

```powershell
docker compose --env-file .env.local -f docker-compose.local.yaml up -d
docker compose --env-file .env.local -f docker-compose.local.yaml ps
```

Open **http://localhost:8080**.

[.env.local](.env.local) holds your namespace and one tag per service:

```
DH_USER=naveenreddy9
CATALOGUE_TAG=latest
CART_TAG=latest
USER_TAG=latest
```

Pin a numeric tag instead of `latest` whenever you need to reason about *which* code is running — numeric tags are never overwritten, so rollback is a one-character edit.

**Teardown:**

```powershell
# stop, keep images cached
docker compose --env-file .env.local -f docker-compose.local.yaml down

# stop and reclaim everything this project pulled (~1.2GB)
docker compose --env-file .env.local -f docker-compose.local.yaml down --rmi all -v --remove-orphans
```

`-v` drops the volumes, so MongoDB and MySQL reseed on the next `up`. `--rmi all` is scoped to this project; `docker system prune -a` is **not** and will delete images from your other work.

### Local — build from source

The upstream path. Builds every service locally and names them `${REPO}/rs-<svc>:${TAG}` from [.env](.env).

```shell
export INSTANA_AGENT_KEY="<your agent key>"   # only needed for the nginx tracing module
docker compose build
docker compose up -d
```

Push to your own registry by editing `.env` first, then `docker compose push`.

### Plain Docker, by hand

Worth doing once — everything after it is a reaction to the problems it creates.

```powershell
docker network create robot-shop-manual

docker run -d --name mongodb   --network robot-shop-manual robotshop/rs-mongodb:2.1.0
docker run -d --name redis     --network robot-shop-manual redis:6.2-alpine
docker run -d --name rabbitmq  --network robot-shop-manual rabbitmq:3.8-management-alpine
docker run -d --name mysql     --network robot-shop-manual robotshop/rs-mysql-db:2.1.0
docker run -d --name catalogue --network robot-shop-manual naveenreddy9/catalogue:latest
docker run -d --name user      --network robot-shop-manual naveenreddy9/user:latest
docker run -d --name cart      --network robot-shop-manual naveenreddy9/cart:latest
docker run -d --name shipping  --network robot-shop-manual naveenreddy9/shipping:latest
docker run -d --name ratings   --network robot-shop-manual naveenreddy9/rating:latest
docker run -d --name payment   --network robot-shop-manual naveenreddy9/payment:latest
docker run -d --name dispatch  --network robot-shop-manual naveenreddy9/dispatch:latest
docker run -d --name web -p 8080:8080 --network robot-shop-manual naveenreddy9/web:latest
```

The container **names** are what makes this work — Docker's embedded DNS on a user-defined bridge resolves them as hostnames. Name one `catalogue-1` and nginx can't find it. That is all "service discovery" means at this level.

### Kubernetes (minikube / kind)

```shell
minikube start --cpus 4 --memory 8192
helm install robot-shop --namespace robot-shop --create-namespace K8s/helm --set nodeport=true

kubectl -n robot-shop get pods -w
minikube service web -n robot-shop --url
```

Optional extras in [K8s/](K8s/):

```shell
kubectl -n robot-shop apply -f K8s/resource-quota.yaml    # namespace quotas
./K8s/autoscale.sh                                        # HPA on the deployments
kubectl apply -f K8s/Istio/                               # service mesh, canary routing
```

> [K8s/helm](K8s/helm/) still uses the upstream global `image.repo`/`image.version` scheme, so it deploys **upstream's** images, not yours. [EKS/helm](EKS/helm/) is the one refactored to per-service images.

### AWS EKS

Five numbered setup docs in [EKS/](EKS/), then deploy. Full walkthrough with the gaps filled in **[DEVOPS_ROADMAP.md § 8.1](DEVOPS_ROADMAP.md)**.

```bash
eksctl create cluster --name demo-cluster-three-tier-1 --region us-east-1   # ~20 min, bills immediately
aws eks update-kubeconfig --name demo-cluster-three-tier-1 --region us-east-1

eksctl utils associate-iam-oidc-provider --cluster demo-cluster-three-tier-1 --approve   # IRSA
# then: ALB controller (04), EBS CSI driver (05)

helm install robot-shop --namespace robot-shop --create-namespace EKS/helm --set nodeport=true
kubectl -n robot-shop apply -f EKS/helm/ingress.yaml      # NOT in templates/ — Helm ignores it
kubectl -n robot-shop get ingress robot-shop -w
```

**Always render before deploying.** Two seconds, no cluster needed, catches every image and tag mistake:

```bash
helm template rs EKS/helm | grep "image:" | sort -u
```

**Three traps:**

1. [EKS/helm/ingress.yaml](EKS/helm/ingress.yaml) sits in the chart root, not `templates/`. Helm only renders `templates/`, so `helm install` ignores it — you must `kubectl apply` it yourself. Same for `AKS/helm/ingress.yaml` and `GKE/helm/gclb.yaml`.
2. Pass `--set nodeport=true`, or the `web` Service defaults to `type: LoadBalancer` and you provision a **second** cloud load balancer competing with your ALB — and pay for both.
3. **Tear down the same day.** Delete the Ingress first so the ALB goes with it, then check the console for orphaned load balancers and EBS volumes. Set an AWS Budgets alert *before* you start.

```bash
kubectl -n robot-shop delete ingress robot-shop
helm uninstall robot-shop -n robot-shop
eksctl delete cluster --name demo-cluster-three-tier-1 --region us-east-1
```

### AKS / GKE

Same chart shape, cloud-specific ingress and storage class.

```shell
helm install robot-shop --namespace robot-shop --create-namespace AKS/helm
kubectl -n robot-shop apply -f AKS/helm/ingress.yaml

helm install robot-shop --namespace robot-shop --create-namespace GKE/helm
kubectl -n robot-shop apply -f GKE/helm/gclb.yaml
```

> Across `K8s`, `AKS` and `GKE` exactly one line differs — `redis.storageClassName` (`standard`/`default`/`default`) — and the cloud charts **hardcode** it in `templates/redis-statefulset.yaml`, so `--set redis.storageClassName=...` is silently ignored. Four near-identical charts maintained by hand is itself a defect worth fixing.

### OpenShift

```shell
./OpenShift/setup.sh            # creates the project, grants anyuid + privileged
oc project robot-shop
helm install robot-shop --set openshift=true --set nodeport=true K8s/helm
```

OpenShift refuses to run containers as root and assigns a random UID. This app expects otherwise — [payment/Dockerfile](payment/Dockerfile) sets `USER root` and `ratings` does `chmod -R 777` — which is why `setup.sh` grants the extra SCCs. Meeting that failure is the most valuable thing OpenShift teaches here.

### Docker Swarm

```shell
./Swarm/create-swarm.sh         # docker-machine nodes
./Swarm/deploy.sh               # docker stack deploy from docker-compose.yaml
docker service ls
```

### DC/OS Marathon

Manifests in [DCOS/manifest/](DCOS/manifest/), built against DC/OS 1.11.

```shell
./DCOS/deploy.sh
./DCOS/destroy.sh
```

---

## CI/CD

Eight GitHub Actions workflows in [.github/workflows/](.github/workflows/), one per service. Each triggers on changes to its own path, builds the image, and pushes **two tags** — `:latest` and `:${{ github.run_number }}`.

| Workflow | Triggers on | Pushes |
|---|---|---|
| `cart.yaml` | `cart/**` | `naveenreddy9/cart` |
| `catalogue.yaml` | `catalogue/**` | `naveenreddy9/catalogue` |
| `user.yaml` | `user/**` | `naveenreddy9/user` |
| `web.yml` | `web/**` | `naveenreddy9/web` |
| `rating.yaml` | `ratings/html/**`, `ratings/Dockerfile` | `naveenreddy9/rating` |
| `dispatch.yaml` | `**.go`, `dispatch/**` | `naveenreddy9/dispatch` |
| `payment.yaml` | `**.py` | `naveenreddy9/payment` |
| `shipping.yaml` | `**.java`, `shipping/Dockerfile` | `naveenreddy9/shipping` |

**Secrets required:** `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`, under Settings → Secrets and variables → Actions.

**Known gaps** — the full audit is in **[DEVOPS_ROADMAP.md § 3.4](DEVOPS_ROADMAP.md)**:

- Six of eight build **and push** on `pull_request`. Only `web.yml` guards it with `if: github.event_name == 'push'`.
- Seven of eight use `docker login -p`, which leaks the token into the process list. Only `cart.yaml` pipes it via `--password-stdin`.
- `payment`, `dispatch` and `shipping` still filter on file **extension** (`**.py`, `**.go`, `**.java`) rather than directory — so unrelated edits trigger builds, and a change to the service's own Dockerfile may not.
- No `permissions:` block, no SHA-pinned actions, no layer caching.

---

## Observability

### Prometheus

`cart` and `payment` expose `/metrics`:

```shell
curl http://<host>:8080/api/cart/metrics
curl http://<host>:8080/api/payment/metrics
```

- **cart** — counter of items added to the cart
- **payment** — counter of items purchased, histogram of items per cart, histogram of cart value

### Logs

[fluentd/](fluentd/) ships log-forwarding configs for both Compose ([fluentd/Docker-Compose/](fluentd/Docker-Compose/)) and Kubernetes ([fluentd/Kubernetes/](fluentd/Kubernetes/)).

### Tracing and End-User Monitoring

Every service ships with Instana components pre-installed. For EUM, set `INSTANA_EUM_KEY` and `INSTANA_EUM_REPORTING_URL` on the `web` service — the JavaScript fragment is injected automatically by [web/entrypoint.sh](web/entrypoint.sh). On Kubernetes, the Helm chart takes `eum.key` and `eum.url`.

---

## Load generation

A [Locust](https://locust.io)-based generator in [load-gen/](load-gen/). Not started automatically.

```shell
# as a Compose overlay
docker compose -f docker-compose.yaml -f docker-compose-load.yaml up -d

# on Kubernetes
kubectl -n robot-shop apply -f K8s/load-deployment.yaml
```

See [load-gen/README.md](load-gen/README.md) for the command-line options.

---

## Learning path

**[DEVOPS_ROADMAP.md](DEVOPS_ROADMAP.md)** is a 12-part progression built around one question at each step: *what does this tool fix that the previous one couldn't?*

| Part | Topic |
|---|---|
| 1 | How this project is designed |
| 2 | Dockerfile design and best practices |
| 3 | How CI works |
| 4 | The registry: image identity and tags |
| 5 | Deploy with plain Docker — and what it doesn't solve |
| 6 | Docker Compose — and what it doesn't solve |
| 7 | Kubernetes — and what it costs you |
| 8 | Different environments: AWS EKS, OpenShift |
| 9 | Observability: Prometheus + Grafana, ELK |
| 10 | Provisioning: Terraform and Ansible |
| 11 | GitOps with Argo CD |
| 12 | Suggested order |

---

## Credits

Forked from [instana/robot-shop](https://github.com/instana/robot-shop). Licensed under Apache 2.0 — see [LICENSE](LICENSE).
