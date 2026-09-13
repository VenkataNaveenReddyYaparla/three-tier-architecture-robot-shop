# 📚 DevOps Learning Path — Stan's Robot Shop

A single progression built around one question at each step: **what does this tool fix that the previous one couldn't?**

Design of the app → Dockerfiles → CI → the registry → plain Docker → Compose → Kubernetes → EKS/OpenShift → observability → Terraform/Ansible → Argo CD.

**Your setup (assumed throughout):** Windows 11 + Docker Desktop (WSL2 backend), **Oracle Linux on WSL** as the Linux workstation, and an AWS account. Commands marked `powershell` run on Windows; commands marked `bash` run in your WSL Oracle Linux shell — every `.sh` file in this repo is POSIX shell and belongs there. Docker Desktop shares its engine with WSL, so `docker` works in both (enable it under Settings → Resources → WSL Integration).

> [!NOTE]
> **Status:** the CI workflows that were originally broken (`dispatch`, `payment`, `shipping`, `user`) have all been **fixed**, and the two that were missing (`ratings`, `web`) have been **written**. All eight now build and push to the `naveenreddy9` Docker Hub namespace. Parts 3.3–3.5 keep the original diagnosis as the reference material — they're written as solved exercises, not pending ones.

---

# 🏗️ Part 1 — How this project is designed

## 1.1 The shape of it

Stan's Robot Shop is a deliberately conventional microservices app: a browser talks to **one** entry point, and that entry point fans out to eight services, each owning its own data store.

```
                    browser  (http://localhost:8080)
                        |
                  +-----------+
                  |    web    |   nginx, serves the AngularJS UI
                  +-----------+   and reverse-proxies every /api/* path
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

## 1.2 Who talks to what

| Caller | Depends on | Failure style if the dependency dies |
|---|---|---|
| web | catalogue, user, shipping, payment | 502s on those routes; the UI still loads |
| catalogue | MongoDB | hard — product listing fails |
| user | MongoDB, Redis | hard — login/session fails |
| cart | Redis | hard — cart dies instantly |
| shipping | MySQL | hangs rather than fails fast |
| ratings | MySQL | hard — ratings fail |
| payment | RabbitMQ | **soft** — payment still succeeds, message queues up |
| dispatch | RabbitMQ | silent — it simply stops consuming |

That payment/dispatch split is the most instructive pair in the app: it's asynchronous, so a RabbitMQ outage produces no user-visible error at all — only a growing queue and a silent consumer. Learning to notice that kind of failure is Part 9.

## 1.3 The routing table

Everything external goes through nginx. `web/default.conf.template` is the real routing table, rendered at container start by `envsubst` in `entrypoint.sh`:

| Path | Backend |
|---|---|
| `/api/catalogue/` | `catalogue:8080` |
| `/api/user/` | `user:8080` |
| `/api/cart/` | `cart:8080` |
| `/api/shipping/` | `shipping:8080` |
| `/api/payment/` | `payment:8080` |
| `/api/ratings/` | `ratings:80` ← **port 80, the only one** |
| `/nginx_status` | nginx stub status |

## 1.4 What's deliberately (or accidentally) wrong with it

The upstream README says it plainly: error handling is patchy and there's no security. That's what makes it good practice material — these are all real findings you'd write up in a review:

- **No volumes anywhere in `docker-compose.yaml`.** Every database writes to the container's writable layer, so `docker compose down` destroys the data.
- **Only the app services define healthchecks.** MongoDB, MySQL, Redis, RabbitMQ and `dispatch` have none, so they never report `healthy`.
- **Plaintext passwords.** `user/server.js:129` compares with `==`, and `mongo/users.js` seeds users like `user/password` and `stan/bigbrain`. MySQL ships `MYSQL_PASSWORD=secret`, RabbitMQ uses `guest/guest`.
- **Several base images are end-of-life:** `php:7.4` (ratings), `python:3.9` (payment), `nginx:1.21.6` (web), and `maven:3.6.3-jdk-8` in shipping's build stage. The Node services and shipping's *runtime* stage have since been moved to supported versions — see Part 2.4.
- **`.gitignore` is five lines** and excludes none of `.env`, `*.pem`, `*.key`, or `kubeconfig` — while `EKS/04-alb-configuration.md` tells you to download `iam_policy.json` straight into the repo.

Keep a running list as you go. The habit of recording *what was wrong and why* is most of what separates someone who "did a tutorial" from someone who can audit a system.

---

# 🐳 Part 2 — Dockerfile design and best practices

## 2.1 What a Dockerfile actually is

Three ideas explain almost everything:

1. **Every instruction creates a layer**, and layers are cached. If a layer's inputs are unchanged, Docker reuses it and skips the work.
2. **The cache invalidates top-down.** Change something at line 4 and every layer after it rebuilds, regardless of whether it needed to.
3. **The build context** — the directory you pass to `docker build` — is uploaded to the daemon before the build starts. That's separate from what ends up in the image.

Consequence, and the single most important design rule: **copy your dependency manifest and install dependencies *before* copying your source code.** Source changes constantly; dependencies rarely do.

## 2.2 Case studies from this repo

### `cart/Dockerfile` — cache ordering done right

```dockerfile
FROM node:20-alpine
EXPOSE 8080
WORKDIR /opt/server
COPY package.json /opt/server/     # 1. manifest only
RUN npm install                    # 2. cached unless package.json changes
COPY server.js /opt/server/        # 3. source last
CMD ["node", "server.js"]
```
Editing `server.js` re-runs only the last two steps; `npm install` stays cached. This is the pattern to internalise.

The base was originally `node:14` — EOL since April 2023 — and is now `node:20-alpine`. That change is written up with measurements in Part 2.4; `catalogue` and `user` got the same treatment.

**Still wrong with it:** `npm install` should be `npm ci` (reproducible, uses the lockfile); it runs as root; there's no `HEALTHCHECK` and no `.dockerignore`. And note what alpine removes — there's no `curl` in the image any more, while `docker-compose.yaml`'s healthcheck for this service still shells out to `curl`. Either add `RUN apk add --no-cache curl` or switch the healthcheck to `wget -q --spider` (busybox `wget` *is* present).

### `shipping/Dockerfile` — multi-stage, with the cache trick undone

```dockerfile
FROM  maven:3.6.3-jdk-8 AS build
WORKDIR /opt/shipping
COPY pom.xml /opt/shipping/
COPY src /opt/shipping/src/
RUN mvn package

FROM eclipse-temurin:25-jre-alpine                    # runtime stage
EXPOSE 8080
WORKDIR /opt/shipping
ENV CART_ENDPOINT=cart:8080
ENV DB_HOST=mysql
COPY --from=build /opt/shipping/target/shipping-1.0.jar shipping.jar
CMD [ "java", "-Xmn256m", "-Xmx768m", "-jar", "shipping.jar" ]
```
The multi-stage split is the thing to copy: Maven and the whole JDK (~500MB of build tooling) never reach the final image, which is a JRE on alpine. Using `maven:3.6.3-jdk-8` as the build stage also replaced a hand-rolled `apt-get install maven` — letting an official image supply the toolchain is almost always better than installing it yourself.

**The regression worth spotting:** `COPY pom.xml` sits immediately above `COPY src`, with **no `RUN mvn dependency:resolve` between them**. Splitting the copies only helps if something *consumes* the manifest before the source arrives. As written, editing any `.java` file invalidates the `COPY src` layer, and `mvn package` re-downloads every dependency from scratch. The fix is one line:

```dockerfile
COPY pom.xml /opt/shipping/
RUN mvn dependency:go-offline        # <-- dependency layer, cached
COPY src /opt/shipping/src/
RUN mvn package -o
```

**Also still wrong:** the build stage is JDK 8 while the runtime is JRE 25 — compiling against an ancient JDK to run on a modern JVM is backwards, and `maven:3.6.3-jdk-8` is long EOL. The fixed heap flags (`-Xmn256m -Xmx768m`) predate JVM container awareness; modern JVMs read cgroup limits automatically, so these override something the JVM would get right on its own.

### `dispatch/Dockerfile` — the refactor, done

This was the worst file in the repo: single-stage `golang:1.23`, shipping the entire Go toolchain to run a ~10MB binary, with `go mod init` generating the module at build time and a shell-form `CMD`. It now reads:

```dockerfile
FROM golang:1.24 AS builder
WORKDIR /go/src/app
COPY *.go .
RUN go mod init dispatch                              # creates go.mod
RUN go mod tidy                                       # downloads deps, creates go.sum
RUN CGO_ENABLED=0 go build -o dispatch .              # static binary, no runtime libs

