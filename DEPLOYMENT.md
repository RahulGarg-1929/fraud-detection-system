# Deployment Guide — FraudShield AI

This document covers every step needed to run the fraud detection system locally, containerise it with Docker, and deploy it to Railway (backend) and Vercel (frontend).

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Local Development Setup](#local-development-setup)
3. [Docker Build & Run](#docker-build--run)
4. [Environment Variables](#environment-variables)
5. [Railway Deployment (Backend)](#railway-deployment-backend)
6. [Vercel Deployment (Frontend)](#vercel-deployment-frontend)
7. [GitHub Actions CI/CD](#github-actions-cicd)
8. [Monitoring & Troubleshooting](#monitoring--troubleshooting)
9. [Cost Estimation](#cost-estimation)
10. [Performance Optimisation](#performance-optimisation)

---

## Project Overview

| Component | Technology | Hosting |
|-----------|-----------|---------|
| Backend API | FastAPI (Python 3.11) | Railway |
| Frontend | React 19 + Vite | Vercel |
| ML Models | LightGBM / XGBoost | Bundled in Docker image |
| CI/CD | GitHub Actions | — |

---

## Local Development Setup

### Prerequisites

- Python 3.11+
- Node.js 20+
- Git with [Git LFS](https://git-lfs.github.com/) installed

### 1. Clone the repository

```bash
git clone https://github.com/RahulGarg-1929/fraud-detection-system.git
cd fraud-detection-system
git lfs pull   # download large model files
```

### 2. Backend

```bash
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env            # fill in your values
python scripts/run_api.py
```

API available at <http://localhost:8000>.  
Interactive docs at <http://localhost:8000/docs>.

### 3. Frontend

```bash
cd frontend
npm install
# Create a local env file
echo "VITE_API_URL=http://localhost:8000" > .env.local
npm run dev
```

Frontend available at <http://localhost:5173>.

---

## Docker Build & Run

### Single service

```bash
# Build the API image
docker build -t fraud-detection-api .

# Run it
docker run -p 8000:8000 \
  -e ENVIRONMENT=development \
  fraud-detection-api
```

### Local development stack (hot-reload)

```bash
cp .env.example .env   # edit .env first
docker compose up --build
```

| Service | URL |
|---------|-----|
| Backend API | <http://localhost:8000> |
| API docs | <http://localhost:8000/docs> |
| Frontend | <http://localhost:5173> |

### Production-like stack

```bash
# Build the frontend first
cd frontend && npm run build && cd ..

docker compose -f docker-compose.prod.yml up --build -d
```

---

## Environment Variables

Copy `.env.example` to `.env` and fill in every value before running.

| Variable | Required | Description |
|----------|----------|-------------|
| `API_HOST` | yes | API bind address (default `0.0.0.0`) |
| `API_PORT` | yes | API port (default `8000`) |
| `ENVIRONMENT` | yes | `development` / `staging` / `production` |
| `VITE_API_URL` | yes | Backend URL consumed by the frontend |
| `MODEL_NAME` | yes | Model file to load (`lightgbm_best`, `xgboost_best`, …) |
| `DEFAULT_THRESHOLD` | no | Fraud decision threshold (default `0.5`) |
| `SECRET_KEY` | yes | Random secret for token signing |
| `ALLOWED_ORIGINS` | yes | Comma-separated CORS origins |
| `LOG_LEVEL` | no | Logging verbosity (default `INFO`) |
| `RAILWAY_TOKEN` | CI only | Railway API token (store as GitHub secret) |
| `VERCEL_TOKEN` | CI only | Vercel API token (store as GitHub secret) |
| `VERCEL_ORG_ID` | CI only | Vercel organisation ID |
| `VERCEL_PROJECT_ID` | CI only | Vercel project ID |

> **Security tip**: Never commit `.env` to version control. The `.gitignore` already excludes it.

---

## Railway Deployment (Backend)

### First-time setup

1. Create a free account at <https://railway.app>.
2. Install the Railway CLI:
   ```bash
   npm install -g @railway/cli
   ```
3. Log in:
   ```bash
   railway login
   ```
4. Create a new project:
   ```bash
   railway init
   ```
5. Link to the project (if already created in the dashboard):
   ```bash
   railway link
   ```

### Deploy

```bash
railway up
```

Railway reads `railway.toml` / `railway.json` for build & deploy settings.  
The service will be available at a URL like `https://fraud-api-production.up.railway.app`.

### Set environment variables on Railway

```bash
railway variables set ENVIRONMENT=production
railway variables set MODEL_NAME=lightgbm_best
railway variables set SECRET_KEY=<your-secret>
railway variables set ALLOWED_ORIGINS=https://your-app.vercel.app
```

Or use the **Railway dashboard → Variables** tab.

### Verify the deployment

```bash
curl https://<your-railway-url>/health
# Expected: {"status":"healthy","model_loaded":true,...}
```

---

## Vercel Deployment (Frontend)

### First-time setup

1. Create a free account at <https://vercel.com>.
2. Install the Vercel CLI:
   ```bash
   npm install -g vercel
   ```
3. From the `frontend/` directory:
   ```bash
   cd frontend
   vercel
   ```
   Follow the interactive prompts to link or create a project.

### Set environment variables

In the Vercel dashboard (or via CLI):

```bash
vercel env add VITE_API_URL production
# Enter: https://<your-railway-url>
```

### Deploy

```bash
# From the repo root
vercel --cwd frontend --prod
```

`vercel.json` configures:
- **SPA rewrites** so React Router handles all routes.
- **API proxy** — requests to `/api/*` are forwarded to the Railway backend.
- **Cache headers** for hashed static assets.

---

## GitHub Actions CI/CD

Two workflow files live in `.github/workflows/`:

| File | Trigger | Jobs |
|------|---------|------|
| `ci-cd.yml` | Push / PR to `main` or `develop` | Backend imports test, health-check smoke test, frontend lint + build, Docker build |
| `deploy-railway.yml` | Push to `main` | Deploy backend to Railway, deploy frontend to Vercel |

### Required GitHub repository secrets

Go to **Settings → Secrets and variables → Actions** and add:

| Secret | Description |
|--------|-------------|
| `RAILWAY_TOKEN` | Railway API token (`railway token create`) |
| `VITE_API_URL` | Production backend URL for the frontend build |
| `VERCEL_TOKEN` | Vercel personal access token |
| `VERCEL_ORG_ID` | Vercel org/user ID (from Vercel dashboard) |
| `VERCEL_PROJECT_ID` | Vercel project ID (from Vercel dashboard) |

---

## Monitoring & Troubleshooting

### API health check

```bash
curl https://<your-railway-url>/health
```

### View Railway logs

```bash
railway logs
# or from the dashboard: Deployments → latest → View logs
```

### Common issues

| Problem | Likely cause | Fix |
|---------|-------------|-----|
| `503 Model not loaded` | Model `.pkl` file missing or corrupt | Ensure Git LFS files were pulled; re-deploy |
| `CORS error` in browser | `ALLOWED_ORIGINS` not set correctly | Add the Vercel URL to `ALLOWED_ORIGINS` |
| Frontend shows blank page | `VITE_API_URL` wrong or missing | Update env var in Vercel dashboard and redeploy |
| Docker build OOM | Not enough RAM for TensorFlow | Use `tensorflow-cpu` (already in `requirements.txt`) |
| Railway build timeout | Slow dependency install | The multi-stage Dockerfile caches deps; re-trigger the build |

---

## Cost Estimation

| Service | Free tier | Paid |
|---------|-----------|------|
| Railway | 500 execution hours/month | ~$5/month for hobby plan |
| Vercel | Unlimited for personal projects | Free for static frontends |
| GitHub Actions | 2000 min/month (public repos: unlimited) | — |

The entire stack can be run **free of charge** for personal/demo use on the free tiers.

---

## Performance Optimisation

### Backend

- **Workers**: Increase `workers` in `scripts/run_api.py` (`uvicorn` supports `--workers N`).
- **Model caching**: Models are loaded once at startup; avoid reloading per request.
- **Batch endpoint**: Use `POST /predict/batch` for bulk scoring — far more efficient than looping over `/predict`.

### Frontend

- Vite produces code-split bundles with hashed filenames; served as immutable from Vercel's CDN.
- Update `vercel.json` cache headers if you add more asset types.

### Docker

- The multi-stage `Dockerfile` keeps the final image lean by copying only the virtual environment — no build tools in production.
- Pin the base image digest in production for reproducible builds:
  ```dockerfile
  FROM python:3.11-slim@sha256:<digest> AS builder
  ```
