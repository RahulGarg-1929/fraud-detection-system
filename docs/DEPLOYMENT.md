# Deployment Guide

## Overview

FraudShield AI consists of two services:

| Service | Technology | Recommended Host |
|---------|-----------|-----------------|
| **Backend API** | Python / FastAPI | [Railway](https://railway.app) |
| **Frontend** | React / Vite | [Vercel](https://vercel.com) |

---

## Prerequisites

- Docker Desktop (local development)
- Node.js ≥ 20 (frontend)
- Python 3.11 (API)
- A [Railway](https://railway.app) account (backend production)
- A [Vercel](https://vercel.com) account (frontend production)

---

## Local Development with Docker Compose

```bash
# 1. Copy the environment template
cp .env.example .env

# 2. Start all services
docker compose up --build

# 3. Access the app
#    API:      http://localhost:8000
#    API docs: http://localhost:8000/docs
#    Frontend: http://localhost:5173
```

To stop services:

```bash
docker compose down
```

---

## Production: Railway (Backend)

### One-time setup

1. Create a new project on [Railway](https://railway.app/new).
2. Link your GitHub repository.
3. Railway auto-detects `railway.json` and builds using the `Dockerfile`.
4. Set environment variables under **Variables** in the Railway dashboard:

   | Variable | Value |
   |----------|-------|
   | `API_PORT` | `8000` |
   | `MODEL_NAME` | `lightgbm_best` |
   | `LOG_LEVEL` | `info` |

5. Railway will expose a public URL like `https://fraudshield-api-xxxx.railway.app`.

### Automated deployment (CI/CD)

Add a `RAILWAY_TOKEN` secret to your GitHub repository:

1. In Railway: **Account Settings → Tokens → New Token**.
2. In GitHub: **Settings → Secrets → Actions → New repository secret** → name it `RAILWAY_TOKEN`.

Every push to `main` now triggers `.github/workflows/deploy.yml`, which deploys automatically after tests pass.

---

## Production: Vercel (Frontend)

### One-time setup

1. Go to [vercel.com/new](https://vercel.com/new) and import your GitHub repo.
2. Set the **Root Directory** to `frontend`.
3. Add the environment variable:

   | Variable | Value |
   |----------|-------|
   | `VITE_API_URL` | `https://your-api.railway.app` |

4. Click **Deploy**.

### Automated deployment (CI/CD)

Add three secrets to GitHub:

| Secret | Where to find it |
|--------|-----------------|
| `VERCEL_TOKEN` | Vercel → Account Settings → Tokens |
| `VERCEL_ORG_ID` | Vercel project JSON or `vercel link` output |
| `VERCEL_PROJECT_ID` | Vercel project JSON or `vercel link` output |
| `VITE_API_URL` | Your Railway backend URL |

---

## CI/CD Pipeline

Two workflows are included:

| Workflow | File | Trigger |
|----------|------|---------|
| **Tests** | `.github/workflows/test.yml` | Every push / PR |
| **Deploy** | `.github/workflows/deploy.yml` | Push to `main` only |

The deploy workflow calls the test workflow as a dependency — it will not deploy if tests fail.

---

## Health Check

Once deployed, verify the API is running:

```bash
curl https://your-api.railway.app/health
```

Expected response:

```json
{
  "status": "healthy",
  "model_loaded": true,
  "model_name": "lightgbm_best",
  "version": "1.0.0"
}
```

---

## Production Checklist

- [ ] Environment variables set on Railway and Vercel
- [ ] `RAILWAY_TOKEN`, `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID`, `VITE_API_URL` added as GitHub secrets
- [ ] `/health` endpoint returns `200 OK`
- [ ] Frontend `VITE_API_URL` points to live Railway URL
- [ ] CORS origins restricted in `api/main.py` (replace `"*"` with your Vercel domain)

---

## Troubleshooting

### API container fails to start

Check Railway logs:

```bash
railway logs --service fraudshield-api
```

Common causes:
- Model `.pkl` files missing from `models/` — ensure they are committed or mounted.
- Missing `requirements.txt` package — check build logs.

### Frontend shows "Network Error"

- Confirm `VITE_API_URL` is set correctly in Vercel environment variables.
- Confirm the Railway backend is running and `/health` returns `200`.
- Check browser console for CORS errors; restrict `allow_origins` in `api/main.py`.

### Docker build fails locally

```bash
docker build --no-cache -t fraudshield-api .
```

If the issue is a dependency conflict, update `requirements.txt` and rebuild.
