<div align="center">
  <h1>🤖 Stan's Robot Shop</h1>
  <p><i>A modern, polyglot microservices e-commerce application.</i></p>

  <p>
    <img alt="Docker" src="https://img.shields.io/badge/Docker-2CA5E0?style=for-the-badge&logo=docker&logoColor=white" />
    <img alt="Kubernetes" src="https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white" />
    <img alt="Node.js" src="https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white" />
    <img alt="Java" src="https://img.shields.io/badge/Java-ED8B00?style=for-the-badge&logo=java&logoColor=white" />
    <img alt="Python" src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" />
    <img alt="Go" src="https://img.shields.io/badge/Go-00ADD8?style=for-the-badge&logo=go&logoColor=white" />
  </p>
</div>

---

## 📸 App Preview

![Stan's Robot Shop App](./screenshot.png)

> **Note:** Please make sure the attached screenshot is saved as `screenshot.png` in the root of this repository.

## 🚀 Overview

**Stan's Robot Shop** is a sample microservice application used as a hands-on sandbox for container orchestration, CI/CD, and observability. It consists of **eight services** written in **six languages**, backed by **four data stores**. 

This repository includes:
- 🔄 **CI Pipelines** publishing to personal registries.
- 🐳 **Local-Run paths** for day-to-day development.
- 📦 **Refactored EKS charts** for Kubernetes deployments.
- 📚 **Comprehensive Learning Path** in [**DEVOPS_ROADMAP.md**](DEVOPS_ROADMAP.md).

> **Disclaimer:** Error handling is patchy and there is no security built in. This is deliberate — the defects are the teaching material!

## 🏗️ Architecture

A robust microservices mesh where each service owns its data layer:

| Service | Language | Data Store | Port |
|---------|----------|------------|------|
| **🕸️ Web** | NGINX + AngularJS | — | `8080` |
| **📦 Catalogue** | Node.js 20 | MongoDB | `8080` |
| **👤 User** | Node.js 20 | MongoDB + Redis | `8080` |
| **🛒 Cart** | Node.js 20 | Redis | `8080` |
| **🚚 Shipping** | Java (Spring Boot) | MySQL | `8080` |
| **⭐ Ratings** | PHP (Symfony) | MySQL | `80` |
| **💳 Payment** | Python (Flask) | RabbitMQ (Pub) | `8080` |
| **📨 Dispatch** | Go | RabbitMQ (Sub) | — |

## 🛠️ Quick Start

**1. Clone & Configure**
```bash
# Update .env.local with your Docker Hub username
DH_USER=your_username
```

**2. Run Locally (Fastest)**
```bash
docker compose --env-file .env.local -f docker-compose.local.yaml up -d
```
🌐 Open [http://localhost:8080](http://localhost:8080) to view the shop!

**3. Teardown**
```bash
docker compose --env-file .env.local -f docker-compose.local.yaml down -v
```

## ☸️ Advanced Deployment

Ready to scale? Robot Shop can be deployed to:
- **Kubernetes (Minikube/Kind)**
- **AWS EKS**
- **Azure AKS & Google GKE**
- **Docker Swarm & OpenShift**

Detailed instructions for all platforms are available in the [**DevOps Roadmap**](DEVOPS_ROADMAP.md).

## 📖 Learning & Practice

Take your DevOps skills to the next level with our curated learning materials:

- 👉 **[PRACTICE_GUIDE.md](PRACTICE_GUIDE.md):** Quick overview of DevOps practices.
- 👉 **[DEVOPS_ROADMAP.md](DEVOPS_ROADMAP.md):** A 12-part comprehensive learning path from Docker to GitOps with ArgoCD.

---
<div align="center">
  <i>Forked from <a href="https://github.com/instana/robot-shop">instana/robot-shop</a>. Licensed under Apache 2.0.</i>
</div>
