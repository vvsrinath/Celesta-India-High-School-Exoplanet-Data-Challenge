# ─────────────────────────────────────────────────────────────────────────────
#  Celesta — Multi-stage Docker build
#  Developed by Srinath Vatchavari Venkateshan
#
#  Stage 1  trainer  : trains the XGBoost+HGB+RF ensemble, writes model.joblib
#  Stage 2  runtime  : lean image — no training deps, serves via gunicorn
#
#  Build:   docker build -t celesta .
#  Run:     docker run -p 5000:5000 celesta
# ─────────────────────────────────────────────────────────────────────────────

# ── Stage 1: train ───────────────────────────────────────────────────────────
FROM python:3.11-slim AS trainer

WORKDIR /build

# build-essential compiles C extensions; libgomp1 is required by XGBoost
RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# Layer cache: install heavy deps before copying source
COPY requirements.txt requirements-train.txt ./
RUN pip install --no-cache-dir -r requirements.txt \
 && pip install --no-cache-dir -r requirements-train.txt

# Copy only what training needs
COPY train_model.py model_utils.py ./
COPY data/koi_stripped.csv data/koi_stripped.csv

# Train — writes model.joblib and data/stats.json
RUN python3 train_model.py

# ── Stage 2: runtime ─────────────────────────────────────────────────────────
FROM python:3.11-slim AS runtime

LABEL maintainer="Srinath Vatchavari Venkateshan"
LABEL description="Celesta — Kepler Exoplanet Classifier"

WORKDIR /app

# XGBoost needs libgomp1 at runtime to deserialise model.joblib
RUN apt-get update && apt-get install -y --no-install-recommends \
        libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# Runtime dependencies only (no shap, no imbalanced-learn)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Application source
COPY app.py model_utils.py ./
COPY templates/ templates/
COPY static/    static/

# Trained artefacts from stage 1
COPY --from=trainer /build/model.joblib      ./model.joblib
COPY --from=trainer /build/data/stats.json   ./data/stats.json

# koi_stripped.csv is needed if app.py ever reads it at startup
COPY data/koi_stripped.csv data/koi_stripped.csv

EXPOSE 5000

# Optional environment variables for the Explore section (Google CSE)
ENV GOOGLE_CSE_CX=""
ENV GOOGLE_CSE_API_KEY=""
ENV FLASK_ENV="production"

# 2 workers; 120 s timeout covers cold model deserialisation (~3 s)
CMD ["gunicorn", \
     "--bind",    "0.0.0.0:5000", \
     "--workers", "2", \
     "--timeout", "120", \
     "--access-logfile", "-", \
     "--error-logfile",  "-", \
     "app:app"]
