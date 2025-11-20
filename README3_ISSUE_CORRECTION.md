This is a comprehensive guide and final document summarizing the complete setup of your **Django DevSecOps CI/CD Pipeline** using GitHub Actions, AWS ECR, and Docker Compose, including all errors encountered and their specific solutions.

-----

## 🏗️ 1. Project Goal & Initial Setup

The goal was to create an automated pipeline that takes your Django application code, runs security checks, builds a Docker image, pushes it to AWS ECR, and finally deploys the multi-container application (Django + PostgreSQL) to an AWS EC2 instance using Docker Compose.

### Core Tools Used:

  * **Repository:** GitHub
  * **CI/CD Runner:** GitHub Actions
  * **Container Registry:** AWS Elastic Container Registry (ECR)
  * **Deployment Target:** AWS EC2
  * **Orchestration:** Docker Compose
  * **Application:** Django with PostgreSQL database (using `psycopg2`)

-----

## 2\. Final CI/CD Pipeline Configuration

The following YAML code represents the final, corrected state of your GitHub Actions workflow (`.github/workflows/django-cicd.yml`). It includes four stages: Security, Build/Push, Image Scan, and Deployment.

```yaml
name: Django CI/CD Pipeline

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  # JOB 1: Security Checks (Bandit & Trivy Dependency/Secret Scan)
  security-check:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - name: Install Bandit
        run: pip install bandit
      - name: Run Bandit Security Check (SAST)
        run: bandit -lll -r . -x ./venv
      - name: Run Trivy Dependency and Secret Scan (FS Mode)
        uses: aquasecurity/trivy-action@master
        with:
          scan-ref: './helloworld'
          scan-type: 'fs'
          format: 'table'
          scanners: 'vuln,secret'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

---
  # JOB 2: Build and Push to ECR
  build-and-push:
    needs: security-check
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
        with:
          mask-password: true
      - name: Build, tag, and push image to Amazon ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: ${{ secrets.AWS_ECR_REPO_URI }}
          IMAGE_TAG: latest
        run: |
          docker build -t $ECR_REPOSITORY:$IMAGE_TAG ./helloworld
          docker push $ECR_REPOSITORY:$IMAGE_TAG

---
  # JOB 3: Trivy Scan of Final Docker Image
  trivy-scan-image:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}
      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v1
        with:
          mask-password: true
      - name: Run Trivy Image Scan (Container OS and Packages)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ secrets.AWS_ECR_REPO_URI }}:latest
          scan-type: 'image'
          format: 'table'
          scanners: 'vuln,config'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

---
  # JOB 4: Deploy to EC2 (Uses Docker Compose)
  deploy:
    needs: trivy-scan-image
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        
      # FIX: Copies the required docker-compose.yml file to EC2
      - name: Copy deployment files to EC2
        uses: appleboy/scp-action@v0.1.4
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          port: 22
          # FIX: Uses the correct path for the file found in the repo
          source: "helloworld/docker-compose.yml" 
          target: "/home/${{ secrets.EC2_USER }}/django-app-deploy"

      # FIX: Executes deployment using docker compose commands
      - name: Deploy to EC2 via SSH
        uses: appleboy/ssh-action@v0.1.6
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          port: 22
          script: |
            APP_DIR="/home/${{ secrets.EC2_USER }}/django-app-deploy"
            
            # Ensure the deployment directory exists
            mkdir -p $APP_DIR

            # 1. Login to ECR
            aws ecr get-login-password --region ${{ secrets.AWS_REGION }} | docker login --username AWS --password-stdin ${{ secrets.AWS_ECR_REPO_URI }}

            # 2. Navigate to the directory where docker-compose.yml was copied
            # FIX: Navigates into the nested folder where the SCP action placed the file
            cd $APP_DIR/helloworld 

            # 3. Pull the latest images for the stack (App from ECR, DB from Docker Hub/Registry)
            docker compose pull
            
            # 4. Gracefully stop and remove the old stack (if running)
            docker compose down || true

            # 5. Start the new stack in detached mode
            docker compose up -d
```

