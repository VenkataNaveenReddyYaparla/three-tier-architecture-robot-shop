# DevOps Practice Roadmap — Stan's Robot Shop

**How to use this doc:** it lists **many different ways to practice DevOps against this one app**. Each way has two parts:

- **How** — exact steps, verified against the files actually in this repo
- **Learn** — the potential learning: what skill it builds and where it shows up in real work

Work through them in any order you like; Part 3 has a suggested sequence. Every command here was checked against the real repo contents, including the parts of the repo that are stale or broken — those are called out rather than hidden, because debugging them is some of the best practice available.

> **Shell note (Windows):** blocks marked `powershell` run in PowerShell. Blocks marked `bash` are POSIX shell — run them in **Git Bash or WSL2**, not PowerShell. Every `.sh` script in this repo is POSIX shell.

> **The four broken CI workflows are broken on purpose.** `dispatch.yaml`, `payment.yaml`, `shipping.yaml`, `user.yaml` stay broken as a debugging exercise (Way 15 is the answer key). Don't "fix" them casually.

---

# Part 0 — Know the ground

## 0.1 The services

| Service | Tech | Base image(s) | Multi-stage? | CI workflow |
|---|---|---|---|---|
| web | Nginx + AngularJS 1.x | `nginx:1.21.6` | No | **none** |
| cart | Node/Express | `node:14` | No | yes (closest to correct) |
| catalogue | Node/Express | `node:14` | No | yes (closest to correct) |
| user | Node/Express | `node:14` | No | broken on purpose |
| shipping | Java / Spring Boot | `debian:10 AS build` → `openjdk:8-jdk` | **Yes — the only one** | broken on purpose |
| ratings | PHP 7.4 / Apache | `php:7.4-apache` (+ `COPY --from=composer`) | No | **none** |
| payment | Python / Flask | `python:3.9` | No | broken on purpose |
| dispatch | Go | `golang:1.23` | No — ships the whole Go toolchain | broken on purpose |
| load-gen | Python / Locust | built separately as `robotshop/rs-load` | No | n/a |
| fluentd | Log forwarder | `fluentd` + elasticsearch plugin | No | n/a |

No Dockerfile in this repo declares a single `ARG`. `docker-compose.yaml` passes `INSTANA_AGENT_KEY` as a build arg to `web`, but `web/Dockerfile` ignores it — safe to leave unset.

## 0.2 Data stores and messaging

| Component | Used by | Notes |
|---|---|---|
| MongoDB | catalogue, user | Seeded by [mongo/](mongo/) — includes plaintext demo users |
| MySQL | shipping, ratings | Seeded by [mysql/](mysql/), includes MaxMind geo data |
| Redis | cart, user | Base image, no dedicated dir |
| RabbitMQ | payment → dispatch | Base image, no dedicated dir |

**Important: `docker-compose.yaml` defines NO volumes at all.** There is no top-level `volumes:` key and no service mounts anything. All database data is ephemeral — `docker compose down` destroys it. This is a defect from an ops point of view, and it's one of the better practice exercises in the repo (Way 10).

## 0.3 What's genuinely ready vs. what's a template

| Area | Reality |
|---|---|
| Docker Compose | Fully working. Start here. |
| K8s Helm chart ([K8s/helm/](K8s/helm/)) | Complete — 28 templates, 12 Deployments + Redis StatefulSet. No Ingress template. |
| Istio ([K8s/Istio/](K8s/Istio/)) | Complete and coherent, but undocumented ordering (Way 33). |
| CI ([.github/workflows/](.github/workflows/)) | 6 of 8 services; 4 broken by design; the other 2 have real bugs too. |
| EKS docs ([EKS/](EKS/)) | 5 thin docs that set up a cluster but **never deploy the app** — the last mile is missing. |
| AKS / GKE | Helm charts only, zero setup docs. GKE's `gclb.yaml` is broken. |
| Swarm / DC/OS | Scripts that depend on dead tooling (`docker-machine`) or an external cluster. |
| OpenShift | README + script assuming a local dev cluster with passwordless `system:admin`. |
| Terraform / Ansible / observability stack | **Not in the repo** — you build these yourself. |

## 0.4 Known traps in this repo (verified)

Hitting these accidentally wastes hours; hitting them deliberately is practice.

1. **`.github/workflows/cart.yaml` uses `actions/checkout@v7` — that tag does not exist.** The workflow fails instantly with "Unable to resolve action". Despite the commit history calling it fixed, it is not green.
2. **`catalogue.yaml` uses `docker login -p`**, which leaks the token into the runner's process list, and `actions/checkout@v2`, which is deprecated.
3. **No volumes in Compose** — database data does not survive a `down`.
4. **Only app services have healthchecks.** mongodb, mysql, redis, rabbitmq and dispatch have none, so they never report `healthy` in `docker compose ps`. Don't wait for it.
5. **`K8s/README.md` line 11 uses Helm 2 syntax** (`helm install --name robot-shop ...`), which fails on Helm 3. [K8s/helm/README.md](K8s/helm/README.md) has the correct form.
6. **`EKS/helm/ingress.yaml`, `AKS/helm/ingress.yaml` and `GKE/helm/gclb.yaml` sit in the chart root, not `templates/`** — `helm install` never renders them. You must `kubectl apply -f` them separately. No doc in the repo says this.
7. **The cloud charts hardcode `storageClassName`** in `templates/redis-statefulset.yaml` (EKS `gp2`, AKS/GKE `default`), so `--set redis.storageClassName=...` is silently ignored there. Only `K8s/helm` keeps it templated.
8. **`GKE/helm/gclb.yaml` is invalid** — declares `networking.k8s.io/v1` but uses a v1beta1 body, and points at a service named `robot-shop` that doesn't exist (it's `web`). Use `GKE/helm/ingress.yaml` instead.
9. **`web-service.yaml` defaults to `type: LoadBalancer`** — on a cloud cluster a plain install provisions a load balancer that duplicates your Ingress. Use `--set nodeport=true` when you're using Ingress.
10. **`Swarm/create-swarm.sh` needs `docker-machine`**, which is end-of-life and unmaintained, plus VirtualBox. Way 47 gives a modern path instead.
11. **`DCOS/manifest/mysql.json` references image `robotshop/rs-shipping-db`** while Compose builds `rs-mysql-db`, and sets `containerPort: 0`.
12. **`.gitignore` is 5 lines** and excludes none of: `.env`, `*.pem`, `*.key`, `kubeconfig`, `iam_policy.json`. `EKS/04-alb-configuration.md` tells you to `curl -O iam_policy.json` straight into the repo. Way 20 hardens this.
13. **Demo credentials are committed on purpose** — `MYSQL_PASSWORD=secret`, mongo seed users (`user/password`, `stan/bigbrain`, `partner-57/worktogether`), RabbitMQ `guest/guest`, and `user/server.js` compares passwords in plaintext with `==`. Never a secret leak to worry about, always a "what not to do in prod" exhibit.

---

# Part 1 — The ways to practice

## Track A — Containers and images

### Way 1 — Build every image by hand

**How** (PowerShell, from the repo root):
```powershell
docker build -t rs-shipping  ./shipping    # multi-stage: debian build -> openjdk:8-jdk runtime
docker build -t rs-dispatch  ./dispatch    # single-stage on golang:1.23
docker build -t rs-payment   ./payment     # python:3.9
docker build -t rs-ratings   ./ratings     # php:7.4-apache, pulls composer at build time
docker build -t rs-cart      ./cart
docker build -t rs-catalogue ./catalogue
docker build -t rs-user      ./user
docker build -t rs-web       ./web
docker images | Select-String "rs-"
```
Then compare them:
```powershell
docker history rs-shipping
docker history rs-dispatch
```

**Learn**
- Why `shipping` (the only multi-stage build here) ends up leaner than `dispatch`, even though Go produces a tiny static binary — `dispatch` ships the entire `golang:1.23` toolchain in its final image because nobody wrote a second stage.
- Layer caching: which instruction ordering causes a full rebuild vs. a cache hit.
- That `ratings` pulls the unpinned `composer:latest` image during build (`COPY --from=composer`) — an unpinned, network-dependent build is a real supply-chain smell.
- Reading a Dockerfile as the specification of what a service needs at runtime.

### Way 2 — Run the whole stack with raw `docker run` (no Compose)

The highest-value beginner exercise in the repo: hand-wire what Compose does for you.

