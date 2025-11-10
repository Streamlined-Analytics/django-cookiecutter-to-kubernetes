# Kubernetes Deployment Guide for DigitalOcean

This guide provides step-by-step instructions for deploying the Django application to a DigitalOcean Kubernetes cluster using the manifests in this directory.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Cluster Setup](#cluster-setup)
3. [Container Registry Setup](#container-registry-setup)
4. [Build and Push Docker Images](#build-and-push-docker-images)
5. [Configure Environment Variables](#configure-environment-variables)
6. [Deploy to Kubernetes](#deploy-to-kubernetes)
7. [Verify Deployment](#verify-deployment)
8. [Access Your Application](#access-your-application)
9. [Troubleshooting](#troubleshooting)
10. [Cleanup](#cleanup)

## Prerequisites

Before you begin, ensure you have the following:

### Required Tools

1. **DigitalOcean Account**: Sign up at [DigitalOcean](https://www.digitalocean.com/)
2. **doctl**: DigitalOcean command-line tool
   ```bash
   # macOS
   brew install doctl
   
   # Linux
   cd ~
   wget https://github.com/digitalocean/doctl/releases/download/v1.98.1/doctl-1.98.1-linux-amd64.tar.gz
   tar xf doctl-*.tar.gz
   sudo mv doctl /usr/local/bin
   
   # Authenticate
   doctl auth init
   ```

3. **kubectl**: Kubernetes command-line tool
   ```bash
   # macOS
   brew install kubectl
   
   # Linux
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
   ```

4. **Docker**: For building container images
   ```bash
   # Follow instructions at https://docs.docker.com/get-docker/
   ```

### Domain Name (Optional but Recommended)

For production use, you should have a domain name configured to point to your cluster's load balancer.

## Cluster Setup

### 1. Create a Kubernetes Cluster

Create a cluster using the DigitalOcean web interface or CLI:

**Using Web Interface:**
1. Go to [DigitalOcean Dashboard](https://cloud.digitalocean.com/)
2. Navigate to "Kubernetes" in the left sidebar
3. Click "Create Kubernetes Cluster"
4. Configure:
   - Choose a datacenter region
   - Select Kubernetes version (latest stable recommended)
   - Choose node pool: at least 2 nodes with 2GB RAM each (e.g., Basic $12/month nodes)
   - Name your cluster (e.g., `django-k8s-cluster`)
5. Click "Create Cluster"

**Using CLI:**
```bash
doctl kubernetes cluster create django-k8s-cluster \
  --region nyc1 \
  --version latest \
  --node-pool "name=worker-pool;size=s-2vcpu-2gb;count=2"
```

### 2. Connect to Your Cluster

Download the kubeconfig file:

```bash
# List your clusters
doctl kubernetes cluster list

# Download kubeconfig
doctl kubernetes cluster kubeconfig save django-k8s-cluster

# Verify connection
kubectl get nodes
```

You should see your cluster nodes listed.

## Container Registry Setup

You need a container registry to store your Docker images. DigitalOcean provides a container registry service.

### 1. Create a Container Registry

**Using Web Interface:**
1. Go to "Container Registry" in the DigitalOcean dashboard
2. Click "Create Registry"
3. Choose a name (e.g., `django-registry`)
4. Select a subscription plan (Basic/Starter is usually sufficient)

**Using CLI:**
```bash
doctl registry create django-registry
```

### 2. Authenticate Docker with the Registry

```bash
doctl registry login
```

### 3. Link Registry to Your Cluster

This allows your cluster to pull images from your private registry:

```bash
doctl kubernetes cluster registry add django-k8s-cluster
```

## Build and Push Docker Images

### 1. Set Your Registry URL

```bash
export REGISTRY_URL="registry.digitalocean.com/django-registry"
```

### 2. Build Docker Images

Navigate to the project root and build all production images:

```bash
# Build Django image
docker build -f compose/production/django/Dockerfile -t ${REGISTRY_URL}/kubernetes_test_v2_production_django:latest .

# Build Postgres image
docker build -f compose/production/postgres/Dockerfile -t ${REGISTRY_URL}/kubernetes_test_v2_production_postgres:latest .

# Build Traefik image
docker build -f compose/production/traefik/Dockerfile -t ${REGISTRY_URL}/kubernetes_test_v2_production_traefik:latest .

# Build Nginx image
docker build -f compose/production/nginx/Dockerfile -t ${REGISTRY_URL}/kubernetes_test_v2_production_nginx:latest .

# Celery services use the same Django image, so tag them
docker tag ${REGISTRY_URL}/kubernetes_test_v2_production_django:latest ${REGISTRY_URL}/kubernetes_test_v2_production_celeryworker:latest
docker tag ${REGISTRY_URL}/kubernetes_test_v2_production_django:latest ${REGISTRY_URL}/kubernetes_test_v2_production_celerybeat:latest
docker tag ${REGISTRY_URL}/kubernetes_test_v2_production_django:latest ${REGISTRY_URL}/kubernetes_test_v2_production_flower:latest
```

### 3. Push Images to Registry

```bash
docker push ${REGISTRY_URL}/kubernetes_test_v2_production_django:latest
docker push ${REGISTRY_URL}/kubernetes_test_v2_production_postgres:latest
docker push ${REGISTRY_URL}/kubernetes_test_v2_production_traefik:latest
docker push ${REGISTRY_URL}/kubernetes_test_v2_production_nginx:latest
docker push ${REGISTRY_URL}/kubernetes_test_v2_production_celeryworker:latest
docker push ${REGISTRY_URL}/kubernetes_test_v2_production_celerybeat:latest
docker push ${REGISTRY_URL}/kubernetes_test_v2_production_flower:latest
```

### 4. Update Kubernetes Manifests

Update the image references in your deployment files to use your registry URL. You need to update these files:

- `django-deployment.yaml`
- `postgres-deployment.yaml`
- `traefik-deployment.yaml`
- `nginx-deployment.yaml`
- `celeryworker-deployment.yaml`
- `celerybeat-deployment.yaml`
- `flower-deployment.yaml`

Replace `image: kubernetes_test_v2_production_*` with `image: registry.digitalocean.com/django-registry/kubernetes_test_v2_production_*:latest`

**Example using sed:**
```bash
cd k8s
sed -i 's|image: kubernetes_test_v2_production_|image: registry.digitalocean.com/django-registry/kubernetes_test_v2_production_|g' *-deployment.yaml
```

## Configure Environment Variables

### 1. Create Kubernetes Secrets

Instead of using ConfigMaps for sensitive data, create Kubernetes Secrets:

```bash
# Generate secure passwords and keys
export POSTGRES_PASSWORD=$(openssl rand -base64 32)
export DJANGO_SECRET_KEY=$(openssl rand -base64 64)
export CELERY_FLOWER_PASSWORD=$(openssl rand -base64 32)

# Create PostgreSQL secret
kubectl create secret generic postgres-secrets \
  --from-literal=POSTGRES_DB=kubernetes_test_v2 \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=${POSTGRES_PASSWORD} \
  --from-literal=POSTGRES_HOST=postgres \
  --from-literal=POSTGRES_PORT=5432

# Create Django secret
kubectl create secret generic django-secrets \
  --from-literal=DJANGO_SECRET_KEY=${DJANGO_SECRET_KEY} \
  --from-literal=CELERY_FLOWER_PASSWORD=${CELERY_FLOWER_PASSWORD}
```

### 2. Update ConfigMaps

Edit the ConfigMap files to set your configuration:

**Edit `envs--production--django-configmap.yaml`:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envs--production--django
data:
  DJANGO_SETTINGS_MODULE: config.settings.production
  DJANGO_ALLOWED_HOSTS: ".yourdomain.com"  # Update with your domain
  DJANGO_ADMIN_URL: "admin/"  # Change this for security
  DJANGO_ACCOUNT_ALLOW_REGISTRATION: "False"  # Set to True if you want open registration
  DJANGO_SECURE_SSL_REDIRECT: "True"  # Set to True for HTTPS
  REDIS_URL: "redis://redis:6379/0"
  WEB_CONCURRENCY: "4"
  CELERY_FLOWER_USER: "admin"  # Change this
  # Optional: Configure email (Mailgun example)
  # MAILGUN_API_KEY: "your-mailgun-api-key"
  # MAILGUN_DOMAIN: "your-mailgun-domain"
  # DJANGO_SERVER_EMAIL: "noreply@yourdomain.com"
  # Optional: Configure Sentry for error tracking
  # SENTRY_DSN: "your-sentry-dsn"
```

**Edit `envs--production--postgres-configmap.yaml`:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envs--production--postgres
data:
  POSTGRES_HOST: postgres
  POSTGRES_PORT: "5432"
  POSTGRES_DB: kubernetes_test_v2
  POSTGRES_USER: postgres
```

**Note**: The actual passwords will come from the secrets created above.

### 3. Update Deployments to Use Secrets

You'll need to modify the deployment files to use both ConfigMaps and Secrets. Edit each deployment file that has `envFrom` to include both:

**Example for `django-deployment.yaml`:**
```yaml
spec:
  containers:
    - name: django
      envFrom:
        - configMapRef:
            name: envs--production--django
        - secretRef:
            name: django-secrets
        - secretRef:
            name: postgres-secrets
```

Apply this pattern to:
- `django-deployment.yaml`
- `celeryworker-deployment.yaml`
- `celerybeat-deployment.yaml`
- `flower-deployment.yaml`

For `postgres-deployment.yaml`:
```yaml
spec:
  containers:
    - name: postgres
      envFrom:
        - secretRef:
            name: postgres-secrets
```

## Deploy to Kubernetes

### 1. Apply Persistent Volume Claims

Create persistent storage for your data:

```bash
kubectl apply -f production-django-media-persistentvolumeclaim.yaml
kubectl apply -f production-postgres-data-persistentvolumeclaim.yaml
kubectl apply -f production-postgres-data-backups-persistentvolumeclaim.yaml
kubectl apply -f production-redis-data-persistentvolumeclaim.yaml
kubectl apply -f production-traefik-persistentvolumeclaim.yaml
```

### 2. Apply ConfigMaps

```bash
kubectl apply -f envs--production--django-configmap.yaml
kubectl apply -f envs--production--postgres-configmap.yaml
```

### 3. Deploy Database and Redis

Deploy PostgreSQL and Redis first:

```bash
kubectl apply -f postgres-deployment.yaml
kubectl apply -f redis-deployment.yaml
```

Wait for these to be ready:

```bash
kubectl wait --for=condition=available --timeout=300s deployment/postgres
kubectl wait --for=condition=available --timeout=300s deployment/redis
```

### 4. Run Database Migrations

Before deploying the Django app, run migrations:

```bash
# Get the postgres pod name
POSTGRES_POD=$(kubectl get pod -l io.kompose.service=postgres -o jsonpath="{.items[0].metadata.name}")

# First deploy Django temporarily to run migrations
kubectl apply -f django-deployment.yaml

# Wait for Django pod to be ready
kubectl wait --for=condition=ready --timeout=300s pod -l io.kompose.service=django

# Get Django pod name
DJANGO_POD=$(kubectl get pod -l io.kompose.service=django -o jsonpath="{.items[0].metadata.name}")

# Run migrations
kubectl exec -it ${DJANGO_POD} -- python manage.py migrate

# Create superuser (interactive)
kubectl exec -it ${DJANGO_POD} -- python manage.py createsuperuser

# Collect static files
kubectl exec -it ${DJANGO_POD} -- python manage.py collectstatic --noinput
```

### 5. Deploy All Services

Deploy the remaining services:

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f celeryworker-deployment.yaml
kubectl apply -f celerybeat-deployment.yaml
kubectl apply -f flower-deployment.yaml
kubectl apply -f traefik-deployment.yaml
kubectl apply -f traefik-service.yaml
```

### 6. Expose Your Application

Create a LoadBalancer service to expose Traefik:

```bash
kubectl patch service traefik -p '{"spec": {"type": "LoadBalancer"}}'
```

Or create a new service file `traefik-loadbalancer.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik-lb
spec:
  type: LoadBalancer
  selector:
    io.kompose.service: traefik
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: https
      port: 443
      targetPort: 443
```

Apply it:
```bash
kubectl apply -f traefik-loadbalancer.yaml
```

## Verify Deployment

### 1. Check Pod Status

```bash
kubectl get pods
```

All pods should show `Running` status.

### 2. Check Services

```bash
kubectl get services
```

Look for the `EXTERNAL-IP` of your LoadBalancer service.

### 3. Check Logs

If any pod is not running correctly:

```bash
kubectl logs <pod-name>
kubectl describe pod <pod-name>
```

### 4. View Application Logs

```bash
# Django logs
kubectl logs -f deployment/django

# Celery worker logs
kubectl logs -f deployment/celeryworker

# Postgres logs
kubectl logs -f deployment/postgres
```

## Access Your Application

### 1. Get Load Balancer IP

```bash
kubectl get service traefik-lb
```

Or if you patched the existing service:
```bash
kubectl get service traefik
```

Note the `EXTERNAL-IP` value.

### 2. Configure DNS

If you have a domain name:

1. Go to your DNS provider
2. Create an A record pointing to the LoadBalancer IP
3. Example: `your-app.yourdomain.com` → `EXTERNAL-IP`

### 3. Access the Application

- **Web Application**: `http://your-app.yourdomain.com` (or `http://EXTERNAL-IP`)
- **Admin Interface**: `http://your-app.yourdomain.com/admin/`
- **Flower (Celery monitoring)**: `http://your-app.yourdomain.com:5555`

### 4. Configure HTTPS (Optional but Recommended)

Traefik can automatically obtain Let's Encrypt SSL certificates. Ensure your domain is properly configured, then Traefik will handle certificate generation.

Update `envs--production--django-configmap.yaml`:
```yaml
DJANGO_SECURE_SSL_REDIRECT: "True"
```

Then reapply:
```bash
kubectl apply -f envs--production--django-configmap.yaml
kubectl rollout restart deployment/django
```

## Troubleshooting

### Pods Not Starting

```bash
# Check pod status
kubectl get pods

# Describe pod for events
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>
```

### Image Pull Errors

If you see `ImagePullBackOff`:

1. Verify your registry is linked to the cluster:
   ```bash
   doctl kubernetes cluster registry add django-k8s-cluster
   ```

2. Check image names in deployment files match your registry

3. Verify images exist in your registry:
   ```bash
   doctl registry repository list-v2
   ```

### Database Connection Issues

```bash
# Check if postgres is running
kubectl get pods -l io.kompose.service=postgres

# Check postgres logs
kubectl logs deployment/postgres

# Verify database credentials
kubectl get secret postgres-secrets -o yaml
```

### Persistent Volume Issues

```bash
# Check PVCs
kubectl get pvc

# Describe PVC for issues
kubectl describe pvc production-postgres-data
```

### Service Not Accessible

```bash
# Check service status
kubectl get services

# Check if LoadBalancer has external IP
kubectl get service traefik-lb

# Check traefik logs
kubectl logs deployment/traefik
```

### Reset Database

If you need to reset the database:

```bash
# Delete postgres deployment
kubectl delete deployment postgres

# Delete PVC (WARNING: This deletes all data)
kubectl delete pvc production-postgres-data

# Recreate PVC and deployment
kubectl apply -f production-postgres-data-persistentvolumeclaim.yaml
kubectl apply -f postgres-deployment.yaml

# Re-run migrations
```

## Scaling

### Scale Deployments

```bash
# Scale Django workers
kubectl scale deployment django --replicas=3

# Scale Celery workers
kubectl scale deployment celeryworker --replicas=2

# Scale Nginx
kubectl scale deployment nginx --replicas=2
```

### Check Resource Usage

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods
```

## Monitoring and Maintenance

### View All Resources

```bash
kubectl get all
```

### Update Application

After making code changes:

1. Build and push new images with a new tag (e.g., `v2`)
2. Update deployment files with new tag
3. Apply changes:
   ```bash
   kubectl apply -f django-deployment.yaml
   ```

Or use rolling update:
```bash
kubectl set image deployment/django django=registry.digitalocean.com/django-registry/kubernetes_test_v2_production_django:v2
```

### Backup Database

```bash
# Execute backup inside postgres container
kubectl exec deployment/postgres -- pg_dump -U postgres kubernetes_test_v2 > backup.sql

# Or use the backup volume
kubectl exec deployment/postgres -- pg_dump -U postgres kubernetes_test_v2 > /backups/backup-$(date +%Y%m%d).sql
```

## Cleanup

### Delete All Resources

To remove everything from your cluster:

```bash
# Delete all deployments
kubectl delete -f .

# Delete secrets
kubectl delete secret postgres-secrets django-secrets

# Delete PVCs (WARNING: This deletes all data)
kubectl delete pvc --all
```

### Delete Cluster

To completely remove your Kubernetes cluster:

**Using CLI:**
```bash
doctl kubernetes cluster delete django-k8s-cluster
```

**Using Web Interface:**
1. Go to Kubernetes in DigitalOcean dashboard
2. Select your cluster
3. Click "Destroy"

### Delete Container Registry

```bash
doctl registry delete django-registry
```

## Additional Resources

- [DigitalOcean Kubernetes Documentation](https://docs.digitalocean.com/products/kubernetes/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Django Deployment Checklist](https://docs.djangoproject.com/en/stable/howto/deployment/checklist/)
- [Cookiecutter Django Documentation](https://cookiecutter-django.readthedocs.io/)

## Support

For issues specific to this deployment:
1. Check pod logs: `kubectl logs <pod-name>`
2. Check pod events: `kubectl describe pod <pod-name>`
3. Review the troubleshooting section above
4. Check DigitalOcean status page for service issues

For application issues, refer to the main [README.md](../README.md) in the project root.