FROM alpine:latest
WORKDIR /app
COPY --from=builder go/src/app/dispatch .
CMD ["./dispatch"]
```
Three of the four original problems are gone: there's a second stage, `CGO_ENABLED=0` produces a static binary that needs no runtime libraries, and `CMD` is exec form so the process gets PID 1 and receives signals.

**What's still open:**
- **`go mod init` / `go mod tidy` still run at build time**, so there's still no committed `go.mod`/`go.sum`. Dependency versions are resolved fresh on every build — not reproducible, and nothing to audit. Running `go mod init && go mod tidy` locally and committing both files is the real fix.
- **`FROM alpine:latest`** is unpinned, and a static binary doesn't need alpine at all. `gcr.io/distroless/static-debian12` or even `FROM scratch` would work and cut the last few MB.
- **No `USER`** — it still runs as root.

### `ratings/Dockerfile` — the security exhibit

```dockerfile
COPY --from=composer /usr/bin/composer /usr/bin/composer   # UNPINNED image
RUN composer install
RUN rm -Rf /var/www/var/*
RUN chown -R www-data /var/www
RUN chmod -R 777 /var/www                                   # world-writable
```
`COPY --from=composer` with no tag silently pulls `composer:latest` — a different build on a different day gets different tooling. And `chmod 777` is world-writable; the file even carries a comment admitting it's a shortcut for the demo. Three consecutive `RUN`s also create three layers where one would do.

### `web/Dockerfile` — configuration at runtime, not build time

```dockerfile
ENV CATALOGUE_HOST=catalogue USER_HOST=user CART_HOST=cart ...
COPY entrypoint.sh /root/
ENTRYPOINT ["/root/entrypoint.sh"]
COPY default.conf.template /etc/nginx/conf.d/default.conf.template
```
The nginx config is a *template*; the entrypoint runs `envsubst` at container start. So one image works in every environment, configured by environment variables. This is the twelve-factor pattern, and it's why the same image runs unchanged under Compose, Kubernetes and EKS.

### `payment/Dockerfile` — the anti-pattern in one line

`USER root` is set explicitly. Containers should drop to an unprivileged user; this one goes the other way.

## 2.3 The checklist

Derived from the six files above — every item has a real example in this repo:

| Practice | Where this repo gets it right / wrong |
|---|---|
| Manifest before source, for cache | cart ✅, catalogue ✅, user ✅; shipping ❌ (no dependency-resolve step between the copies) |
| Multi-stage for compiled languages | shipping ✅, dispatch ✅ |
| Let an official image supply the toolchain | shipping ✅ (`maven:3.6.3-jdk-8` build stage) |
| Pin every base image, including `COPY --from` | ratings ❌ (`composer:latest`), dispatch ❌ (`alpine:latest`) |
| Commit lockfiles, don't generate deps at build | dispatch ❌ (`go mod init`/`go mod tidy` at build) |
| Run as a non-root user | all ❌; payment explicitly `USER root` |
| Exec-form `CMD`/`ENTRYPOINT` | cart ✅, dispatch ✅, web ✅ |
| Runtime config via env vars, not baked in | web ✅, shipping ✅ |
| Least-privilege file permissions | ratings ❌ (`chmod 777`) |
| Keep base images in support | cart/catalogue/user ✅ (node:20); shipping runtime ✅ (temurin:25) — shipping *build* ❌ (jdk-8), ratings ❌ (php:7.4), payment ❌ (python:3.9), web ❌ (nginx:1.21.6) |
| Small runtime base | cart/catalogue/user ✅, shipping ✅, dispatch ✅ (alpine) |
| `.dockerignore` to shrink build context | none have one ❌ |

## 2.4 Exercises — and what came of them

**1. Refactor `dispatch` to multi-stage. ✅ Done.** Covered in the case study above. Three of four problems fixed; the uncommitted `go.mod` is still open.

**3. Bump `cart` off `node:14`. ✅ Done — and further than asked.** `cart`, `catalogue` and `user` all went to `node:20-alpine` in one commit. The measured result:

| Image | Before | After | Change |
|---|---|---|---|
| `cart` | 424 MB (`node:20`) | **65 MB** | −85% |
| `catalogue` | 420 MB (`node:20`) | **61 MB** | −85% |
| `user` | never built (`node:14`, CI broken) | **62 MB** | — |

Two things worth taking from that:

- **The saving is the base, not your code.** These services are a single `server.js` plus `node_modules`. Nearly everything in the original 420MB was Debian userland and build toolchain that the application never touches at runtime.
- **The risk that didn't materialise.** All three services depend on `@instana/collector`, which ships native modules. Alpine uses musl rather than glibc, so a missing musl prebuild would mean compiling from source — and alpine has no compiler. It built cleanly on the first CI run. The reason it's safe is that those native modules are declared as *optional* dependencies, so npm continues past a failed build rather than aborting. Worth knowing *why* it worked, not just that it did.
- **What it quietly broke.** Alpine has no `curl`, and `docker-compose.yaml`'s healthchecks for all three services call `curl`. Nobody noticed because the local run uses `docker-compose.local.yaml`, which declares no healthchecks (Part 6.2).

**Still to do:**

2. Add a `.dockerignore` to a Node service; watch the "sending build context" number fall.
4. Pin `COPY --from=composer` to a real version in `ratings` — and pin `alpine:latest` in `dispatch` while you're there.
5. Add a non-root `USER` to `payment` and fix whatever permissions break.
6. Add the missing `RUN mvn dependency:go-offline` to `shipping` and time a rebuild after a one-character source edit, before and after.
7. Scan before and after: `docker scout cves <image>` or `docker run --rm aquasec/trivy image <image>`. Run it against `cart:14` and `cart:15` — both are still in your registry, so you can diff a Debian base against an alpine one directly.

---

# 🔄 Part 3 — How CI works

## 3.1 The mechanics

A CI pipeline is four things, and GitHub Actions names all of them explicitly:

1. **A trigger** — `on: push` / `on: pull_request`, optionally narrowed by `paths:` so a change to one service doesn't rebuild everything.
2. **A runner** — a fresh VM that starts with *nothing*: no repo, no credentials, no state from the last run. `runs-on: ubuntu-latest` rents one from GitHub; `runs-on: self-hosted` means *you* supply the machine.
3. **Steps** — shell commands, or reusable `uses:` actions. Each step is a new shell, so `export FOO=bar` does **not** survive into the next step (you write to `$GITHUB_PATH`/`$GITHUB_ENV` for that).
4. **Secrets** — injected as `${{ secrets.NAME }}`, never committed.

Because the runner starts empty, every pipeline follows the same skeleton: *check out the code → log in to the registry → build the image → push it*.

The one mental model that matters: **the exit code of the last command in a `run:` block decides whether the step passed.** That single rule explains two of the bugs you're about to find.

## 3.2 How this repo wires it up

Eight services, eight workflows, one file each. All of them now build and push:

| Workflow | Triggered by | Pushes | Login style |
|---|---|---|---|
| `cart.yaml` | `cart/**` | `naveenreddy9/cart:<run_number>` | `--password-stdin` ✅ |
| `catalogue.yaml` | `catalogue/**` | `naveenreddy9/catalogue:<run_number>` | `-p` ❌ |
| `user.yaml` | `user/**` | `naveenreddy9/user:<run_number>` | `-p` ❌ |
| `web.yml` | `web/**` | `naveenreddy9/web:<run_number>` | `-p` ❌ |
| `rating.yaml` | `ratings/html/**`, `ratings/Dockerfile` | `naveenreddy9/rating:<run_number>` | `-p` ❌ |
| `dispatch.yaml` | `**.go`, `dispatch/**` | `naveenreddy9/dispatch:<run_number>` | `-p` ❌ |
| `payment.yaml` | `**.py` | `naveenreddy9/payment:<run_number>` | `-p` ❌ |
| `shipping.yaml` | `**.java`, `shipping/Dockerfile` | `naveenreddy9/shipping:<run_number>` | `-p` ❌ |

All eight use the same two secrets: `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` (set under Settings → Secrets and variables → Actions), and all eight use `actions/checkout@v7`.

**The trigger difference is still the sharpest thing in this table.** Five files scope to a **directory** (`cart/**`, `user/**`, `web/**`); three still scope to a **file extension** (`**.go`, `**.py`, `**.java`). Part 3.3 has the diagnosis, and 3.3.1 has a live demonstration of it going wrong.

## 3.3 CI/CD Best Practices Checklist

Every workflow must be actively maintained to prevent silent failures. Follow these practices:

- ❌ **Don't use `docker login -p`:** It writes tokens unencrypted and exposes them to system process lists.
- ✅ **Do use `--password-stdin`:** Securely pipe credentials: `echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u ... --password-stdin`
- ✅ **Pin GitHub Actions:** Use specific SHA hashes instead of `@v7` tags to prevent supply-chain attacks.
- ✅ **Scope triggers to paths, not extensions:** Trigger builds with `paths: ['cart/**']` rather than `'**.js'`, which accidentally triggers backend builds when frontend files change.
- ✅ **Drop default permissions:** Always explicitly set `permissions: {contents: read}` to prevent excess token privileges.
- 🚨 **Never push images on `pull_request`:** Restrict `docker push` only to `push` events on the main branch, otherwise unreviewed PR code will overwrite your registry images!

## 3.6 Then make the CI genuinely good

- **Matrix build** — replace eight near-identical files with one `strategy.matrix.service: [cart, catalogue, ...]`.
- **Layer caching** — `docker/build-push-action` with `cache-from`/`cache-to: type=gha`. Biggest single speed win, most visible on `shipping`.
- **Multi-arch** — `docker/setup-buildx-action` with `platforms: linux/amd64,linux/arm64`.
- **Scan and gate** — fail the build on HIGH/CRITICAL:
  ```yaml
  - uses: aquasecurity/trivy-action@master
    with:
      image-ref: ${{ secrets.DOCKERHUB_USERNAME }}/rs-cart:${{ github.sha }}
      exit-code: '1'
      severity: 'HIGH,CRITICAL'
  ```
- **SBOM** — `docker buildx build --sbom=true`, published as an artifact.
- **Tests** — there are effectively none in this repo (`cart`, `catalogue` and `user` have placeholder `npm test` scripts that just `exit 1`; `shipping` declares test dependencies but has no `src/test` directory). Write a few real ones and gate the build on them.
- **Harden `.gitignore`** before any cloud work puts `iam_policy.json` or a kubeconfig next to your code.

---

# 📦 Part 4 — The registry: image identity and tags

CI produces images. Everything after this part consumes them. The joint between the two is the registry, and it's where a specific class of confusion lives — the kind that produces `manifest unknown` at 11pm, or worse, a deploy that silently runs last week's code.

## 4.1 What your CI actually produced

Eight repositories under the `naveenreddy9` namespace on Docker Hub, each tagged with the `github.run_number` of the run that built it:

| Repo | Tags | Newest |
|---|---|---|
| `naveenreddy9/catalogue` | 6, 7, 8, 9, 10, 11 | **11** |
| `naveenreddy9/cart` | 13, 14, 15 | **15** |
| `naveenreddy9/dispatch` | 7 | **7** |
| `naveenreddy9/shipping` | 5, 6 | **6** |
| `naveenreddy9/user` | 3, 4 | **4** |
| `naveenreddy9/rating` | 1 | **1** |
| `naveenreddy9/payment` | 1 | **1** |
| `naveenreddy9/web` | 1 | **1** |

Stare at that column for a moment, because it contains the whole lesson of this part: **`github.run_number` is a per-workflow counter.** Cart's workflow has run 15 times, payment's once. There is no number that identifies "the current version of the app" — every service is on its own count, and they will never converge.

That's not a bug in the workflows. It's what per-service pipelines *mean*: services release independently. But it breaks every tool that assumes a single version string, and you're about to meet two of them.

**And a run number identifies a *run*, not code.** `user:3` and `user:4` are byte-identical images. Tag 4 exists because a commit added one line to `user.yaml` — and `user.yaml` lists itself in its own `paths:` filter, so editing the workflow rebuilt the service. Nothing about the application changed.

That cuts both ways, and it's worth being clear about which you want:

- **`github.run_number`** answers *"which pipeline run produced this?"* — good for finding logs, useless for knowing whether two tags contain the same code.
- **`github.sha`** answers *"which commit is this?"* — two builds of the same commit get the same tag, and you can go straight from a running container to the exact source. It's the tag GitOps wants (Part 11).

Neither is wrong. But if you find yourself asking "is `user:4` actually different from `user:3`?", the tag scheme has already failed you.

## 4.2 `latest` is not "newest"

This is the single most common misconception about Docker tags, so state it plainly:

> **`latest` is an ordinary tag name with no special behaviour.** Docker never sorts tags, never compares dates, and never picks a highest version. Nothing about `latest` is automatic.

Its one and only property is that it's the name Docker fills in when you *omit* a tag. `docker pull cart` is shorthand for `docker pull cart:latest` — that's the whole feature.

So `naveenreddy9/cart:latest` exists only if something explicitly pushed a tag literally spelled `latest`. Your workflows push `:${{ github.run_number }}` and nothing else. Ask for `latest` and you get:

```
Error response from daemon: manifest unknown
```

Not "tag 15". Not the newest build. Nothing.

**When `latest` does track the newest build**, it's because every run overwrites that name with a new image — the name moves because you keep reassigning it, not because Docker resolved anything. Which gives you the real trade-off:

| | `:latest` | `:<run_number>` or `:<sha>` |
|---|---|---|
| Values to maintain | one, forever | one per service |
| "Which commit is running?" | unanswerable | exact |
| Rollback | impossible — the old image has no name | change one number |
| Re-pull behaviour | `docker compose up` reuses a cached `latest` and silently runs stale code | a new tag always pulls |
| Kubernetes | `imagePullPolicy: IfNotPresent` + `latest` = nodes disagree about what's running | deterministic |

The usual answer is **both**: push `:<run_number>` *and* `:latest` from the same workflow, then choose per situation — `latest` for a throwaway local run, the specific tag for anything you need to reason about later. Two extra lines per workflow: a second `-t` on the build, a second `docker push`.

## 4.3 The four categories of image in this stack

Twelve containers run. They come from four genuinely different places, and conflating them is how people end up with an empty product catalogue and no idea why.

**1. Yours — built by CI from source in this repo.** The eight above. You control the Dockerfile, the tag and the namespace.

**2. Upstream `robotshop/*` — images with data baked in.** `robotshop/rs-mongodb:2.1.0` and `robotshop/rs-mysql-db:2.1.0`. These have Dockerfiles here ([mongo/](mongo/), [mysql/](mysql/)) but **no workflow builds them**, so they come from upstream.

> **These are not interchangeable with `mongo:5` and `mysql:5.7`.** [mongo/Dockerfile](mongo/Dockerfile) copies in `catalogue.js` and `users.js`; [mysql/Dockerfile](mysql/Dockerfile) copies in `scripts/`. That's the seed data — the product catalogue, the demo users, the shipping tables. Swap in the plain official images and every container starts, nothing errors, and the shop is simply empty. A failure with no error message is the worst kind, and this one is two characters away at all times.

**3. Genuinely open source, used unmodified.** `redis:6.2-alpine` and `rabbitmq:3.8-management-alpine`. No Dockerfile in this repo, no customisation, pulled straight from Docker Hub.

**4. Base images — build time only.** The `FROM` lines: `node:20-alpine`, `golang:1.24`, `alpine:latest`, `python:3.9`, `php:7.4-apache`, `maven:3.6.3-jdk-8`, `eclipse-temurin:25-jre-alpine`, `nginx:1.21.6`, plus `composer` and `mongo:5`/`mysql:5.7`. These are consumed on the CI runner and never run as containers in the stack. [pullbaseimages.sh](pullbaseimages.sh) scrapes exactly this set.

Categories 2 and 3 are the ones people merge by mistake. Both are "images someone else built", but one of them carries your data.

## 4.4 Why one version value can't describe this stack

Two files in this repo assume a uniform release tag, the way upstream ships `2.1.0` across every service. Both break the moment you build your own images.

**[.env](.env) — used by [docker-compose.yaml](docker-compose.yaml):**
```
REPO=robotshop
TAG=2.1.0
```
Every service resolves as `${REPO}/rs-<svc>:${TAG}`. One namespace, one tag, one `rs-` prefix. Your images are `naveenreddy9/<svc>:<n>` — different namespace, no prefix, and a different `<n>` per service. No pair of values works.

**`EKS/helm/values.yaml` — as it was:**
```yaml
image:
  repo: robotshop
  version: latest
```
with every template doing `{{ .Values.image.repo }}/rs-<svc>:{{ .Values.image.version }}`.

Changing `repo` to `naveenreddy9` and renaming seven templates got *most* of the way there and produced a chart where **10 of 12 pods would `ImagePullBackOff`**, in three distinct ways:

- `version: latest` — doesn't exist for any of your images (4.2).
- A single `version` can't be 15 for cart and 1 for payment at the same time (4.1).
- `repo` is **global**, so pointing it at your namespace also redirected `mongodb`, `mysql` and `web` — which then resolved to `naveenreddy9/rs-mongodb`, `naveenreddy9/rs-mysql-db` and `naveenreddy9/rs-web`. None exist: the first two only live under `robotshop` (4.3), and yours is called `web`, not `rs-web`.

**The fix is to stop assembling image names from shared parts.** Each workload carries its own complete reference:

```yaml
# EKS/helm/values.yaml
image:
  pullPolicy: IfNotPresent      # still global — this one genuinely is

cart:
  image: naveenreddy9/cart:15
catalogue:
  image: naveenreddy9/catalogue:11
ratings:
  image: naveenreddy9/rating:1  # note: repo is 'rating', service is 'ratings'
mongodb:
  image: robotshop/rs-mongodb:2.1.0   # stays upstream — seed data
```
```yaml
# EKS/helm/templates/cart-deployment.yaml
image: {{ .Values.cart.image }}
```

More verbose, and correct. It also removes the `rs-` prefix problem at the root: nothing is concatenated, so renaming one service can't break another. Override a single service without editing the file:

```bash
helm upgrade robot-shop EKS/helm --set cart.image=naveenreddy9/cart:16
```

**The general principle:** a shared value is only worth sharing if it's genuinely shared. `pullPolicy` is. `repo` and `version` looked like they were, right up until two services needed different ones.

## 4.5 Reading your own registry

You cannot deploy what you can't enumerate. Docker Hub's public API needs no auth for public repos:

```bash
# every repo in your namespace
curl -s "https://hub.docker.com/v2/repositories/naveenreddy9/?page_size=100" \
  | grep -o '"name":"[^"]*"'

# every tag for one repo, newest first
curl -s "https://hub.docker.com/v2/repositories/naveenreddy9/cart/tags?page_size=100" \
  | grep -o '"name":"[^"]*"'
```

Locally:
```powershell
docker images                                   # what's cached here
docker manifest inspect naveenreddy9/cart:15    # does this tag exist remotely, without pulling it
docker history naveenreddy9/cart:15             # where the megabytes went, layer by layer
```

And before any Helm deploy, render the chart and read the image lines — this catches every mistake in 4.4 in about two seconds, for free, without a cluster:

```bash
helm template rs EKS/helm | grep "image:" | sort -u
```

## 4.6 Exercises

1. **Add `:latest` alongside the run number** in one workflow. Two lines. Then deploy from `latest`, push a change, deploy again *without* changing any tag — and work out whether you got the new code. Then explain why `docker compose pull` was necessary.
2. **Break it deliberately.** Set `CART_TAG=999` in `.env.local` and read the error. Then set it to `latest` and read that error. They're different failures; know both on sight.
3. **Prove category 2 matters.** Point `mongodb` at plain `mongo:5` in `docker-compose.local.yaml` and load the site. Note that nothing errors — catalogue returns `[]` and the page is simply blank.
4. **Switch to `github.sha`.** Retag one service by commit SHA instead of run number. What do you gain (a tag that identifies code, not a counter), and what do you lose (human-readable ordering — is `a3f9c2` newer than `7b1e44`)?
5. **Find the drift.** `rating` vs `ratings` appears in the workflow, the compose file, the Helm values and the nginx routing table. Trace every place the name is written and decide where you'd fix it.

---

# 🚀 Part 5 — Deploy with plain Docker

Do this once, properly. Everything after it is a reaction to the problems you're about to feel.

## 5.1 Run the whole stack by hand

```powershell
docker network create robot-shop-manual

docker run -d --name mongodb   --network robot-shop-manual robotshop/rs-mongodb:2.1.0
docker run -d --name redis     --network robot-shop-manual redis:6.2-alpine
docker run -d --name rabbitmq  --network robot-shop-manual rabbitmq:3.8-management-alpine
docker run -d --name mysql     --network robot-shop-manual robotshop/rs-mysql-db:2.1.0
docker run -d --name catalogue --network robot-shop-manual robotshop/rs-catalogue:2.1.0
docker run -d --name user      --network robot-shop-manual robotshop/rs-user:2.1.0
docker run -d --name cart      --network robot-shop-manual robotshop/rs-cart:2.1.0
docker run -d --name shipping  --network robot-shop-manual robotshop/rs-shipping:2.1.0
docker run -d --name ratings   --network robot-shop-manual robotshop/rs-ratings:2.1.0
docker run -d --name payment   --network robot-shop-manual robotshop/rs-payment:2.1.0
docker run -d --name dispatch  --network robot-shop-manual robotshop/rs-dispatch:2.1.0
docker run -d --name web -p 8080:8080 --network robot-shop-manual robotshop/rs-web:2.1.0
```

Open http://localhost:8080.

**The container names are load-bearing.** On a user-defined bridge network, Docker's embedded DNS resolves a container's name as its hostname — and `web/Dockerfile` defaults `CATALOGUE_HOST=catalogue`, `CART_HOST=cart`, and so on. Name a container `catalogue-1` and nginx can't find it. That is all "service discovery" means at this level.

Things to notice while it's running:
```powershell
docker logs -f cart              # start it before redis and watch it crash-loop, then recover
docker exec -it cart sh          # then: wget -qO- http://catalogue:8080/products
docker stats                     # no limits set, so every container can eat the whole host
```

## 5.2 The same thing on a normal Linux server

This is also the answer to "can I run this on a plain server?" — yes, and the repo has **no script for it**; every `.sh` in the repo targets a cluster (Swarm/DC-OS/OpenShift) or runs inside a container. On a fresh Ubuntu/Oracle Linux host (an EC2 instance, or your WSL Oracle Linux):

```bash
sudo dnf install -y docker            # Oracle Linux / RHEL family;  apt-get install docker.io on Ubuntu
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER" && newgrp docker

git clone <your-fork-url>
cd three-tier-architecture-robot-shop
```
Then run the twelve commands above. On EC2, open port 8080 in the security group.

Add `--restart unless-stopped` to every container, or nothing comes back after a reboot. Then reboot the box and see what happens — that lesson is worth the five minutes.

## 5.3 What plain Docker doesn't solve

Now the important part. Write your own list first, then compare:

| Problem | What you just felt |
|---|---|
| **No single source of truth** | The deployment exists only in your shell history. Nobody else can reproduce it, and you can't diff it. |
| **Order and readiness** | You started things in the right order by hand. Get it wrong and services crash-loop until their dependency appears. |
| **No config management** | Every env var, port and network flag is typed by hand, per container, per environment. |
| **Teardown is manual** | Twelve `docker rm -f` commands, and forgetting one leaves a stale name that blocks the next run. |
| **No scaling** | Want three `catalogue` containers? Three more commands, three more names — and nothing load-balances between them. |
| **No restart policy by default** | Reboot the host and the app is gone. |
| **No resource limits** | `docker stats` shows nothing is constrained; one leaking service can take the host down. |
| **No health awareness** | Docker will happily report a container "up" while the app inside it is broken. |

Every one of these is what the next tool fixes.

---

# 🐳 Part 6 — Docker Compose

## 6.1 The same stack, declared once

```powershell
cd D:\robot-shop-project\three-tier-architecture-robot-shop
docker compose pull
docker compose up -d
docker compose ps
```

[docker-compose.yaml](docker-compose.yaml) is the file that replaces Part 5's twelve commands. Read it next to what you typed and match them up: `image:` ← the image argument, `environment:` ← every `-e`, `ports:` ← `-p`, `networks:` ← `--network`, `depends_on:` ← the order you started things in, `healthcheck:` ← the thing Docker had no idea about before.

Compose fixes, directly:

- **Source of truth** — the deployment is a file in git, reviewable and diffable.
- **Ordering** — `depends_on` encodes what you did by hand.
- **One-command lifecycle** — `up`, `down`, `logs`, `ps` across the whole stack.
- **Health** — seven services now declare `healthcheck`, so "up" starts to mean something.
- **Scaling** — `--scale` instead of inventing names.
- **Config layering** — override files instead of retyping flags.

## 6.2 Running *your* images instead of upstream's

[docker-compose.yaml](docker-compose.yaml) builds from source and names everything `${REPO}/rs-<svc>:${TAG}`. That's fine for upstream, whose services all ship under one release tag. It cannot express what your CI produces — Part 4.4 has the reasoning. [docker-compose.local.yaml](docker-compose.local.yaml) is the companion file that can:

```powershell
docker compose --env-file .env.local -f docker-compose.local.yaml up -d
docker compose --env-file .env.local -f docker-compose.local.yaml ps
```

Three things make it different from the stock file, and each one is a deliberate choice worth understanding:

**It pulls, it never builds.** No `build:` key anywhere. The images were built once on a CI runner; rebuilding them locally would produce something *similar* but not identical, which defeats the point of having a registry. What you run locally is bit-for-bit what CI published.

**It carries one tag variable per service.** [.env.local](.env.local):
```
DH_USER=naveenreddy9
CATALOGUE_TAG=11
CART_TAG=15
USER_TAG=3
SHIPPING_TAG=6
DISPATCH_TAG=7
RATING_TAG=1
PAYMENT_TAG=1
WEB_TAG=1
```
Eight variables where the stock file has one, because `github.run_number` counts per workflow (Part 4.1). This is the file that goes stale — every CI run makes one of these numbers wrong, and nothing tells you.

**It mixes all four image categories in one file**, grouped and commented so the boundaries stay visible: your eight from Docker Hub, `robotshop/rs-mongodb` and `rs-mysql-db` from upstream because of the seed data, `redis` and `rabbitmq` straight from their official images.

**Service names are load-bearing, exactly as in Part 5.1.** [web/Dockerfile](web/Dockerfile) bakes in `CATALOGUE_HOST=catalogue`, `CART_HOST=cart` and so on as env defaults, and `entrypoint.sh` runs `envsubst` over them to generate the nginx config at container start. Rename a service in the compose file and nginx returns 502 for that route. Compose gives you DNS by service name for free — but the names still have to match what the image expects.

**What it deliberately leaves out:** healthchecks. The stock file declares seven; this one declares none. That's why nobody noticed alpine had removed `curl` from the Node images (Part 2.4) — the healthchecks that would have failed weren't running. Adding them back, with `wget -q --spider` instead of `curl`, is a good exercise.

**Teardown:**
```powershell
# stop, keep images cached for next time
docker compose --env-file .env.local -f docker-compose.local.yaml down

# stop and reclaim everything this project pulled (~1.2GB)
docker compose --env-file .env.local -f docker-compose.local.yaml down --rmi all -v --remove-orphans
```
`-v` drops the volumes, so MongoDB and MySQL reseed from scratch on the next `up` — which is how you get back to a clean catalogue after experimenting. `--rmi all` is scoped to this project's images; `docker system prune -a` is not, and will happily delete images belonging to your other work.

## 6.3 Day-2 practice

**Logs and inspection**
```powershell
docker compose logs -f payment
docker compose exec cart sh
docker compose config          # render the final merged file without running it
```

**Scaling, and its one limit**
```powershell
docker compose up -d --scale catalogue=3   # works
docker compose up -d --scale web=2         # FAILS
```
`web` is the only service binding a fixed host port (`8080:8080`), and two containers can't share it. That single constraint is the reason Kubernetes has Services and Ingress instead of port mappings.

**Failure injection** — with the load generator running (`docker compose -f docker-compose.yaml -f docker-compose-load.yaml up -d`), kill dependencies one at a time and match reality against the table in Part 1.2:
```powershell
docker compose stop redis      # cart dies instantly — hard failure
docker compose stop rabbitmq   # payments still succeed — soft failure, queue grows
docker compose pause mysql     # shipping hangs instead of failing — the worst kind
docker compose start redis
```

**Fix the persistence gap** — there are no volumes in this compose file, so prove it and repair it:
```powershell
docker compose exec mongodb mongo catalogue --eval "db.products.count()"
docker compose down
docker compose up -d           # data came back from the image, not from storage
```
Create `docker-compose.override.yaml`:
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
Compose merges `docker-compose.override.yaml` automatically. Then practise a backup:
```powershell
docker compose exec mongodb mongodump --archive=/tmp/dump.gz --gzip --db catalogue
docker compose cp mongodb:/tmp/dump.gz ./backup-catalogue.gz
```

**Multi-environment** — build `docker-compose.dev.yaml` and `docker-compose.prod.yaml` differing in replicas, log level and resource limits:
```powershell
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml config
```
This override/merge idea reappears as Helm values files and Argo CD parameters later.

## 6.4 What Compose doesn't solve

| Problem | Why it matters |
|---|---|
| **One host, full stop** | Compose schedules nothing across machines. If that server dies, the app dies. |
| **No self-healing** | A crashed container restarts, but a dead *host* is your problem at 3am. |
| **No rolling updates** | `docker compose up -d` recreates containers. There's no gradual replacement and no rollback. |
| **No reconciliation loop** | Compose applies your file once. Nothing watches for drift or puts things back afterwards. |
| **`web` still can't scale** | The fixed host port caps your entry point at one replica per host. |
| **No real scheduling** | Nothing places workloads by available CPU/memory, or evicts them under pressure. |
| **Secrets are just env vars** | Plain text in the file, visible in `docker inspect`. |
| **No horizontal autoscaling** | Load doubles at 9am; nothing responds. |

Compose is genuinely the right answer for a single-host deployment, a dev environment, or CI. It runs out of road the moment you need more than one machine or zero-downtime deploys.

---

# ☸️ Part 7 — Kubernetes

## 7.1 A local cluster

```powershell
minikube start --cpus=4 --memory=6g --driver=docker
minikube addons enable metrics-server      # needed for autoscaling in 6.4
kubectl get nodes
```
`kind create cluster` is a lighter alternative; Docker Desktop's built-in Kubernetes is the zero-install one. Any of them works — 10+ pods just need the CPU/memory to be allocated up front, or pods sit `Pending` with `Insufficient cpu`.

## 7.2 Deploy it

```powershell
helm install robot-shop --namespace robot-shop --create-namespace K8s/helm --set nodeport=true
kubectl -n robot-shop get pods -w
minikube ip
kubectl -n robot-shop get svc web       # note the NodePort, browse http://<minikube-ip>:<port>
```

Two traps: **[K8s/README.md](K8s/README.md) gives Helm 2 syntax** (`helm install --name robot-shop ...`) which fails on Helm 3 — [K8s/helm/README.md](K8s/helm/README.md) has the correct form. And Helm 3 won't create the namespace without `--create-namespace`.

Then demystify Helm — it is a template engine that emits YAML, nothing more:
```powershell
helm template robot-shop K8s/helm --set nodeport=true > rendered.yaml
```
Read `rendered.yaml`. The chart's 28 templates become 12 Deployments, a Redis StatefulSet, 13 Services, and some RBAC. Rendering first and applying second is a debugging technique that works on any chart.

The real value keys in this chart: `image.repo`, `image.version`, `image.pullPolicy`, `nodeport`, `psp.enabled`, `payment.gateway`, `redis.storageClassName`, `eum.key`, `eum.url`, `ocCreateRoute`, plus a per-service slot for `affinity`/`nodeSelector`/`tolerations`. (`openshift` exists in `values.yaml` but no template references it — a dead key.)

## 7.3 Compose concepts → Kubernetes objects

| Compose | Kubernetes | What changed |
|---|---|---|
| `image:` | Pod template `containers[].image` | same idea |
| `environment:` | `env:` / ConfigMap / Secret | config becomes a first-class object |
| service name on a network | **Service** + cluster DNS | now `catalogue.robot-shop.svc.cluster.local` |
| `ports: 8080:8080` | Service type NodePort/LoadBalancer, or Ingress | the host-port limit disappears |
| `--scale catalogue=3` | Deployment `replicas: 3` | declared, not commanded |
| `depends_on` | *nothing* | pods crash-loop and retry until dependencies appear |
| `healthcheck` | `livenessProbe` / `readinessProbe` | two separate questions: restart me vs. send me traffic |
| `restart: always` | the default | the reconciliation loop *is* the restart policy |
| `volumes:` | PVC + StorageClass | storage becomes a requested resource |
| `docker compose up` | `kubectl apply` / `helm install` | you declare desired state; a controller converges on it |

The row worth pausing on is `depends_on` → nothing. Kubernetes deliberately has no ordering primitive: everything retries until it works. Once that clicks, most of its behaviour makes sense.

## 7.4 Day-2 practice

**Rollout and rollback**
```powershell
helm upgrade robot-shop K8s/helm -n robot-shop --set image.version=2.1.0
kubectl -n robot-shop rollout status deployment/cart
kubectl -n robot-shop rollout history deployment/cart
kubectl -n robot-shop rollout undo deployment/cart
helm history robot-shop -n robot-shop
helm rollback robot-shop 1 -n robot-shop
```
Watch it pod-by-pod with load running and see whether any request fails — that's what readiness probes are for.

**Debugging drills** — break it deliberately, diagnose with kubectl only:
```powershell
helm upgrade robot-shop K8s/helm -n robot-shop --set image.version=does-not-exist
kubectl -n robot-shop get pods                 # ImagePullBackOff
kubectl -n robot-shop describe pod <pod>       # read Events — the answer is almost always here
kubectl -n robot-shop logs <pod> --previous    # logs from the crashed instance
helm rollback robot-shop -n robot-shop
```
The ladder is always: `get` → `describe` (Events) → `logs` → `exec`. Also practise `kubectl port-forward svc/catalogue 8081:8080` and calling `catalogue:8080` from inside another pod.

**Resource quotas**
```powershell
kubectl -n robot-shop apply -f K8s/resource-quota.yaml
kubectl -n robot-shop describe resourcequota robot-shop-quota
kubectl -n robot-shop scale deployment catalogue --replicas=15    # hit the ceiling
```
The file sets `limits.cpu: 4`, `requests.cpu: 2`, `limits.memory: 5Gi`, `requests.memory: 3Gi`, `pods: 20`, with no hardcoded namespace, so `-n` decides where it lands. Side effect worth knowing: once a quota with requests/limits exists, **any pod without them is rejected** — so a quick `kubectl run` debug pod fails. That's a real support ticket in disguise.

**Autoscaling**
```bash
./K8s/autoscale.sh                       # WSL — POSIX script, hardcodes NS="robot-shop"
```
It runs `kubectl autoscale ... --max 2 --min 1 --cpu-percent 50` across the eight app deployments. **Without metrics-server the HPAs report `<unknown>/50%` and never scale.** Then drive load and watch:
```powershell
kubectl -n robot-shop apply -f K8s/load-deployment.yaml
kubectl -n robot-shop get hpa -w
kubectl -n robot-shop top pods
```

**Storage** — Redis is the chart's only stateful workload (`redis-statefulset.yaml`, with `storageClassName` from values, default `standard`):
```powershell
kubectl -n robot-shop get pvc,pv
kubectl -n robot-shop delete pod redis-0     # PVC survives the pod
```
MongoDB and MySQL run as plain Deployments with no storage at all — the same defect as Compose. Giving them PVCs is the natural follow-on exercise.

**Ingress** — the chart has **no Ingress template**; `web` is exposed as a Service only. Add one:
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
Point `robotshop.local` at `minikube ip` in your hosts file. An Ingress object does nothing without a controller watching it — remember that when you hit AKS/GKE later.

## 7.5 What Kubernetes costs you

Being honest about this is part of the skill:

- **Enormous surface area.** You now maintain a cluster, a chart, RBAC, storage classes, an ingress controller, and a metrics pipeline — to run the same twelve containers.
- **YAML everywhere**, and the rendered output is far removed from what you wrote.
- **Debugging gets indirect** — the crash you're chasing may be a probe, a quota, a scheduling constraint, or the app.
- **It's still not a deployment process.** Nothing here decides *when* to deploy, or from what. That's Parts 3 and 11.

Kubernetes buys you: multi-host scheduling, self-healing, rolling updates with rollback, declarative reconciliation, autoscaling, and a portable API across every cloud — which is exactly what Part 8 tests.

---

# ☁️ Part 8 — Different environments

The point of this part is to find out how much of Part 7 actually transfers. Answer: the manifests transfer almost entirely; everything *around* them — identity, storage, ingress — does not.

## 8.1 AWS EKS

The repo ships five numbered docs that build a cluster. They're thin and have gaps, so here's the corrected path with the gaps filled.

**Step 1 — tools** ([EKS/01-prerequisites.md](EKS/01-prerequisites.md) names them but gives no commands). In WSL Oracle Linux: install `awscli`, `eksctl`, `kubectl` and `helm`, then `aws configure`. The `export`/`$( )` syntax in the repo's docs is bash — run all of this from WSL, not PowerShell.

**Step 2 — the cluster** ([EKS/02-eks-cluster-setup.md](EKS/02-eks-cluster-setup.md)):
```bash
eksctl create cluster --name demo-cluster-three-tier-1 --region us-east-1
aws eks update-kubeconfig --name demo-cluster-three-tier-1 --region us-east-1
kubectl get nodes
```
Two warnings the doc doesn't give you: its heading says "Install using Fargate" but there's no `--fargate` flag, so you get a normal managed node group. And this takes ~20 minutes and **starts billing immediately** — the control plane charges by the hour whether or not anything runs on it.

**Step 3 — OIDC / IRSA** ([EKS/03-oidc-IAM.md](EKS/03-oidc-IAM.md)):
```bash
eksctl utils associate-iam-oidc-provider --cluster demo-cluster-three-tier-1 --approve
```
This is the step worth understanding rather than copying. It lets a *Kubernetes service account* assume an *AWS IAM role* — so pods get AWS permissions with no static access keys anywhere. It's the modern cloud-identity pattern and a very common interview question.

**Step 4 — the ALB controller** ([EKS/04-alb-configuration.md](EKS/04-alb-configuration.md)). The doc uses `<your-aws-account-id>` and `<your-vpc-id>` without saying how to find them, and has a stray trailing `\` that breaks copy-paste. Here's how to get them:
```bash
aws sts get-caller-identity --query Account --output text
aws eks describe-cluster --name demo-cluster-three-tier-1 \
  --query cluster.resourcesVpcConfig.vpcId --output text
```

**Step 5 — EBS CSI driver** ([EKS/05-ebs-csi-driver.md](EKS/05-ebs-csi-driver.md)) — gives Redis's PVC something to bind to. (The doc has a copy-paste bug telling you to replace `<AWS-ACCOUNT-ID>` "with the name of your cluster" — ignore that.)

**Step 6 — deploy the app. This doc doesn't exist in the repo.** The five docs build a cluster and stop:
```bash
helm install robot-shop --namespace robot-shop --create-namespace EKS/helm --set nodeport=true
kubectl -n robot-shop apply -f EKS/helm/ingress.yaml
kubectl -n robot-shop get ingress robot-shop -w    # wait for the ALB hostname
```

**Two traps that will cost you an afternoon otherwise:**

1. **`EKS/helm/ingress.yaml` is in the chart root, not `templates/`.** Helm only renders files under `templates/`, so `helm install` ignores it completely. You must `kubectl apply` it yourself. Nothing in the repo says so. (Same for `AKS/helm/ingress.yaml` and `GKE/helm/gclb.yaml`.)
2. **Pass `--set nodeport=true`.** The chart's `web` Service defaults to `type: LoadBalancer`, so without this you provision a *second* cloud load balancer competing with your ALB — and pay for both.

**`EKS/helm` no longer matches the other three charts.** It has been refactored to carry a complete image reference per workload — `{{ .Values.cart.image }}` rather than `{{ .Values.image.repo }}/rs-cart:{{ .Values.image.version }}` — because the global `repo`/`version` pair cannot describe images built by eight independent pipelines. Part 4.4 walks through why, and through the three ways a half-finished version of that change fails.

Always render before you deploy. This costs two seconds and catches every image mistake without a cluster:
```bash
helm template rs EKS/helm | grep "image:" | sort -u
```
Expect `naveenreddy9/*` for the eight services you build, `robotshop/rs-mongodb` and `rs-mysql-db` for the two seeded databases, and `redis`/`rabbitmq` pinned directly in their templates.

Also worth knowing: `K8s`, `AKS` and `GKE` are still on the old global-`repo` scheme and still point at `robotshop/rs-*`. They're internally consistent, so they work — they just deploy upstream's images, not yours. Between those three, exactly one line differs: `redis.storageClassName` (`standard`/`default`/`default`) — and the two cloud charts **hardcode** it in `templates/redis-statefulset.yaml`, so `--set redis.storageClassName=...` is silently ignored there. Four near-identical charts maintained by hand is itself a defect worth fixing, and `EKS/helm` having now diverged from the other three makes that worse, not better. A single chart with a values file per environment is the real answer.

**Tear down the same day:**
```bash
kubectl -n robot-shop delete ingress robot-shop     # delete the ALB first
helm uninstall robot-shop -n robot-shop
eksctl delete cluster --name demo-cluster-three-tier-1 --region us-east-1
```
Then check the console for orphaned load balancers and EBS volumes. Set an AWS Budgets alert *before* you start — the classic surprise bill is a NAT gateway or an ALB left running for a month.

**What EKS teaches that minikube can't:** IRSA/OIDC identity, a real ingress controller provisioning real cloud infrastructure, CSI storage classes, and the cost discipline that comes with all of it.

## 8.2 OpenShift (planned — decide later)

You haven't run this yet, so here are the two entry points and what's actually different. Nothing else in this document depends on it.

**Option A — Red Hat Developer Sandbox.** Free, hosted, nothing to install, renewable. You get a restricted namespace rather than cluster-admin, which is enough to meet SCCs, Routes and Projects.

**Option B — OpenShift Local (CRC).** A full single-node cluster on your laptop with cluster-admin, but it wants ~9–16GB RAM free, which is heavy next to Docker Desktop.

```bash
crc setup && crc start
oc login -u developer https://api.crc.testing:6443
oc new-project robot-shop
helm install robot-shop --set openshift=true --set nodeport=true K8s/helm
```

**What's genuinely different from vanilla Kubernetes:**

- **Security Context Constraints.** OpenShift refuses to run containers as root by default and assigns a random UID. This app's images expect otherwise — `payment/Dockerfile` literally sets `USER root`, and `ratings` does `chmod -R 777`. That's why [OpenShift/setup.sh](OpenShift/setup.sh) grants `anyuid` and `privileged`. Meeting this failure is the most valuable thing OpenShift teaches, and it ties straight back to the Dockerfile non-root practice in Part 2.3.
- **Routes instead of Ingress**, and **Projects** as namespaces with extra policy.
- Repo notes: [OpenShift/README.md](OpenShift/README.md) covers OCP 3.x and 4.x with correct Helm 3 syntax; `setup.sh` assumes a dev cluster where `system:admin` logs in without a password, and its two `add-scc-to-user` calls omit `-n robot-shop`, so they hit whatever project is current. The chart's `ocCreateRoute` value is the one that creates a Route (`openshift` is a dead key).

---

# 📊 Part 9 — Observability

Three questions, three tools: **what is happening** (metrics), **what happened** (logs), and **who told me** (alerts).

## 9.1 What this app already exposes

These are the real metric names — `cart` and `payment` are the only instrumented services:

| Service | Metric | Type | Meaning |
|---|---|---|---|
| cart | `items_added` | Counter | items added to carts. It's on a custom registry, so there are **no** default Node.js metrics |
| payment | `sold_count` | Counter | items sold |
| payment | `units_sold` | Histogram | units per sale (buckets 1, 2, 5, 10, 100) |
| payment | `cart_value` | Histogram | value per sale (buckets 100 … 10000) |

```powershell
curl http://localhost:8080/api/cart/metrics
curl http://localhost:8080/api/payment/metrics
```
Note these are *business* metrics, not CPU graphs — "are we selling anything" is usually a better alert than "is CPU high".

## 9.2 Prometheus + Grafana on the Compose stack

`observability/prometheus.yml`:
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
`observability/docker-compose.yaml`:
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
    name: three-tier-architecture-robot-shop_robot-shop   # confirm with: docker network ls
```
Joining the app's own network is more reliable than `host.docker.internal` on Docker Desktop, and lets Prometheus scrape services by name directly rather than through nginx.

```powershell
docker compose -f observability/docker-compose.yaml up -d
# Prometheus http://localhost:9090    Grafana http://localhost:3000  (admin/admin)
```
Add Prometheus as a Grafana data source, then build panels on:
```promql
rate(items_added[5m])
rate(sold_count[5m])
histogram_quantile(0.95, rate(cart_value_bucket[5m]))
```
The concepts that matter here: Prometheus **pulls** (targets don't push); counters only go up, so you always wrap them in `rate()`; and a histogram is really a set of `_bucket` series, which is why `histogram_quantile` needs them.

## 9.3 Prometheus on Kubernetes

```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80
```
Then swap static targets for label-based discovery with a ServiceMonitor:
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
Check the real labels and port names first with `kubectl -n robot-shop get svc payment -o yaml`. This is the Operator pattern: a CRD turns "edit a config file" into "create an object", and discovery becomes label-driven.

## 9.4 Logs with ELK

Good news — the repo's [fluentd/](fluentd/) already uses the **Elasticsearch output plugin**, so ELK is the natural fit. It just points at a Humio endpoint with placeholder credentials, which means it ships nothing until you repoint it.

Two configs ship in the repo, and they collect logs in **two different ways** — worth understanding before editing either:
- `fluentd/Docker-Compose/fluent.conf` — `@type forward` on port 24224. It doesn't read files; containers *push* to it via Docker's fluentd logging driver.
- `fluentd/Kubernetes/fluentd.yaml` — a DaemonSet that **tails** `/var/log/containers/*.log`, enriches with `kubernetes_metadata`, and drops `kube-system`.

**Local ELK for the Compose stack** — `observability/elk-compose.yaml`:
```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports: ["9200:9200"]
    networks: [robot-shop]
  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports: ["5601:5601"]
    networks: [robot-shop]
  fluentd:
    image: robotshop/fluentd:elastic          # built by fluentd/build.sh
    volumes:
      - ./fluent.conf:/fluentd/etc/fluent.conf
    ports: ["24224:24224"]
    networks: [robot-shop]
networks:
  robot-shop:
    external: true
    name: three-tier-architecture-robot-shop_robot-shop
```
Copy `fluentd/Docker-Compose/fluent.conf` to `observability/fluent.conf` and repoint its `<match>` block at your local Elasticsearch:
```
<match **>
  @type elasticsearch
  host elasticsearch
  port 9200
  scheme http
  logstash_format true
  logstash_prefix robotshop
</match>
```
Then make the app actually ship its logs — add to `docker-compose.override.yaml`:
```yaml
services:
  cart:
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: robotshop.cart
```
Repeat per service. Open Kibana at http://localhost:5601, create a data view on `robotshop-*`, and search.

**One caution worth experiencing deliberately:** with the fluentd logging driver, if the log collector is unreachable, containers can **fail to start**. Stop fluentd and try `docker compose up` — it's a memorable lesson in not putting a hard dependency in the logging path (`fluentd-async: true` is the mitigation).

On Kubernetes, deploy `fluentd/Kubernetes/fluentd.yaml` with the output repointed at your Elasticsearch, and compare: the DaemonSet tails node log files, so applications need no configuration at all. That contrast — push vs. tail — is the main architectural lesson in logging.

## 9.5 Alerting, and watching a real failure

Prometheus rules:
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
Now put it all together. With load running and both dashboards open:
```powershell
docker compose stop rabbitmq     # payments still "succeed" — sold_count keeps climbing, dispatch goes silent
docker compose stop redis        # cart fails instantly and loudly
docker compose pause mysql       # shipping hangs — slowest to detect, worst to diagnose
```
For each, write a short timeline: what the dashboard showed, which log line in Kibana was the first real signal, how long detection took, and what alert *should* have caught it. That exercise — correlating a metric, a log and a symptom into one story — is the actual skill.

---

# 🛠️ Part 10 — Provisioning: Terraform and Ansible

Both are "infrastructure as code", and they own different halves:

| | Terraform | Ansible |
|---|---|---|
| Question | *Does this infrastructure exist?* | *Is this machine configured correctly?* |
| Works on | cloud APIs (VPCs, clusters, IAM) | hosts over SSH |
| Model | declarative, with state | idempotent tasks, no state file |
| Here | creates the EKS cluster | configures EC2 hosts to run the app |

## 10.1 Terraform — build the EKS cluster as code

Everything you did by hand in Part 8.1 becomes a file. Create `terraform/main.tf`:

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

  # the ALB controller discovers subnets by these tags
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

  enable_irsa = true          # the OIDC step from Part 8.1, declared

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
Each argument goes on its own line — comma-separated assignments on one line are invalid HCL.

```bash
cd terraform
terraform init
terraform plan -out=tfplan     # read this carefully — it is the whole point
terraform apply tfplan
aws eks update-kubeconfig --name <cluster_name> --region <region>
terraform destroy
```

**What to take away:** `plan` shows you the diff before anything happens — no other tool in this document gives you that. **State** is the file that maps your code to real resources; it must be remote (S3 + DynamoDB locking) the moment more than one person is involved, and drift between state and reality is the classic Terraform incident. And compare honestly against Part 8.1: five documents of manual commands versus one `apply` that is reproducible and destroyable.

## 10.2 Ansible — configure the machines

Your WSL Oracle Linux is the control node; targets are EC2 instances (or local VMs).

```bash
sudo dnf install -y ansible-core
ansible-galaxy collection install community.docker
```

`ansible/inventory.ini`:
```ini
[docker_hosts]
node1 ansible_host=<ec2-public-ip> ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/your-key.pem
```

`ansible/bootstrap-docker-host.yml` — turn a bare instance into a host that can run this app:
```yaml
- hosts: docker_hosts
  become: true
  tasks:
    - name: Install Docker
      package:
        name: docker
        state: present

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

    - name: Pre-pull the base images this repo uses
      community.docker.docker_image:
        name: "{{ item }}"
        source: pull
      loop:
        - nginx:1.21.6
        - node:20-alpine
        - alpine:latest
        - maven:3.6.3-jdk-8
        - eclipse-temurin:25-jre-alpine
        - php:7.4-apache
        - python:3.9
        - golang:1.24
```
That image list is the real set of `FROM` lines in this repo — the same job [pullbaseimages.sh](pullbaseimages.sh) does imperatively, done declaratively.

```bash
ansible-playbook -i ansible/inventory.ini ansible/bootstrap-docker-host.yml
ansible-playbook -i ansible/inventory.ini ansible/bootstrap-docker-host.yml   # expect changed=0
```
**Run it twice.** The second run reporting `changed=0` is idempotency demonstrated rather than claimed — the single most important idea in configuration management.

Then go further: a playbook that copies `docker-compose.yaml` to the host and brings the stack up, which gives you a complete "Terraform creates the server, Ansible configures it, Compose runs the app" pipeline on real infrastructure.

---

# 🔄 Part 11 — GitOps with Argo CD

Everything so far deploys by *pushing*: you (or CI) run `helm upgrade` against a cluster, which means your pipeline holds cluster credentials and nothing notices if someone changes the cluster by hand afterwards.

GitOps inverts it. A controller **inside** the cluster watches git and continuously reconciles reality to match it.

```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd port-forward svc/argocd-server 8081:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"   # base64-decode this
```

Register this repo's chart:
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
    automated:
      prune: true
      selfHeal: true
```

Two experiments that make the idea land:

1. **Drift correction** — `kubectl -n robot-shop delete deployment cart`, then watch Argo CD put it back within seconds. Nobody ran a pipeline.
2. **Git as the deploy trigger** — change `image.version` in `K8s/helm/values.yaml`, commit, push. The cluster converges on its own.

**How this closes the loop with Part 3:** CI's job becomes *build the image and push it with an immutable tag* (`github.sha`). The deploy step becomes *update the tag in git*. CI never touches the cluster, cluster credentials leave your pipeline entirely, and git becomes the audit log of every production change — including rollbacks, which are now just `git revert`.

---

# 🗺️ Part 12 — Suggested order

| Stage | Parts | Notes |
|---|---|---|
| 1 | **1 — App design** | One sitting. Read the compose file and the nginx template side by side. |
| 2 | **2 — Dockerfiles** | ✅ dispatch multi-stage and the alpine move are done. The open items in 2.4 are the remaining work. |
| 3 | **3 — CI** | ✅ All eight pipelines build and push. 3.4 is the list of what's still wrong with them — start there. |
| 4 | **4 — Registry** | Short, and it prevents a whole category of confusion in every part after it. Don't skip it. |
| 5 | **5 — Plain Docker** | Don't skip this either. Everything after is a reaction to the pain here. |
| 6 | **6 — Compose** | Add the missing volumes; add healthchecks to the local file; run the failure-injection drills. |
| 7 | **7 — Kubernetes** | The biggest block. `helm template` early — it demystifies everything after. |
| 8 | **9 — Observability** | Deliberately before the cloud: it's free locally, and it makes Part 8 far more interesting. |
| 9 | **8.1 — EKS** | Costs money. Set a budget alert first, tear down the same day. |
| 10 | **10 — Terraform + Ansible** | Terraform rebuilds Part 8.1 as code; Ansible configures hosts for Part 5. |
| 11 | **11 — Argo CD** | Needs Parts 3 and 7 working. Closes the loop. |
| — | **8.2 — OpenShift** | Optional, whenever you want it. Nothing else depends on it. |

Two habits worth keeping the whole way through:

1. **Keep a defects log.** Every broken thing you find in this repo (Part 1.4 is a starting list) is interview material — you found it, diagnosed it, and can explain the fix.
2. **Tear cloud resources down the same day.** It's the one mistake here that costs money rather than time.

Three habits this repo has already taught the hard way, worth carrying forward:

3. **After a push, check the workflow you expected actually ran.** A trigger that doesn't match produces no output at all — indistinguishable from "nothing needed doing" (Part 3.3.1).
4. **Render before you deploy.** `helm template` and `docker compose config` both show you the final resolved result for free. Most image and tag mistakes are visible there (Parts 4.5, 8.1).
5. **Distrust a shared value the moment two consumers need different ones.** `REPO`/`TAG` and Helm's `image.version` were fine until they weren't, and the failure mode was ten pods in `ImagePullBackOff` (Part 4.4).
