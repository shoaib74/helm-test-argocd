# Quick Installation Guide

## Step 1: Install Prerequisites

### Install Argo Rollouts Controller

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### Verify Installation

```bash
kubectl get pods -n argo-rollouts
```

## Step 2: Create ACR Secret (Azure Container Registry)

```bash
kubectl create secret docker-registry acr-secret \
  --docker-server=acrdemonnbs.azurecr.io \
  --docker-username=<ACR_USERNAME> \
  --docker-password=<ACR_PASSWORD> \
  --docker-email=<EMAIL>
```

Or use Azure AD integration:

```bash
# Attach ACR to AKS cluster
az aks update -n aks-demo-001 -g rg-demo-001 --attach-acr acrdemonnbs
```

## Step 3: Deploy Using Helm

### Option 1: Canary Deployment

```bash
helm install my-java-app ./java-app-rollout \
  --values ./java-app-rollout/examples/values-canary.yaml \
  --set image.tag=v1.0.0 \
  --namespace default
```

### Option 2: Blue-Green Deployment

```bash
helm install my-java-app ./java-app-rollout \
  --values ./java-app-rollout/examples/values-bluegreen.yaml \
  --set image.tag=v1.0.0 \
  --namespace default
```

### Option 3: Production Setup with Ingress

```bash
helm install my-java-app ./java-app-rollout \
  --values ./java-app-rollout/examples/values-production.yaml \
  --set image.tag=v1.0.0 \
  --namespace production \
  --create-namespace
```

## Step 4: Verify Deployment

```bash
# Check rollout status
kubectl argo rollouts get rollout my-java-app

# Check pods
kubectl get pods -l app.kubernetes.io/name=java-app-rollout

# Check services
kubectl get svc -l app.kubernetes.io/name=java-app-rollout

# Get LoadBalancer IP (if using LoadBalancer service)
kubectl get svc my-java-app-stable
```

## Step 5: Test Application

```bash
# Port forward to test locally
kubectl port-forward svc/my-java-app-stable 8080:80

# Test application
curl http://localhost:8080

# Test health endpoint
curl http://localhost:8081/actuator/health
```

## Step 6: Deploy New Version

### Update Image Tag

```bash
helm upgrade my-java-app ./java-app-rollout \
  --set image.tag=v2.0.0 \
  --reuse-values
```

### Monitor Rollout

```bash
kubectl argo rollouts get rollout my-java-app --watch
```

### Promote When Ready

#### Canary
```bash
# Promote to next step
kubectl argo rollouts promote my-java-app

# Skip all steps and promote immediately
kubectl argo rollouts promote my-java-app --full
```

#### Blue-Green
```bash
# Promote preview to active
kubectl argo rollouts promote my-java-app
```

### Rollback if Needed

```bash
# Abort current rollout
kubectl argo rollouts abort my-java-app

# Undo to previous version
kubectl argo rollouts undo my-java-app
```

## Step 7: Access Argo Rollouts Dashboard

```bash
kubectl argo rollouts dashboard
```

Open browser to: http://localhost:3100/rollouts

## Common Commands

### View Rollout History

```bash
kubectl argo rollouts history my-java-app
```

### Set Image Manually

```bash
kubectl argo rollouts set image my-java-app \
  java-app-rollout=acrdemonnbs.azurecr.io/my-java-app:v3.0.0
```

### Pause Rollout

```bash
kubectl argo rollouts pause my-java-app
```

### Resume Rollout

```bash
kubectl argo rollouts resume my-java-app
```

### Restart Rollout

```bash
kubectl argo rollouts restart my-java-app
```

## Troubleshooting

### Check Rollout Events

```bash
kubectl describe rollout my-java-app
```

### Check Analysis Runs

```bash
kubectl get analysisrun
kubectl logs -l analysisrun=my-java-app-<hash>
```

### View Pod Logs

```bash
kubectl logs -l app.kubernetes.io/name=java-app-rollout -f
```

### Debug Pod Issues

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

## Cleanup

```bash
# Uninstall the application
helm uninstall my-java-app

# Delete namespace (if created)
kubectl delete namespace production
```

## Next Steps

1. Configure Prometheus for automated analysis
2. Set up Ingress for production traffic
3. Configure CI/CD pipeline to automate deployments
4. Set up monitoring and alerting
5. Configure backup and disaster recovery

## Support

For issues or questions, contact the DevOps team.