**How** (PowerShell):
```powershell
docker network create robot-shop-manual
docker run -d --name mongodb  --network robot-shop-manual robotshop/rs-mongodb:2.1.0
docker run -d --name redis    --network robot-shop-manual redis:6.2-alpine
docker run -d --name rabbitmq --network robot-shop-manual rabbitmq:3.8-management-alpine
docker run -d --name mysql    --network robot-shop-manual robotshop/rs-mysql-db:2.1.0
docker run -d --name catalogue --network robot-shop-manual robotshop/rs-catalogue:2.1.0
docker run -d --name user      --network robot-shop-manual robotshop/rs-user:2.1.0
docker run -d --name cart      --network robot-shop-manual robotshop/rs-cart:2.1.0
docker run -d --name shipping  --network robot-shop-manual robotshop/rs-shipping:2.1.0
docker run -d --name ratings   --network robot-shop-manual robotshop/rs-ratings:2.1.0
docker run -d --name payment   --network robot-shop-manual robotshop/rs-payment:2.1.0
docker run -d --name dispatch  --network robot-shop-manual robotshop/rs-dispatch:2.1.0
docker run -d --name web -p 8080:8080 --network robot-shop-manual robotshop/rs-web:2.1.0
```
Read [docker-compose.yaml](docker-compose.yaml) for the env vars each service expects, and note that container **names** must match the hostnames services look up (`MONGO_URL`, `REDIS_HOST`, `CART_HOST`, …). Get one wrong and the app half-works — which is the lesson.

**Learn**
- Docker's embedded DNS: on a user-defined bridge network, container name *is* the hostname.
- That "service discovery" in Compose and Kubernetes is this, automated.
- Startup ordering: without `depends_on` the services crash-loop until dependencies appear, then recover on their own — exactly how Kubernetes behaves.
- Why declarative orchestration exists at all. After twelve `docker run` lines you never want to do it again.

### Way 3 — Image slimming and layer analysis

**How**
- Rewrite `dispatch/Dockerfile` as a multi-stage build (`FROM golang:1.23 AS build` → `FROM alpine` or `scratch`, copying only the compiled binary) and measure before/after.
- Do the same thinking for the Node services: `node:14` → `node:14-alpine` or distroless.
- Add a `.dockerignore` to a Node service and watch the build context shrink (`docker build` prints how much it sends).
- Pin `COPY --from=composer` in `ratings/Dockerfile` to a real version tag.

**Learn**
- Concrete size reduction — often ~800MB to ~20MB for the Go service — and why that matters for pull time, cold starts and CVE surface.
- Build context vs. image content, a distinction that confuses nearly everyone at first.
- Base-image pinning as a supply-chain control.

### Way 4 — Registry practice: Docker Hub and a local registry

**How**
```powershell
docker login
docker tag rs-cart <your-dockerhub-user>/rs-cart:v1
docker push <your-dockerhub-user>/rs-cart:v1
```
Then run your own registry instead:
```powershell
docker run -d -p 5000:5000 --name registry registry:2
docker tag rs-cart localhost:5000/rs-cart:v1
docker push localhost:5000/rs-cart:v1
curl http://localhost:5000/v2/_catalog
```
Finally point the whole stack at your images by editing [.env](.env) (`REPO=<your-user>`, `TAG=v1`) and running `docker compose build; docker compose push`.

**Learn**
- How an image reference decomposes into `registry/namespace/name:tag`.
- Why `.env`'s `REPO`/`TAG` indirection exists — the same pattern as Helm's `image.repo`/`image.version`.
- Running a private registry, which is what every air-gapped or enterprise environment does.

### Way 5 — Image scanning

**How**
```powershell
docker scout cves robotshop/rs-ratings:2.1.0
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image robotshop/rs-ratings:2.1.0
```
`php:7.4-apache`, `node:14`, `debian:10` and `openjdk:8-jdk` are all end-of-life, so this repo produces a genuinely alarming report — perfect material.

**Learn**
- Reading a CVE report and separating "fixable by a base-image bump" from "won't fix".
- Why EOL base images are the single biggest source of container vulnerabilities.
- The follow-on exercise: bump `node:14` to `node:20`, rebuild, rescan, and find out what breaks in the app.

---

## Track B — Compose and day-2 operations

### Way 6 — Run the stack with Compose

**How**
```powershell
cd D:\robot-shop-project\three-tier-architecture-robot-shop
docker compose pull          # .env pins REPO=robotshop TAG=2.1.0
docker compose up -d
docker compose ps
docker compose logs -f web
```
Open **http://localhost:8080**. Only `web`, `cart`, `catalogue`, `user`, `shipping`, `ratings` and `payment` define healthchecks — the databases and `dispatch` have none and will never report `healthy`, so judge those by their logs.

Verify the API surface (routes confirmed in `web/default.conf.template`):
```powershell
curl http://localhost:8080/api/catalogue/products
curl http://localhost:8080/api/cart/metrics
curl http://localhost:8080/api/payment/metrics
```

**Learn**
- Declarative multi-container orchestration: healthchecks, `depends_on`, one shared network.
- Reading an nginx reverse-proxy config as the app's routing table — note `ratings` proxies to port **80** while everything else uses 8080.
- The difference between `pull` (pre-built images) and `build` (from source), and what `.env` controls.

### Way 7 — Generate load with Locust

**How** — the simple overlay:
```powershell
docker compose -f docker-compose.yaml -f docker-compose-load.yaml up -d
```
Or the script, for real control (Git Bash/WSL, must be run from `load-gen/`):
```bash
cd load-gen
./build.sh                                     # builds ${REPO}/rs-load:${TAG}
./load-gen.sh -d -n 10 -t 5m -h http://localhost:8080
```
Flags: `-n` clients, `-t` runtime, `-h` target URL, `-d` detached, `-e` error-injection mode.

`load-gen.sh` uses `--network=host`, which is unreliable on Docker Desktop for Windows. If it can't reach the app, attach it to the app's network instead:
```powershell
docker network ls    # find the generated name, e.g. three-tier-architecture-robot-shop_robot-shop
docker run -d --rm --name loadgen --network <that-network> -e HOST=http://web:8080/ -e NUM_CLIENTS=10 robotshop/rs-load:2.1.0
```

**Learn**
- Load generation as a first-class ops tool — everything downstream (autoscaling, dashboards, alerting) is meaningless without traffic.
- Reading someone else's entrypoint script to find the real interface: `HOST`, `NUM_CLIENTS`, `RUN_TIME`, `ERROR` and `SILENT` are consumed by `entrypoint.sh`, not by the Python (which only reads `ERROR`).
- Why `--network=host` behaves differently on Linux and Docker Desktop.

### Way 8 — Failure injection: map the blast radius

**How** — stop one dependency at a time, with load running, and record what breaks:
```powershell
docker compose stop redis      # cart + user session state
docker compose stop mongodb    # catalogue + user
docker compose stop rabbitmq   # payment -> dispatch pipeline
docker compose stop mysql      # shipping + ratings
docker compose logs --tail=50 cart
docker compose start redis
```

**Learn**
- Dependency mapping from observed behaviour rather than from documentation.
- Hard vs. soft failure: cart dies without Redis; payment keeps taking orders while dispatch simply stops consuming.
- Recovery behaviour — what self-heals when a dependency returns and what needs a restart. This is the core skill of incident response.

### Way 9 — Scaling, and its limits

**How**
```powershell
docker compose up -d --scale catalogue=3 --scale cart=2
docker compose ps
docker compose up -d --scale web=2     # this FAILS — and that is the point
```
`web` is the only service binding a fixed host port (`8080:8080`), so it cannot scale. No service sets `container_name`, so everything else scales cleanly.

**Learn**
- Why a fixed host-port binding is incompatible with horizontal scaling — precisely the reason Kubernetes has Services and Ingress instead of port mappings.
- Compose's round-robin DNS across replicas.
- Reading a compose file and predicting what will and won't scale before trying it.

### Way 10 — Persistence: fix the missing volumes

This repo keeps **all** database data in the container's writable layer. Prove it, then fix it.

**How**
```powershell
docker compose exec mongodb mongo catalogue --eval "db.products.count()"
docker compose down
docker compose up -d
# data came back from the image, not from storage — any runtime change is gone
```
Add persistence yourself in a new `docker-compose.override.yaml`:
```yaml
services:
  mongodb:
    volumes: [ "mongo-data:/data/db" ]
  mysql:
    volumes: [ "mysql-data:/var/lib/mysql" ]
volumes:
  mongo-data:
  mysql-data:
```
Then practice backup and restore:
```powershell
docker compose exec mongodb mongodump --archive=/tmp/dump.gz --gzip --db catalogue
docker compose cp mongodb:/tmp/dump.gz ./backup-catalogue.gz
docker compose exec mysql sh -c "mysqldump -u root cities > /tmp/cities.sql"
```

**Learn**
- Stateless vs. stateful containers, and why the writable layer is not storage.
- Named volumes, bind mounts, and how `docker-compose.override.yaml` layers on without editing the original file.
- Backup/restore drills — the ops task everyone skips until the day they can't.
- Direct preparation for PersistentVolumeClaims in Kubernetes (Way 28).

