# Perch Mount Deploy

This directory is the minimum viable deployment repository for the Perch Mount system.

## Purpose

- Keep deployment configuration separate from frontend and backend application code
- Deploy frontend and backend containers independently from their own GitHub repositories
- Preserve PostgreSQL data with a named volume
- Avoid destructive database resets during normal application deployments

## Recommended GitHub Repositories

- `perch-mount-frontend-2`
- `perch-mount-system`
- `perch-mount-deploy`

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
