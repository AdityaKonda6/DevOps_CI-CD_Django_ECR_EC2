# 🚀 Django DevSecOps CI/CD Pipeline: Deployment on AWS EC2 with Docker Compose

This repository documents a complete, production-ready **DevSecOps** pipeline for a **Django** application. The solution uses **Docker Compose** to orchestrate three services—Django (Gunicorn), Nginx, and PostgreSQL—and automates the entire build, security, and deployment process using **GitHub Actions** to an **AWS EC2** instance.

The goal of this setup is **speed, security, and reliability**.

-----

## 💡 1. Architecture Overview

The application is deployed using a standard multi-tier containerized architecture.

| Component | Technology | Purpose |
| :--- | :--- | :--- |
| **Frontend Proxy** | **Nginx (Container)** | Serves as the public-facing entry point (Port 80). It handles all incoming HTTP requests and acts as a **Reverse Proxy**, routing traffic to the internal Gunicorn application server. |
| **Application Server** | **Gunicorn/Django (Container)** | The Python application server that executes the Django code on port **8000**. It runs internally, shielded by Nginx. |
| **Database** | **PostgreSQL (Container)** | Provides persistent storage for the application data, mounted via a Docker Volume. |
| **Containerization** | **Docker Compose** | Orchestrates the startup, linking, and networking of the three containers (`db`, `web`, `nginx`). |
| **CI/CD Platform** | **GitHub Actions** | Automated workflow that executes security scanning, builds the image, and handles the deployment via SSH to AWS. |
| **Image Registry** | **AWS ECR** | Stores the final, production-ready Django Docker image. |
| **Deployment Target** | **AWS EC2** | The virtual machine hosting the Docker environment. |

-----

## 🔒 2. DevSecOps & Security Gateways

Security is integrated directly into the CI/CD pipeline, ensuring no vulnerable code or containers reach production.

| Tool | Job Name | Scan Type | Purpose |
| :--- | :--- | :--- | :--- |
| **Bandit** | `security-check` | **SAST** (Static Application Security Testing) | Scans the Python source code for common security vulnerabilities (e.g., hardcoded passwords, insecure functions). |
| **Trivy (FS)** | `security-check` | **Dependency Scanning** | Scans the `requirements.txt` and other project files for known vulnerabilities in third-party libraries. |
| **Trivy (Image)** | `trivy-scan-image` | **Image Scanning** | Scans the *final Docker image* (layers, OS packages, configurations) for vulnerabilities before deployment. **This is a critical Security Gate.** |

-----

## ⚙️ 3. The CI/CD Pipeline Flow (GitHub Actions)

The workflow is defined in `.github/workflows/main.yml` and consists of four sequential jobs.

| Job Name | Dependency | Key Action | Outcome |
| :--- | :--- | :--- | :--- |
| **1. `security-check`** | None | Runs **Bandit** and **Trivy (FS)** on the source code. | Pipeline fails if critical code or dependency vulnerabilities are found. |
| **2. `build-and-push`** | `security-check` | Builds the Docker image from `helloworld/Dockerfile` and pushes it to **AWS ECR**. | Latest image is available in the ECR repository. |
| **3. `trivy-scan-image`**| `build-and-push` | Pulls the newly pushed image from ECR and runs a deep **Trivy** scan. | Pipeline halts if the container image has Critical/High vulnerabilities. |
| **4. `deploy`** | `trivy-scan-image` | Uses **SSH** to connect to EC2, copies `docker-compose.yml` and Nginx files, replaces the `build: .` tag with the ECR image URI, and runs **`docker compose up -d`**. | Application is updated and running on EC2 via Nginx (Port 80). |

-----

## 💻 4. Local Project Configuration Files

### `helloworld/docker-compose.yml`

