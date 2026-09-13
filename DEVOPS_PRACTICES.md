# 📚 DevOps Practices — Stan's Robot Shop

A single progression built around one question at each step: **what does this tool fix that the previous one couldn't?**

Design of the app → Dockerfiles → CI → the registry → plain Docker → Compose → Kubernetes → EKS → observability → Terraform/Ansible → Argo CD.

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

The base was originally `node:14` — EOL since April 2023 — and is now `node:20-alpine`. That change is written up in Part 2.4; `catalogue` and `user` got the same treatment.

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

## 2.3 Dockerfile Best Practices Checklist

Part 2.2 read the files one at a time. This is the same knowledge turned into rules you can apply to *any* Dockerfile, grouped by what each one actually buys you. Every ❌ below has a live example in this repo.

**Cache and build speed**

- ✅ **Order instructions least-changed to most-changed:** base image → system packages → dependency manifest → dependency install → application source. The cache invalidates top-down, so everything you put above your source code gets rebuilt every time your source changes.
- ❌ **Don't `COPY . .` before installing dependencies:** it makes the install step depend on every file in the repo, so editing a README re-runs `npm install`.
- ✅ **Make sure something *consumes* the manifest before the source arrives:** splitting `COPY pom.xml` from `COPY src` achieves nothing on its own. `shipping` is exactly this mistake — see Part 2.2.
- ✅ **Use BuildKit cache mounts for package managers:** `RUN --mount=type=cache,target=/root/.npm npm ci` keeps the download cache between builds without baking it into a layer.

**Image size**

- ✅ **Multi-stage for anything compiled:** build in a stage that has the toolchain, copy only the artifact into a clean runtime stage. `shipping` and `dispatch` both do this, and the JDK and the Go toolchain never reach the final image.
- ✅ **Pick the smallest runtime that still works:** `-alpine` or `-slim` for interpreted languages; `gcr.io/distroless/static-debian12` or `scratch` for a static binary.
- ✅ **Clean package-manager caches in the *same* `RUN`:** `apk add --no-cache`, `pip install --no-cache-dir`, `apt-get … && rm -rf /var/lib/apt/lists/*`. A `rm` in a *later* layer frees nothing — the bytes are still sitting in the layer underneath.
- ❌ **Don't leave build dependencies installed:** `fluentd/Dockerfile` installs `build-base ruby-dev` as `--virtual .build-dependencies` — the whole point of that tag is to `apk del .build-dependencies` afterwards — and then never deletes it. It also splits the install and the gem build across two `RUN`s, so even adding the delete wouldn't shrink the image.
- ✅ **Add a `.dockerignore` — after checking what your context actually is:** not one build context in this repo has one. Every build here runs from the *service* directory (`working-directory: cart` then `docker build .` in CI, `context: cart` in Compose), so the repo's 83MB `.git` is never in scope — the headline reason you'll read everywhere doesn't apply here. What *is* in scope: a local `mvn package` leaves `shipping/target/` in the context, a local `npm install` leaves `node_modules/`, and **Docker does not read `.gitignore`** (`shipping/.gitignore` already excludes `/target` and it makes no difference to a build).

**Reproducibility**

- ❌ **Never use `latest` or an untagged image** — that includes `FROM`, `COPY --from=`, and CI base images. `ratings` pulls `composer:latest`, `dispatch` runs on `alpine:latest`, `fluentd` is a bare `FROM fluentd`.
- ✅ **Pin by digest for anything you genuinely depend on:** `FROM node:20-alpine@sha256:…`. A tag can be re-pointed at new content; a digest can't.
- ✅ **Commit lockfiles and install from them:** `npm ci` instead of `npm install`, a committed `go.mod`/`go.sum`, pinned versions in `requirements.txt`, `composer install --no-dev`. This repo commits **no lockfile of any kind** — `payment/requirements.txt` is seven unpinned package names, so two builds a week apart can legitimately ship different versions of Flask.
- ❌ **Don't generate the module definition during the build:** `dispatch` runs `go mod init && go mod tidy` inside the build stage. Nothing about that is auditable or repeatable.

