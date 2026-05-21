# StartTech Application Architecture

This document describes how the frontend and backend are structured today and how they interact with the deployed AWS environment.

## System Overview

```text
Browser
  -> CloudFront
  -> S3-hosted frontend assets

Browser
  -> ALB
  -> Go API on EC2
  -> MongoDB Atlas
  -> Redis ElastiCache
```

The frontend and backend are deployed independently but work together through the backend ALB URL configured in the frontend build.

## Frontend Architecture

The frontend is a Vite application built with:

- React
- TypeScript
- TanStack Router
- TanStack Query
- Axios

### Frontend API Access

The frontend reads its backend base URL from:

- `VITE_API_BASE_URL`

Requests are created through `src/lib/apiClient.ts`.

Authentication behavior:

- JWT token stored in `localStorage`
- bearer token sent in `Authorization` header

## Backend Architecture

The backend is a Go service built with:

- Gin
- MongoDB driver
- Redis-backed cache abstraction

Core routes:

- `GET /ping`
- `GET /health`
- auth and user routes
- todo routes

### Health Endpoints

The two health endpoints serve different purposes:

- `/ping`
  lightweight process check used by the ALB target group
- `/health`
  dependency-aware check used by smoke tests and operational debugging

`/health` can return `503` when MongoDB or Redis is unavailable even if `/ping` still returns `200`.

## Deployment Architecture

### Backend

```text
GitHub Actions
  -> build Docker image
  -> push to ECR
  -> invoke deploy-backend.sh
  -> SSM Run Command on EC2
  -> docker pull + docker run
```

The deploy script now logs into the registry derived from the full image URI so it stays aligned with the ECR account actually used by CI.

### Frontend

```text
GitHub Actions
  -> npm ci
  -> create .env
  -> vite build
  -> sync dist/ to S3
  -> serve through CloudFront
```

## Configuration Architecture

Backend runtime configuration comes from AWS SSM Parameter Store.

Required keys:

- `/starttech/cloud/mongo_uri`
- `/starttech/cloud/jwt_secret`
- `/starttech/cloud/db_name`
- `/starttech/cloud/redis_host`

These values are consumed by EC2 userdata and by remote deployment commands.

## CORS Model

The backend allows:

- explicit origins from configuration
- the configured CloudFront frontend origin
- localhost development origin

This lets the same backend serve both local development and deployed frontend traffic.

## Failure Modes Worth Knowing

### Wrong frontend API URL

If `VITE_API_BASE_URL` is wrong at build time, the deployed frontend will continue calling the wrong backend until a new production build is generated and redeployed.

### Missing SSM parameters

If EC2 bootstrap cannot find `/starttech/cloud/*`, the backend container never starts and the ALB eventually returns `502`.

### Dependency outage

If MongoDB or Redis is down, `/ping` may still succeed while `/health` fails. That is expected and is why both checks exist.

## Design Intent

The application architecture separates concerns cleanly:

- Terraform provisions infrastructure
- application CI builds and deploys artifacts
- ALB handles public API entry
- CloudFront handles public frontend delivery
- SSM provides secure runtime configuration
