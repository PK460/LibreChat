# LibreChat Docker Desktop Deployment Guide

## Architecture Overview

The application runs as a multi-container stack:

| Service | Image | Purpose |
|---------|-------|---------|
| **LibreChat API** | `registry.librechat.ai/danny-avila/librechat-dev:latest` | Node.js backend + React frontend |
| **MongoDB** | `mongo:8.0.20` | Primary database |
| **MeiliSearch** | `getmeili/meilisearch:v1.35.1` | Full-text search engine |
| **pgvector** | `pgvector/pgvector:0.8.0-pg15-trixie` | Vector database for RAG |
| **RAG API** | `registry.librechat.ai/danny-avila/librechat-rag-api-dev-lite:latest` | Retrieval-augmented generation service |

---

## Prerequisites

1. Install **Docker Desktop** for Windows and ensure it is running.
2. Allocate at least **8 GB RAM** to Docker (Docker Desktop → Settings → Resources → Memory).
3. Ensure **Git** is installed and the repository is cloned:
   ```powershell
   git clone https://github.com/PK460/LibreChat.git
   cd LibreChat
   ```
4. (Optional) Install **Node.js 20+** if you plan to run config scripts outside Docker.

---

## Step 1: Create the Environment File

```powershell
copy .env.example .env
```

Open `.env` in an editor and configure the following **required** values:

### Generate Security Keys

```powershell
# Generate CREDS_KEY (32 bytes hex)
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

# Generate CREDS_IV (16 bytes hex)
node -e "console.log(require('crypto').randomBytes(16).toString('hex'))"

# Generate JWT_SECRET (32 bytes hex)
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

# Generate JWT_REFRESH_SECRET (32 bytes hex)
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

### Minimum Required .env Settings

```env
HOST=localhost
PORT=3080

MONGO_URI=mongodb://mongodb:27017/LibreChat

DOMAIN_CLIENT=http://localhost:3080
DOMAIN_SERVER=http://localhost:3080

CREDS_KEY=<your-generated-value>
CREDS_IV=<your-generated-value>
JWT_SECRET=<your-generated-value>
JWT_REFRESH_SECRET=<your-generated-value>

# Add API keys for your desired AI providers:
OPENAI_API_KEY=user_provided
# ANTHROPIC_API_KEY=
# GOOGLE_KEY=
```

### Optional: MeiliSearch Master Key

```env
MEILI_MASTER_KEY=<your-meili-key>
```

---

## Step 2: (Optional) Configure librechat.yaml

Copy the example configuration file to customize AI endpoints and models:

```powershell
copy librechat.example.yaml librechat.yaml
```

Edit `librechat.yaml` to configure custom endpoints, models, and features. See [LibreChat Configuration Docs](https://www.librechat.ai/docs/configuration/librechat_yaml) for details.

---

## Step 3: Deploy with Docker Compose

### Option A: Using Pre-built Images (Recommended for Quick Start)

This pulls official images from the LibreChat registry:

```powershell
docker compose up -d
```

The application will be available at **http://localhost:3080**.

### Option B: Build Locally from Source

Create a `docker-compose.override.yml` file:

```yaml
services:
  api:
    image: librechat
    build:
      context: .
      target: node
```

Then build and start:

```powershell
docker compose build
docker compose up -d
```

### Option C: Production Deployment (API + NGINX Split)

Uses `deploy-compose.yml` which separates the API and static frontend (served via NGINX):

```powershell
docker compose -f deploy-compose.yml up -d
```

Or use the convenience npm script:

```powershell
npm run start:deployed
```

This exposes:
- **Port 3080** — API server
- **Port 80/443** — NGINX serving the React client

---

## Step 4: Bind-Mount the Configuration File (If Using librechat.yaml)

If you use `docker-compose.yml` (Option A/B), add a volume mount via `docker-compose.override.yml`:

```yaml
services:
  api:
    volumes:
      - type: bind
        source: ./librechat.yaml
        target: /app/librechat.yaml
```

> **Note:** `deploy-compose.yml` already includes this bind-mount.

---

## Step 5: Verify the Deployment

```powershell
# Check all containers are running
docker compose ps

# View API logs
docker compose logs api

# View all service logs
docker compose logs -f
```

Navigate to **http://localhost:3080**, create your first user account, and start chatting.

---

## Step 6: Stopping the Application

```powershell
# Stop all containers (preserves data)
docker compose down