### Way 11 — Multi-environment configuration

**How** — write `docker-compose.dev.yaml` and `docker-compose.prod.yaml` differing in replica counts, log levels, exposed ports and resource limits:
```powershell
docker compose -f docker-compose.yaml -f docker-compose.dev.yaml up -d
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml config   # render without running
```

**Learn**
- Config composition and override precedence — the same concept as Helm values files and Kustomize overlays.
- `config` as a render/dry-run step, the Compose equivalent of `helm template`.
- Keeping environment differences in files instead of in somebody's shell history.

---

## Track C — Shell scripting and automation

### Way 12 — Read the scripts that already exist

**How** — read and explain line by line: [pullbaseimages.sh](pullbaseimages.sh) (finds every Dockerfile, extracts its `FROM`, pulls it), [Swarm/create-swarm.sh](Swarm/create-swarm.sh), [Swarm/deploy.sh](Swarm/deploy.sh), [DCOS/deploy.sh](DCOS/deploy.sh), [OpenShift/setup.sh](OpenShift/setup.sh), [K8s/autoscale.sh](K8s/autoscale.sh), [load-gen/load-gen.sh](load-gen/load-gen.sh).

For each, answer three questions: what does it assume is already installed? which directory must it run from? what happens if a command in the middle fails?

**Learn**
- `awk`/`egrep` extraction, `getopts` argument parsing (`load-gen.sh`), `eval $(...)` environment injection (the Swarm scripts).
- Spotting implicit dependencies — the skill that stops you running a script that half-executes and leaves a mess.
- Why none of these use `set -euo pipefail`, and what that costs (`shipping.yaml`'s CI bug in Way 15 is exactly this failure class).

### Way 13 — Write your own automation

**How** — create `scripts/` and write these (Git Bash/WSL):

`scripts/wait-for-healthy.sh`
```bash
#!/usr/bin/env bash
set -euo pipefail
TIMEOUT=${1:-180}; ELAPSED=0
while [ "$ELAPSED" -lt "$TIMEOUT" ]; do
  if ! docker compose ps --format json | grep -q '"Health":"starting"'; then
    echo "Stack settled after ${ELAPSED}s"; docker compose ps; exit 0
  fi
  sleep 5; ELAPSED=$((ELAPSED + 5))
done
echo "Timed out"; docker compose logs --tail=50; exit 1
```

`scripts/smoke-test.sh` — curl every route from `web/default.conf.template`, fail on the first non-200:
```bash
#!/usr/bin/env bash
set -euo pipefail
BASE=${1:-http://localhost:8080}
for path in / /api/catalogue/products /api/cart/metrics /api/payment/metrics; do
  code=$(curl -s -o /dev/null -w '%{http_code}' "${BASE}${path}")
  [ "$code" = "200" ] || { echo "FAIL ${path} -> ${code}"; exit 1; }
  echo "OK   ${path}"
done
```

`scripts/reset.sh` — `docker compose down -v`, then `pull`, then `up -d`, then call `wait-for-healthy.sh`.

`scripts/build-all.sh` — loop every service directory, `docker build`, tag with the git SHA, optionally push.

**Learn**
- `set -euo pipefail`, exit codes and timeouts — writing scripts that fail loudly instead of silently.
- That a smoke test is the seed of both a CI gate and a Kubernetes readiness probe: one idea, three contexts.
- Idempotent teardown/setup, which is what every CI job and every Ansible playbook is really doing.

### Way 14 — A task runner

**How** — write a `Makefile` (or `Taskfile.yml`) wrapping the common flows: `make up`, `make down`, `make logs`, `make smoke`, `make build-all`, `make k8s-install`, `make k8s-uninstall`.

**Learn**
- Interface design for a repo — new joiners should run `make up`, not read a wiki page.
- Where a task runner ends and CI begins (ideally the same targets run in both).

---

## Track D — CI/CD

### Way 15 — Debug the four broken workflows (the flagship exercise)

**How:** read [dispatch.yaml](.github/workflows/dispatch.yaml), [payment.yaml](.github/workflows/payment.yaml), [shipping.yaml](.github/workflows/shipping.yaml) and [user.yaml](.github/workflows/user.yaml), and write down every reason each would fail — *before* opening the answer key. Do it on paper. Don't run them and don't fix them.

<details>
<summary><b>Answer key — open only after writing your own list</b></summary>

**Shared by all four**
- `runs-on: [self-hosted]` with no registered runner → the job queues forever and times out after 24h. The inline comment even says "use a standard runner instead" — the code contradicts it.
- No `defaults.run.working-directory`, so every `run:` executes at the repo root. There is **no root Dockerfile**, so `docker build .` fails with "failed to read dockerfile".
- The job key is `payment:` in all four files — copy-paste leftover; only the display `name:` was updated.
- `actions/checkout@v2` is deprecated, and `fetch-depth: 0` clones full history for no reason.
- `docker login -p` puts the token in the runner's process list; it should be `--password-stdin`.
- No `permissions:` block, actions unpinned to a SHA, and the `pull_request` trigger builds and pushes images from unmerged (for forks, untrusted) code.
- **`${name}`** is bash variable syntax, not Actions syntax (`${{ }}`), and is defined nowhere. It expands to empty, so `docker tag payment /payment:latest` fails with "invalid reference format".

**dispatch.yaml**
- `curl -sL <go tarball> |` pipes the archive into `export PATH=...`. `export` is a builtin that reads no stdin, so Go is never installed and the download is thrown away — **and the step exits 0, so it passes silently.** `export` also wouldn't survive into the next step (that needs `$GITHUB_PATH`).
- The Go-install step is unnecessary anyway: `dispatch/Dockerfile` already uses `FROM golang:1.23`.
- It builds `-t payment` — the wrong service name entirely.
- `docker tag dispatch ...` references an image that was never created → "No such image: dispatch:latest".

**payment.yaml**
- The `'**.py'` path filter also matches `load-gen/robot-shop.py`, so editing the load generator triggers a payment build.
- `sudo apt-get install python3` with no `apt-get update` and no `-y` → prompts, then aborts. Also unnecessary (`FROM python:3.9`).

**shipping.yaml**
- `name: Dispatch` on line 1 — so the Actions UI shows two workflows called "Dispatch".
- The step is named "Go install" for a Java service.
- `sudo apt-get install openjdk 17 | java -version` — `openjdk` isn't a package (it's `openjdk-17-jdk`), `17` is a stray second argument, and the trailing `|` pipes apt's output into `java -version`. **The pipeline's exit status comes from `java -version`, so a failed install reports success.**

**user.yaml**
- `'**.js'` is the worst filter in the repo: it matches `cart/server.js`, `catalogue/server.js`, `mongo/catalogue.js`, `mongo/users.js` and `web/static/js/*.js` — so editing cart, catalogue, seed data or the frontend all trigger a `user` build.
- `npm install` at the repo root, where there is no `package.json` → ENOENT. Unnecessary as well.
- `docker user ${name}/user:latest` should be `docker push`; it fails with "'user' is not a docker command".
</details>

**Learn**
- Reading YAML pipelines like a compiler: trigger scope, execution context, working directory, variable syntax, exit-code semantics.
- **Silent-pass bugs** — `dispatch`'s dangling pipe and `shipping`'s piped apt both exit 0 while doing nothing. This is the most dangerous class of CI defect, because green stops meaning working.
- Path-filter design, and how an over-broad filter turns a per-service pipeline into a repo-wide one.
- Why `${{ }}` and `${}` are not interchangeable, and why secrets belong on stdin.

### Way 16 — Audit the two "reference" workflows too

They are closer to correct, but not correct.

**How** — read [cart.yaml](.github/workflows/cart.yaml) and [catalogue.yaml](.github/workflows/catalogue.yaml) and find what's left:
- `cart.yaml` pins `actions/checkout@v7` — **that tag does not exist** (checkout tops out in the v5 range), so the workflow fails immediately with "Unable to resolve action". It only *looks* fixed.
- `cart.yaml` pushes `:latest` only, so there is no immutable tag to roll back to.
- `catalogue.yaml` uses `docker login -p` (token in the process list) and `actions/checkout@v2` (deprecated), and still carries a stale comment claiming the image is "tagged as 'payment'".
- Neither declares `permissions:`, and both build and push on `pull_request`.

**Learn**
- That "reviewed and merged" is not evidence a pipeline works — only a green run against the real action registry is.
- Mutable (`latest`) vs. immutable (`run_number`, git SHA, semver) tags, and what each costs during a 2am rollback.
- Least-privilege `GITHUB_TOKEN` permissions, and why unpinned third-party actions are a supply-chain risk.

