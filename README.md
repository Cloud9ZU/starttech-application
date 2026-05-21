# StartTech Application

StartTech is a full-stack task management application with a React frontend and a Go backend. This repository owns the application code, build pipelines, and deployment scripts.

## Stack

- frontend: React, TypeScript, Vite
- backend: Go, Gin
- containerization: Docker
- backend deployment: ECR -> EC2 via SSM
- frontend deployment: built assets -> S3 bucket behind CloudFront

## Repository Layout

```text
starttech-application/
|-- .github/workflows/
|   |-- backend-ci-cd.yml
|   `-- frontend-ci-cd.yml
|-- backend/
|-- frontend/
`-- scripts/
    |-- deploy-backend.sh
    |-- deploy-frontend.sh
    |-- health-check.sh
    `-- rollback.sh
```

## Local Development

### Backend

```bash
cd starttech-application/backend
go mod download
go run ./cmd/api/main.go
```

The API serves:

- `GET /ping` for basic readiness
- `GET /health` for dependency health

### Frontend

```bash
cd starttech-application/frontend
npm install
npm run dev
```

For production builds:

```bash
npm run build
```

## Required Frontend Environment

The frontend expects:

```env
VITE_API_BASE_URL=http://<alb-dns>
```

This is injected in CI from repository secrets before the Vite build runs.

## Deployment Flows

### Backend

1. run Go tests
2. build Docker image
3. push image to ECR
4. send SSM commands to backend EC2 instances
5. wait for ALB readiness on `/ping`
6. run smoke test on `/health`

### Frontend

1. install Node dependencies
2. generate `.env`
3. run Vite build
4. sync `frontend/dist/` to the provisioned S3 bucket

## Scripts

- `scripts/deploy-backend.sh`
  Sends the backend rollout command to running EC2 instances registered in SSM.
- `scripts/deploy-frontend.sh`
  Syncs `frontend/dist/` to the target S3 bucket.
- `scripts/health-check.sh`
  Polls `GET /health` and fails after retries.
- `scripts/rollback.sh`
  Redeploys a previous backend image by full ECR URI or tag.

## Runtime Notes

- backend configuration comes from AWS Systems Manager Parameter Store under `/starttech/cloud/*`
- backend instances must be able to log in to the ECR registry from the image URI they are asked to deploy
- ALB readiness and smoke testing are intentionally different:
  - `/ping` proves the API process is up
  - `/health` proves the API can reach dependencies

## Related Docs

- [ARCHITECTURE.md](/C:/Sch.Cl/Amazu/starttech-application/ARCHITECTURE.md)
- [RUNBOOK.md](/C:/Sch.Cl/Amazu/starttech-application/RUNBOOK.md)
