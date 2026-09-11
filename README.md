# DevOps Intern Take-Home Assessment

This repository contains the completed DevOps implementation for the Debyez Technologies DevOps Intern Take-Home Assessment.

The application consists of a React + TypeScript frontend, FastAPI backend, PostgreSQL database, and S3-compatible object storage. The application-level business logic was provided as part of the assessment. The DevOps work focuses on containerization, infrastructure, storage integration, reverse proxying, CI/CD, production deployment, versioned images, rollback, and operational verification.

---

## 1. Application Overview

The application provides:

* React + TypeScript frontend
* FastAPI backend
* PostgreSQL database
* S3-compatible object storage using LocalStack
* Item CRUD operations
* File upload, listing, retrieval, and deletion
* Application and dependency health checks

The application provides enough functionality to verify:

* Containerized frontend and backend
* Database connectivity
* Object-storage connectivity
* Inter-service communication
* HTTP routing
* Environment configuration
* CI/CD
* Production deployment
* Version identification
* Rollback

---

# 2. Repository Structure

```text
.
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── database.py
│   │   ├── models/
│   │   ├── schemas/
│   │   ├── routes/
│   │   └── services/
│   │       └── storage.py
│   ├── Dockerfile
│   ├── .dockerignore
│   └── requirements.txt
│
├── frontend/
│   ├── src/
│   ├── public/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── nginx.conf
│   ├── package.json
│   ├── package-lock.json
│   ├── vite.config.ts
│   └── tsconfig.json
│
├── .github/
│   └── workflows/
│       └── ci.yml
│
├── .env.prod.example
├── .gitignore
├── docker-compose.yml
├── docker-compose.prod.yml
└── README.md
```

The real `.env.prod` file is intentionally excluded from Git.

---

# 3. Architecture

```text
                         Browser
                            |
              +-------------+-------------+
              |                           |
     app.debyez.localhost       api.debyez.localhost
              |                           |
              +-------------+-------------+
                            |
                         Traefik
                         Port 80
                            |
              +-------------+-------------+
              |                           |
          Frontend                     Backend
        Nginx container             FastAPI container
                                          |
                         +----------------+----------------+
                         |                                 |
                     PostgreSQL                        LocalStack
                                                        S3
                                                         |
                                               s3.debyez.localhost
```

### Routing

| Hostname               | Destination     |
| ---------------------- | --------------- |
| `app.debyez.localhost` | Frontend        |
| `api.debyez.localhost` | FastAPI backend |
| `s3.debyez.localhost`  | LocalStack S3   |

Only Traefik publishes port `80` to the host.

---

# 4. Level 1 — Containerization & Application Infrastructure

## Backend Container

The backend is containerized using Python 3.12.

The container:

* Installs dependencies from `requirements.txt`
* Copies the FastAPI application
* Exposes port `8000`
* Runs Uvicorn
* Includes a Docker health check through Docker Compose

Backend startup command:

```text
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

## Frontend Container

The frontend uses a multi-stage Docker build:

1. Node.js build stage
2. Nginx runtime stage

The production frontend is served by Nginx on port `80`.

The API URL is provided during the image build using:

```text
VITE_API_BASE_URL
```

## Docker Compose

The development Compose stack contains:

* PostgreSQL
* LocalStack
* FastAPI backend
* React frontend
* Traefik

Start the development environment:

```powershell
docker compose up -d --build
```

Check services:

```powershell
docker compose ps
```

Stop the environment:

```powershell
docker compose down
```

Persistent data is stored in Docker volumes:

```text
postgres_data
localstack_data
```

---

# 5. Health Checks

The backend provides the following health endpoints:

```text
GET /api/health
GET /api/health/db
GET /api/health/s3
GET /api/health/full
```

The full health endpoint verifies the application and its dependencies.

Through Traefik:

```text
http://api.debyez.localhost/api/health
```

```text
http://api.debyez.localhost/api/health/full
```

Docker Compose also uses health checks for:

* PostgreSQL
* LocalStack
* Backend

The backend depends on healthy PostgreSQL and LocalStack services before starting.

---

# 6. Level 2 — Object Storage Integration

LocalStack provides an S3-compatible storage service for local and production-style deployment.

The backend uses two S3 endpoint configurations.

### Internal endpoint

Used by the backend container:

```text
http://localstack:4566
```

### Public endpoint

Used by the browser for presigned URLs:

```text
http://s3.debyez.localhost
```

This separation is necessary because the backend runs inside the Docker network while the browser accesses services through the host-facing Traefik route.

## Storage Operations

The application supports:

* File upload
* File listing
* File retrieval
* File deletion
* Presigned download URLs

The backend automatically ensures that the configured bucket is available.

---

# 7. Level 3 — Reverse Proxy Implementation

Traefik is used as the reverse proxy.

Traefik discovers Docker services through Docker labels.

The following routes are configured:

```text
app.debyez.localhost
        |
        +--> frontend:80