### Way 17 — Write the two missing workflows

`ratings` and `web` have no workflow at all. Build them from a blank file.

**How** — `.github/workflows/ratings.yaml`:
```yaml
name: Ratings
on:
  push:
    branches: [master]
    paths: ['ratings/**', '.github/workflows/ratings.yaml']
  pull_request:
    branches: [master]
    paths: ['ratings/**', '.github/workflows/ratings.yaml']
permissions:
  contents: read
jobs:
  ratings:
    name: Build and push ratings
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ratings
    steps:
      - uses: actions/checkout@v4
      - name: Docker login
        run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u ${{ secrets.DOCKERHUB_USERNAME }} --password-stdin
      - name: Build
        run: docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/rs-ratings:${{ github.sha }} .
      - name: Push
        run: docker push ${{ secrets.DOCKERHUB_USERNAME }}/rs-ratings:${{ github.sha }}
```
Do the same for `web`. Neither Dockerfile needs build args, but both need their own directory as build context (`ratings` copies `status.conf` and `html/`; `web` depends on `entrypoint.sh` keeping its executable bit).

**Learn**
- Writing a pipeline from scratch rather than copy-pasting one — the difference between understanding and cargo-culting.
- Secrets configuration (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN` under Settings → Secrets and variables → Actions).
- Choosing a tagging scheme on purpose instead of inheriting one.

### Way 18 — Level up the pipelines

**How** — rebuild CI properly, one technique at a time:
- **Matrix build** — one workflow with `strategy.matrix.service: [cart, catalogue, user, shipping, ratings, payment, dispatch, web]` replacing eight near-identical files.
- **`docker/build-push-action`** with `cache-from`/`cache-to: type=gha` for layer caching across runs.
- **Multi-arch** via `docker/setup-buildx-action` and `platforms: linux/amd64,linux/arm64`.
- **Change detection** with `dorny/paths-filter` so the matrix only builds what actually changed.
- **SHA-pin** every third-party action.
- **Branch protection** requiring the workflow to pass before merge.

**Learn**
- DRY pipeline design, and the trade-off a matrix makes (one file to maintain, harder failure isolation).
- Remote build caching, usually the single biggest CI speed win.
- Why multi-arch matters now that ARM runners and Apple Silicon are everywhere.

### Way 19 — Add tests, then gate CI on them

There are effectively **no tests** in this repo: `cart`, `catalogue` and `user` have placeholder `npm test` scripts that just `exit 1`; `shipping` declares `spring-boot-starter-test` but has no `src/test` directory and no surefire config; `payment`, `dispatch` and `ratings` have nothing.

**How**
- Write a few real tests — a Jest test for a `cart` route, a JUnit test for a `shipping` class, a `go test` for a `dispatch` helper.
- Wire them into the workflow **before** the Docker build, so a failing test blocks the image.
- Add a post-deploy gate: bring up a Compose stack inside the CI job and run `scripts/smoke-test.sh` (Way 13) against it.

**Learn**
- The test pyramid in a CI context: fast unit tests gate the build, slower smoke tests gate the deploy.
- Using `docker compose up` inside a runner as a throwaway integration environment.
- Why "we'll add tests later" becomes permanent — you are currently doing the archaeology.

### Way 20 — Supply chain and secrets hygiene

**How**
- Add a Trivy gate that fails the build on HIGH/CRITICAL:
  ```yaml
  - uses: aquasecurity/trivy-action@master
    with:
      image-ref: ${{ secrets.DOCKERHUB_USERNAME }}/rs-cart:${{ github.sha }}
      exit-code: '1'
      severity: 'HIGH,CRITICAL'
  ```
- Generate an SBOM (`docker buildx build --sbom=true`, or `syft`) and publish it as a build artifact.
- Enable GitHub secret scanning, and run `gitleaks` across the history.
- **Harden `.gitignore`.** It is five lines and excludes none of `.env`, `*.pem`, `*.key`, `kubeconfig`, `iam_policy.json` — and `EKS/04-alb-configuration.md` explicitly tells you to `curl -O iam_policy.json` into the working directory.
- Catalogue the deliberately-bad credentials (`MYSQL_PASSWORD=secret`, mongo's plaintext seed users, RabbitMQ `guest/guest`, the plaintext password comparison at `user/server.js:129`) and write up how each should be handled in production.

**Learn**
- Shifting security left: catching CVEs at build time instead of in production.
- SBOMs, and why customers and regulators increasingly demand them.
- Telling a real secret leak apart from demo credentials — an actual judgement call in security review.

---

## Track E — Kubernetes

### Way 21 — Stand up a local cluster

**How** — pick one and know why:
```powershell
minikube start --cpus=4 --memory=6g --driver=docker
minikube addons enable metrics-server      # required later for Way 26
```
Alternatives: `kind create cluster` (fastest, no VM, but no built-in LoadBalancer or addons), or Docker Desktop's built-in Kubernetes (zero setup, hardest to reset cleanly).

**Learn**
- What a local cluster actually is — a control plane and kubelet in a container or VM.
- Why 10+ pods need explicit CPU/memory allocation, and what happens when you under-provision (pods stuck `Pending` with `Insufficient cpu`).
- The differences between distributions, which is exactly the question "which cluster do we use for CI?" in real teams.

### Way 22 — Deploy with the Helm chart

**How**
```powershell
helm install robot-shop --namespace robot-shop --create-namespace K8s/helm --set nodeport=true
kubectl -n robot-shop get pods -w
minikube ip
kubectl -n robot-shop get svc web      # note the NodePort
# browse http://<minikube-ip>:<nodeport>
```
Two traps: **`K8s/README.md` gives Helm 2 syntax** (`helm install --name robot-shop ...`) which fails on Helm 3 — use the form above or the one in [K8s/helm/README.md](K8s/helm/README.md). And Helm 3 does not create the namespace unless you pass `--create-namespace`.

Real value keys in this chart (verified against [K8s/helm/values.yaml](K8s/helm/values.yaml)): `image.repo`, `image.version`, `image.pullPolicy`, `nodeport`, `psp.enabled`, `payment.gateway`, `redis.storageClassName`, `eum.key`, `eum.url`, `ocCreateRoute`, plus a per-workload `{}` slot for each service to hold `affinity`/`nodeSelector`/`tolerations`. Note `openshift` exists in values.yaml but is never referenced by any template — a dead key.

**Learn**
- Helm release lifecycle: `install`, `upgrade`, `rollback`, `uninstall`, `history`.
- `--set` vs. a values file, and how `{{ .Values.x }}` flows into rendered manifests.
- That a chart's `apiVersion: v1` (Helm 2 era) still installs fine under Helm 3 — versioning in the ecosystem is messier than tutorials suggest.
- Service types: this chart's `web` Service is `LoadBalancer` by default and `NodePort` only when `nodeport=true`.

### Way 23 — Deploy the same app without Helm

**How**
```powershell
helm template robot-shop K8s/helm --set nodeport=true > rendered.yaml
kubectl create namespace robot-shop
kubectl -n robot-shop apply -f rendered.yaml
```
Read `rendered.yaml` — 28 templates become plain Deployments, Services, a StatefulSet for Redis, plus ServiceAccount/ClusterRole/ClusterRoleBinding/PodSecurityPolicy.

**Learn**
- Demystifying Helm — it is a template engine that produces YAML, nothing more.
- Reading raw Kubernetes objects and mapping each back to its chart template.
- A debugging technique that works on any chart: render first, apply second, so you can see exactly what you are about to create.

### Way 24 — Write your own manifests, then Kustomize

**How**
- Write a Deployment + Service for one service (say `catalogue`) by hand, from scratch, without copying the chart. Compare yours with the chart's version.
- Then build a Kustomize structure: a `base/` with the rendered manifests and `overlays/dev` + `overlays/prod` patching replica counts and resource limits.
```powershell
kubectl apply -k overlays/dev
```

**Learn**
- The anatomy of a Deployment: selector/template label matching (the single most common beginner error), probes, resources, env.
- Kustomize's patch model vs. Helm's template model — the two dominant approaches, and why teams argue about them.
- That "overlays" are the same idea as the Compose override file from Way 11.

### Way 25 — Resource quotas

**How**
```powershell
kubectl -n robot-shop apply -f K8s/resource-quota.yaml
kubectl -n robot-shop describe resourcequota robot-shop-quota
```
The file sets `limits.cpu: 4`, `requests.cpu: 2`, `limits.memory: 5Gi`, `requests.memory: 3Gi`, `pods: 20`, and deliberately has **no hardcoded namespace**, so the `-n` flag decides where it lands. Now hit the ceiling on purpose:
```powershell
kubectl -n robot-shop scale deployment catalogue --replicas=15
kubectl -n robot-shop get events --sort-by=.lastTimestamp
```
Then try a debug pod with no resource spec — it will be **rejected**, because once a quota with `requests`/`limits` exists, every pod must declare them:
```powershell
kubectl -n robot-shop run tmp --image=busybox --restart=Never -- sleep 3600
```

**Learn**
- Requests vs. limits, and how the scheduler uses requests while the kubelet enforces limits.
- Why quotas cause "mysterious" pod-creation failures — a genuinely common production support ticket.
- Namespace-level multi-tenancy, the basis of platform teams' capacity model.

### Way 26 — Horizontal autoscaling under real load

**How**
```bash
# Git Bash/WSL — POSIX script, hardcodes NS="robot-shop"
./K8s/autoscale.sh
```
It runs `kubectl autoscale deployment <svc> --max 2 --min 1 --cpu-percent 50` across the eight app deployments (cart, catalogue, dispatch, payment, ratings, shipping, user, web) and then prints the HPAs. **It requires metrics-server** — without it, HPAs report `<unknown>/50%` and never scale. Then drive load:
```powershell
kubectl -n robot-shop apply -f K8s/load-deployment.yaml    # rs-load, 15 clients, error injection on
kubectl -n robot-shop get hpa -w
kubectl -n robot-shop top pods
```

**Learn**
- The full autoscaling loop: metrics-server → HPA controller → replica change → scheduler.
- Why an HPA silently does nothing without a metrics pipeline, and how to diagnose `<unknown>` targets.
- That `--max 2` makes this a demonstration, not a capacity plan — a good prompt to reason about what the real max should be.

### Way 27 — kubectl debugging drills

**How** — break things deliberately, then diagnose only with kubectl:
```powershell
helm upgrade robot-shop K8s/helm --set image.version=does-not-exist -n robot-shop
kubectl -n robot-shop get pods                  # ImagePullBackOff
kubectl -n robot-shop describe pod <pod>        # read the Events section
helm rollback robot-shop -n robot-shop
```
Other drills: `kubectl logs --previous` after a crash-loop; `kubectl exec -it <cart-pod> -- sh` then curl `catalogue:8080` from inside the cluster; `kubectl port-forward svc/catalogue 8081:8080`; delete a pod and watch the ReplicaSet replace it; `kubectl rollout restart deployment/cart`.

**Learn**
- The diagnostic ladder: `get` → `describe` (Events) → `logs` → `exec`. This is the on-call muscle memory.
- Cluster-internal DNS (`<service>.<namespace>.svc.cluster.local`) proven from inside a pod.
- `port-forward` as a debugging tool that bypasses Ingress entirely.

### Way 28 — Persistent storage in Kubernetes

**How** — the chart's only stateful workload is Redis (`redis-statefulset.yaml`, with `storageClassName` from `.Values.redis.storageClassName`, default `standard`):
```powershell
kubectl -n robot-shop get pvc,pv
kubectl -n robot-shop delete pod redis-0
kubectl -n robot-shop get pvc        # PVC survives; data persists across the pod's life
```
Then extend it: give MongoDB and MySQL real PVCs too — the chart runs both as plain Deployments with no storage, the same defect as Compose (Way 10).

**Learn**
- StatefulSet vs. Deployment, and why stable identity plus per-pod volumes matter for databases.
- PVC/PV/StorageClass binding, and how `storageClassName` becomes `gp2` on EKS or `default` on AKS.
- That this sample app is not production-safe for data — recognising that is the skill.

### Way 29 — Ingress (the chart has none)

**How** — there is no Ingress template anywhere in `K8s/helm/templates/`. Add one:
```powershell
minikube addons enable ingress
```
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: robot-shop
  namespace: robot-shop
spec:
  ingressClassName: nginx
  rules:
    - host: robotshop.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 8080
```
Add `robotshop.local` to your hosts file pointing at `minikube ip`.

**Learn**
- Ingress vs. Service type LoadBalancer/NodePort — three different ways to expose the same app, with different costs.
- IngressClass and controllers: an Ingress object does nothing without a controller watching it (exactly why the AKS manifest in Way 38 hangs `<pending>` without AGIC).
- Name-based routing, the foundation for the cloud ingress work later.

---

## Track F — Progressive delivery, service mesh, GitOps

### Way 30 — Rolling updates and rollback

**How**
```powershell
helm upgrade robot-shop K8s/helm -n robot-shop --set image.version=2.1.0
kubectl -n robot-shop rollout status deployment/cart
kubectl -n robot-shop rollout history deployment/cart
kubectl -n robot-shop rollout undo deployment/cart
helm history robot-shop -n robot-shop
helm rollback robot-shop 1 -n robot-shop
```
Watch the replacement pod-by-pod with `kubectl get pods -w` while load-gen runs, and see whether any requests fail.

**Learn**
- `maxSurge`/`maxUnavailable` and why a rolling update can still drop requests without correct readiness probes.
- Two rollback layers — Kubernetes' own rollout history and Helm's release history — and when each applies.
- Why immutable image tags (Way 16) are what makes rollback meaningful.

### Way 31 — Manual blue-green

**How** — run two versions of `web` side by side and flip the Service selector:
- Deploy `web-blue` (label `version: blue`) and `web-green` (label `version: green`).
- Point the `web` Service's selector at `version: blue`, verify, then patch it to `green`:
```bash
kubectl -n robot-shop patch svc web -p '{"spec":{"selector":{"service":"web","version":"green"}}}'
```
In PowerShell the inner quotes must be escaped for the native binary — a quoting lesson in its own right:
```powershell
kubectl -n robot-shop patch svc web -p '{\"spec\":{\"selector\":{\"service\":\"web\",\"version\":\"green\"}}}'
```

**Learn**
- That a Service is just a label selector — the entire mechanism behind blue-green.
- Instant cutover and instant rollback, versus a rolling update's gradual replacement.
- The cost: double the resources during the switch.

### Way 32 — Istio: install and gateway

**How**
```powershell
istioctl install --set profile=demo -y
kubectl label namespace robot-shop istio-injection=enabled
kubectl -n robot-shop rollout restart deployment      # required — existing pods have no sidecar
kubectl -n robot-shop get pods                        # expect 2/2 containers per pod
kubectl -n robot-shop apply -f K8s/Istio/gateway.yaml
```
`gateway.yaml` declares Gateway `robotshop-gateway` (selector `istio: ingressgateway`, port 80, hosts `*`) and VirtualService `robotshop` routing to `web.robot-shop.svc.cluster.local:8080` — the destination namespace is **hardcoded**, so deploying elsewhere means editing the file. On minikube the ingress gateway is a LoadBalancer, so you need a tunnel:
```powershell
minikube tunnel        # leave running in a second terminal
kubectl -n istio-system get svc istio-ingressgateway
```

**Learn**
- Sidecar injection — and the fact that labelling a namespace does nothing until pods restart, a classic first-time mesh mistake.
- Gateway + VirtualService as the mesh's replacement for Ingress.
- Why `2/2` in `kubectl get pods` is the quickest confirmation that a mesh is actually in the request path.

### Way 33 — Istio canary release (order matters)

The repo has everything needed, but nothing documents the required order. Apply it wrong and it silently fails.

**How** — after Way 32:
```powershell
kubectl -n robot-shop apply -f K8s/Istio/payment-deployment-fix.yaml   # FIRST: the canary workload
kubectl -n robot-shop apply -f K8s/Istio/canary.yaml                   # THEN: the traffic split
kubectl -n robot-shop get pods -l service=payment                      # expect payment + payment-fix
```
Why the order matters: `canary.yaml` defines a DestinationRule with subsets `production` (label `stage: prod`) and `canary` (label `stage: test`), plus a VirtualService splitting **99% / 1%**. The chart's `payment-deployment.yaml` carries `stage: prod`, so the production subset resolves. The `canary` subset is served **only** by `payment-deployment-fix.yaml` (Deployment `payment-fix`, image `robotshop/rs-payment-fix:latest`, labels `service: payment` + `stage: test`). Apply `canary.yaml` alone and the canary subset has zero endpoints, so ~1% of payment calls simply fail.

Then shift the weights in `canary.yaml` (99/1 → 75/25 → 50/50 → 0/100), re-apply, and watch the split with load running.

**Learn**
- Subset-based traffic splitting: DestinationRule defines *who*, VirtualService defines *how much*.
- That one Service fronting two Deployments is what makes canarying possible — the `payment` Service selects only `service: payment`, deliberately ignoring `stage`.
- Debugging an empty subset — "no healthy upstream" is the symptom you will meet in production.
- Progressive delivery as a percentage dial rather than an all-or-nothing deploy.

### Way 34 — GitOps with Argo CD

**How**
```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd port-forward svc/argocd-server 8081:443
```
Then register this repo's chart:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: robot-shop
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <your-fork-url>
    path: K8s/helm
    targetRevision: master
    helm:
      parameters:
        - name: nodeport
          value: "true"
  destination:
    server: https://kubernetes.default.svc
    namespace: robot-shop
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```
Prove self-healing: `kubectl -n robot-shop delete deployment cart` and watch Argo CD put it back. Then change a value in `K8s/helm/values.yaml`, push, and watch it reconcile.

**Learn**
- Push-based CI deploys vs. pull-based GitOps reconciliation, and why the latter removes cluster credentials from CI entirely.
- Drift detection and self-healing — the cluster converging on git rather than on whoever ran kubectl last.
- That the repo becomes the audit log of every production change.

---

## Track G — Cloud and Infrastructure as Code

### Way 35 — EKS the documented way (and finish the missing step)

**How** — work through the five docs in order, filling their gaps:

| Doc | What it does | Gap you must fill |
|---|---|---|
| [01-prerequisites.md](EKS/01-prerequisites.md) | Names kubectl, eksctl, AWS CLI | No commands at all; no `aws configure`; never mentions Helm even though doc 04 needs it |
| [02-eks-cluster-setup.md](EKS/02-eks-cluster-setup.md) | `eksctl create cluster --name demo-cluster-three-tier-1 --region us-east-1` | Heading says "Fargate" but the command creates a **managed node group** — no `--fargate` flag. Add `aws eks update-kubeconfig`, expect ~20 minutes, and note this starts billing |
| [03-oidc-IAM.md](EKS/03-oidc-IAM.md) | Associates the OIDC provider for IRSA | Bash-only (`export`, `$( )` break in PowerShell — use Git Bash/WSL). Never explains *why* OIDC is needed |
| [04-alb-configuration.md](EKS/04-alb-configuration.md) | IAM policy + IRSA service account + Helm-installs the AWS Load Balancer Controller | Doesn't tell you how to find the account ID or VPC ID, and there's a **stray trailing `\`** that breaks copy-paste. Subnets also need `kubernetes.io/role/elb=1` tags |
| [05-ebs-csi-driver.md](EKS/05-ebs-csi-driver.md) | IRSA role + EBS CSI addon | Prose bug: says "replace `<AWS-ACCOUNT-ID>` with the name of your cluster". No verification step |

Discover the placeholders the docs never explain:
```bash
aws sts get-caller-identity --query Account --output text
aws eks describe-cluster --name demo-cluster-three-tier-1 --query cluster.resourcesVpcConfig.vpcId --output text
```
**There is no doc 06 — the EKS docs never actually deploy Robot Shop.** Here is the missing last mile:
```powershell
aws eks update-kubeconfig --name demo-cluster-three-tier-1 --region us-east-1
helm install robot-shop --namespace robot-shop --create-namespace EKS/helm --set nodeport=true
kubectl -n robot-shop apply -f EKS/helm/ingress.yaml      # NOT rendered by helm - see Way 37
kubectl -n robot-shop get ingress robot-shop -w           # wait for the ALB hostname
```
Tear down when finished: `helm uninstall`, `kubectl delete ingress`, then `eksctl delete cluster --name demo-cluster-three-tier-1 --region us-east-1`.

**Learn**
- A real managed-Kubernetes setup end to end: control plane, node group, kubeconfig, IRSA, ingress controller, CSI driver.
- IRSA/OIDC — how a pod assumes an AWS IAM role without static keys. This is the modern cloud-identity pattern and a very common interview topic.
- Reading a thin internal runbook critically and filling its gaps, which is most of the job in a real team.

### Way 36 — EKS with Terraform instead

**How** — create `terraform/` and provision what docs 01–03 do by hand:
```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = var.aws_region
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.cluster_name}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["${var.aws_region}a", "${var.aws_region}b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  # required so the ALB controller can discover subnets
  public_subnet_tags  = { "kubernetes.io/role/elb"          = 1 }
  private_subnet_tags = { "kubernetes.io/role/internal-elb" = 1 }
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = "1.29"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  enable_irsa = true

  eks_managed_node_groups = {
    default = {
      instance_types = ["t3.medium"]
      min_size       = 1
      max_size       = 3
      desired_size   = 2
    }
  }
}
```
Each argument goes on its own line — comma-separated assignments on one line are invalid HCL. Then:
```powershell
cd terraform
terraform init
terraform plan -out=tfplan
terraform apply tfplan
terraform destroy
```
Extensions: remote state in S3 with DynamoDB locking; `terraform fmt`/`validate` in CI; a `modules/` split; workspaces for dev/prod.

**Learn**
- Declarative infrastructure and the plan/apply cycle — seeing a diff before it happens.
- State: what it is, why it must be remote and locked for a team, and what breaks when it drifts.
- Module composition and provider versioning.
- The real contrast with Way 35: the same cluster, reproducible in one command instead of five documents.

### Way 37 — Cloud ingress, and the trap that hides it

**How** — the ingress manifests live in the **chart root**, not `templates/`, so `helm install` never renders them. They must be applied separately:
```powershell
kubectl -n robot-shop apply -f EKS/helm/ingress.yaml    # ALB
kubectl -n robot-shop apply -f AKS/helm/ingress.yaml    # Application Gateway
kubectl -n robot-shop apply -f GKE/helm/ingress.yaml    # GCE  (use this, NOT gclb.yaml)
```
What each needs:
- **EKS** — `kubernetes.io/ingress.class: alb`, `scheme: internet-facing`, `target-type: ip`, backend `web:8080`. Requires the AWS Load Balancer Controller (doc 04) and elb-tagged subnets. Uses the deprecated class annotation rather than `ingressClassName`.
- **AKS** — `ingressClassName: azure-application-gateway`, no annotations. Requires the AGIC add-on and an existing Application Gateway, or it sits `<pending>` forever.
- **GKE** — use `GKE/helm/ingress.yaml` (valid v1, `ingressClassName: gce`, backend `web:8080`). **`GKE/helm/gclb.yaml` is broken**: it declares `networking.k8s.io/v1` but uses a v1beta1 body (`spec.backend.serviceName`), and targets a service called `robot-shop` that doesn't exist — the service is `web`.

Also pass `--set nodeport=true`, or the chart's default `type: LoadBalancer` on `web` provisions a second cloud load balancer competing with your Ingress.

**Learn**
- Helm's rendering rule — only files under `templates/` become manifests — which explains an entire category of "I installed the chart but nothing happened".
- Cloud ingress controllers as the bridge between a Kubernetes object and a real cloud load balancer.
- Reading a manifest's apiVersion against its body to spot version-skew bugs (the `gclb.yaml` case) before the cluster rejects it.

### Way 38 — Port it to AKS and GKE

The charts exist; the setup docs do not. Write them.

**How** — AKS:
```powershell
az group create --name robot-shop-rg --location eastus
az aks create --resource-group robot-shop-rg --name robot-shop-aks --node-count 2 --generate-ssh-keys
az aks get-credentials --resource-group robot-shop-rg --name robot-shop-aks
helm install robot-shop --namespace robot-shop --create-namespace AKS/helm --set nodeport=true
```
GKE:
```powershell
gcloud container clusters create robot-shop-gke --num-nodes=2 --zone=us-central1-a
gcloud container clusters get-credentials robot-shop-gke --zone=us-central1-a
helm install robot-shop --namespace robot-shop --create-namespace GKE/helm --set nodeport=true
```
**Important:** across all four charts (`K8s`, `EKS`, `AKS`, `GKE`) exactly one line differs — `redis.storageClassName` (`standard` / `gp2` / `default` / `default`). Worse, the three cloud charts **hardcode** it inside `templates/redis-statefulset.yaml`, so `--set redis.storageClassName=...` is silently ignored there; only `K8s/helm` keeps it templated. Fixing that (making all four charts parameterised, or collapsing them into one chart with per-cloud values files) is itself an excellent exercise.

Then write the missing `AKS/01-…` and `GKE/01-…` docs in the same numbered style as `EKS/`, and add Terraform modules using `azurerm_kubernetes_cluster` / `google_container_cluster`.

**Learn**
- Where cloud abstractions leak: storage classes, ingress controllers, and identity (IRSA vs. Workload Identity vs. Managed Identity).
- Four near-identical charts is a maintenance anti-pattern — recognising duplication as a defect is a platform-engineering instinct.
- Writing the docs you wished existed, against a template you have already followed.

### Way 39 — Ansible for configuration management

**How** — create `ansible/` and bootstrap a host that Terraform created:

`ansible/inventory.ini`
```ini
[docker_hosts]
node1 ansible_host=<public-ip> ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
```

`ansible/bootstrap-docker-host.yml`
```yaml
- hosts: docker_hosts
  become: true
  tasks:
    - name: Install Docker
      apt:
        name: docker.io
        state: present
        update_cache: true

    - name: Ensure Docker is running
      systemd:
        name: docker
        state: started
        enabled: true

    - name: Allow the login user to run Docker
      user:
        name: "{{ ansible_user }}"
        groups: docker
        append: true

    - name: Pre-pull the base images this repo actually uses
      community.docker.docker_image:
        name: "{{ item }}"
        source: pull
      loop:
        - nginx:1.21.6
        - node:14
        - debian:10
        - openjdk:8-jdk
        - php:7.4-apache
        - python:3.9
        - golang:1.23