**Security**

- ❌ **Don't run as root — and never `USER root` deliberately:** `payment` does exactly that. Every service in this repo currently runs as root.
- ✅ **Create a user and switch to it:** `RUN adduser -D -u 10001 app` then `USER 10001`. Use the numeric UID so Kubernetes' `runAsNonRoot` can verify it without resolving `/etc/passwd`.
- ❌ **Never `chmod -R 777`:** `ratings` does, and the comment in the file admits it's a shortcut. If you need to tolerate an arbitrary runtime UID (OpenShift does this), make the paths group-writable and `chgrp -R 0` — group 0, not world.
- 🚨 **Never bake a secret into `ENV` or `ARG`:** the value persists in the image, and `docker history --no-trunc <image>` prints it back to anyone who can pull. `mysql/Dockerfile` ships `MYSQL_PASSWORD=secret` this way — the same class of mistake as `docker login -p` in Part 3.3, just stored on disk instead of in a process list.
- ✅ **Use BuildKit secret mounts for build-time credentials:** `RUN --mount=type=secret,id=npmrc …` makes the value available during that one `RUN` and never writes it to a layer.
- ✅ **Scan the image in CI and fail the build on HIGH/CRITICAL:** `trivy image` or `docker scout cves`. On the EOL bases in this repo it will have a great deal to say.
- ✅ **Drop privileges at run time too:** `read_only: true`, `cap_drop: [ALL]`, `security_opt: ["no-new-privileges:true"]` in Compose, `securityContext` in Kubernetes. The Dockerfile is only half the job.

**Runtime behaviour**

- ✅ **Exec form for `CMD` and `ENTRYPOINT`:** `CMD ["node", "server.js"]`. Shell form wraps the process in `/bin/sh -c`, which becomes PID 1 and swallows `SIGTERM`, so the container takes the full kill timeout to stop on every single deploy.
- ✅ **Configure at run time, not build time:** `web` renders its nginx config from environment variables at container start, which is exactly why one image runs unchanged under Compose, Kubernetes and EKS.
- ✅ **Add a `HEALTHCHECK` — and confirm the tool it calls exists in the image:** moving the Node services to alpine removed `curl`, which the Compose healthchecks still invoke. Busybox `wget -q --spider` is already in there.
- ✅ **`EXPOSE` the port** (documentation, and it makes `-P` work) and use **`WORKDIR`** rather than `RUN cd`, which doesn't persist to the next layer.
- ✅ **Prefer absolute paths in `COPY --from`:** `COPY --from=builder go/src/app/dispatch .` works — relative sources resolve from the root of the source stage — but `/go/src/app/dispatch` says what it means.
- ✅ **Reap zombies if your process won't:** `docker run --init`, or `tini` as the entrypoint. Node and the JVM handle it; shell-script entrypoints that spawn children do not.

**Provenance**

- ✅ **Label the image with where it came from:** `LABEL org.opencontainers.image.source=… org.opencontainers.image.revision=$GIT_SHA`. Six months on, "which commit is this image?" is a question you will actually have to answer — Part 4 is entirely about that problem.

**Two checks you can run right now**

```bash
# BuildKit's built-in linter — nothing to install
docker build --check ./cart

# The stricter third-party linter
docker run --rm -i hadolint/hadolint < cart/Dockerfile
```

Worth knowing what they miss: `docker build --check ./cart` reports **no warnings**, on a file that installs without a lockfile and runs as root. The linters catch syntax-level rules; the design rules above are still yours to apply.

### How this repo scores against it

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

## 2.4 What this repo still needs

Two parts: the base-image move that's already been made and what it cost, then the backlog of everything that hasn't. **Nothing in the backlog has been applied** — each one is written as a proposal with the exact change, so it stays a piece of work you can do and verify yourself.

### The move that's already done

