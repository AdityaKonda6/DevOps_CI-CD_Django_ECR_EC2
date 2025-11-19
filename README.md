![Build Status](https://github.com/AdityaKonda6/django-devsecops-pipeline/actions/workflows/django-cicd.yml/badge.svg)
![Python](https://img.shields.io/badge/Python-3.10-blue)
![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20ECR-orange)
![Docker](https://img.shields.io/badge/Docker-Container-blue)
# django-devsecops-pipeline
Purpose: Dockerize a Django app, scan the image, push to AWS ECR, and deploy to EC2.
This README collects **everything**: file descriptions, local setup, Docker, CloudShell/ECR steps, EC2 pull & run, image scanning (Trivy HTML), SAST (Bandit), DAST (OWASP ZAP), and helpful troubleshooting.

---
<img width="1536" height="1024" alt="ChatGPT Image Nov 19, 2025, 11_26_46 AM" src="https://github.com/user-attachments/assets/fc3890ca-1a97-4424-9329-0f1ebae5d455" />

---
## Table of contents

1. [Project files & purpose](#project-files--purpose)
2. [Prerequisites](#prerequisites)
3. [Local setup & run (Docker)](#local-setup--run-docker)
4. [Files — explanation and example snippets](#files--explanation-and-example-snippets)
5. [Push image to AWS ECR using CloudShell](#push-image-to-aws-ecr-using-cloudshell)
6. [Pull image on EC2 & run](#pull-image-on-ec2--run)
7. [CI/CD notes (GitHub Actions, OIDC)](#cicd-notes-github-actions-oidc)
8. [Image scanning — Trivy (HTML report)](#image-scanning---trivy-html-report)
9. [SAST — Bandit (Python security static analysis)](#sast---bandit-python-security-static-analysis)
10. [DAST — OWASP ZAP (dynamic scan via Docker)](#dast---owasp-zap-dynamic-scan-via-docker)
11. [Troubleshooting & tips](#troubleshooting--tips)
12. [Cleanup](#cleanup)

---

## Project files & purpose

Your repository should contain (or you should create) the following:

* `Dockerfile` — builds the production image (gunicorn, dependencies).
* `docker-compose.yml` — local dev stack (Django + Postgres).
* `helloworld/settings.py` — Django settings; should be configured to read DB/SECRET/DEBUG from env vars.
* `.dockerignore` — files not needed in image (venv, .git, **pycache**).
* `.env` — local environment variables for docker-compose (never commit secrets).
* `entrypoint.sh` or `setup.sh` — runs migrations, collectstatic, starts server.
* `requirements.txt` — Python package list.
* `manage.py`, app code, templates, static, etc.

---

## Prerequisites

* Docker Desktop (Linux containers) on Windows or Docker on Linux/macOS
* `docker-compose` (if not using the Compose v2 plugin)
* An AWS account and permissions to create ECR repos, IAM roles, and EC2 instances
* A keypair (PEM) for SSH to EC2 (or use Session Manager/SSM)
* Optional: CloudShell (no local AWS CLI required) — [https://console.aws.amazon.com/cloudshell](https://console.aws.amazon.com/cloudshell)

---

## Local setup & run (Docker)

1. Clone the repo:

```bash
git clone https://github.com/AdityaKonda6/DevOps_CI-CD_Django_ECR_EC2.git
cd DevOps_CI-CD_Django_ECR_EC2
```

2. Create `.env` (example):

```
SECRET_KEY=changemeforsure
DEBUG=1
DATABASE_NAME=hellodb
DATABASE_USER=hello
DATABASE_PASSWORD=hello123
DATABASE_HOST=db
DATABASE_PORT=5432
```

> **Never commit** real secrets to Git.

3. Build & run locally with Docker Compose:

```bash
docker-compose up -d --build
```

4. Confirm services are running:

```bash
docker-compose ps
docker-compose logs -f web
```

5. Stop & remove:

```bash
docker-compose down
```

6. Common local tasks:

* Run migrations:

```bash
docker-compose exec web python manage.py migrate
```

* Create superuser:

```bash
docker-compose exec web python manage.py createsuperuser
```

---

## Files — explanation and example snippets

### `Dockerfile`

Purpose: create a production-ready image (use gunicorn + optional WhiteNoise).
Minimal example:

```dockerfile
FROM python:3.11-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
RUN apt-get update && apt-get install -y gcc libpq-dev && rm -rf /var/lib/apt/lists/*
COPY requirements.txt /app/
RUN pip install --upgrade pip && pip install -r requirements.txt
COPY . /app
COPY entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh
EXPOSE 8000
CMD ["/app/entrypoint.sh"]
```

### `docker-compose.yml`

Purpose: development stack (Postgres + Web).
Example (already in repo):

```yaml
version: "3.9"
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: hellodb
      POSTGRES_USER: hello
      POSTGRES_PASSWORD: hello123
    volumes:
      - db_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  web:
    build: .
    command: >
      sh -c "python manage.py migrate --noinput &&
             python manage.py runserver 0.0.0.0:8000"
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    environment:
      - DEBUG=1
      - DATABASE_NAME=hellodb
      - DATABASE_USER=hello
      - DATABASE_PASSWORD=hello123
      - DATABASE_HOST=db
      - DATABASE_PORT=5432
    depends_on:
      - db

volumes:
  db_data:
```

### `settings.py`

Make DB and secrets configurable via environment:

```python
import os
SECRET_KEY = os.getenv('SECRET_KEY', 'dev-secret')
DEBUG = os.getenv('DEBUG', '1') == '1'
ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS', '*').split(',')
DATABASES = {
  'default': {
    'ENGINE': 'django.db.backends.postgresql',
    'NAME': os.getenv('DATABASE_NAME', 'hellodb'),
    'USER': os.getenv('DATABASE_USER', 'hello'),
    'PASSWORD': os.getenv('DATABASE_PASSWORD', 'hello123'),
    'HOST': os.getenv('DATABASE_HOST', 'db'),
    'PORT': os.getenv('DATABASE_PORT', '5432'),
  }
}
```

### `.dockerignore`

Example:

```
__pycache__
*.pyc
*.pyo
*.pyd
venv
env
.env
*.sqlite3
*.log
.git
.DS_Store
```

### `.env`

Use for docker-compose local env vars. Have it in `.gitignore`.

### `entrypoint.sh` (or `setup.sh`)

Example:

```bash
#!/bin/sh
set -e
# optional DB wait here
echo "Running migrations..."
python manage.py migrate --noinput
if [ "$DJANGO_COLLECTSTATIC" = "1" ]; then
  python manage.py collectstatic --noinput
fi
exec gunicorn helloworld.wsgi:application --bind 0.0.0.0:8000 --workers 3
```

### `requirements.txt`

List dependencies:

```
Django>=4.2
gunicorn
psycopg2-binary
whitenoise
```

---

## Push image to AWS ECR using CloudShell (step-by-step)

> Use CloudShell if you cannot install AWS CLI locally.

1. Zip your project locally (or push repo to GitHub so CloudShell can `git clone`).
2. Open **AWS CloudShell**: [https://console.aws.amazon.com/cloudshell](https://console.aws.amazon.com/cloudshell)
3. Upload your project ZIP (CloudShell UI → Upload) and unzip:

```bash
unzip myproject.zip
cd myproject
```

4. Create an ECR repository (or create from Console):

```bash
aws ecr create-repository --repository-name helloworld --region us-east-1
```

5. Authenticate Docker to ECR (replace ACCOUNT_ID & region):

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

6. Build and tag image (in CloudShell):

```bash
docker build -t helloworld-web:latest .
docker tag helloworld-web:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/helloworld:latest
```

7. Push to ECR:

```bash
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/helloworld:latest
```

8. Verify in Console → ECR → Repositories → `helloworld`.

> If push fails with `Denied` cross-account error, ensure you are operating in the **same AWS account** that owns the ECR repo or add a repo policy to allow your principal.

---

## Pull image on EC2 & run

1. Launch an EC2 instance (Ubuntu recommended). Ensure Security Group allows SSH and app port (8000 or 80).
2. Attach IAM role `AmazonEC2ContainerRegistryReadOnly` to the instance (recommended).
3. SSH into EC2:

```bash
ssh -i /path/to/key.pem ubuntu@<ec2-public-ip>
```

4. Install Docker (Ubuntu):

```bash
sudo apt update -y
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

5. Login to ECR & pull:

```bash
aws ecr get-login-password --region us-east-1 | sudo docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
sudo docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/helloworld:latest
```

6. Run container:

```bash
sudo docker run -d --name helloworld -p 8000:8000 \
  -e DEBUG=0 -e SECRET_KEY='supersecret' \
  --restart unless-stopped \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/helloworld:latest
```

7. Open `http://<ec2-public-ip>:8000/` in browser.

**Alternative if you can't attach role:** use CloudShell to perform remote `docker login` on EC2 by piping credentials over SSH (see previous conversation for one-liner).

---

## CI/CD notes (GitHub Actions & OIDC)

Recommended flow:

* Use GitHub OIDC (fine-grained, no long-lived keys) — create IAM role that trusts `token.actions.githubusercontent.com` with conditions limited to your repo.
* Grant that role ECR permissions (create repo, push).
* Workflow builds image on runner, runs Trivy, tags and pushes to ECR, then deploys to EC2 (SSH or SSM).
* Store deploy SSH private key in GitHub Secrets.

I provided a sample `.github/workflows/ci-cd.yml` previously — add it to your repo and set necessary GitHub Secrets.

---

## Image scanning — Trivy (HTML report)

### Pull Trivy image (optional – you may already have it)

```bash
docker pull aquasec/trivy:latest
```

### Scan local image and produce HTML (PowerShell / Linux examples)

**Download HTML template (official):**

```bash
curl -LO https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl
```

**Scan and create `trivy-report.html`:**

```bash
# Linux / WSL / CloudShell
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$(pwd):/report" \
  aquasec/trivy:latest \
  image --format template --template "@/report/html.tpl" -o /report/trivy-report.html helloworld-web:latest
```

**Fail the build on HIGH/CRITICAL:**

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$(pwd):/report" \
  aquasec/trivy:latest \
  image --exit-code 1 --severity HIGH,CRITICAL --format template --template "@/report/html.tpl" -o /report/trivy-report.html helloworld-web:latest
```

Open `trivy-report.html` in your browser or download from CloudShell.

---

## SAST — Bandit (Python static analysis)

1. Install (locally or in CI):

```bash
python -m pip install --upgrade pip
pip install bandit
```

2. Run scan (produce HTML report):

```bash
# from project root (exclude venv and large files)
bandit -r . -f html -o bandit-report.html --exclude venv,__pycache__,node_modules
```

3. Review `bandit-report.html`.

> Add Bandit to your CI pipeline to fail on findings or produce artifacts.

---

## DAST — OWASP ZAP (dynamic scan)

1. Pull ZAP Docker image:

```bash
docker pull zaproxy/zap-stable
```

2. Run ZAP full scan against a running instance (example assumes target accessible at `http://localhost` on same Docker network):

```bash
# If your web container is in network 'my_network' and reachable as 'web:8000'
docker run --rm \
  --network container:nginx-gsquare \
  -v "$(pwd)":/zap/wrk:rw \
  zaproxy/zap-stable \
  zap-full-scan.py -t http://localhost -r /zap/wrk/zap_report.html
```

3. Result: `zap_report.html` in current directory.

> For Docker Desktop local development, run your app and run ZAP against `http://host.docker.internal:8000` if necessary.

---

## Recommended workflow (local → CloudShell → ECR → EC2)

1. Develop locally, verify `docker-compose up -d --build` works.
2. Run Bandit & Trivy locally; fix issues.
3. Zip/upload project to CloudShell (or push to GitHub and let CI do the build).
4. In CloudShell, build, tag, and push to ECR.
5. On EC2 (with ECR read role), `docker pull` and run the container.
6. For production, use RDS (Postgres) instead of SQLite and serve static via S3/Nginx.

---

## Troubleshooting & tips

* **`no basic auth credentials` on `docker pull`** — Docker not authenticated to ECR; run `aws ecr get-login-password ... | docker login ...` on that host (requires instance role or AWS creds).
* **ECR cross-account push denied** — ensure you are pushing from the same AWS account or add repository policy to allow the other account.
* **Large image sizes** — keep `requirements.txt` lean, use slim base images, use multi-stage builds to shrink final image.
* **Persisting DB** — Docker container filesystem is ephemeral; use RDS or Docker volumes for DB persistence.
* **Secrets** — never commit `.env` with real secrets. Use GitHub Secrets, AWS Secrets Manager, or SSM Parameter Store.
* **CI artifacts** — upload `trivy-report.html` & `bandit-report.html` as artifacts in your workflow for review.

---

## Cleanup

* Remove containers/images locally:

```bash
docker-compose down --rmi all --volumes
docker image prune -a
```

* Delete ECR repository (careful — this is destructive):

```bash
aws ecr delete-repository --repository-name helloworld --force --region us-east-1
```

* Terminate EC2 instance via EC2 Console.

---

## Sample quick commands cheat-sheet

Build & run locally:

```bash
docker-compose up -d --build
docker-compose down
```

Build → tag → push (CloudShell):

```bash
docker build -t helloworld-web:latest .
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
docker tag helloworld-web:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/helloworld:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/helloworld:latest
```

Pull & run on EC2:

```bash
aws ecr get-login-password --region us-east-1 | sudo docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
sudo docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/helloworld:latest
sudo docker run -d -p 8000:8000 --name helloworld --restart unless-stopped 123456789012.dkr.ecr.us-east-1.amazonaws.com/helloworld:latest
```

Trivy quick scan & HTML:

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v "$(pwd):/report" aquasec/trivy:latest image --format template --template "@/report/html.tpl" -o /report/trivy-report.html helloworld-web:latest
```

Bandit:

```bash
pip install bandit
bandit -r . -f html -o bandit-report.html --exclude venv,__pycache__
```

ZAP (DAST):

```bash
docker pull zaproxy/zap-stable
docker run --rm --network container:nginx-gsquare -v "$(pwd)":/zap/wrk:rw zaproxy/zap-stable zap-full-scan.py -t http://localhost -r /zap/wrk/zap_report.html
```

---

## Final notes

* This README is designed to be a single source of truth for your full local → cloud pipeline.
* If you want, I can:

  * Generate the exact `trivy-html.tpl` into your repo,
  * Create a ready-to-use GitHub Actions workflow tailored to your repo structure, or
  * Provide step-by-step screenshots for the AWS Console IAM/OIDC setup.

Tell me which of those you want next and I’ll produce the files/commands.
