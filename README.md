# Container Registry Tracker

A lightweight **DevOps capstone project** — a Go REST API that tracks container **registries**, the **images** inside them, and **where those images are deployed** (prod / dev / qa). It is backed by PostgreSQL, containerized with a multi-stage Docker build, orchestrated with Docker Compose, and shipped through a Jenkins CI/CD pipeline to Docker Hub.

> **The pitch:** Organisations running containers at scale lose track of which image version is deployed where. This app solves that with a simple REST API backed by PostgreSQL — and demonstrates the full build → push → deploy lifecycle.

---

## Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Highlights](#highlights)
- [Project Structure](#project-structure)
- [Data Model](#data-model)
- [API Reference](#api-reference)
- [Getting Started](#getting-started)
- [CI/CD Pipeline](#cicd-pipeline)
- [Docker Image Optimization](#docker-image-optimization)
- [Screenshots](#screenshots)
- [Author](#author)

---

## Architecture

```
GitHub  →  Jenkins  →  Docker Build  →  Docker Hub  →  Docker Compose  →  Running App + PostgreSQL
```

```
            ┌────────────────────────────────────────────┐
            │              Docker Compose                 │
            │                                             │
   :8081 ───┼──►  registry-tracker-app  ───►  postgres    │
            │     (Go + Gin + GORM)         (postgres:16) │
            │            host=postgres        :5432       │
            └────────────────────────────────────────────┘
```

Inside Docker Compose, the app reaches the database using the **service name** `postgres` as the hostname (not `localhost`) — each container is its own network namespace.

---

## Tech Stack

| Layer            | Technology                                  |
| ---------------- | ------------------------------------------- |
| Language         | Go 1.25                                     |
| Web framework    | [Gin](https://github.com/gin-gonic/gin)     |
| ORM              | [GORM](https://gorm.io)                      |
| Database         | PostgreSQL 16                               |
| Containerization | Docker (multi-stage build)                  |
| Orchestration    | Docker Compose                              |
| Registry         | Docker Hub (`likith0129/registry-tracker`)  |
| CI/CD            | Jenkins (Declarative Pipeline)              |
| SCM              | Git / GitHub                                |

---

## Highlights

- **~96% smaller images** — multi-stage Docker build shrinks the image from **1.33 GB → 46 MB** by shipping only the compiled static binary on Alpine.
- **Resilient startup** — the app retries the database connection up to 10 times (5s apart), so it survives PostgreSQL not being ready yet inside Compose.
- **Auto-migration** — GORM creates/updates tables from the Go structs on boot; no manual SQL.
- **Fully automated delivery** — a Jenkins pipeline builds, logs in, and pushes the image to Docker Hub on every change.
- **Twelve-factor config** — the database DSN is injected via the `DATABASE_URL` environment variable.

---

## Project Structure

```
container-registry-tracker/
├── cmd/
│   └── main.go            # Entry point: connect DB, auto-migrate, register routes, start server (:8081)
├── config/
│   └── database.go        # PostgreSQL connection via GORM, with 10x retry loop
├── models/
│   ├── registry.go        # Registry struct  → registries table
│   ├── image.go           # Image struct     → images table (FK: RegistryID)
│   └── deployment.go      # Deployment struct → deployments table (FK: ImageID)
├── handlers/
│   ├── registry.go        # Create / list registries
│   ├── image.go           # Create / list / get-by-id images
│   └── deployment.go      # Create / list / filter-by-environment deployments
├── routes/
│   └── routes.go          # Maps HTTP routes → handler functions
├── Dockerfile             # Multi-stage build (golang:1.25 → alpine)
├── docker-compose.yml     # App + PostgreSQL services
├── Jenkinsfile            # CI/CD pipeline
├── go.mod / go.sum        # Module definition & dependency checksums
└── README.md
```

---

## Data Model

All models embed `gorm.Model`, so every record also carries `ID`, `CreatedAt`, `UpdatedAt`, and `DeletedAt`.

```
Registry (1) ──< Image (1) ──< Deployment
```

| Model          | Fields                                            | Notes                                  |
| -------------- | ------------------------------------------------- | -------------------------------------- |
| **Registry**   | `name`, `type`                                    | e.g. `Docker Hub`, type `public`       |
| **Image**      | `name`, `version`, `registry_id`                  | belongs to a Registry                  |
| **Deployment** | `image_id`, `environment`                         | environment = `prod` / `dev` / `qa`    |

The three tables are auto-created by GORM on first boot (verified with `\dt` inside the PostgreSQL container):

![PostgreSQL auto-migrated tables](https://raw.githubusercontent.com/LikithKumar0112/Docker-Capstone-Project/Images/docs/screenshots/postgres-tables.png)

---

## API Reference

Base URL: `http://localhost:8081`

### Registries

| Method | Endpoint       | Description            |
| ------ | -------------- | ---------------------- |
| POST   | `/registries`  | Create a registry      |
| GET    | `/registries`  | List all registries    |

### Images

| Method | Endpoint        | Description            |
| ------ | --------------- | ---------------------- |
| POST   | `/images`       | Create an image        |
| GET    | `/images`       | List all images        |
| GET    | `/images/:id`   | Get one image by ID    |

### Deployments

| Method | Endpoint                                  | Description                          |
| ------ | ----------------------------------------- | ------------------------------------ |
| POST   | `/deployments`                            | Create a deployment                  |
| GET    | `/deployments`                            | List all deployments                 |
| GET    | `/deployments/environment/:environment`   | List deployments in an environment   |

### Example requests

```bash
# 1. Create a registry
curl -X POST http://localhost:8081/registries \
  -H "Content-Type: application/json" \
  -d '{"name": "Docker Hub", "type": "public"}'

# 2. Create an image in registry 1
curl -X POST http://localhost:8081/images \
  -H "Content-Type: application/json" \
  -d '{"name": "registry-tracker", "version": "v1.0", "registry_id": 1}'

# 3. Deploy image 1 to production
curl -X POST http://localhost:8081/deployments \
  -H "Content-Type: application/json" \
  -d '{"image_id": 1, "environment": "prod"}'

# 4. List everything deployed to prod
curl http://localhost:8081/deployments/environment/prod
```

---

## Getting Started

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) & Docker Compose
- (Optional, for local non-Docker runs) Go 1.25+ and a local PostgreSQL

### Run with Docker Compose (recommended)

```bash
git clone https://github.com/LikithKumar0112/container-registry-tracker.git
cd container-registry-tracker

docker compose up -d
```

This starts:
- **postgres** — PostgreSQL 16 on port `5432` (data persisted in the `postgres_data` volume)
- **registry-tracker-app** — the API on [http://localhost:8081](http://localhost:8081)

Both containers running, with the app connected to the database and all routes registered:

![docker ps and application logs](https://raw.githubusercontent.com/LikithKumar0112/Docker-Capstone-Project/Images/docs/screenshots/docker-ps-and-app-logs.png)

Stop and clean up:

```bash
docker compose down
```

### Run locally without Docker

```bash
# Start a local PostgreSQL with database 'registrydb' (user/pass: registryuser/registrypass)
go mod download
go run ./cmd
```

If `DATABASE_URL` is not set, the app falls back to a local DSN:
`host=localhost user=registryuser password=registrypass dbname=registrydb port=5432 sslmode=disable`

### Configuration

| Variable        | Description                          | Default (local fallback)                   |
| --------------- | ------------------------------------ | ------------------------------------------ |
| `DATABASE_URL`  | PostgreSQL connection string (DSN)   | `host=localhost ... dbname=registrydb ...` |

---

## CI/CD Pipeline

The [`Jenkinsfile`](./Jenkinsfile) defines a declarative pipeline:

| Stage                  | What it does                                                      |
| ---------------------- | ---------------------------------------------------------------- |
| **Checkout**           | Pulls the latest code from SCM                                   |
| **Build Docker Image** | `docker build -t likith0129/registry-tracker:latest .`           |
| **Docker Login**       | Authenticates to Docker Hub using `dockerhub-creds` (stored in Jenkins Credentials, piped via `--password-stdin`) |
| **Push Docker Image**  | `docker push likith0129/registry-tracker:latest`                 |

Jenkins **Stage View** showing the pipeline running through every stage:

![Jenkins pipeline stage view](https://raw.githubusercontent.com/LikithKumar0112/Docker-Capstone-Project/Images/docs/screenshots/jenkins-stage-view.png)

The resulting image published to Docker Hub:

![Docker Hub repository](https://raw.githubusercontent.com/LikithKumar0112/Docker-Capstone-Project/Images/docs/screenshots/dockerhub-repository.png)

**Server prerequisite:** Jenkins must be able to talk to the Docker daemon:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins docker
```

> When deploying via Compose in the pipeline, run `docker compose down || true` before `docker compose up -d` to avoid the *"container name already in use"* conflict from a previous deployment.

---

## Docker Image Optimization

The [`Dockerfile`](./Dockerfile) uses a **multi-stage build**:

| Approach                       | Image Size | What it contains                        |
| ------------------------------ | ---------- | --------------------------------------- |
| Single stage (`golang:1.25`)   | **1.33 GB** | Go SDK + compiler + source + binary     |
| Multi-stage (Alpine runtime)   | **46 MB**   | Alpine OS + compiled static binary only |

The builder stage compiles a static binary (`CGO_ENABLED=0`), and the runtime stage copies only that binary onto a minimal `alpine:latest` base — roughly a **96% size reduction**.

The single-stage build (`registry-tracker:v1`) weighing in at **1.33 GB**:

![docker images showing v1 at 1.33 GB](https://raw.githubusercontent.com/LikithKumar0112/Docker-Capstone-Project/Images/docs/screenshots/docker-images-v1-1.33gb.png)

After the multi-stage rebuild — `v1` (1.33 GB) next to `v2` (46 MB):

![docker images size comparison v1 vs v2](https://raw.githubusercontent.com/LikithKumar0112/Docker-Capstone-Project/Images/docs/screenshots/docker-image-size-comparison.png)

---

## Screenshots

All screenshots live in [`docs/screenshots/`](https://github.com/LikithKumar0112/Docker-Capstone-Project/tree/Images/docs/screenshots) and are referenced throughout this README. The full lifecycle at a glance:

| What | Screenshot |
| ---- | ---------- |
| App + database running, routes registered | [`docker-ps-and-app-logs.png`](https://raw.githubusercontent.com/LikithKumar0112/Docker-Capstone-Project/Images/docs/screenshots/docker-ps-and-app-logs.png) |
| Image size: single-stage build at 1.33 GB | [`docker-images-v1-1.33gb.png`](https://raw.githubusercontent.com/LikithKumar0112/Docker-Capstone-Project/Images/docs/screenshots/docker-images-v1-1.33gb.png) |
| Image size: v1 (1.33 GB) vs v2 (46 MB) | [`docker-image-size-comparison.png`](https://raw.githubusercontent.com/LikithKumar0112/Docker-Capstone-Project/Images/docs/screenshots/docker-image-size-comparison.png) |
| Image published to Docker Hub | [`dockerhub-repository.png`](https://raw.githubusercontent.com/LikithKumar0112/Docker-Capstone-Project/Images/docs/screenshots/dockerhub-repository.png) |
| Jenkins pipeline stage view | [`jenkins-stage-view.png`](https://raw.githubusercontent.com/LikithKumar0112/Docker-Capstone-Project/Images/docs/screenshots/jenkins-stage-view.png) |
| PostgreSQL auto-migrated tables | [`postgres-tables.png`](https://raw.githubusercontent.com/LikithKumar0112/Docker-Capstone-Project/Images/docs/screenshots/postgres-tables.png) |

---

## Author

**Likith Kumar**
DevOps Capstone Project — *Container Registry Tracker*

Docker Hub image: [`likith0129/registry-tracker`](https://hub.docker.com/r/likith0129/registry-tracker)
