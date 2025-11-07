# Quick Start Guide - Java App with Argo Rollouts

## 🚀 5-Minute Setup

### 1. Install Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### 2. Create ACR Secret (if using Azure)

```bash
kubectl create secret docker-registry acr-secret \
  --docker-server=acrdemonnbs.azurecr.io \
  --docker-username=<USERNAME> \
  --docker-password=<PASSWORD>
```

### 3. Deploy Application

**Canary Deployment:**
```bash
helm install my-app ./java-app-rollout \
  --set image.repository=acrdemonnbs.azurecr.io/my-java-app \
  --set image.tag=1.0.0
```

**Blue-Green Deployment:**
```bash
helm install my-app ./java-app-rollout \
  --set strategy.type=blueGreen \
  --set image.repository=acrdemonnbs.azurecr.io/my-java-app \
  --set image.tag=1.0.0
```

### 4. Monitor Deployment

```bash
kubectl argo rollouts get rollout my-app --watch
```

---

## 📊 Deploy New Version

### Update Image

```bash
helm upgrade my-app ./java-app-rollout \
  --set image.tag=2.0.0 \
  --reuse-values
```

### Promote (Manual)

```bash
# Canary: Promote to next step
kubectl argo rollouts promote my-app

# Blue-Green: Switch traffic
kubectl argo rollouts promote my-app
```

### Rollback

```bash
kubectl argo rollouts abort my-app
kubectl argo rollouts undo my-app
```

---

## 🎯 Common Use Cases

### Use Case 1: Canary with Manual Approval

```yaml
# values-custom.yaml
strategy:
  type: canary

canary:
  steps:
    - setWeight: 25
    - pause: {}  # Manual approval required
    - setWeight: 50
    - pause: {}
    - setWeight: 75
    - pause: {}
```

```bash
helm install my-app ./java-app-rollout -f values-custom.yaml
```

### Use Case 2: Blue-Green with Auto-Promotion

```yaml
# values-custom.yaml
strategy:
  type: blueGreen

blueGreen:
  autoPromotionEnabled: true
  autoPromotionSeconds: 300  # 5 minutes
```

### Use Case 3: Canary with Prometheus Analysis

```yaml
# values-custom.yaml
canary:
  analysis:
    enabled: true
    prometheus:
      address: "http://prometheus-server.monitoring.svc.cluster.local"
    metrics:
      successRate:
        threshold: 0.95
```

---

## 🔍 Essential Commands

| Action | Command |
|--------|---------|
| **Watch Rollout** | `kubectl argo rollouts get rollout my-app --watch` |
| **Promote** | `kubectl argo rollouts promote my-app` |
| **Abort** | `kubectl argo rollouts abort my-app` |
| **Rollback** | `kubectl argo rollouts undo my-app` |
| **Dashboard** | `kubectl argo rollouts dashboard` |
| **History** | `kubectl argo rollouts history my-app` |
| **Logs** | `kubectl logs -l app.kubernetes.io/name=java-app-rollout -f` |
| **Pods** | `kubectl get pods -l app.kubernetes.io/name=java-app-rollout` |

---

## 🎨 Customization Examples

### Change JVM Settings

```bash
helm upgrade my-app ./java-app-rollout \
  --set java.javaOpts="-Xms2g -Xmx4g -XX:+UseG1GC" \
  --reuse-values
```

### Scale Replicas

```bash
helm upgrade my-app ./java-app-rollout \
  --set replicaCount=5 \
  --reuse-values
```

### Enable HPA

```bash
helm upgrade my-app ./java-app-rollout \
  --set autoscaling.enabled=true \
  --set autoscaling.minReplicas=3 \
  --set autoscaling.maxReplicas=10 \
  --reuse-values
```

### Add Environment Variable

```bash
helm upgrade my-app ./java-app-rollout \
  --set env[0].name=MY_VAR \
  --set env[0].value=my-value \
  --reuse-values
```

---

## 📁 Directory Structure

```
java-app-rollout/
├── Chart.yaml                          # Chart metadata
├── values.yaml                         # Default values
├── templates/
│   ├── rollout.yaml                    # Argo Rollout resource
│   ├── service.yaml                    # Services (stable/canary/preview)
│   ├── ingress.yaml                    # Ingress (optional)
│   ├── analysistemplate.yaml           # Prometheus analysis
│   ├── hpa.yaml                        # Horizontal Pod Autoscaler
│   ├── poddisruptionbudget.yaml        # PDB for HA
│   ├── serviceaccount.yaml             # Service Account
│   ├── configmap.yaml                  # ConfigMap
│   └── secret.yaml                     # Secret
├── examples/
│   ├── values-canary.yaml              # Canary example
│   ├── values-bluegreen.yaml           # Blue-Green example
│   └── values-production.yaml          # Production example
├── README.md                           # Full documentation
├── INSTALL.md                          # Installation guide
└── QUICK-START.md                      # This file
```

---

## ✅ Checklist

- [ ] Argo Rollouts controller installed
- [ ] kubectl argo-rollouts plugin installed
- [ ] ACR secret created (if using private registry)
- [ ] Values customized for your environment
- [ ] Application has health endpoints (`/actuator/health`)
- [ ] Prometheus installed (if using analysis)
- [ ] Ingress controller installed (if using ingress)

---

## 🆘 Troubleshooting

### Rollout Stuck in Progressing

```bash
# Check rollout status
kubectl describe rollout my-app

# Check pods
kubectl get pods -l app.kubernetes.io/name=java-app-rollout

# Check pod logs
kubectl logs <pod-name>
```

### Health Probes Failing

```bash
# Port forward and test manually
kubectl port-forward <pod-name> 8081:8081
curl http://localhost:8081/actuator/health
```

### Analysis Failing

```bash
# Check analysis run
kubectl get analysisrun
kubectl describe analysisrun <name>

# Verify Prometheus is accessible
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://prometheus-server.monitoring.svc.cluster.local
```

---

## 📚 More Resources

- **Full Documentation**: [README.md](./README.md)
- **Installation Guide**: [INSTALL.md](./INSTALL.md)
- **Argo Rollouts Docs**: https://argo-rollouts.readthedocs.io/
- **Examples**: [examples/](./examples/)

---

**Need Help?** Contact DevOps Team