`cart`, `catalogue` and `user` went from `node:14` (EOL April 2023) to `node:20-alpine`, and `shipping`'s runtime stage went to `eclipse-temurin:25-jre-alpine`. Two consequences, both of which generalise to every alpine migration you will ever do:

**1. It's a large saving, and it's worth measuring rather than taking on faith.**

```bash
docker pull node:14 && docker pull node:20-alpine
docker images --format '{{.Repository}}:{{.Tag}}\t{{.Size}}' node
# then compare the built services, not just the bases:
docker images --format '{{.Repository}}:{{.Tag}}\t{{.Size}}' | grep -E 'cart|catalogue|user'
```

The base-image gap is roughly an order of magnitude. The gap between the *built* images is much smaller, because `node_modules` is the same size either way — which is the more useful lesson: a smaller base only helps up to the point where your own dependencies start to dominate.

**2. Alpine is smaller because things are missing.** `curl` is gone from the Node images, and `docker-compose.yaml` still healthchecks those services with `curl`. Nobody noticed, because `docker-compose.local.yaml` declares no healthchecks at all (Part 6.2). The fix is one of:

```dockerfile
RUN apk add --no-cache curl          # keep the healthcheck as written
```
```yaml
test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/health"]   # busybox wget is already there
```

Prefer the second — don't add a package to an image just to satisfy a health probe.

### The backlog, in the order worth doing it

Ordered by value-per-minute, not by severity. The first six are an afternoon in total and they cover most of what a reviewer would flag.

| # | Change | Where | Why it matters |
|---|---|---|---|
| 1 | Add a `.dockerignore` to every build context | all 12 | Keeps locally-built junk (`target/`, `node_modules/`, Symfony cache) out of the context and — for ratings — out of the image. Docker ignores `.gitignore` |
| 2 | `npm ci` + commit `package-lock.json` | cart, catalogue, user | Reproducible installs — today two builds can resolve different versions |
| 3 | Commit `go.mod`/`go.sum`, drop `go mod init` from the build | dispatch | Same reason, plus the dependency set becomes auditable |
| 4 | Pin every floating tag | ratings (`composer`), dispatch (`alpine:latest`), fluentd (`FROM fluentd`) | A build that worked yesterday failing today with no commit in between is the worst debugging experience on this list |
| 5 | Stop baking `MYSQL_PASSWORD` into the image | mysql | Readable with `docker history` by anyone who can pull the image |
| 6 | Add a non-root `USER` | all — start with payment, which sets `USER root` | Prerequisite for `runAsNonRoot` in Part 7, and the most common review finding there is |
| 7 | Add `RUN mvn dependency:go-offline` between the two `COPY`s | shipping | Turns a full dependency re-download into a cache hit on every source change |
| 8 | Replace `chmod -R 777` with `chgrp -R 0` + `chmod -R g=u` | ratings | Same arbitrary-UID tolerance, nothing world-writable |
| 9 | Delete the build deps in the same `RUN` that installs them | fluentd | ~200MB of `build-base` currently ships to production |
| 10 | Move off the EOL bases | ratings (php 7.4), payment (python 3.9), web (nginx 1.21.6), shipping build stage (maven 3.6.3-jdk-8), mysql 5.7, mongo 5 | Real work, not a tag bump — see the table below |
| 11 | `dispatch` → `distroless/static` or `scratch` | dispatch | `CGO_ENABLED=0` already produces a static binary; alpine is dead weight under it |
| 12 | OCI labels + a `trivy`/`docker scout` step in CI | all, plus `.github/workflows/*` | Ties an image back to a commit (Part 4) and catches the CVEs the EOL bases carry |

### The patches for items 1–8

**1 — `.dockerignore`.** One file per build context, and the contents differ per service because the contexts are already narrow (10–64KB each; `web` is 1.2MB of static assets and `mysql` 16MB of SQL, both of which the image needs). This is the cheapest item on the list, but be clear-eyed about what it's worth *here*:

