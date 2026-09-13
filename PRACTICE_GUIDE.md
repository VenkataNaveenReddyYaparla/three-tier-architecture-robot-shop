<div align="center">
  <h1>🛠️ DevOps Cheat Sheet</h1>
  <p><i>Actionable DevOps best practices extracted from Stan's Robot Shop</i></p>
</div>

---

This guide highlights the core DevOps practices implemented in this project. While the full **[DEVOPS_PRACTICES.md](DEVOPS_PRACTICES.md)** provides deep dives into *why* these tools are used, this document shows you *how* we apply them.

## 🐳 1. Dockerfile Design Practices

### Cache Optimization
Every instruction in a Dockerfile creates a layer, and Docker caches these layers top-down. 
> [!IMPORTANT]  
> **Rule:** Always copy dependency manifests (like `package.json` or `pom.xml`) and install dependencies **before** copying your source code.

**Example from `cart/Dockerfile`:**
```dockerfile
COPY package.json /opt/server/     # 1. Manifest only
RUN npm install                    # 2. Cached unless package.json changes
COPY server.js /opt/server/        # 3. Source last (changes frequently)
```

### Multi-Stage Builds
Don't ship your build tools in your final image!
**Example from `shipping/Dockerfile`:**
We use a heavy `maven` image to compile the Java code, but the final runtime image is a tiny `eclipse-temurin:25-jre-alpine`. This drops over 500MB of unnecessary bloat and reduces the attack surface.

## 🔄 2. CI/CD Pipeline Practices

### Path Filtering
When working in a monorepo with multiple microservices, CI shouldn't build the `cart` service if only the `payment` code changed.
**Example from `.github/workflows/cart.yaml`:**
```yaml
on:
  push:
    paths:
      - 'cart/**'  # Only trigger on changes to the cart service
```

### Secure Secret Handling
Never pass passwords or tokens via command line arguments (e.g., `docker login -p`), as they can be exposed in system process lists.
> [!TIP]
> **Best Practice:** Pipe secrets securely via standard input.
```bash
echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u ${{ secrets.USERNAME }} --password-stdin
```

## 📊 3. Observability & Reliability

### Metrics Endpoints
Modern microservices must expose their health and performance metrics to scrapers like Prometheus.
- The `cart` and `payment` services expose a standard `/metrics` endpoint, allowing Prometheus to track "items in cart" and "purchases made."

### Asynchronous Fault Tolerance
Microservices should degrade gracefully. 
- In this app, the `payment` service (Python) sends successful transactions to a RabbitMQ queue, which is then consumed by the `dispatch` service (Go). 
- If `dispatch` crashes, the payment still goes through! The queue simply grows silently until `dispatch` is brought back online.

## ☸️ 4. Orchestration Gotchas

### Volume Persistence
In plain `docker-compose.yaml`, the MongoDB and MySQL containers do not use named volumes. 
> [!WARNING]
> This means running `docker compose down` will permanently destroy your product catalog and user data. Always use volumes for stateful services in production!

### Avoiding Outdated Resource Limits
You might see hardcoded JVM heap flags (`-Xmn256m`, `-Xmx768m`) in older Java Dockerfiles. Modern JVMs (running in Kubernetes or Docker) are cgroup-aware and calculate heap sizes automatically based on the container's RAM limits. Avoid hardcoding these where possible!

---
💡 **Want to learn how to build this from scratch?** Check out the full [**DevOps Practices**](DEVOPS_PRACTICES.md).