-----

## 3\. Errors Encountered and Solutions Applied

| Error/Problem | Cause | Solution Implemented |
| :--- | :--- | :--- |
| **`permission denied while trying to connect to the docker API`** | The `ubuntu` user lacked permissions to access the Docker socket. | Used `sudo su` to switch to the `root` user for manual testing. **(Note: CI/CD uses the `root` access granted by `appleboy/ssh-action` to run commands.)** |
| **`Exited (1) ... psycopg2.OperationalError: connection to server at "localhost" (::1), port 5432 failed: Connection refused`** | The Django container was running in isolation (`docker run`) and couldn't find a PostgreSQL server on its own `localhost`. | **CRITICAL FIX:** Switched deployment from single `docker run` to **`docker compose up -d`** to launch both the Django and PostgreSQL containers on a shared network. |
| **`psycopg2.OperationalError: could not translate host name "db" to address: Name does not resolve`** | The container was still run in isolation (`docker run`), which doesn't know what "db" means. This name is only resolvable within a Docker Compose network. | Enforced the use of **`docker compose up -d`** in the deployment script (Job 4) and removed any standalone `docker run` attempts. |
| **`bash: line 9: cd: /home/***/app: No such file or directory`** | The deployment script failed to find the `docker-compose.yml` file because the file was **not** on the EC2 host. The build process only uploads the Docker image to ECR. | **CRITICAL FIX:** Added the **`appleboy/scp-action`** step to explicitly copy the `docker-compose.yml` file from the GitHub runner to the EC2 target directory. |
| **`tar: empty archive`** | The `appleboy/scp-action` could not find the file specified in `source:`. | Corrected the `source` path in the `scp-action` from the repository root to the correct nested path: **`helloworld/docker-compose.yml`**. |
| **`cat: /home/***/django-app-deploy/docker-compose.yml: No such file or directory`** | The SCP action recursively copied the file, resulting in a nested structure (`/django-app-deploy/helloworld/docker-compose.yml`). | **CRITICAL FIX:** Modified the `script` block in the SSH action to navigate **into the nested directory** before running Docker Compose: `cd $APP_DIR/helloworld`. |
| **`can't open file '/usr/local/bin/wait-for-it.sh': [Errno 2] No such file or directory`** | The database startup command in `docker-compose.yml` was calling a shell script that was not included in the Docker image. | **RECOMMENDED FIX:** Removed the explicit `wait-for-it.sh` call from the `command` and relied on the **`depends_on: service_healthy`** condition on the `db` service to handle the dependency wait in the Docker Compose file (as shown in the next section). |

-----

## 4\. Final `docker-compose.yml` Configuration

The database connectivity issues required adding a health check to the `db` service and removing the unreliable `wait-for-it.sh` script from the `web` service.

```yaml
version: "3.9"

services:
  # 1. PostgreSQL Database Service
  db:
    image: postgres:16-alpine
    restart: always # Keep DB running reliably
    environment:
      POSTGRES_DB: hellodb
      POSTGRES_USER: hello
      POSTGRES_PASSWORD: hello123
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    
    # Ensures the DB is ready before the application tries to connect
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U hello -d hellodb"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s

  # 2. Django Web Application Service
  web:
    image: 534232118663.dkr.ecr.ap-south-1.amazonaws.com/mydjango:latest # YOUR ECR IMAGE
    
    # Simplified command relies on the service_healthy dependency for waiting
    command: >
      sh -c "
      python manage.py migrate --noinput &&
      gunicorn helloworld.wsgi:application --bind 0.0.0.0:8000
      "
    ports:
      - "8000:8000"
      
    environment:
      # These must be read by your Django settings.py file
      - DATABASE_NAME=hellodb
      - DATABASE_USER=hello
      - DATABASE_PASSWORD=hello123
      - DATABASE_HOST=db       # Hostname is the service name 'db'
      - DATABASE_PORT=5432

    # CRITICAL: Waits for the 'db' service to pass its health check before starting
    depends_on:
      db:
        condition: service_healthy

volumes:
  postgres_data:
```