```
(That list is the real set of `FROM` images in this repo — the same job [pullbaseimages.sh](pullbaseimages.sh) does imperatively.)
```powershell
ansible-playbook -i ansible/inventory.ini ansible/bootstrap-docker-host.yml
ansible-playbook -i ansible/inventory.ini ansible/bootstrap-docker-host.yml   # expect "changed=0"
```
Then go further: a playbook that copies the compose file to the host and runs the stack, and a rewrite of [OpenShift/setup.sh](OpenShift/setup.sh) as an idempotent playbook.

**Learn**
- Idempotency — proven, not assumed, by the second run reporting zero changes.
- The Terraform/Ansible boundary: provisioning infrastructure vs. configuring what runs on it.
- Inventory, become/privilege escalation, and modules vs. raw `shell` commands (and why reaching for `shell` is usually a smell).

### Way 40 — Cost control and teardown discipline

**How**
- Before any cloud track, set an AWS Budgets alert (or GCP/Azure equivalent) at a threshold you'd hate to exceed.
- Write `scripts/teardown-eks.sh` that removes things in dependency order: ingress (so the ALB is deleted) → Helm release → cluster.
- Know what bills even when idle: the EKS control plane charges per hour whether or not workloads run; NAT gateways, load balancers, and EBS volumes all charge while they exist.
- Use `eksctl delete cluster` and confirm in the console — orphaned load balancers and volumes are the classic surprise charge.

**Learn**
- FinOps basics — the cost model of managed Kubernetes, and why "I only ran it for a weekend" still produces a bill.
- Dependency-ordered teardown, the same reasoning as `terraform destroy`'s graph.
- The professional habit of provisioning with an exit plan.

---

## Track H — Observability

### Way 41 — Prometheus and Grafana against the Compose stack

First, know what this app actually exports — these are the real metric names, not generic examples:

| Service | Metric | Type | Meaning |
|---|---|---|---|
| cart | `items_added` | Counter | items added to carts (the *only* metric cart exports — it uses a custom registry, so there are no default Node.js metrics) |
| payment | `sold_count` | Counter | items sold |
| payment | `units_sold` | Histogram | units per sale (buckets 1, 2, 5, 10, 100) |
| payment | `cart_value` | Histogram | value per sale (buckets 100 … 10000) |

**How** — create `observability/prometheus.yml`:
```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: cart
    metrics_path: /metrics
    static_configs:
      - targets: ['cart:8080']
  - job_name: payment
    metrics_path: /metrics
    static_configs:
      - targets: ['payment:8080']
