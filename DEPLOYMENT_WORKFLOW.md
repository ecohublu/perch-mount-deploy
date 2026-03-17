# Perch Mount Deployment Workflow

This document describes the current Perch Mount deployment setup, including:

- Repository responsibilities
- WSL folder structure
- CI/CD architecture
- Automatic deployment flow
- Manual deployment flow inside WSL
- Hotfix and branch rules

## 1. Repository Layout

Perch Mount is currently split into three repositories:

### Frontend

- Repository: `ecohublu/perch-mount-frontend-2`
- Local development path: `D:\perch-mount-frontend-2`
- Responsibility:
  - Vue frontend application
  - Frontend Docker image build
  - Frontend GitHub Actions CI/CD

### Backend

- Repository: `ecohublu/perch-mount-system`
- Local development path: `D:\perch-mount-system`
- Responsibility:
  - Flask backend application
  - Database migrations
  - Backend Docker image build
  - Backend GitHub Actions CI/CD

### Deploy

- Repository: `ecohublu/perch-mount-deploy`
- Local development path: `D:\perch-mount-deploy`
- WSL working copy: `~/repos/perch-mount-deploy`
- Responsibility:
  - `docker-compose.deploy.yml`
  - deployment workflow
  - PostgreSQL runtime image with `pg_cron`
  - orchestration of frontend, backend, PostgreSQL, and Redis

## 2. WSL Folder Structure

The current recommended WSL layout is:

```text
~/actions-runners/
  website/
  perch-mount-deploy/

~/deploy-runtime/
  perch-mount/
    .env
    secrets/
      flask_secret
      jwt_secret

~/repos/
  perch-mount-deploy/
```

## 3. What Each WSL Folder Does

### `~/actions-runners/`

Stores self-hosted GitHub Actions runners.

- `website/`: runner for the old website repository
- `perch-mount-deploy/`: runner for the deploy repository

These directories are for GitHub job execution only. They are not the source of truth for deployment config.

### `~/deploy-runtime/perch-mount/`

This is the source of truth for deployment runtime values in WSL.

Files:

- `~/deploy-runtime/perch-mount/.env`
- `~/deploy-runtime/perch-mount/secrets/flask_secret`
- `~/deploy-runtime/perch-mount/secrets/jwt_secret`

You should edit this location when changing:

- ports
- API base URL
- auth toggle flags
- image tags
- DB connection values
- runtime secrets

### `~/repos/perch-mount-deploy/`

This is a Git working copy of the deploy repository.

Use it for:

- reading `docker-compose.deploy.yml`
- manual `docker compose` commands
- testing deployment changes locally

This directory is not the runtime source of truth. It is only a working copy.

## 4. Why There Are Two `.env` / `secrets` Locations

There are two roles:

### Source of truth

- `~/deploy-runtime/perch-mount/.env`
- `~/deploy-runtime/perch-mount/secrets/*`

### Manual execution copy

- `~/repos/perch-mount-deploy/.env`
- `~/repos/perch-mount-deploy/secrets/*`

Why this exists:

- GitHub Actions copies runtime files from `~/deploy-runtime/perch-mount`
- manual `docker compose` expects `.env` and `./secrets` inside the repo working directory

This means:

- `deploy-runtime` is the master copy
- `repos/perch-mount-deploy` only needs a copy when you run Docker Compose manually

## 5. Deployment Architecture

```text
Frontend repo main
  -> build frontend image
  -> push ghcr.io/ecohublu/perch-mount-frontend-2:latest

Backend repo main
  -> build backend image
  -> push ghcr.io/ecohublu/perch-mount-system:latest

Deploy repo main
  -> self-hosted runner on WSL
  -> copy runtime files from ~/deploy-runtime/perch-mount
  -> docker login ghcr.io
  -> pull frontend/backend/redis
  -> build postgres with pg_cron
  -> start postgres + redis
  -> run backend migration
  -> restart frontend + backend
```

## 6. Branch Strategy

All three repositories follow the same branch model:

- `dev`: integration branch
- `main`: release branch

Normal flow:

```text
feature/* -> dev -> main
```

Hotfix flow:

```text
hotfix or release fix -> main
main fix must be synced back to dev
```

Important rule:

- If a maintainer patches `main` directly to restore deployment, that change must be merged or replayed back into `dev`

## 7. CI/CD Rules

### Frontend repo

- `dev` / PR:
  - run build validation
- `main`:
  - run build validation
  - build Docker image
  - publish `latest` and `sha-*` tags to GHCR

### Backend repo

- `dev` / PR:
  - run backend validation
- `main`:
  - build Docker image
  - publish `latest` and `sha-*` tags to GHCR

### Deploy repo

- `main`:
  - run deployment on the self-hosted WSL runner

## 8. Automatic Deployment Flow

Use this when deploying normally.

### Step 1: Merge frontend changes to `main`

Frontend fixes must reach:

- `ecohublu/perch-mount-frontend-2` -> `main`

### Step 2: Merge backend changes to `main`

Backend fixes must reach:

- `ecohublu/perch-mount-system` -> `main`

### Step 3: Confirm app images are published

The following images must exist:

- `ghcr.io/ecohublu/perch-mount-frontend-2:latest`
- `ghcr.io/ecohublu/perch-mount-system:latest`

### Step 4: Confirm WSL runtime config

Check:

