# Agape Reverse Proxy

Nginx reverse proxy configuration for Agape V1 and V2 backends.

## Architecture

```
CloudFlare → Nginx Reverse Proxy (this repo)
                ├─> /api/v1/* → backend-v1:5002 (login.py - 226 endpoints)
                ├─> /api/v2/* → backend:5001 (hexagonal architecture - PRODUCTION)
                └─> /api/v2-staging/* → backend-v2-staging:5003 (testing environment)
```

## Files

- `docker-compose.yml` - Docker Compose configuration for all services
- `nginx/nginx.conf` - Nginx reverse proxy configuration

## Deployment

Deployment is automated via GitHub Actions. On push to `master`:

1. Files are copied to EC2 server at `/home/ubuntu/delejove-v2-backend/`
2. Nginx configuration is validated
3. Nginx container is restarted to apply changes

## Endpoints

### V1 API (Legacy login.py)
- Base URL: `https://agape.penwin.cloud/api/v1/*`
- Health check: `https://agape.penwin.cloud/api/v1/health`
- Container: `agape-backend-v1`
- Port: 5002
- Repository: [orioletfarras/agape-v1](https://github.com/orioletfarras/agape-v1)

### V2 API (Hexagonal Architecture - Production)
- Base URL: `https://agape.penwin.cloud/api/v2/*`
- Health check: `https://agape.penwin.cloud/api/v2/health`
- Container: `delejove-v2-backend`
- Port: 5001
- Repository: [orioletfarras/delejove-v2-backend](https://github.com/orioletfarras/delejove-v2-backend)
- Branch: `main`

### V2 Staging API (Testing Environment)
- Base URL: `https://agape.penwin.cloud/api/v2-staging/*`
- Health check: `https://agape.penwin.cloud/api/v2-staging/health`
- Container: `delejove-v2-backend-staging`
- Port: 5003
- Repository: [orioletfarras/delejove-v2-backend](https://github.com/orioletfarras/delejove-v2-backend)
- Branch: `staging`
- Purpose: Test new features before deploying to production

## Manual Deployment

If you need to deploy manually:

```bash
# Copy files to server
scp docker-compose.yml ubuntu@51.94.1.69:~/delejove-v2-backend/
scp nginx/nginx.conf ubuntu@51.94.1.69:~/delejove-v2-backend/nginx/

# SSH to server
ssh ubuntu@51.94.1.69

# Restart nginx
cd ~/delejove-v2-backend
docker compose restart nginx
```

## Configuration Changes

To update nginx configuration:

1. Edit `nginx/nginx.conf` locally
2. Commit and push to master
3. GitHub Actions will automatically deploy the changes

## Server Details

- Server: DeleJove-v2-Production
- IP: 51.94.1.69 (Elastic IP)
- Region: eu-south-2 (Spain)
- Domain: agape.penwin.cloud (CloudFlare)