```
and `observability/docker-compose.yaml`:
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports: ["9090:9090"]
    networks: [robot-shop]
  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    networks: [robot-shop]
networks:
  robot-shop:
    external: true
    name: three-tier-architecture-robot-shop_robot-shop
```
Joining the app's own network (check the real name with `docker network ls`) is more reliable than `host.docker.internal` on Docker Desktop, and lets you scrape services directly by name instead of going through nginx. Then:
```powershell
docker compose -f observability/docker-compose.yaml up -d
# Prometheus http://localhost:9090   Grafana http://localhost:3000 (admin/admin)
```
Build panels on: `rate(items_added[5m])`, `rate(sold_count[5m])`, `histogram_quantile(0.95, rate(cart_value_bucket[5m]))`.

**Learn**
- The pull model — Prometheus scrapes targets; targets don't push.
- Counters vs. histograms, `rate()`, and why `histogram_quantile` needs the `_bucket` series.
- Container networking for monitoring tools: why joining the app network beats host networking on Docker Desktop.
- That a business metric (`cart_value`) is often more useful than a system metric — this app deliberately exposes both kinds.

### Way 42 — kube-prometheus-stack on Kubernetes

**How**
```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80
```
Then add ServiceMonitors so Prometheus discovers cart and payment:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: robot-shop-payment
  namespace: monitoring
  labels:
    release: monitoring
