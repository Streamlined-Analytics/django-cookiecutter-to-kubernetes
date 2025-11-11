# Kubernetes Deployment Guide for DigitalOcean (Managed Services)

This guide provides step-by-step instructions for deploying the Django application to DigitalOcean Kubernetes using **managed services** including:
- **DigitalOcean Managed PostgreSQL Database**
- **DigitalOcean Spaces (S3-compatible object storage) for media files**
- **DigitalOcean Load Balancer** (automatically provisioned)

This approach eliminates the ReadWriteOnce volume limitations and provides better scalability, reliability, and managed backups.

**Platform Support**: This guide includes instructions for Windows (PowerShell and Command Prompt), Linux, and macOS.

**Registry Requirements**: This deployment uses **4 container repositories** and works with DigitalOcean's Basic Container Registry plan (which includes 5 repositories).

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Cluster Setup](#cluster-setup)
3. [Managed Database Setup](#managed-database-setup)
4. [Spaces (Object Storage) Setup](#spaces-object-storage-setup)
5. [Container Registry Setup](#container-registry-setup)
6. [Build and Push Docker Images](#build-and-push-docker-images)
7. [Configure Environment Variables](#configure-environment-variables)
8. [Deploy to Kubernetes](#deploy-to-kubernetes)
9. [Configure Load Balancer](#configure-load-balancer)
10. [Verify Deployment](#verify-deployment)
11. [Access Your Application](#access-your-application)
12. [Monitoring and Maintenance](#monitoring-and-maintenance)
13. [Troubleshooting](#troubleshooting)
14. [Cleanup](#cleanup)

## Prerequisites

### Required Tools

1. **DigitalOcean Account**: Sign up at [DigitalOcean](https://www.digitalocean.com/)
2. **doctl**: DigitalOcean command-line tool (see main [DEPLOYMENT.md](DEPLOYMENT.md) for installation)
3. **kubectl**: Kubernetes command-line tool (see main [DEPLOYMENT.md](DEPLOYMENT.md) for installation)
4. **Docker**: For building container images (see main [DEPLOYMENT.md](DEPLOYMENT.md) for installation)

### Domain Name (Recommended)

For production use, you should have a domain name to configure with your load balancer.

## Cluster Setup

Follow the same cluster setup steps from the main [DEPLOYMENT.md](DEPLOYMENT.md#cluster-setup) guide.

Quick reference:
```bash
doctl kubernetes cluster create django-k8s-cluster \
  --region nyc1 \
  --version latest \
  --node-pool "name=worker-pool;size=s-2vcpu-2gb;count=2"

# Connect to cluster
doctl kubernetes cluster kubeconfig save django-k8s-cluster
```

## Managed Database Setup

### 1. Create Managed PostgreSQL Database

**Using Web Interface:**
1. Go to [DigitalOcean Databases](https://cloud.digitalocean.com/databases)
2. Click "Create Database"
3. Select PostgreSQL
4. Choose configuration:
   - **Database cluster name**: `django-postgres-db`
   - **Version**: PostgreSQL 15 or later
   - **Datacenter region**: Same as your cluster (e.g., NYC1)
   - **Node plan**: Basic (1GB RAM / 1 vCPU / 10GB disk is sufficient for testing)
   - **Number of nodes**: 1 (or 2+ for high availability)
5. Click "Create Database Cluster"

**Using CLI:**
```bash
doctl databases create django-postgres-db \
  --engine pg \
  --version 15 \
  --region nyc1 \
  --size db-s-1vcpu-1gb \
  --num-nodes 1
```

### 2. Configure Database

Wait for the database to be created (takes 5-10 minutes):

```bash
# Check status
doctl databases list
```

Once ready, get connection details:

```bash
# Get connection info
doctl databases connection django-postgres-db --format Host,Port,User,Password,Database

# Or view in web console: Databases > django-postgres-db > Connection Details
```

Save these values - you'll need them for configuring Kubernetes secrets:
- **Host**: (e.g., `django-postgres-db-do-user-xxxxx.db.ondigitalocean.com`)
- **Port**: `25060` (default for managed database)
- **User**: `doadmin`
- **Password**: (auto-generated)
- **Database**: `defaultdb` (we'll create our app database)

### 3. Create Application Database

**Using doctl:**
```bash
# Create the application database
doctl databases db create django-postgres-db kubernetes_test_v2
```

**Or using psql (if installed):**
```bash
# Get connection string
doctl databases connection django-postgres-db

# Connect and create database
psql "postgresql://doadmin:PASSWORD@HOST:25060/defaultdb?sslmode=require"
CREATE DATABASE kubernetes_test_v2;
\q
```

### 4. Configure Trusted Sources

Add your Kubernetes cluster to the database's trusted sources:

**Using Web Interface:**
1. Go to your database in the DigitalOcean console
2. Click "Settings" tab
3. Under "Trusted Sources", click "Edit"
4. Add your Kubernetes cluster
5. Save changes

**Using CLI:**
```bash
# Get your cluster's VPC network UUID
doctl kubernetes cluster get django-k8s-cluster --format VPCUUID

# Add cluster to trusted sources
doctl databases firewalls append django-postgres-db --rule k8s:YOUR_CLUSTER_UUID
```

## Spaces (Object Storage) Setup

### 1. Create DigitalOcean Space

**Using Web Interface:**
1. Go to [DigitalOcean Spaces](https://cloud.digitalocean.com/spaces)
2. Click "Create Space"
3. Configure:
   - **Datacenter region**: Same as cluster (e.g., NYC3)
   - **Enable CDN**: Yes (recommended for serving media)
   - **Space name**: `django-media-storage` (must be globally unique)
   - **Choose File Listing**: Restricted (recommended for security)
4. Click "Create Space"

**Using CLI:**
```bash
# Create Space (note: CDN must be enabled via web console)
doctl spaces create django-media-storage --region nyc3
```

### 2. Create API Keys for Spaces

**Using Web Interface:**
1. Go to API section in DigitalOcean console
2. Click "Spaces Keys" tab
3. Click "Generate New Key"
4. Name: `django-spaces-key`
5. Save the **Access Key** and **Secret Key** (shown only once!)

### 3. Configure Space Permissions (Optional)

For production, you may want to create a CORS policy:

1. Go to your Space in the console
2. Click "Settings"
3. Under "CORS Configurations", add:
   ```json
   [
     {
       "AllowedOrigins": ["https://yourdomain.com"],
       "AllowedMethods": ["GET", "HEAD"],
       "AllowedHeaders": ["*"],
       "MaxAgeSeconds": 3000
     }
   ]
   ```

## Container Registry Setup

Follow the same container registry setup from the main [DEPLOYMENT.md](DEPLOYMENT.md#container-registry-setup) guide.

Quick reference:
```bash
# Create registry
doctl registry create django-registry

# Authenticate
doctl registry login

# Link to cluster
doctl kubernetes cluster registry add django-k8s-cluster
```

## Build and Push Docker Images

Follow the same build and push process from the main [DEPLOYMENT.md](DEPLOYMENT.md#build-and-push-docker-images) guide, but you'll also need to install the `django-storages` package for Spaces integration.

### 1. Update Django Requirements

Before building, add `django-storages` and `boto3` to your requirements:

**Edit `requirements/production.txt`:**
```txt
# Add these lines
django-storages[s3]==1.14.2
boto3==1.34.17
```

### 2. Build and Push Images

**Linux/macOS:**
```bash
export REGISTRY_URL="registry.digitalocean.com/django-registry"

# Build Django image with updated requirements
docker build -f compose/production/django/Dockerfile -t ${REGISTRY_URL}/django:latest .
docker push ${REGISTRY_URL}/django:latest

# Build other images
docker build -f compose/production/nginx/Dockerfile -t ${REGISTRY_URL}/nginx:latest .
docker push ${REGISTRY_URL}/nginx:latest

docker build -f compose/production/traefik/Dockerfile -t ${REGISTRY_URL}/traefik:latest .
docker push ${REGISTRY_URL}/traefik:latest

# Note: We're not building postgres as we're using managed database
```

**Windows (PowerShell):**
```powershell
$env:REGISTRY_URL="registry.digitalocean.com/django-registry"

docker build -f compose/production/django/Dockerfile -t "$env:REGISTRY_URL/django:latest" .
docker push "$env:REGISTRY_URL/django:latest"

docker build -f compose/production/nginx/Dockerfile -t "$env:REGISTRY_URL/nginx:latest" .
docker push "$env:REGISTRY_URL/nginx:latest"

docker build -f compose/production/traefik/Dockerfile -t "$env:REGISTRY_URL/traefik:latest" .
docker push "$env:REGISTRY_URL/traefik:latest"
```

### 3. Update Deployment Manifests

Update the image references in your deployment files to use your registry URL:

```bash
cd k8s

# Linux/macOS
sed -i 's|image: kubernetes_test_v2_production_django|image: registry.digitalocean.com/django-registry/django:latest|g' django-deployment.yaml
sed -i 's|image: kubernetes_test_v2_production_celeryworker|image: registry.digitalocean.com/django-registry/django:latest|g' celeryworker-deployment.yaml
sed -i 's|image: kubernetes_test_v2_production_celerybeat|image: registry.digitalocean.com/django-registry/django:latest|g' celerybeat-deployment.yaml
sed -i 's|image: kubernetes_test_v2_production_flower|image: registry.digitalocean.com/django-registry/django:latest|g' flower-deployment.yaml
sed -i 's|image: kubernetes_test_v2_production_nginx|image: registry.digitalocean.com/django-registry/nginx:latest|g' nginx-deployment.yaml
sed -i 's|image: kubernetes_test_v2_production_traefik|image: registry.digitalocean.com/django-registry/traefik:latest|g' traefik-deployment.yaml
```

## Configure Environment Variables

### 1. Create Kubernetes Secrets

Create secrets for database, Django, and Spaces credentials.

**Important**: Use the connection details from your managed database and Spaces API keys.

**Linux/macOS:**
```bash
# Database credentials from managed PostgreSQL
export DB_HOST="django-postgres-db-do-user-xxxxx.db.ondigitalocean.com"
export DB_PORT="25060"
export DB_NAME="kubernetes_test_v2"
export DB_USER="doadmin"
export DB_PASSWORD="your-database-password-from-digitalocean"

# Generate Django secrets
export DJANGO_SECRET_KEY=$(openssl rand -base64 64)
export CELERY_FLOWER_PASSWORD=$(openssl rand -base64 32)

# Spaces credentials (from API keys)
export SPACES_ACCESS_KEY="your-spaces-access-key"
export SPACES_SECRET_KEY="your-spaces-secret-key"

# Create database secret
kubectl create secret generic postgres-secrets \
  --from-literal=POSTGRES_DB=${DB_NAME} \
  --from-literal=POSTGRES_USER=${DB_USER} \
  --from-literal=POSTGRES_PASSWORD=${DB_PASSWORD} \
  --from-literal=POSTGRES_HOST=${DB_HOST} \
  --from-literal=POSTGRES_PORT=${DB_PORT}

# Create Django secret
kubectl create secret generic django-secrets \
  --from-literal=DJANGO_SECRET_KEY=${DJANGO_SECRET_KEY} \
  --from-literal=CELERY_FLOWER_PASSWORD=${CELERY_FLOWER_PASSWORD}

# Create Spaces secret
kubectl create secret generic spaces-secrets \
  --from-literal=AWS_ACCESS_KEY_ID=${SPACES_ACCESS_KEY} \
  --from-literal=AWS_SECRET_ACCESS_KEY=${SPACES_SECRET_KEY}
```

**Windows (PowerShell):**
```powershell
# Database credentials from managed PostgreSQL
$env:DB_HOST="django-postgres-db-do-user-xxxxx.db.ondigitalocean.com"
$env:DB_PORT="25060"
$env:DB_NAME="kubernetes_test_v2"
$env:DB_USER="doadmin"
$env:DB_PASSWORD="your-database-password-from-digitalocean"

# Generate Django secrets
$env:DJANGO_SECRET_KEY = (openssl rand -base64 64)
$env:CELERY_FLOWER_PASSWORD = (openssl rand -base64 32)

# Spaces credentials
$env:SPACES_ACCESS_KEY="your-spaces-access-key"
$env:SPACES_SECRET_KEY="your-spaces-secret-key"

# Create database secret
kubectl create secret generic postgres-secrets `
  --from-literal=POSTGRES_DB=$env:DB_NAME `
  --from-literal=POSTGRES_USER=$env:DB_USER `
  --from-literal=POSTGRES_PASSWORD=$env:DB_PASSWORD `
  --from-literal=POSTGRES_HOST=$env:DB_HOST `
  --from-literal=POSTGRES_PORT=$env:DB_PORT

# Create Django secret
kubectl create secret generic django-secrets `
  --from-literal=DJANGO_SECRET_KEY=$env:DJANGO_SECRET_KEY `
  --from-literal=CELERY_FLOWER_PASSWORD=$env:CELERY_FLOWER_PASSWORD

# Create Spaces secret
kubectl create secret generic spaces-secrets `
  --from-literal=AWS_ACCESS_KEY_ID=$env:SPACES_ACCESS_KEY `
  --from-literal=AWS_SECRET_ACCESS_KEY=$env:SPACES_SECRET_KEY
```

### 2. Create ConfigMaps

Create a ConfigMap for Django with Spaces configuration:

**Create `k8s/envs--production--django-managed-configmap.yaml`:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envs--production--django
data:
  DJANGO_SETTINGS_MODULE: config.settings.production
  DJANGO_ALLOWED_HOSTS: ".yourdomain.com"  # Update with your domain
  DJANGO_ADMIN_URL: "admin/"  # Change for security
  DJANGO_ACCOUNT_ALLOW_REGISTRATION: "False"
  DJANGO_SECURE_SSL_REDIRECT: "True"
  WEB_CONCURRENCY: "4"
  CELERY_FLOWER_USER: "admin"
  
  # Redis connection
  REDIS_URL: "redis://redis:6379/0"
  
  # Database URL (will use managed database credentials from secret)
  # Format: postgres://USER:PASSWORD@HOST:PORT/DATABASE
  DATABASE_URL: "postgres://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@$(POSTGRES_HOST):$(POSTGRES_PORT)/$(POSTGRES_DB)?sslmode=require"
  
  # DigitalOcean Spaces configuration
  USE_SPACES: "true"
  AWS_STORAGE_BUCKET_NAME: "django-media-storage"  # Your Space name
  AWS_S3_REGION_NAME: "nyc3"  # Your Space region
  AWS_S3_ENDPOINT_URL: "https://nyc3.digitaloceanspaces.com"  # Region endpoint
  AWS_S3_CUSTOM_DOMAIN: "django-media-storage.nyc3.cdn.digitaloceanspaces.com"  # CDN endpoint
  AWS_DEFAULT_ACL: "public-read"
  AWS_S3_OBJECT_PARAMETERS: "CacheControl:max-age=86400"
  
  # Email configuration (optional)
  # MAILGUN_API_KEY: ""
  # MAILGUN_DOMAIN: ""
  # DJANGO_SERVER_EMAIL: ""
  
  # Sentry (optional)
  # SENTRY_DSN: ""
```

Apply the ConfigMap:
```bash
kubectl apply -f k8s/envs--production--django-managed-configmap.yaml
```

### 3. Update Deployments to Use Secrets

Update your Django deployment to include all three secrets:

**Edit `k8s/django-deployment.yaml`:**

```yaml
spec:
  containers:
    - name: django
      envFrom:
        - configMapRef:
            name: envs--production--django
        - secretRef:
            name: postgres-secrets
        - secretRef:
            name: django-secrets
        - secretRef:
            name: spaces-secrets
      # Remove volumeMounts for media since we're using Spaces
      # volumeMounts:
      #   - mountPath: /app/kubernetes_test_v2/media
      #     name: production-django-media
  # Remove volumes section for media
  # volumes:
  #   - name: production-django-media
  #     persistentVolumeClaim:
  #       claimName: production-django-media
```

Apply the same pattern to `celeryworker-deployment.yaml`, `celerybeat-deployment.yaml`, and `flower-deployment.yaml`.

**For Nginx**, since media is served from Spaces/CDN, you can either:
- Remove the nginx deployment entirely (Spaces CDN serves files)
- Or keep it for serving static files only

## Deploy to Kubernetes

### 1. Apply Persistent Volume Claims

We only need volumes for Redis and Traefik (no media volume needed):

```bash
kubectl apply -f k8s/production-redis-data-persistentvolumeclaim.yaml
kubectl apply -f k8s/production-traefik-persistentvolumeclaim.yaml
```

**Note**: No postgres or media volumes needed since we're using managed services!

### 2. Apply ConfigMaps

```bash
kubectl apply -f k8s/envs--production--django-managed-configmap.yaml
```

### 3. Deploy Redis

```bash
kubectl apply -f k8s/redis-deployment.yaml
kubectl wait --for=condition=available --timeout=300s deployment/redis
```

### 4. Run Database Migrations

Deploy Django and run migrations against the managed database:

**Linux/macOS:**
```bash
kubectl apply -f k8s/django-deployment.yaml
kubectl wait --for=condition=ready --timeout=300s pod -l io.kompose.service=django

# Get Django pod name
DJANGO_POD=$(kubectl get pod -l io.kompose.service=django -o jsonpath="{.items[0].metadata.name}")

# Run migrations
kubectl exec -it ${DJANGO_POD} -- python manage.py migrate

# Create superuser
kubectl exec -it ${DJANGO_POD} -- python manage.py createsuperuser

# Collect static files (if not using Spaces for static)
kubectl exec -it ${DJANGO_POD} -- python manage.py collectstatic --noinput
```

**Windows (PowerShell):**
```powershell
kubectl apply -f k8s/django-deployment.yaml
kubectl wait --for=condition=ready --timeout=300s pod -l io.kompose.service=django

$DJANGO_POD = kubectl get pod -l io.kompose.service=django -o jsonpath="{.items[0].metadata.name}"

kubectl exec -it $DJANGO_POD -- python manage.py migrate
kubectl exec -it $DJANGO_POD -- python manage.py createsuperuser
kubectl exec -it $DJANGO_POD -- python manage.py collectstatic --noinput
```

### 5. Deploy Celery Services

```bash
kubectl apply -f k8s/celeryworker-deployment.yaml
kubectl apply -f k8s/celerybeat-deployment.yaml
kubectl apply -f k8s/flower-deployment.yaml
```

### 6. Create Kubernetes Services

**Important**: Traefik needs Kubernetes Services to route traffic to the pods. Create services for Django and Flower:

```bash
kubectl apply -f k8s/django-service.yaml
kubectl apply -f k8s/flower-service.yaml
# Note: nginx-service.yaml is only needed if you're using Nginx for media files
# kubectl apply -f k8s/nginx-service.yaml
```

These services allow Traefik to discover and route traffic to your application pods.

### 7. Deploy Traefik (Load Balancer)

```bash
kubectl apply -f k8s/traefik-deployment.yaml
kubectl apply -f k8s/traefik-service.yaml
```

## Configure Load Balancer

### 1. Expose Traefik via LoadBalancer

Patch the Traefik service to use LoadBalancer type:

**Linux/macOS:**
```bash
kubectl patch service traefik -p '{"spec": {"type": "LoadBalancer"}}'
```

**Windows (PowerShell):**
```powershell
kubectl patch service traefik -p '{\"spec\": {\"type\": \"LoadBalancer\"}}'
```

**Windows (Command Prompt):**
```cmd
kubectl patch service traefik -p "{\"spec\": {\"type\": \"LoadBalancer\"}}"
```

This will automatically provision a DigitalOcean Load Balancer.

### 2. Get Load Balancer IP

Wait for the external IP to be assigned (takes 2-3 minutes):

```bash
kubectl get service traefik -w
```

Once you see an `EXTERNAL-IP`, note it down:

```bash
kubectl get service traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

### 3. Configure DNS

Point your domain to the Load Balancer IP:

1. Go to your DNS provider
2. Create an A record:
   - **Name**: `@` (for root domain) or subdomain
   - **Value**: Load Balancer IP
   - **TTL**: 300 (or default)

Example: `yourdomain.com` → `167.99.123.45`

### 4. Configure HTTPS (Optional but Recommended)

Update your Django ConfigMap to enable SSL redirect:

```yaml
DJANGO_SECURE_SSL_REDIRECT: "True"
```

Apply changes:
```bash
kubectl apply -f k8s/envs--production--django-managed-configmap.yaml
kubectl rollout restart deployment/django
```

Traefik will automatically obtain Let's Encrypt certificates for your domain.

## Verify Deployment

### 1. Check All Pods

```bash
kubectl get pods
```

All pods should show `Running` status:
- django
- celeryworker
- celerybeat  
- flower
- redis
- traefik

### 2. Check Services

```bash
kubectl get services
```

The `traefik` service should have an `EXTERNAL-IP` (Load Balancer IP).

### 3. Test Database Connection

```bash
# Get Django pod name
DJANGO_POD=$(kubectl get pod -l io.kompose.service=django -o jsonpath="{.items[0].metadata.name}")

# Test database connection
kubectl exec -it ${DJANGO_POD} -- python manage.py dbshell
```

### 4. Test Spaces Connection

Upload a test file through Django admin to verify Spaces integration is working.

### 5. View Logs

```bash
kubectl logs -f deployment/django
kubectl logs -f deployment/celeryworker
```

## Access Your Application

### Application URLs

- **Main Application**: `https://yourdomain.com`
- **Admin Interface**: `https://yourdomain.com/admin/`
- **Flower (Celery)**: `https://yourdomain.com:5555`
- **Media Files**: Served from `https://django-media-storage.nyc3.cdn.digitaloceanspaces.com/`

### Managed Services Console

Monitor your managed services in DigitalOcean:
- **Database**: Monitor queries, connections, CPU/memory usage
- **Spaces**: View storage usage, bandwidth
- **Load Balancer**: Monitor traffic, health checks

## Monitoring and Maintenance

### Database Backups

DigitalOcean Managed Databases include automatic daily backups:

1. Go to your database in the console
2. Click "Backups" tab
3. View available backups
4. Restore from backup if needed

### Manual Database Backup

```bash
# Get database connection string
doctl databases connection django-postgres-db

# Create backup
pg_dump "postgresql://doadmin:PASSWORD@HOST:25060/kubernetes_test_v2?sslmode=require" > backup.sql
```

### Spaces Backup

Spaces data is automatically replicated across DigitalOcean's infrastructure. For additional backup:

1. Use `s3cmd` or AWS CLI to sync Spaces to local storage
2. Configure lifecycle policies in Spaces settings

### Scaling

**Scale Django workers:**
```bash
kubectl scale deployment django --replicas=3
```

**Scale Celery workers:**
```bash
kubectl scale deployment celeryworker --replicas=2
```

**Scale Database:**
- Go to database in console → Settings → Resize
- Choose larger node size or add read replicas

### Update Application

```bash
# Build new image
docker build -f compose/production/django/Dockerfile -t registry.digitalocean.com/django-registry/django:v2 .
docker push registry.digitalocean.com/django-registry/django:v2

# Update deployment
kubectl set image deployment/django django=registry.digitalocean.com/django-registry/django:v2

# Or edit deployment file and apply
kubectl apply -f k8s/django-deployment.yaml
```

## Troubleshooting

### Bad Gateway (502) Errors

If you're getting "Bad Gateway" errors when accessing your domain:

**1. Check that Kubernetes Services exist:**
```bash
kubectl get services
```

You should see services for: `django`, `flower`, `traefik`, and `redis`.

**Missing Services?** Apply them:
```bash
kubectl apply -f k8s/django-service.yaml
kubectl apply -f k8s/flower-service.yaml
```

**2. Verify Traefik can reach Django:**
```bash
# Get Traefik pod
TRAEFIK_POD=$(kubectl get pod -l io.kompose.service=traefik -o jsonpath="{.items[0].metadata.name}")

# Check if Django service resolves
kubectl exec -it ${TRAEFIK_POD} -- nslookup django

# Test connection to Django
kubectl exec -it ${TRAEFIK_POD} -- wget -O- http://django:5000 2>&1 | head
```

**3. Check Django is listening on port 5000:**
```bash
DJANGO_POD=$(kubectl get pod -l io.kompose.service=django -o jsonpath="{.items[0].metadata.name}")
kubectl exec -it ${DJANGO_POD} -- netstat -tlnp | grep 5000
```

**4. Verify Traefik configuration:**
```bash
kubectl logs deployment/traefik | grep -i error
```

**5. Check ALLOWED_HOSTS:**
Ensure your domain is in ALLOWED_HOSTS. Update ConfigMap:
```yaml
DJANGO_ALLOWED_HOSTS: ".yourdomain.com"  # or "*" for testing
```

Then restart Django:
```bash
kubectl rollout restart deployment/django
```

**6. Check Traefik routing rules:**
The Traefik configuration expects:
- Django service at `http://django:5000`
- Flower service at `http://flower:5555`
- These must match your Kubernetes Service definitions

### Database Connection Issues

**Check connection from pod:**
```bash
DJANGO_POD=$(kubectl get pod -l io.kompose.service=django -o jsonpath="{.items[0].metadata.name}")
kubectl exec -it ${DJANGO_POD} -- python manage.py check --database default
```

**Verify database is in trusted sources:**
```bash
doctl databases firewalls list django-postgres-db
```

### Spaces Upload Failures

**Check Spaces secrets:**
```bash
kubectl get secret spaces-secrets -o yaml
```

**Test Spaces connection:**
```bash
# Install AWS CLI or s3cmd
aws configure set aws_access_key_id YOUR_KEY
aws configure set aws_secret_access_key YOUR_SECRET

# Test connection
aws s3 ls s3://django-media-storage --endpoint-url=https://nyc3.digitaloceanspaces.com
```

### Load Balancer Not Getting External IP

```bash
# Check service status
kubectl describe service traefik

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp
```

Common issues:
- Cluster not properly configured
- Resource quota exceeded
- Region-specific issues (try different region)

### Pod Crashes

```bash
# View pod logs
kubectl logs -f POD_NAME

# Describe pod for events
kubectl describe pod POD_NAME

# Check resource usage
kubectl top pods
```

## Cleanup

### Delete Kubernetes Resources

```bash
# Delete all deployments
kubectl delete -f k8s/

# Delete secrets
kubectl delete secret postgres-secrets django-secrets spaces-secrets

# Delete PVCs
kubectl delete pvc production-redis-data production-traefik
```

### Delete Managed Services

**Database:**
```bash
doctl databases delete django-postgres-db
```

Or via web console: Databases → django-postgres-db → Settings → Destroy

**Spaces:**
```bash
# List and delete objects first
doctl spaces ls django-media-storage

# Delete Space
doctl spaces delete django-media-storage
```

Or via web console: Spaces → django-media-storage → Settings → Destroy

**Load Balancer:**

Load Balancer is automatically deleted when you delete the Kubernetes service:
```bash
kubectl delete service traefik
```

Or delete manually via console: Networking → Load Balancers

### Delete Cluster

```bash
doctl kubernetes cluster delete django-k8s-cluster
```

### Delete Container Registry

```bash
doctl registry delete django-registry
```

## Additional Resources

- [DigitalOcean Kubernetes Documentation](https://docs.digitalocean.com/products/kubernetes/)
- [DigitalOcean Managed Databases](https://docs.digitalocean.com/products/databases/)
- [DigitalOcean Spaces Documentation](https://docs.digitalocean.com/products/spaces/)
- [Django Storages Documentation](https://django-storages.readthedocs.io/)
- [Django Deployment Checklist](https://docs.djangoproject.com/en/stable/howto/deployment/checklist/)

## Benefits of Managed Services Approach

✅ **No volume conflicts** - Media stored in Spaces (S3-compatible)
✅ **Better scalability** - Scale pods independently without storage constraints
✅ **Managed backups** - Automatic daily database backups
✅ **High availability** - Database replication and failover
✅ **CDN integration** - Fast media delivery via DigitalOcean CDN
✅ **Simplified operations** - Less infrastructure to manage
✅ **Cost effective** - Pay for what you use, scale as needed

---

For the basic deployment without managed services, see [DEPLOYMENT.md](DEPLOYMENT.md).
