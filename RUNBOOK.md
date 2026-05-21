# StartTech Application Runbook

This runbook covers common operational tasks for building, deploying, and debugging the application repository.

## Pipelines

### Backend pipeline

File:

- `.github/workflows/backend-ci-cd.yml`

Stages:

1. test
2. build and push image
3. deploy to EC2 through SSM
4. wait for `/ping`
5. smoke test `/health`

### Frontend pipeline

File:

- `.github/workflows/frontend-ci-cd.yml`

Stages:

1. install dependencies
2. create `.env`
3. build `dist/`
4. sync to S3

## Local Build Commands

Backend:

```bash
cd starttech-application/backend
go test ./... -v
go vet ./...
```

Frontend:

```bash
cd starttech-application/frontend
npm install
npm run build
```

## Deploy Scripts

Backend deploy:

```bash
./scripts/deploy-backend.sh <full-image-uri>
```

Frontend deploy:

```bash
./scripts/deploy-frontend.sh <frontend-bucket-name>
```

Rollback:

```bash
./scripts/rollback.sh <image-uri-or-tag>
```

Health smoke test:

```bash
./scripts/health-check.sh http://<alb-dns>
```

## Backend Health Expectations

- `GET /ping` should return `200` when the API process is listening
- `GET /health` should return `200` only when MongoDB and Redis checks pass

If `/ping` works and `/health` fails, the app is up but a dependency is degraded.

## Common Failure: Wait for Backend Health Fails

If the pipeline loops on the readiness step:

1. confirm the ALB DNS value is correct
2. confirm the target group health path is still `/ping`
3. inspect target health in AWS
4. inspect the backend EC2 instances through SSM

Useful AWS checks:

```bash
aws elbv2 describe-target-health --target-group-arn <target-group-arn>
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names cloud-backend-asg
aws ssm describe-instance-information
```

## Common Failure: ALB Returns 502

What it usually means:

- no healthy targets
- backend container did not start
- userdata bootstrap failed
- EC2 instance cannot reach configuration or image dependencies

High-value checks:

```bash
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name AWS-RunShellScript \
  --parameters commands="sudo docker ps -a"
```

```bash
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name AWS-RunShellScript \
  --parameters commands="sudo tail -n 200 /var/log/cloud-init-output.log"
```

```bash
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name AWS-RunShellScript \
  --parameters commands="curl -i http://localhost:8080/ping"
```

## Known Critical Configuration

The backend deployment expects SSM values under:

- `/starttech/cloud/mongo_uri`
- `/starttech/cloud/jwt_secret`
- `/starttech/cloud/db_name`
- `/starttech/cloud/redis_host`

If these paths drift, instance bootstrap fails.

## Frontend Deployment Checks

If the frontend is calling the wrong backend:

1. verify `VITE_API_BASE_URL` secret
2. rebuild the frontend
3. redeploy `dist/`
4. invalidate CloudFront if the old bundle is still cached

If the frontend deploy succeeded but the browser still looks stale, check whether the latest asset hashes are present in `frontend/dist/` before syncing.

## Safe Release Checklist

1. run backend tests
2. build frontend locally if frontend code changed
3. confirm infrastructure outputs are current
4. deploy backend
5. wait for `/ping`
6. verify `/health`
7. verify login and todo flows in the frontend