api.debyez.localhost
        |
        +--> backend:8000

s3.debyez.localhost
        |
        +--> localstack:4566
```

Traefik listens on:

```text
0.0.0.0:80
```

The application can therefore be accessed without exposing individual application container ports.

---

# 8. Local Application URLs

Frontend:

```text
http://app.debyez.localhost
```

Backend health:

```text
http://api.debyez.localhost/api/health
```

Full backend health:

```text
http://api.debyez.localhost/api/health/full
```

S3 endpoint:

```text
http://s3.debyez.localhost
```

---

# 9. Level 4 — CI/CD

GitHub Actions automatically builds and publishes Docker images whenever code is pushed to `main`.

Workflow:

```text
Git push
   |
   v
GitHub Actions
   |
   +--> Checkout
   |
   +--> Configure Docker Buildx
   |
   +--> Authenticate to Docker Hub
   |
   +--> Build backend image
   |
   +--> Push backend image
   |
   +--> Build frontend image
   |
   +--> Push frontend image
```

Workflow file:

```text
.github/workflows/ci.yml
```

## Docker Image Repositories

Backend:

```text
aksck/devops-assessment-backend
```

Frontend:

```text
aksck/devops-assessment-frontend
```

Each CI run publishes images using the exact Git commit SHA.

For example:

```text
aksck/devops-assessment-backend:<COMMIT_SHA>
aksck/devops-assessment-frontend:<COMMIT_SHA>
```

The workflow also maintains a `latest` tag.

---

# 10. GitHub Secrets

Docker Hub authentication is configured using GitHub repository secrets.

Configured secrets:

```text
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN
```

The Docker Hub token is not stored in the repository.

No credentials are committed to Git.

---

# 11. CI/CD Verification

The following CI runs were successfully completed:

| Commit                                     | Purpose                                | CI Status  |
| ------------------------------------------ | -------------------------------------- | ---------- |
| `f9eb56f1dd9b8d61f55c4efd079053d1edba229c` | GitHub Actions Docker CI pipeline      | Successful |
| `9772851f289facc7d3071f506e565b0355628467` | SHA-pinned production deployment       | Successful |
| `fbd2df5ffdf36c7a8b5f87b6da1e53addcff8659` | Production image version configuration | Successful |

The latest CI run successfully produced the final Docker images.

---

# 12. Production Deployment

Production deployment is separated from image building.

Production uses:

```text
docker-compose.prod.yml
```

Unlike the development Compose file, the production Compose file does not build application images locally.

The backend and frontend use:

```yaml
image: aksck/devops-assessment-backend:${IMAGE_TAG}
```

and:

```yaml
image: aksck/devops-assessment-frontend:${IMAGE_TAG}
```

The deployment therefore uses an already published CI image.

---

# 13. Production Environment Configuration

A template is provided:

```text
.env.prod.example
```

Contents:

```env
IMAGE_TAG=<CI_COMMIT_SHA>
```

The actual deployment file is:

```text
.env.prod
```

Example:

```env
IMAGE_TAG=fbd2df5ffdf36c7a8b5f87b6da1e53addcff8659
```

`.env.prod` is ignored by Git and is not committed.

---

# 14. Production Deployment Commands

Validate the resolved production configuration:

```powershell
docker compose -f docker-compose.prod.yml --env-file .env.prod config
```

Pull the CI-produced images:

```powershell
docker compose -f docker-compose.prod.yml --env-file .env.prod pull
```

Start the production stack:

```powershell
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d
```

Check service status:

```powershell
docker compose -f docker-compose.prod.yml --env-file .env.prod ps
```

---

# 15. Final Production Deployment

The final production deployment uses commit:

```text
fbd2df5ffdf36c7a8b5f87b6da1e53addcff8659
```

The deployed application images are:

```text
aksck/devops-assessment-backend:fbd2df5ffdf36c7a8b5f87b6da1e53addcff8659
```

```text
aksck/devops-assessment-frontend:fbd2df5ffdf36c7a8b5f87b6da1e53addcff8659
```

The final deployment was verified with Docker Compose.

Final service state:

| Service    | Status  |
| ---------- | ------- |
| Backend    | Healthy |
| Frontend   | Running |
| PostgreSQL | Healthy |
| LocalStack | Healthy |
| Traefik    | Running |

Traefik exposes:

```text
0.0.0.0:80 -> 80/tcp
```

No application image was rebuilt during production deployment. The images were pulled from Docker Hub.

---

# 16. Deployment Version Identification

The currently deployed application version can be identified from:

```powershell
docker compose -f docker-compose.prod.yml --env-file .env.prod ps
```

The backend and frontend image names contain the exact Git commit SHA.

The deployment configuration can also be inspected with:

```powershell
docker compose -f docker-compose.prod.yml --env-file .env.prod config
```

This makes it possible to trace a deployed container back to the exact Git commit that produced the Docker image.

---

# 17. Rollback

Rollback uses an already published Docker image.

No image rebuild is performed during rollback.

A previously verified working version was:

```text
f9eb56f1dd9b8d61f55c4efd079053d1edba229c
```

To roll back, update `.env.prod`:

```env
IMAGE_TAG=f9eb56f1dd9b8d61f55c4efd079053d1edba229c
```

Pull the previously published images:

```powershell
docker compose -f docker-compose.prod.yml --env-file .env.prod pull
```

Deploy:

```powershell
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d
```

Verify:

```powershell
docker compose -f docker-compose.prod.yml --env-file .env.prod ps
```

---

# 18. Rollback Verification

Rollback was actually tested using the previously published CI image:

```text
f9eb56f1dd9b8d61f55c4efd079053d1edba229c
```

The production backend and frontend were successfully switched from:

```text
9772851f289facc7d3071f506e565b0355628467
```

to:

```text
f9eb56f1dd9b8d61f55c4efd079053d1edba229c
```

After rollback:

* Backend became healthy
* Frontend started successfully
* PostgreSQL remained healthy
* LocalStack remained healthy
* Traefik remained operational

Functional verification was also performed after rollback:

* Create item — successful
* Confirm created item — successful
* File upload — successful
* File listing/retrieval — successful

This demonstrated that the rollback restored a working application version using the previously published Docker images.

---

# 19. Git Versioning

Meaningful Git commits were used throughout the implementation.

Current commit history includes:

```text
fbd2df5  Improve production image version configuration
9772851  Add SHA-pinned production deployment
f9eb56f  Add GitHub Actions Docker CI pipeline
43411c9  Add Traefik reverse proxy and production routing
aa58d58  Add service health checks and restart policies
4e0eeeb  Add Docker containerization and compose setup
671b73c  repo setup
```

This provides a clear progression of the DevOps implementation.

---

# 20. Security and Configuration Practices

The implementation follows these practices:

* Production environment file is excluded from Git
* Docker Hub credentials are stored in GitHub Secrets
* No Docker Hub token is committed
* Docker images are deployed using commit-SHA tags
* Production does not rebuild application images
* Application services are not directly exposed to the host
* Traefik is the public entry point
* Docker Compose health checks are used for critical services
* Production configuration is separated from the application image
* `.env.prod.example` provides a safe configuration template

---

# 21. Troubleshooting

## Check all containers

```powershell
docker compose ps
```

For production:

```powershell
docker compose -f docker-compose.prod.yml --env-file .env.prod ps
```

## Check backend logs

```powershell
docker compose logs backend
```

Production:

```powershell
docker compose -f docker-compose.prod.yml logs backend
```

## Check frontend logs

```powershell
docker compose logs frontend
```

## Check Traefik logs

```powershell
docker compose logs traefik
```

## Check LocalStack logs

```powershell
docker compose logs localstack
```

## Inspect a container

```powershell
docker inspect <container-name>
```

## Check backend health

```text
http://api.debyez.localhost/api/health/full
```

## Verify production image version

```powershell
docker compose -f docker-compose.prod.yml --env-file .env.prod ps
```

---

# 22. Verification Checklist

## Level 1 — Containerization

* [x] Backend Dockerfile
* [x] Frontend Dockerfile
* [x] Docker Compose
* [x] PostgreSQL
* [x] Health checks
* [x] Restart policies
* [x] Persistent volumes

## Level 2 — Object Storage

* [x] LocalStack S3
* [x] S3 bucket integration
* [x] File upload
* [x] File listing
* [x] File retrieval
* [x] File deletion
* [x] Presigned URLs

## Level 3 — Reverse Proxy

* [x] Traefik
* [x] Frontend hostname
* [x] Backend hostname
* [x] S3 hostname
* [x] Docker label-based routing
* [x] Port 80 entry point

## Level 4 — CI/CD and Deployment

* [x] GitHub Actions
* [x] Docker Hub authentication
* [x] Backend image build
* [x] Frontend image build
* [x] Commit-SHA image tags
* [x] Production Compose
* [x] Production image pull
* [x] Production deployment
* [x] Deployed version identification
* [x] Rollback using previously published image
* [x] Post-rollback verification
* [x] Functional verification after rollback

---

# 23. Final Result

The assessment implementation provides a complete DevOps workflow:

```text
Developer
   |
   v
GitHub
   |
   v
GitHub Actions
   |
   +--------------------+
   |                    |
   v                    v
Backend Image       Frontend Image
   |                    |
   +---------+----------+
             |
             v
        Docker Hub
             |
             v
   Production Compose
             |
             v
          Traefik
             |
       +-----+-----+
       |           |
       v           v
   Frontend     Backend
                   |
             +-----+------+
             |            |
             v            v
        PostgreSQL     LocalStack
                         S3
```

The final deployment is traceable to a specific Git commit, production deployment uses previously published images, and rollback to a known-good commit-SHA image has been successfully demonstrated and functionally verified.
