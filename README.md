# Flask DevOps App — Docker + GitHub Actions CI

A small Flask web app, containerized with Docker, served behind an Nginx reverse proxy, with a CI pipeline that lints, tests, builds, and pushes the image to Docker Hub.

## Project structure

```
.
├── app/
│   ├── app.py              # Flask application (/ and /health endpoints)
│   ├── requirements.txt    # Python dependencies (Flask, Gunicorn)
│   └── test_app.py         # Unit tests (pytest)
├── nginx/
│   └── nginx.conf          # Reverse proxy config
├── .github/workflows/
│   └── ci.yml              # CI pipeline
├── Dockerfile              # Multi-stage, non-root, slim image
├── docker-compose.yml      # App + Nginx
└── .dockerignore
```

## Run locally

**Requirements:** Docker and Docker Compose.

```bash
docker compose up -d --build
```

Then open:

- http://localhost:8080 returns `{"message": "Hello from a Dockerized Flask app!"}`
- http://localhost:8080/health returns `{"status": "ok"}`

Stop it with:

```bash
docker compose down
```

### Run the app without Nginx

```bash
docker build -t flask-devops-app .
docker run -p 5000:5000 flask-devops-app
```

## Dockerfile highlights

- **Multi-stage build:** dependencies are installed in a builder stage, and only the result is copied into the final image, which keeps it small.
- **`python:3.12-slim`** base image.
- **Runs as a non-root user** (`appuser`).
- **Gunicorn** production server instead of Flask's dev server.
- **HEALTHCHECK** on `/health`. Docker Compose waits for the app to be healthy before starting Nginx.

## CI pipeline (GitHub Actions)

The workflow runs on every push and pull request to `main`:

| Job | Steps |
|-----|-------|
| `lint-and-test` | Set up Python 3.12, then lint with `flake8`, then run tests with `pytest` |
| `build-and-push` | Runs only if the first job passes. Builds the Docker image and, **on push to main only**, pushes it to Docker Hub |

Images are tagged with `latest` and the commit SHA, for example `username/flask-devops-app:3f2a1b...`.
Pull requests build the image but don't push it.

### Setup: Docker Hub secrets

1. On Docker Hub, go to **Account Settings → Personal access tokens** and create a token with **Read & Write** access.
2. In the GitHub repo, go to **Settings → Secrets and variables → Actions → New repository secret** and add:
   - `DOCKERHUB_USERNAME`: your Docker Hub username
   - `DOCKERHUB_TOKEN`: the token from step 1
3. Push to `main` and check the **Actions** tab.

## Run tests locally

```bash
pip install -r app/requirements.txt pytest flake8
flake8 app --max-line-length=100
cd app && pytest -v
```
