# Railway Deployment Guide for Metaculus

This guide walks you through deploying the Metaculus forecasting platform on [Railway](https://railway.app/), a modern platform-as-a-service that simplifies deployment and scaling.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Quick Start](#quick-start)
- [Detailed Deployment Steps](#detailed-deployment-steps)
- [Environment Variables](#environment-variables)
- [Service Configuration](#service-configuration)
- [Database Setup](#database-setup)
- [Redis Setup](#redis-setup)
- [Health Checks](#health-checks)
- [Scaling](#scaling)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before deploying to Railway, ensure you have:

1. **Railway Account**: Sign up at [railway.app](https://railway.app/)
2. **Railway CLI** (optional but recommended):
   ```bash
   npm install -g @railway/cli
   # or
   curl -fsSL https://railway.app/install.sh | sh
   ```
3. **GitHub Account**: Your repository should be connected to GitHub
4. **Domain** (optional): For custom domain configuration

## Architecture Overview

The Metaculus application consists of several components:

- **Web Service**: Combined Django backend (Gunicorn on port 8000) + Next.js frontend (PM2 on port 3000) behind Nginx (port 8080)
- **Dramatiq Worker Service**: Background job processing for async tasks
- **Django Cron Service**: Scheduled tasks (scoring, metrics evaluation, notifications)
- **PostgreSQL Database**: With pgvector extension for vector similarity searches
- **Redis**: For caching and Dramatiq task queue

## Quick Start

### Option 1: Deploy via Railway Dashboard

1. Go to [Railway Dashboard](https://railway.app/dashboard)
2. Click **"New Project"** → **"Deploy from GitHub repo"**
3. Select your forked `metaculus` repository
4. Railway will automatically detect the `Dockerfile` and `railway.json`
5. Add required environment variables (see [Environment Variables](#environment-variables))
6. Add PostgreSQL database: Click **"+ New"** → **"Database"** → **"PostgreSQL"**
7. Add Redis: Click **"+ New"** → **"Database"** → **"Redis"**
8. Deploy!

### Option 2: Deploy via Railway CLI

```bash
# Login to Railway
railway login

# Initialize a new project
railway init

# Link to existing project (if already created)
railway link

# Add PostgreSQL database
railway add --database postgresql

# Add Redis
railway add --database redis

# Set environment variables
railway variables set SECRET_KEY="your-secret-key-here"
railway variables set PUBLIC_APP_URL="https://your-app.railway.app"
# ... (see full list below)

# Deploy
railway up
```

## Detailed Deployment Steps

### Step 1: Create Railway Project

1. Visit [Railway Dashboard](https://railway.app/dashboard)
2. Click **"New Project"**
3. Choose **"Deploy from GitHub repo"**
4. Authorize Railway to access your GitHub account (if not already done)
5. Select the `metaculus` repository

### Step 2: Configure the Main Web Service

Railway will automatically detect the `Dockerfile` and create a service. Configure it:

1. Click on the service
2. Go to **Settings**
3. Set the following:
   - **Service Name**: `web`
   - **Start Command**: `sh -c 'scripts/prod/startapp.sh'` (or leave blank to use Dockerfile CMD)
   - **Build**: Use `web` target from Dockerfile
   - **Port**: `8080` (Railway will auto-detect from EXPOSE directive)

### Step 3: Add PostgreSQL Database

1. Click **"+ New"** in your project
2. Select **"Database"** → **"PostgreSQL"**
3. Railway will automatically create a PostgreSQL instance
4. **Important**: Enable pgvector extension:
   ```bash
   # Connect via Railway CLI
   railway connect postgres
   
   # In psql shell:
   CREATE EXTENSION IF NOT EXISTS vector;
   \q
   ```

### Step 4: Add Redis

1. Click **"+ New"** in your project
2. Select **"Database"** → **"Redis"**
3. Railway will automatically create a Redis instance

### Step 5: Add Dramatiq Worker Service

1. Click **"+ New"** → **"GitHub Repo"**
2. Select the same `metaculus` repository
3. Configure the service:
   - **Service Name**: `dramatiq-worker`
   - **Dockerfile Path**: `Dockerfile`
   - **Docker Build Target**: `dramatiq_worker`
   - **Start Command**: `sh -c 'scripts/prod/run_dramatiq.sh'`
4. Add the same environment variables as the web service

### Step 6: Add Django Cron Service

1. Click **"+ New"** → **"GitHub Repo"**
2. Select the same `metaculus` repository
3. Configure the service:
   - **Service Name**: `django-cron`
   - **Dockerfile Path**: `Dockerfile`
   - **Docker Build Target**: `django_cron`
   - **Start Command**: `sh -c 'scripts/prod/django_cron.sh'`
4. Add the same environment variables as the web service

### Step 7: Run Database Migrations

Before your app can run, you need to run migrations:

```bash
# Option 1: Via Railway CLI
railway run python manage.py migrate

# Option 2: Create a one-off job in Railway Dashboard
# Go to your web service → "Deploy" tab → "Run a command"
# Enter: python manage.py migrate
```

## Environment Variables

Configure these environment variables for all services (web, worker, cron):

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | Auto-provided by Railway PostgreSQL |
| `REDIS_URL` | Redis connection string | Auto-provided by Railway Redis |
| `SECRET_KEY` | Django secret key (generate a secure random string) | `django-insecure-...` |
| `PUBLIC_APP_URL` | Frontend URL | `https://your-app.railway.app` |
| `PUBLIC_API_BASE_URL` | Backend API URL | `https://your-app.railway.app/api` |

### Optional Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `DEBUG` | Enable Django debug mode (DO NOT enable in production) | `false` |
| `ALLOWED_HOSTS` | Comma-separated list of allowed hosts | `your-app.railway.app,*.railway.app` |
| `AWS_STORAGE_BUCKET_NAME` | S3 bucket for media storage | `metaculus-media` |
| `AWS_ACCESS_KEY_ID` | AWS access key | `AKIAIOSFODNN7EXAMPLE` |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| `MAILGUN_API_KEY` | Mailgun API key for email | `key-...` |
| `MAILGUN_DOMAIN` | Mailgun domain | `mg.example.com` |
| `EMAIL_HOST_USER` | Email sender address | `noreply@metaculus.com` |
| `EMAIL_NOTIFICATIONS_USER` | Email address for notifications | `notifications@metaculus.com` |
| `GUNICORN_WORKERS` | Number of Gunicorn workers | `4` |
| `NODE_INSTANCES` | Number of PM2 Node.js instances | `1` |
| `NODE_HEAP_SIZE` | Node.js heap size in MB | `1024` |
| `DRAMATIQ_PROCESSES` | Number of Dramatiq worker processes | `8` |
| `DRAMATIQ_THREADS` | Number of threads per Dramatiq process | `16` |
| `APP_DOMAIN` | Restrict Nginx to specific domain (optional) | `www.metaculus.com` |
| `SENTRY_DSN` | Sentry error tracking DSN (optional) | `https://...@sentry.io/...` |

### Setting Environment Variables

**Via Railway Dashboard:**
1. Go to your service
2. Click **"Variables"** tab
3. Click **"+ New Variable"**
4. Enter variable name and value
5. Click **"Add"**

**Via Railway CLI:**
```bash
railway variables set SECRET_KEY="your-secret-key-here"
railway variables set PUBLIC_APP_URL="https://your-app.railway.app"
# etc...
```

**Using Railway Reference Variables:**
Railway automatically provides reference variables for connected services:
- `DATABASE_URL`: `${{Postgres.DATABASE_URL}}`
- `REDIS_URL`: `${{Redis.REDIS_URL}}`

## Service Configuration

### Web Service

The web service runs both the Django backend and Next.js frontend behind an Nginx reverse proxy:

- **Build Target**: `web` (from Dockerfile)
- **Port**: `8080` (Nginx listens on this port)
- **Start Command**: `sh -c 'scripts/prod/startapp.sh'`
- **Health Check**: `/api/healthcheck/`
- **Resources**: Recommended 2GB RAM minimum, 2 vCPU

### Dramatiq Worker Service

Processes background jobs asynchronously:

- **Build Target**: `dramatiq_worker`
- **Start Command**: `sh -c 'scripts/prod/run_dramatiq.sh'`
- **No Port Exposed**: This is a background worker
- **Resources**: Recommended 1-2GB RAM, scales horizontally

### Django Cron Service

Runs scheduled tasks periodically:

- **Build Target**: `django_cron`
- **Start Command**: `sh -c 'scripts/prod/django_cron.sh'`
- **No Port Exposed**: This is a cron job runner
- **Resources**: Recommended 512MB-1GB RAM

## Database Setup

### PostgreSQL with pgvector

The Metaculus application requires the `pgvector` extension for vector similarity searches.

**Enable pgvector:**

```bash
# Option 1: Via Railway CLI
railway connect postgres

# In the psql shell:
CREATE EXTENSION IF NOT EXISTS vector;
\q

# Option 2: Via Railway Dashboard Query Editor
# Go to PostgreSQL service → "Query" tab
# Run: CREATE EXTENSION IF NOT EXISTS vector;
```

**Verify the extension:**
```sql
SELECT * FROM pg_extension WHERE extname = 'vector';
```

**Run migrations:**
```bash
railway run python manage.py migrate
```

### Database Backups

Railway automatically backs up PostgreSQL databases. You can also create manual backups:

```bash
# Create a backup
railway run pg_dump -Fc > backup.dump

# Restore from backup
railway run pg_restore -d $DATABASE_URL backup.dump
```

## Redis Setup

Railway provides Redis out of the box. No additional configuration is needed.

**Verify Redis connection:**
```bash
railway connect redis

# In redis-cli:
PING
# Should return: PONG
```

## Health Checks

Railway supports health checks to ensure your service is running properly:

**Configure in Railway Dashboard:**
1. Go to web service → **"Settings"**
2. Scroll to **"Health Check"**
3. Set:
   - **Path**: `/api/healthcheck/`
   - **Timeout**: `300` seconds
   - **Interval**: `30` seconds

**Note**: The Django application has a health check endpoint at `/api/healthcheck/` (defined in `utils/middlewares.py`).

## Scaling

### Vertical Scaling (More Resources)

1. Go to service → **"Settings"**
2. Adjust **"Resources"**:
   - CPU: 1-8 vCPUs
   - Memory: 512MB-32GB

### Horizontal Scaling (More Instances)

Railway Pro plan supports horizontal scaling:

1. Go to service → **"Settings"**
2. Set **"Replicas"**: 1-20 instances
3. Railway automatically load balances traffic

**Recommended scaling configuration:**
- **Web Service**: 2-4 replicas (depending on traffic)
- **Dramatiq Worker**: Scale horizontally based on queue size (1-10 workers)
- **Django Cron**: Keep at 1 replica (to avoid duplicate cron jobs)

## Troubleshooting

### Common Issues

#### 1. Build Fails with "Cannot find Docker target"

**Problem**: Railway can't find the specified Dockerfile target.

**Solution**: Ensure the service is configured with the correct target:
- Web: `web`
- Worker: `dramatiq_worker`
- Cron: `django_cron`

#### 2. Application Crashes on Startup

**Problem**: Missing environment variables or database connection issues.

**Solution**:
1. Check service logs: Railway Dashboard → Service → "Logs" tab
2. Verify all required environment variables are set
3. Ensure `DATABASE_URL` and `REDIS_URL` are correctly configured
4. Check that pgvector extension is enabled in PostgreSQL

#### 3. Static Files Not Loading

**Problem**: Static files (CSS, JS) return 404 errors.

**Solution**:
1. Ensure `collectstatic` is run during build (it's in the Dockerfile)
2. Check `AWS_*` environment variables if using S3 for static files
3. Verify Nginx configuration is correct

#### 4. Database Migration Errors

**Problem**: Migrations fail with "relation does not exist" or similar errors.

**Solution**:
1. Ensure pgvector extension is installed: `CREATE EXTENSION vector;`
2. Run migrations in order: `railway run python manage.py migrate`
3. Check PostgreSQL logs for specific errors

#### 5. Background Jobs Not Processing

**Problem**: Dramatiq worker service is not processing tasks.

**Solution**:
1. Check worker logs: Railway Dashboard → dramatiq-worker → "Logs"
2. Verify `REDIS_URL` is correctly set
3. Ensure worker service is running (not crashed)
4. Check Redis connection: `railway connect redis` → `PING`

#### 6. High Memory Usage

**Problem**: Service runs out of memory and crashes.

**Solution**:
1. Reduce `GUNICORN_WORKERS`, `NODE_INSTANCES`, or `DRAMATIQ_PROCESSES`
2. Increase service memory in Railway Settings → Resources
3. Check for memory leaks in application logs

#### 7. Slow Initial Deployment

**Problem**: First deployment takes a long time (10-20 minutes).

**Solution**:
- This is normal! The Dockerfile builds both Python and Node.js dependencies
- Subsequent deployments use cached layers and are much faster
- Consider using Railway's build cache to speed up builds

#### 8. Nginx Returns 502 Bad Gateway

**Problem**: Nginx can't connect to backend services.

**Solution**:
1. Check that Gunicorn and Next.js are starting correctly
2. Review startup script logs for errors
3. Verify port configuration (Gunicorn: 8000, Next.js: 3000, Nginx: 8080)
4. Ensure enough startup time (healthcheck timeout: 300s)

### Viewing Logs

**Via Dashboard:**
1. Go to Railway Dashboard
2. Click on the service
3. Click **"Logs"** tab
4. Filter by severity or search for specific errors

**Via CLI:**
```bash
# View logs for all services
railway logs

# View logs for specific service
railway logs --service web
```

### Debugging with Railway CLI

```bash
# Connect to service shell
railway shell

# Run commands in service context
railway run python manage.py shell

# Connect to database
railway connect postgres

# Connect to Redis
railway connect redis
```

### Getting Help

- **Railway Documentation**: [docs.railway.app](https://docs.railway.app/)
- **Railway Discord**: [discord.gg/railway](https://discord.gg/railway)
- **Metaculus GitHub Issues**: [github.com/Metaculus/metaculus/issues](https://github.com/Metaculus/metaculus/issues)

## Performance Tips

1. **Enable Railway's CDN**: For static assets, enable Railway's CDN in Settings
2. **Database Connection Pooling**: Consider using pgbouncer for connection pooling
3. **Redis Persistence**: Configure Redis persistence if needed for critical data
4. **Monitoring**: Set up Sentry or similar for error tracking
5. **Caching**: Ensure Django caching is properly configured with Redis

## Cost Estimation

Railway pricing is usage-based:

- **Hobby Plan** ($5/month): Good for development/testing
  - $5 included usage
  - Pay-as-you-go beyond that

- **Pro Plan** ($20/month): Recommended for production
  - $20 included usage
  - Horizontal scaling support
  - Team collaboration features

**Estimated monthly cost for production deployment:**
- Web service (2 replicas): ~$15-30
- Worker service: ~$10-20
- Cron service: ~$5-10
- PostgreSQL: ~$10-15
- Redis: ~$5-10
- **Total**: ~$45-85/month (varies with traffic and resource usage)

## Security Checklist

- [ ] Set `DEBUG=false` in production
- [ ] Use strong `SECRET_KEY` (minimum 50 characters)
- [ ] Configure `ALLOWED_HOSTS` properly
- [ ] Enable HTTPS (Railway provides SSL by default)
- [ ] Set up environment variable encryption
- [ ] Regularly update dependencies
- [ ] Enable database backups
- [ ] Set up monitoring and alerts
- [ ] Review Railway security best practices

## Next Steps

After successful deployment:

1. **Set up custom domain**: Railway Dashboard → Service → Settings → Domains
2. **Configure monitoring**: Integrate with Sentry, Datadog, or similar
3. **Set up CI/CD**: Configure automatic deployments on git push
4. **Load test**: Use k6 or similar to test under load
5. **Optimize resources**: Monitor usage and adjust resources accordingly
6. **Set up staging environment**: Create a separate Railway project for staging

---

**Note**: This guide assumes you're using the latest version of the Metaculus codebase. Some configuration details may vary based on your specific setup. Always refer to the main [README.md](../README.md) for the most up-to-date development setup instructions.