# Stop the deploy-compose stack
npm run stop:deployed

# Stop and remove all volumes (⚠️ DATA LOSS)
docker compose down -v
```

---

## Building and Pushing a Custom Docker Image

### Single-Stage Build (Simpler, Larger Image)

Uses the main `Dockerfile` — builds everything in one stage:

```powershell
docker build -t your-registry/librechat:latest .
```

### Multi-Stage Build (Production-Optimized, Smaller Image)

Uses `Dockerfile.multi` — separates build and runtime layers:

```powershell
docker build -f Dockerfile.multi --target api-build -t your-registry/librechat-api:latest .
```

### Customize Build Arguments

```powershell
# Increase Node.js heap size during build (default: 6144 MB)
docker build --build-arg NODE_MAX_OLD_SPACE_SIZE=8192 -t your-registry/librechat:latest .

# Increase npm install timeout (default: 1500 seconds)
docker build --build-arg NPM_CI_TIMEOUT_SECONDS=2000 -t your-registry/librechat:latest .
```

### Tag the Image

```powershell
docker tag your-registry/librechat:latest your-registry/librechat:v0.8.6-rc1
```

### Push to Docker Hub

```powershell
docker login
docker push your-registry/librechat:latest
docker push your-registry/librechat:v0.8.6-rc1
```

### Push to Azure Container Registry (ACR)

```powershell
az acr login --name yourACRName
docker tag librechat yourACRName.azurecr.io/librechat:latest
docker push yourACRName.azurecr.io/librechat:latest
```

### Push to GitHub Container Registry (GHCR)

```powershell
echo $env:GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
docker tag librechat ghcr.io/your-org/librechat:latest
docker push ghcr.io/your-org/librechat:latest
```

### Use Your Custom Image

Update `docker-compose.override.yml`:

```yaml
services:
  api:
    image: your-registry/librechat:latest
```

Then restart:

```powershell
docker compose up -d
```

---

## Updating the Application

### Update with Pre-built Images

```powershell
docker compose pull
docker compose up -d
```

### Update with Local Build

```powershell
git pull
docker compose build
docker compose up -d
```

### Update via npm Script

```powershell
npm run update:docker
```

---

## Troubleshooting

### Container Won't Start

```powershell
# Check container logs
docker compose logs api

# Check if .env file exists and is properly configured
Test-Path .env
```

### Port Conflicts

If port 3080 is in use, change the `PORT` variable in `.env`:

```env
PORT=3081
```

### MongoDB Connection Issues

Ensure MongoDB container is running:

```powershell
docker compose ps mongodb
docker compose logs mongodb
```

### Memory Issues During Build

Increase Docker Desktop memory allocation and/or the build argument:

```powershell
docker build --build-arg NODE_MAX_OLD_SPACE_SIZE=8192 -t librechat .
```

### Reset Everything

```powershell
docker compose down -v
Remove-Item -Recurse -Force ./data-node, ./meili_data_v1.35.1, ./logs, ./uploads, ./images
docker compose up -d
```

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `Dockerfile` | Single-stage build (all-in-one API + client) |
| `Dockerfile.multi` | Multi-stage build (optimized, smaller production image) |
| `docker-compose.yml` | Default stack (development / self-hosted) |
| `deploy-compose.yml` | Production stack (API + NGINX split) |
| `docker-compose.override.yml.example` | Override examples (local build, config, etc.) |
| `.env.example` | Environment variable template |
| `librechat.example.yaml` | AI endpoint/model configuration template |
| `rag.yml` | Standalone RAG API + pgvector stack |
| `client/nginx.conf` | NGINX config for serving the React client |

---

## Docker Compose Service Ports

| Service | Internal Port | External Port | Notes |
|---------|---------------|---------------|-------|
| API | 3080 | 3080 | Configurable via `PORT` env var |
| MongoDB | 27017 | Not exposed | Internal only by default |
| MeiliSearch | 7700 | Not exposed | Internal only by default |
| pgvector | 5432 | Not exposed | Internal only by default |
| RAG API | 8000 | Not exposed | Configurable via `RAG_PORT` |
| NGINX (deploy) | 80, 443 | 80, 443 | Only in `deploy-compose.yml` |