| Context | Put in `.dockerignore` | What it actually prevents |
|---|---|---|
| cart, catalogue, user | `node_modules`, `npm-debug.log` | A local `npm install` ships its `node_modules` to the daemon on every build. It never reaches the image — the Dockerfile copies only `package.json` and `server.js` — so this is upload time, not image size |
| shipping | `target/`, `.classpath`, `.settings` | A local `mvn package` leaves tens of MB in the context. `shipping/.gitignore` already lists `/target`; Docker has never read that file |
| ratings | `html/var/cache/*`, `html/var/log/*`, `html/vendor` | **The one that reaches the image.** `COPY html/ /var/www/html` copies your local Symfony cache and logs straight in — see the note below |
| all | `.env`, `*.pem`, `*.key` | Nothing today, but it's the guard that stops a future `COPY . .` from baking in a credential |

**The ratings one is worth understanding**, because it's a bug hiding behind a permissions fix. The Dockerfile tries to clear the cache with:

```dockerfile
RUN rm -Rf /var/www/var/*
```

but `COPY html/ /var/www/html` put the app — and its `var/cache` and `var/log` — at `/var/www/html/var`. `/var/www/var` is a different directory that doesn't exist in `php:7.4-apache`, so the `rm` matches nothing and quietly succeeds. Today the checked-in `var/` holds only two `.gitkeep` files, so nothing leaks. Run the app locally once and it won't be empty any more.

**2 — reproducible Node installs.** Run `npm install` once locally in `cart/`, `catalogue/` and `user/`, commit the resulting `package-lock.json`, then:

```dockerfile
COPY package.json package-lock.json /opt/server/
RUN npm ci --omit=dev
```

`npm ci` fails loudly when the lockfile and `package.json` disagree, which is exactly the behaviour you want in CI.

**3 — dispatch's module files.** Locally, in `dispatch/`: `go mod init dispatch && go mod tidy`, commit both files, then:

```dockerfile
COPY go.mod go.sum ./
RUN go mod download          # cached dependency layer
COPY *.go .
RUN CGO_ENABLED=0 go build -o /dispatch .
```

This also fixes the cache ordering, which the current file gets wrong — today a one-character change to `main.go` re-downloads every dependency.

**5 — the MySQL password.** Delete the `ENV MYSQL_PASSWORD=secret` line from `mysql/Dockerfile` and supply it at run time instead: `environment:` in Compose backed by `.env`, a Kubernetes `Secret` in Part 7. The image then carries no credential at all. Prove the problem to yourself first:

```bash
docker history --no-trunc robotshop/rs-mysql-db:2.1.0 | grep -i password
```

**6 — a non-root user.** For the alpine-based services:

```dockerfile
RUN adduser -D -u 10001 app && chown -R 10001 /opt/server
USER 10001
```