This file defines the three services and their connectivity.

  * **`web` service:** Uses the image tag (`image: ...:latest`) provided by the pipeline (after the `sed` command runs in the deploy job). It runs Gunicorn on port **8000** internally.
  * **`db` service:** Uses `postgres:15` with a persistent volume (`db_data`).
  * **`nginx` service:** Exposes port **80** to the host machine. It mounts the `nginx.conf` file and uses a custom entrypoint script to prevent race conditions.

### `helloworld/nginx/nginx.conf`

This is the Nginx configuration for the Reverse Proxy.

```nginx
server {
    listen 80;
    server_name _; 

    location / {
        # Forward traffic to the internal 'web' service (Django)
        proxy_pass http://web:8000;
        
        # Preserve Host and IP info for Django
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### `helloworld/nginx/docker-entrypoint.sh`

This script is crucial for deployment stability. The `depends_on: - web` flag only guarantees the container *starts*, not that the application *inside* is ready. Nginx will fail if it starts and cannot resolve the `web` host immediately.

```bash
#!/bin/sh
set -e

# Uses netcat (nc) to repeatedly check if the 'web' service is listening on port 8000.
while ! nc -z web 8000; do
  sleep 0.5
done

# Once the web service is ready, start Nginx.
exec nginx -g "daemon off;"
```

-----

## 🛠 5. AWS Infrastructure and EC2 Setup

The pipeline requires specific AWS prerequisites:

### IAM Permissions

The **EC2 Instance Profile** must be set to the IAM Role (`EC2-ECR-Pull-Role`). This role must contain policies allowing the EC2 host to authenticate with and pull images from **ECR**.

### Security Groups

The security group attached to the EC2 instance must have Inbound Rules open for:

1.  **SSH (Port 22):** For access by the developer and the GitHub Actions runner.
2.  **HTTP (Port 80):** **Crucial** for the public-facing Nginx service.

### EC2 Manual Configuration (The First-Run Setup) ⚠️

When launching a new Ubuntu EC2 instance, these commands **must** be run once via SSH to prepare the environment for Docker Compose:

1.  **Install Docker, AWS CLI, and Netcat:**
    ```bash
    sudo apt update
    sudo apt install docker.io awscli netcat -y
    sudo usermod -aG docker ubuntu
    ```
2.  **Install Docker Compose (for the deploy job):**
    ```bash
    # Check current latest version for the correct URL
    sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
    ```

-----

## ⚠️ 6. Troubleshooting and Maintenance

### Common Errors Solved in this Project

| Error/Issue | Root Cause | Solution Implemented |
| :--- | :--- | :--- |
| **`address already in use`** (Port 80) | A system-level Nginx service was running on the EC2 host, blocking Docker. | `sudo systemctl stop nginx` and `sudo systemctl disable nginx` on the host. |
| **`host not found in upstream "web"`** | Nginx container started before the Django container's internal networking was ready. | Implemented the **`docker-entrypoint.sh`** script with the `netcat` check to force Nginx to wait. |
| **Missing files in `deploy`** | `scp-action` was not explicitly copying the `nginx` directory and `docker-compose.yml`. | Updated `scp-action` source: `source: "helloworld/docker-compose.yml,helloworld/nginx"`. |
| **`bash: aws: command not found`** | AWS CLI was not installed on the fresh EC2 instance. | Manual installation: `sudo apt install awscli -y`. |

### **The Restart Checklist (For Cost-Saving)**

If you delete your AWS resources (EC2, ECR) and need to restart the project later, follow these three steps precisely:

1.  **Create New AWS Resources:** Launch a new EC2 instance (remembering the **IAM Role** and opening **Port 80**). Create a new ECR repository.
2.  **Update GitHub Secrets:** **ALWAYS** update the `EC2_HOST` (new IP), `AWS_ECR_REPO_URI`, and `EC2_SSH_KEY` (if new key pair was used).
3.  **Run EC2 Manual Configuration:** **Crucially**, SSH into the new EC2 and reinstall **Docker, AWS CLI, and Docker Compose** (see Section 5).