spec:
  namespaceSelector:
    matchNames: [robot-shop]
  selector:
    matchLabels:
      service: payment
  endpoints:
    - port: http
      path: /metrics
```
(Check the chart's actual service labels and port names with `kubectl -n robot-shop get svc payment -o yaml` and adjust.)

**Learn**
- The Operator pattern — CRDs like ServiceMonitor turn "edit a config file" into "create an object".
- Label-selector-driven service discovery, versus the static target list from Way 41.
- Why cluster monitoring is installed as a platform capability rather than per-app.

### Way 43 — Logging with the repo's fluentd

**How** — read what's actually here first: [fluentd/](fluentd/) builds `robotshop/fluentd:elastic` and ships **two** configs.
- `fluentd/Docker-Compose/fluent.conf` uses `@type forward` on port 24224 (it does *not* tail files), adds container metadata, and outputs via the **elasticsearch** plugin to `cloud.humio.com:9200` with **placeholder credentials** — it ships nothing until you edit them. To use it you must also switch the app's logging driver in `docker-compose.yaml` to `fluentd` with `fluentd-address: localhost:24224`.
- `fluentd/Kubernetes/fluentd.yaml` is a DaemonSet in namespace `logging` that tails `/var/log/containers/*.log`, enriches with `kubernetes_metadata`, drops `kube-system`, and targets the same Humio endpoint.

The practical exercise is to repoint it at something you can actually run:
```yaml
# replace the elasticsearch <match> block with Loki
<match **>
  @type loki
  url http://loki:3100
  <label>
    container $.docker.container_id
  </label>
</match>
```
Run Loki + Grafana locally, then query logs in Grafana beside the metrics from Way 41.

**Learn**
- The logging pipeline shape: source → filter/enrich → output, and why the forward driver and the tail source solve different problems.
- Docker logging drivers, and the risk that an unreachable log backend blocks container startup.
- Structured logging and metadata enrichment — how a log line gets attributed to a pod, container and namespace.

### Way 44 — Alerting

**How** — add rules to Prometheus:
```yaml
groups:
  - name: robot-shop
    rules:
      - alert: NoSalesForTenMinutes
        expr: rate(sold_count[10m]) == 0
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "No purchases completed in 10 minutes"

      - alert: CartServiceDown
        expr: up{job="cart"} == 0
        for: 2m
        labels: { severity: critical }
```
Wire Alertmanager to a webhook, then trigger both deliberately: stop RabbitMQ (payments stop completing) and stop cart.

**Learn**
- Symptom-based alerting (`no sales`) vs. cause-based (`service down`) — and why the business-level alert is usually the one that matters.
- `for:` durations and why they suppress flapping.
- The `up` metric, which you get free for every scrape target.

### Way 45 — SLOs and error budgets

**How** — define an SLI from what you can actually measure here: successful checkout rate, derived from `sold_count` against load-gen's request volume, or availability from `up`. Set an SLO (say 99% over 30 days), then build a Grafana panel showing the remaining error budget. Break it on purpose with Way 8 and watch the budget burn.

**Learn**
- SLI/SLO/error-budget vocabulary, and translating a business goal into a query.
- Why error budgets are a release-decision tool, not a reporting tool.
- Choosing measurable indicators — often the hardest part, and this app's limited metric set makes that constraint real.

### Way 46 — Chaos with observability on

**How** — with dashboards (Way 41) and load (Way 7) both running, inject failures and correlate:
```powershell
docker compose stop rabbitmq      # payment keeps accepting; dispatch stops consuming
docker compose pause mysql        # shipping/ratings hang rather than fail fast
docker compose stop redis         # cart fails immediately
```
Write an incident timeline for each: what the dashboard showed, which log line was the first real signal, how long detection took, and what the alert *should* have been. On Kubernetes, go further with `kubectl delete pod` loops, or a CPU-stress sidecar to trigger the HPA.

**Learn**
- Correlating a metric dip, a log spike and a user-visible symptom into one story — the actual skill of an incident responder.
- The difference between a crash (fails fast) and a hang (`pause` — much harder to detect), and why timeouts matter.
- Writing blameless postmortems from evidence you collected yourself.

---

## Track I — Alternate orchestrators

### Way 47 — Docker Swarm (modern path)

Skip [Swarm/create-swarm.sh](Swarm/create-swarm.sh) for your first run: it depends on `docker-machine`, which is end-of-life and needs VirtualBox. Use your existing Docker Desktop engine instead.

**How**
```powershell
docker swarm init
docker stack deploy -c docker-compose.yaml robot-shop
docker stack services robot-shop
docker service ls
docker service logs robot-shop_web
docker service scale robot-shop_catalogue=3
docker service update --image robotshop/rs-cart:2.1.0 robot-shop_cart    # rolling update
docker service rollback robot-shop_cart
docker stack rm robot-shop
docker swarm leave --force
```
Expect warnings: `docker stack deploy` **ignores** several Compose keys, notably `build:`, `depends_on` and `container_name`. That is not a bug in the repo — it is Swarm's model, and reading which keys were dropped is the exercise. `docker-compose.yaml` is `version: '3'`, which stack deploy accepts.

Once that works, read [Swarm/deploy.sh](Swarm/deploy.sh) to see the scripted version (it sources `../.env` and runs `docker stack deploy -c ../docker-compose.yaml`, so it must run from inside `Swarm/` in Git Bash).

**Learn**
- A second orchestrator's vocabulary — services, tasks, stacks — mapped onto Kubernetes' Deployments, Pods and releases.
- Built-in rolling updates and rollback with far less machinery than Kubernetes.
- Which Compose semantics survive the jump to a scheduler and which don't — a sharp lesson in why `depends_on` has no meaning in a distributed scheduler.
- Why the industry consolidated on Kubernetes anyway.

### Way 48 — OpenShift

**How** — use OpenShift Local (CRC), the current replacement for the Minishift the repo assumes:
```powershell
crc setup
crc start
oc login -u developer https://api.crc.testing:6443
oc new-project robot-shop
helm install robot-shop --set openshift=true --set nodeport=true K8s/helm
```
[OpenShift/README.md](OpenShift/README.md) covers both OCP 3.x and 4.x and uses correct Helm 3 syntax. [OpenShift/setup.sh](OpenShift/setup.sh) assumes a dev cluster where `system:admin` logs in without a password, and its two `add-scc-to-user` calls omit `-n robot-shop`, so they apply to whatever project is current — read it before running it. Note also that `openshift: false` exists in `values.yaml` but is referenced by no template; `ocCreateRoute` is the key that actually does something.

**Learn**
- Security Context Constraints — why an image that runs fine on vanilla Kubernetes is rejected here for running as root. This is the single biggest practical difference.
- Routes vs. Ingress, Projects vs. Namespaces.
- Reading a vendor distribution's opinions as constraints rather than bugs.

### Way 49 — DC/OS and Marathon (read-only study)

Standing up DC/OS is not worth it today, but the manifests are worth reading.

**How** — read [DCOS/deploy.sh](DCOS/deploy.sh) (loops `dcos marathon app add` over `manifest/*.json`, requires the `dcos` CLI already attached to a cluster, and must run from `DCOS/`), [DCOS/destroy.sh](DCOS/destroy.sh) (a hardcoded list of eleven removals that will drift), and two or three manifests. Compare a Marathon app definition against the equivalent Kubernetes Deployment.

Spot the defects while you're there: `manifest/mysql.json` references image `robotshop/rs-shipping-db` (Compose builds `rs-mysql-db`) and sets `containerPort: 0`; every service-to-service hostname is a hardcoded `*.marathon.l4lb.thisdcos.directory` VIP; and images are pinned to `:latest` with no `${REPO}`/`${TAG}` indirection, so you cannot repoint them at your own registry without editing all eleven files.

**Learn**
- That container scheduling concepts predate Kubernetes and largely transfer (health checks, upgrade strategy, resource requests, service VIPs).
- How hardcoded service addresses make an app deployment-target-specific — the problem service discovery abstractions exist to solve.
- Recognising drift between a deployment manifest and the build system that produces the images.

---

# Part 2 — What each microservice teaches you

The same app, but each service is the best teacher for a different lesson.

| Service | The lesson it uniquely teaches |
|---|---|
| **web** | The only service with a fixed host port, so it's the one that *can't* scale (Way 9) — the clearest demonstration of why port mappings and horizontal scaling conflict. Also the app's routing table: `web/default.conf.template` shows every `/api/*` prefix and its backend, rendered at runtime by `envsubst` in `entrypoint.sh`. And it has **no CI workflow** — a blank slate to write one (Way 17). |
| **cart** | Session state in Redis: stop Redis and cart dies instantly (Way 8), the textbook example of a hard dependency. Exports `items_added`, your first Prometheus counter. Its CI workflow is the "reference" one — and pins `actions/checkout@v7`, a tag that doesn't exist, so it teaches that a merged pipeline can still be broken. |
| **catalogue** | MongoDB-backed reads, and the CI file that gets image tagging *right* (`github.run_number`, immutable) while getting login *wrong* (`docker login -p`). Good vs. bad practice in one 40-line file. |
| **user** | Auth and sessions across both MongoDB and Redis. `user/server.js:129` compares passwords in plaintext with `==` — a security exhibit you can point at in a review. Its workflow has the worst path filter in the repo (`'**.js'` fires on cart, catalogue, mongo seeds and the frontend). |
| **shipping** | The only **multi-stage** Dockerfile (`debian:10` build → `openjdk:8-jdk` runtime) and by far the slowest build — so it's where build caching actually pays (Way 18). Its workflow carries the `name: Dispatch` collision and the piped-apt bug whose exit code comes from the *last* command, so a failed install reports success. |
| **ratings** | The only service with **no CI workflow at all**, and the only one whose container listens on **port 80** rather than 8080 (see its healthcheck and the nginx route). Its Dockerfile pulls unpinned `composer:latest` at build time — a live supply-chain smell (Way 3, Way 20). |
| **payment** | The richest observability target: `sold_count`, `units_sold` and `cart_value` are *business* metrics, not CPU graphs (Way 41, Way 44). It's also the RabbitMQ producer, so stopping RabbitMQ shows a soft failure. And it's the Istio canary subject — `payment-deployment-fix.yaml` is the v2 workload behind the 99/1 split (Way 33). |
| **dispatch** | The Go service that ships its entire `golang:1.23` toolchain because nobody added a second build stage — the best multi-stage refactor exercise (Way 3). As RabbitMQ's *consumer* it has no healthcheck in Compose, so it's invisible in `docker compose ps` health terms. Its workflow has the silent-pass `curl | export` bug. |
| **load-gen** | Not a service to deploy — the tool that makes every other track meaningful. Nothing about autoscaling, dashboards, alerting or chaos means anything without traffic. |
| **fluentd** | Log forwarding as a standalone concern, with two different collection models (forward vs. tail) for two different platforms, and placeholder credentials that force you to understand the output plugin before anything ships. |
| **mongo / mysql** | Seed data and demo credentials (`MYSQL_PASSWORD=secret`, plaintext mongo users). Neither has a volume in Compose or a PVC in the chart — the persistence gap you fix in Ways 10 and 28. |

---

# Part 3 — A suggested order

You can jump around, but this sequence means every step builds on something already working.

| Stage | Ways | Why here |
|---|---|---|
| **1. Containers** | 1 → 5 | Everything downstream is images. Way 2 (raw `docker run`) is the single most clarifying exercise in this list. |
| **2. Compose and day-2** | 6 → 11 | A working stack you can break, scale, back up and re-provision at will. |
| **3. Automation** | 12 → 14 | Scripts make the earlier steps repeatable; you'll reuse the smoke test in CI and as a readiness probe. |
| **4. CI/CD** | 15 → 20 | Way 15 is the flagship exercise. Do it on paper before the answer key. |
| **5. Kubernetes** | 21 → 29 | The largest block. Don't skip Way 23 (`helm template`) — it demystifies everything after it. |
| **6. Delivery and mesh** | 30 → 34 | Needs a working cluster. Way 33's ordering trap is the kind of thing that only teaches once. |
| **7. Cloud and IaC** | 35 → 40 | Costs real money — read Way 40 *first*, and set the budget alert before the cluster. |
| **8. Observability** | 41 → 46 | Can be done any time after Stage 2, but it's most valuable once there's load and autoscaling to watch. |
| **9. Other orchestrators** | 47 → 49 | Comparison and perspective, once Kubernetes vocabulary is second nature. |

Two habits worth keeping throughout:

1. **Write down what broke and why.** The repo's stale corners (Part 0.4) are the highest-value material in it, and they only pay off if you record the diagnosis.
2. **Tear down cloud resources the same day you create them.** Way 40 exists because this is the one mistake that costs money rather than time.