Debian-based ones use `useradd -r -u 10001 app`. For `payment`, deleting `USER root` is the first half; the second half is making sure uwsgi owns its socket path. For `ratings`, `php:apache` already ships `www-data` — the image simply never switches to it, and Apache listening on port 80 is why (a non-root process can't bind below 1024). That's also why ratings is the only service nginx proxies to on port 80 rather than 8080 (Part 1.3), so moving it to 8080 is part of the same change.

**7 — shipping's cache.** Exactly as written in Part 2.2:

```dockerfile
COPY pom.xml /opt/shipping/
RUN mvn dependency:go-offline
COPY src /opt/shipping/src/
RUN mvn package -o
```

**8 — ratings' permissions.**

```dockerfile
RUN rm -Rf /var/www/var/* \
    && chown -R www-data /var/www \
    && chgrp -R 0 /var/www \
    && chmod -R g=u /var/www
```

Three `RUN`s collapse into one, and nothing ends up world-writable.

### On item 10 — the EOL base images

This is the one place where "just bump the tag" is wrong, and it's worth knowing why before you try it:

| Service | Current | Realistic target | What breaks |
|---|---|---|---|
| web | `nginx:1.21.6` | `nginx:1.27-alpine` | Nothing — `envsubst` is still present. Do this one first. |
| payment | `python:3.9` | `python:3.12-slim` | `uwsgi` needs a compiler to build its wheel on slim — add `build-essential` in a builder stage, or move to `gunicorn` |
| shipping (build) | `maven:3.6.3-jdk-8` | `maven:3.9-eclipse-temurin-21` | Compiling on JDK 8 to run on JRE 25 is backwards; bumping may surface real source-level deprecations |
| mongo | `mongo:5` | `mongo:7` | Data files aren't backward compatible — irrelevant here only because there are no volumes (Part 1.4), which is itself the bug |
| mysql | `mysql:5.7` | `mysql:8.0` | `config.sh` rewrites `my.cnf`; both the 8.0 config layout and the default auth plugin differ |
| ratings | `php:7.4-apache` | `php:8.3-apache` | Genuinely hard — `composer.json` pins `symfony/* ^5.2` and `php ^7.4`. A framework upgrade wearing a base-image costume; leave it last |

The general rule: bumping a base image is a one-line change only when nothing in the image depends on the old version's behaviour. Sorting that table by "what breaks" *before* touching anything is the actual skill.

### What isn't a Dockerfile problem

Three findings that look like Dockerfile bugs but have to be fixed somewhere else — worth separating, because putting a fix in the wrong layer is its own mistake:

- **No volumes**, so every database loses its data on `docker compose down`. That's `docker-compose.yaml` (Part 6), not the images.
- **Healthchecks calling a missing `curl`.** Compose declares them, so the probe is where the fix belongs; adding `curl` back to the image would be fixing it in the wrong place.
- **Plaintext credentials in `mongo/users.js` and RabbitMQ's `guest/guest`.** Seed data and runtime configuration — Part 7's `Secret` objects are where that actually gets solved.

---

# 🔄 Part 3 — How CI works

## 3.1 The mechanics

A CI pipeline is four things, and GitHub Actions names all of them explicitly:

1. **A trigger** — `on: push` / `on: pull_request`, optionally narrowed by `paths:` so a change to one service doesn't rebuild everything.
2. **A runner** — a fresh VM that starts with *nothing*: no repo, no credentials, no state from the last run. `runs-on: ubuntu-latest` rents one from GitHub; `runs-on: self-hosted` means *you* supply the machine.
3. **Steps** — shell commands, or reusable `uses:` actions. Each step is a new shell, so `export FOO=bar` does **not** survive into the next step (you write to `$GITHUB_PATH`/`$GITHUB_ENV` for that).
4. **Secrets** — injected as `${{ secrets.NAME }}`, never committed.

Because the runner starts empty, every pipeline follows the same skeleton: *check out the code → log in to the registry → build the image → push it*.

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

## 3.3 CI/CD Best Practices Checklist

Every workflow must be actively maintained to prevent silent failures. Follow these practices:

- ❌ **Don't use `docker login -p`:** It writes tokens unencrypted and exposes them to system process lists.
- ✅ **Do use `--password-stdin`:** Securely pipe credentials: `echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u ... --password-stdin`
- ✅ **Pin GitHub Actions:** Use specific SHA hashes instead of `@v7` tags to prevent supply-chain attacks.
- ✅ **Scope triggers to paths, not extensions:** Trigger builds with `paths: ['cart/**']` rather than `'**.js'`, which accidentally triggers backend builds when frontend files change.
- ✅ **Drop default permissions:** Always explicitly set `permissions: {contents: read}` to prevent excess token privileges.
- 🚨 **Never push images on `pull_request`:** Restrict `docker push` only to `push` events on the main branch, otherwise unreviewed PR code will overwrite your registry images!

---

# 📦 Part 4 — The registry: image identity and tags

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

---

# 🚀 Part 5 — Deploy with plain Docker

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

Everything you did by hand in Part 8.1 becomes a file. A `terraform/main.tf` uses community modules to declare:

- **VPC module** (`terraform-aws-modules/vpc/aws`): Creates the VPC with public and private subnets across two AZs, a NAT gateway, and the ALB subnet discovery tags (`kubernetes.io/role/elb`).
- **EKS module** (`terraform-aws-modules/eks/aws`): Creates the EKS control plane, enables IRSA (the OIDC step from Part 8.1, declared as code), and provisions a managed node group with `t3.medium` instances (min 1, max 3).

The workflow is always the same four commands:
```bash
cd terraform
terraform init                 # download providers and modules
terraform plan -out=tfplan     # read this carefully — it is the whole point
terraform apply tfplan          # create everything
terraform destroy               # tear it all down
```

**What to take away:** `plan` shows you the diff before anything happens — no other tool in this document gives you that. **State** is the file that maps your code to real resources; it must be remote (S3 + DynamoDB locking) the moment more than one person is involved, and drift between state and reality is the classic Terraform incident. Compare honestly against Part 8.1: five documents of manual commands versus one `apply` that is reproducible and destroyable.

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

**Setup:**
```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd port-forward svc/argocd-server 8081:443
```

Register this repo's Helm chart as an Argo CD `Application` resource pointing at `K8s/helm` with `syncPolicy.automated` enabled. Once connected:

- **Drift correction:** Delete a deployment manually (`kubectl delete deployment cart`) — Argo CD puts it back within seconds. Nobody ran a pipeline.
- **Git as the deploy trigger:** Change a tag in `values.yaml`, commit, push. The cluster converges on its own.

**How this closes the loop with CI:** CI's job becomes *build the image and push it with an immutable tag* (`github.sha`). The deploy step becomes *update the tag in git*. CI never touches the cluster, and git becomes the audit log of every production change — including rollbacks, which are now just `git revert`.

---

# 🗺️ Part 12 — Suggested order

| Stage | Parts | Notes |
|---|---|---|
| 1 | **1 — App design** | One sitting. Read the compose file and the nginx template side by side. |
| 2 | **2 — Dockerfiles** | ✅ dispatch multi-stage and the alpine move are done. 2.3 is the checklist; the backlog in 2.4 is the remaining work. |
| 3 | **3 — CI** | ✅ All eight pipelines build and push. 3.3 is the best practices checklist — start there. |
| 4 | **4 — Registry** | Short, and it prevents a whole category of confusion in every part after it. Don't skip it. |
| 5 | **5 — Plain Docker** | Don't skip this either. Everything after is a reaction to the pain here. |
| 6 | **6 — Compose** | Add the missing volumes; add healthchecks to the local file; run the failure-injection drills. |
| 7 | **7 — Kubernetes** | The biggest block. `helm template` early — it demystifies everything after. |
| 8 | **9 — Observability** | Deliberately before the cloud: it's free locally, and it makes Part 8 far more interesting. |
| 9 | **8 — EKS** | Costs money. Set a budget alert first, tear down the same day. |
| 10 | **10 — Terraform + Ansible** | Terraform rebuilds Part 8 as code; Ansible configures hosts for Part 5. |
| 11 | **11 — Argo CD** | Needs Parts 3 and 7 working. Closes the loop. |

Two habits worth keeping the whole way through:

1. **Keep a defects log.** Every broken thing you find in this repo (Part 1.4 is a starting list) is interview material — you found it, diagnosed it, and can explain the fix.
2. **Tear cloud resources down the same day.** It's the one mistake here that costs money rather than time.

Three habits this repo has already taught the hard way, worth carrying forward:

3. **After a push, check the workflow you expected actually ran.** A trigger that doesn't match produces no output at all — indistinguishable from "nothing needed doing" (Part 3.3.1).
4. **Render before you deploy.** `helm template` and `docker compose config` both show you the final resolved result for free. Most image and tag mistakes are visible there (Parts 4.5, 8.1).
5. **Distrust a shared value the moment two consumers need different ones.** `REPO`/`TAG` and Helm's `image.version` were fine until they weren't, and the failure mode was ten pods in `ImagePullBackOff` (Part 4.4).
