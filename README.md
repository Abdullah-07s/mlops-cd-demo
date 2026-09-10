# mlops-cd-demo

A minimal Flask inference API demonstrating a full **Continuous Delivery (CD)** pipeline: automated staging deployment, a health-check smoke test gate, manual approval before production, and rollback via immutable versioned artifacts.

Built as part of an MLOps course assignment, following the tutorial *"Continuous Delivery (CD) for an ML Application."*

---

## What this demonstrates

- **CI vs CD vs Continuous Deployment** — pull requests run tests only; a semantic version tag (`v1.2.0`) triggers the full delivery pipeline.
- **Build once, deploy many** — a single Docker image is built per release and promoted through staging → production, never rebuilt per environment.
- **Immutable, addressable artifacts** — every release is tagged both in Git (`v1.2.0`) and in the container registry (`ghcr.io/.../mlops-cd-demo:1.2.0`), enabling deterministic rollback.
- **Automated staging + smoke test gate** — the pipeline deploys to staging automatically, then verifies `/health` responds correctly before anything is allowed to proceed.
- **Manual production approval** — production deployment pauses and requires an authorized reviewer to approve it in GitHub before going live.
- **Rollback** — demonstrated by redeploying a known-good prior image tag (`1.0.0`) after a simulated bad release (`1.1.0`).

---

## API

| Route | Method | Description |
|---|---|---|
| `/` | GET | Basic service status |
| `/health` | GET | Health check — returns application version, model version, and status |
| `/predict` | POST | Dummy prediction endpoint (`{"value": <number>}` → `value * 2`) |

**`/health` response schema:**
```json
{
  "application_version": "1.2.0",
  "model_version": "1.1",
  "status": "healthy"
}
```

`application_version` is injected at container runtime via the `APP_VERSION` environment variable, set by the CD pipeline to match the released Git tag.

---

## Project structure

```
mlops-cd-demo/
├── app.py                      # Flask inference API
├── requirements.txt
├── Dockerfile
├── VERSION                     # Manual reference version (source of truth is the Git tag)
├── tests/
│   └── test_app.py
└── .github/
    └── workflows/
        └── cd.yml              # Continuous Delivery pipeline
```

---

## Running locally

```bash
python -m venv .venv
.venv\Scripts\Activate.ps1        # Windows PowerShell
pip install -r requirements.txt
python -m pytest
python app.py
```

```bash
curl http://localhost:5000/health
```

## Running with Docker

```bash
docker build -t mlops-cd-demo:local .
docker run --rm -e APP_VERSION=1.2.0 -p 5000:5000 mlops-cd-demo:local
```

---

## CD Pipeline

Defined in [`.github/workflows/cd.yml`](.github/workflows/cd.yml), triggered on any tag matching `v*.*.*`:

```
tag push (v1.2.0)
      |
      v
    test  ──────────  pytest
      |
      v
    build  ─────────  build & push image to GHCR
      |               (versioned tag + latest)
      v
 deploy-staging  ───  deploy container, run /health smoke test
      |
      v
  [ manual approval required ]
      |
      v
 deploy-production  ─ deploy container, verify /health
```

> **Note on environments:** this project has no dedicated staging/production VM, so "deploy" runs the container directly on the GitHub Actions runner and verifies it via `localhost`. The pipeline logic — build once, promote the same artifact, gate on health, require approval — is identical to a real VM-based deployment; only the deployment target differs.

### Releasing a new version

```bash
git checkout main
git pull
git tag v1.3.0
git push origin v1.3.0
```

Watch the run under **Actions → Continuous Delivery**. Approve the `production` environment deployment when prompted.

### Rolling back

Because every release is an immutable, independently addressable image, rollback doesn't require a rebuild:

```bash
docker pull ghcr.io/<owner>/mlops-cd-demo:1.0.0
docker rm -f mlops-api-prod
docker run -d --name mlops-api-prod -p 5000:5000 ghcr.io/<owner>/mlops-cd-demo:1.0.0
curl http://localhost:5000/health
```

---

## Versioning

| Git tag | Image tag |
|---|---|
| `v1.0.0` | `ghcr.io/.../mlops-cd-demo:1.0.0` |
| `v1.1.0` | `ghcr.io/.../mlops-cd-demo:1.1.0` |
| `v1.2.0` | `ghcr.io/.../mlops-cd-demo:1.2.0` |

`latest` always points to the most recently released version and is used for convenience only — production references should always use an explicit version tag for reproducibility.
