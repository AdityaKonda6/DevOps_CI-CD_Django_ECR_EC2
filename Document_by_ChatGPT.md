# Django DevSecOps Pipeline – Full Project Documentation

This document provides a complete, end‑to‑end explanation of the **django-devsecops-pipeline** project, including its architecture, components, workflow, tools, deployment strategy, and security integrations. It is written so that even after months, you can read it and immediately understand how the project works and how everything fits together.

---

# 1. Project Overview

This repository contains a **Django web application** packaged inside containers and deployed using a **DevSecOps pipeline**. The pipeline includes:

* **Docker** & **Docker Compose** for containerization
* **GitHub Actions** for CI/CD
* **AWS ECR** for container image storage
* **AWS EC2** for running the application in production
* **PostgreSQL** as the database
* **Nginx** as a reverse proxy in front of Django
* **Trivy (Image Scanning)** for container vulnerability scanning
* **Bandit (SAST)** for Python security analysis
* **OWASP ZAP (DAST)** for dynamic security testing
* **Security best practices** through folder structure and configs

The goal is to create a secure, automated, cloud‑ready deployment pipeline.

---

# 2. Repository Structure

A typical structure for the project:

```
project-root/
│
├── helloworld/               # Django project directory
│   ├── settings.py           # Configurable via environment variables
│   ├── urls.py
│   ├── wsgi.py
│   └── ...
│
├── manage.py
├── Dockerfile                # Builds Django app image
├── docker-compose.yml        # Runs web + db + nginx
├── requirements.txt          # Python dependencies
├── nginx/
│   └── nginx.conf            # Reverse proxy configuration
│
├── .github/
│   └── workflows/
│       └── ci-cd.yml         # GitHub Actions pipeline
│
├── DevOps_CICD_Guide.md      # Additional docs
├── Issues_correction_steps.md
├── SECURITY.md               # Security policy
└── README.md
```

Each file plays a specific role in the DevSecOps workflow.

---

# 3. Django Application

The Django application lives inside the `helloworld/` folder.

### Key Configuration – `settings.py`

* **All sensitive values** (DB credentials, secret key, debug flag) come from **environment variables**.
* **PostgreSQL** is used via Docker.
* Example structure:

  * `SECRET_KEY = os.getenv("SECRET_KEY")`
  * `DEBUG = os.getenv("DEBUG") == "1"`
  * `ALLOWED_HOSTS` includes EC2 IP and localhost

This makes the application secure and cloud‑ready.

---

# 4. Containerization

Containerization is implemented using **Docker** and **Docker Compose**.

## 4.1 Dockerfile

Purpose:

* Build the Django app image
* Install dependencies
* Expose port `8000`
* Run the application (either via `runserver` or Gunicorn)

## 4.2 docker-compose.yml

A multi‑container environment:

### Services:

1. **db** – PostgreSQL 16
2. **web** – Django application (your ECR-pushed image)
3. **nginx** – Reverse proxy in front of Django

Network communication:

* `nginx` → `web:8000`
* `web` → `db:5432`

This mirrors what is deployed on EC2.

---

# 5. Adding Nginx Reverse Proxy

To make the app production-ready, Nginx is used.

### nginx/nginx.conf

* Proxies all incoming traffic on **port 80** to the Django container on **port 8000**.
* Handles incoming Host headers, forwarded IPs, and supports WebSockets.

Benefits:

* Performance
* Security
* Production best practice (run Django behind a proxy)

---

# 6. AWS Deployment

Deployment happens on an **EC2 Instance** using **ECR images** pushed from GitHub.

## 6.1 AWS ECR (Elastic Container Registry)

* Stores the Docker images for the Django app.
* GitHub Actions authenticates using **OIDC**, avoiding long‑lived credentials.
* CI/CD workflow tags and pushes images automatically.

## 6.2 AWS EC2

* Runs Docker + Docker Compose
* Pulls latest image
* Runs three services: `db`, `web`, `nginx`
* Accessible publicly via `http://EC2-IP/`

Security is tightened using:

* Security groups
* IAM permissions
* OIDC for GitHub Actions authentication

---

# 7. CI/CD Pipeline (GitHub Actions)

The CI/CD workflow performs:

### 7.1 Build

* Check out code
* Install dependencies
* Build Docker image for Django

### 7.2 Security Scanning (DevSecOps)

1. **Trivy**

   * Scans image for vulnerabilities
   * Generates HTML report

2. **Bandit**

   * Scans Python source code for security risks

3. **OWASP ZAP**

   * Performs DAST scanning on running app (if configured)

### 7.3 Push to AWS ECR

* Authenticate via GitHub OIDC
* Push built image

### 7.4 Deploy to EC2

* Connect via SSH/SSM
* Run `docker compose pull`
* Run `docker compose up -d`

This ensures automatic deployment after every commit.

---

# 8. Security Features (DevSecOps)

This project integrates **security at every stage**.

### Static Security (SAST)

* **Bandit** checks for Python code vulnerabilities

### Container Security

* **Trivy** scans images for known CVEs

### Runtime Security (DAST)

* **OWASP ZAP** tests for application‑level vulnerabilities

### Secret Management

* All sensitive values are passed via **environment variables**, not committed to Git

### Docker Best Practices

* Using slim Python base images
* Copying only required files
* Non-root execution (if configured)

### AWS Security

* ECR access controlled by IAM
* OIDC avoids storing AWS keys
* EC2 uses limited permissions

---

# 9. Running the Project Locally

### Build and start all services:

```
docker compose up --build
```

### Access:

* Django via Nginx: `http://localhost/`
* Django direct (dev only): `http://localhost:8000/`
* PostgreSQL will be running internally

---

# 10. Deployment Workflow Summary

1. **You push code to GitHub**
2. **GitHub Actions triggers CI/CD**:

   * Build → Scan → Push → Deploy
3. **Image stored in ECR**
4. **EC2 pulls new image**
5. **Docker Compose restarts containers**
6. **Nginx serves the Django app on port 80**

This is a full DevSecOps pipeline.

---

# 11. Future Improvements

You may later add:

* HTTPS using Let's Encrypt + Certbot
* Gunicorn instead of runserver
* Centralized logging (CloudWatch)
* Enhanced ZAP automation
* Terraform for Infrastructure as Code

---

# 12. Conclusion

This repository demonstrates a complete **production-grade DevSecOps workflow** for a Django application using:

* Docker
* Docker Compose
* Nginx
* PostgreSQL
* AWS (EC2 + ECR)
* GitHub Actions
* Security scanning tools (Trivy, Bandit, ZAP)

Everything is automated, secure, and cloud-friendly.

This document allows you to revisit the project anytime and instantly understand:

* Architecture
* Deployment strategy
* CI/CD workflow
* Security layers
* Folder structure

You can copy this file into your repository as `PROJECT_DOCUMENTATION.md` or similar.