```bash
grep -E 'FRONTEND_IMAGE|FRONTEND_TAG|BACKEND_IMAGE|BACKEND_TAG|API_BASE_URL|DISABLE_AUTH|FRONTEND_PORT' ~/deploy-runtime/perch-mount/.env
```

Typical values:

```env
FRONTEND_IMAGE=ghcr.io/ecohublu/perch-mount-frontend-2
FRONTEND_TAG=latest
BACKEND_IMAGE=ghcr.io/ecohublu/perch-mount-system
BACKEND_TAG=latest
FRONTEND_PORT=8081
API_BASE_URL=http://localhost:5000
DISABLE_AUTH=true
```

### Step 5: Trigger deploy

Go to:

- `https://github.com/ecohublu/perch-mount-deploy/actions`

Then:

- rerun the latest `main` deployment workflow

### Step 6: Verify deployment

In WSL:

```bash
docker ps
curl http://localhost:5000/ping
curl http://localhost:8081/
```

In browser:

- `http://localhost:8081/`

## 9. What the Deploy Workflow Actually Does

The deploy workflow in `.github/workflows/cicd.yml` performs:

1. checkout deploy repository
2. copy `~/deploy-runtime/perch-mount/.env` to the workflow working directory
3. copy runtime secrets into local `./secrets`
4. login to GHCR
5. pull frontend, backend, and redis images
6. build PostgreSQL image with `pg_cron`
7. start PostgreSQL and Redis
8. run backend migrations
9. restart frontend and backend containers

## 10. Manual Deployment Flow in WSL

Use this for debugging or testing.

### Step 1: Update runtime config

Edit:

```bash
nano ~/deploy-runtime/perch-mount/.env
```

### Step 2: Copy runtime config into repo working copy

```bash
cp ~/deploy-runtime/perch-mount/.env ~/repos/perch-mount-deploy/.env
mkdir -p ~/repos/perch-mount-deploy/secrets
cp ~/deploy-runtime/perch-mount/secrets/flask_secret ~/repos/perch-mount-deploy/secrets/
cp ~/deploy-runtime/perch-mount/secrets/jwt_secret ~/repos/perch-mount-deploy/secrets/
```

### Step 3: Run Docker Compose manually

```bash
cd ~/repos/perch-mount-deploy
docker compose --env-file .env -f docker-compose.deploy.yml up -d
```

### Restart only frontend

```bash
cd ~/repos/perch-mount-deploy
docker compose --env-file .env -f docker-compose.deploy.yml up -d --force-recreate --no-deps frontend
```

### Run migrations manually

```bash
cd ~/repos/perch-mount-deploy
docker compose --env-file .env -f docker-compose.deploy.yml run --rm backend flask --app run.py db upgrade
```

## 11. When to Use Automatic vs Manual Deployment

### Use automatic deployment when:

- you want the normal production-like flow
- you want deployment history in GitHub Actions
- app images were already published from frontend/backend `main`

### Use manual deployment when:

- you are debugging CORS
- you are checking routing
- you are testing `.env` changes
- you only want to restart one service locally

## 12. Known Important Runtime Variables

### Frontend

- `FRONTEND_IMAGE`
- `FRONTEND_TAG`
- `FRONTEND_PORT`
- `API_BASE_URL`
- `GOOGLE_CLIENT_ID`
- `DISABLE_AUTH`

### Backend / DB

- `BACKEND_IMAGE`
- `BACKEND_TAG`
- `POSTGRES_DB`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `PERCH_MOUNT_FLASK_SECRET`
- `PERCH_MOUNT_JWT_SECRET`
- `PERCH_MOUNT_POSTGRESQL_*`
- `PERCH_MOUNT_ACCESS_CONTROL_ALLOW_ORIGIN`

## 13. Common Failure Cases

### Frontend still shows Google login

Check:

- frontend `main` image is updated
- `DISABLE_AUTH=true` exists in `~/deploy-runtime/perch-mount/.env`
- deploy repo passes `DISABLE_AUTH` into frontend container
- deploy was rerun after all changes

### Frontend route returns nginx 404

Check:

- frontend image includes SPA fallback config
- frontend `main` was rebuilt after the Nginx config fix

### API calls go to `/api/api/...`

Check:

- `API_BASE_URL` should be `http://localhost:5000`
- not `http://localhost:5000/api`

### Deploy pulls wrong image path

Check:

- `FRONTEND_IMAGE`
- `BACKEND_IMAGE`

They must not contain `your-github-account`.

### Secrets not found

Check:

- `~/deploy-runtime/perch-mount/secrets/flask_secret`
- `~/deploy-runtime/perch-mount/secrets/jwt_secret`

## 14. Recommended Release Checklist

1. Merge frontend `dev` into `main`
2. Merge backend `dev` into `main`
3. Confirm frontend `main` action succeeded
4. Confirm backend `main` action succeeded
5. Confirm WSL runtime `.env` values
6. Rerun deploy workflow
7. Check `docker ps`
8. Open `http://localhost:8081/`

## 15. Summary

The current deployment model is:

- app code is developed on Windows under `D:\...`
- deploy orchestration is defined in `perch-mount-deploy`
- runtime values live in WSL under `~/deploy-runtime/perch-mount`
- GitHub Actions handles automatic deployment
- WSL manual Docker Compose remains available for debugging

This gives you:

- clear repo responsibilities
- reproducible deployment steps
- persistent database storage
- a usable path for both full automation and manual debugging
