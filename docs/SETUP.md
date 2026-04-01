# Local Development Setup

## Requirements

| Tool | Minimum Version |
|------|----------------|
| Python | 3.11 |
| Node.js | 20 |
| npm | 9 |
| Docker Desktop | 24 (optional) |

---

## Quick Start (without Docker)

### 1. Clone the repository

```bash
git clone https://github.com/RahulGarg-1929/fraud-detection-system.git
cd fraud-detection-system
```

### 2. Create a Python virtual environment

```bash
python -m venv .venv

# macOS / Linux
source .venv/bin/activate

# Windows (PowerShell)
.venv\Scripts\Activate.ps1
```

### 3. Install Python dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

### 4. Set up environment variables

```bash
cp .env.example .env
# Edit .env if needed (defaults work for local development)
```

### 5. Start the API server

```bash
python scripts/run_api.py
```

The API will be available at:
- **API** → `http://localhost:8000`
- **Swagger UI** → `http://localhost:8000/docs`
- **ReDoc** → `http://localhost:8000/redoc`

> If the trained model files are present in `models/`, the API loads them automatically.
> Without model files it starts in **demo mode** using a heuristic scorer.

### 6. Start the React frontend

Open a second terminal:

```bash
cd frontend
npm install
npm run dev
```

The frontend will be available at `http://localhost:5173`.

---

## Quick Start (with Docker Compose)

```bash
cp .env.example .env
docker compose up --build
```

| Service | URL |
|---------|-----|
| API | http://localhost:8000 |
| API docs | http://localhost:8000/docs |
| Frontend | http://localhost:5173 |

---

## Project Structure

```
fraud-detection-system/
├── api/                  # FastAPI application
│   ├── main.py           # Routes and app factory
│   ├── model_service.py  # Model loading & inference
│   └── schemas.py        # Pydantic request/response models
├── src/                  # Core ML pipeline
│   ├── config.py         # Paths, hyperparameters, features
│   ├── data_loader.py    # Data loading utilities
│   ├── preprocessing.py  # Feature preprocessing
│   ├── feature_engineering.py
│   ├── model_training.py
│   ├── evaluation.py
│   └── explainability.py # SHAP explanations
├── scripts/              # Entry-point scripts
│   ├── run_api.py        # Start the API server
│   ├── run_pipeline.py   # Train models end-to-end
│   └── run_eda.py        # Exploratory data analysis
├── models/               # Trained model artifacts (.pkl)
├── frontend/             # React + Vite UI
├── docs/                 # Documentation
├── Dockerfile            # Multi-stage production image
├── docker-compose.yml    # Local development stack
├── railway.json          # Railway deployment config
├── vercel.json           # Vercel deployment config
└── requirements.txt      # Python dependencies
```

---

## Training Models from Scratch

> **Note:** Training requires the IEEE-CIS Fraud Detection dataset from Kaggle.
> Download it and place the CSV files in the project root before running.

```bash
# Run the full ML pipeline (data loading → training → evaluation → saving)
python scripts/run_pipeline.py
```

Trained model artifacts are saved to `models/`.

---

## Running EDA

```bash
python scripts/run_eda.py
```

Plots and reports are saved to `reports/figures/`.

---

## Linting (Frontend)

```bash
cd frontend
npm run lint
```

---

## Environment Variables Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `API_HOST` | `0.0.0.0` | Host the API binds to |
| `API_PORT` | `8000` | Port the API listens on |
| `MODEL_NAME` | `lightgbm_best` | Model file loaded at startup |
| `LOG_LEVEL` | `info` | Logging verbosity |
| `VITE_API_URL` | `http://localhost:8000` | Backend URL used by the frontend |
