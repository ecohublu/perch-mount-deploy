# Perch Mount Deploy

This directory is the minimum viable deployment repository for the Perch Mount system.

For the full deployment, WSL layout, and CI/CD workflow guide, see [DEPLOYMENT_WORKFLOW.md](DEPLOYMENT_WORKFLOW.md).

## Purpose

- Keep deployment configuration separate from frontend and backend application code
- Deploy frontend and backend containers independently from their own GitHub repositories
- Preserve PostgreSQL data with a named volume
- Avoid destructive database resets during normal application deployments

## Recommended GitHub Repositories

- `perch-mount-frontend-2`
- `perch-mount-system`
- `perch-mount-deploy`

## Branch Flow

- `dev` is the integration branch for deployment and infrastructure changes.
- `main` is the release branch used by the deployment workflow.

## Recommended Workflow

1. Create an issue for each deployment or infrastructure task.
2. Create a feature or fix branch from `dev`.
3. Open a pull request back to `dev`.
4. After validation, merge `dev` into `main` to trigger release deployment.

## Hotfix Rule

- If production deployment is blocked, a maintainer may patch `main` directly.
- Every deploy hotfix pushed to `main` must be synced back to `dev`.

## First-Time Setup

1. Copy `.env.example` to `.env`
2. Fill in image names, ports, and backend environment variables
3. Create the `secrets` directory files:
   - `secrets/flask_secret`
   - `secrets/jwt_secret`
4. Update image names to the actual GHCR repositories you publish from GitHub

## Deployment Commands

Start or update infrastructure and application containers:

```bash
docker compose --env-file .env -f docker-compose.deploy.yml up -d
```

Run backend database migrations without deleting data:

```bash
docker compose --env-file .env -f docker-compose.deploy.yml run --rm backend flask --app run.py db upgrade
```

## Notes

- Do not use `docker compose down -v` in production
- PostgreSQL data is stored in the `postgres_data` named volume
- Frontend and backend images are expected to be published to GHCR by their own repositories
