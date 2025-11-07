# Java Application Helm Chart with Argo Rollouts

A production-ready Helm chart for deploying Java applications (Spring Boot) on Kubernetes with support for progressive delivery using Argo Rollouts.

## Features

- ✅ **Progressive Delivery Strategies**
  - Canary Deployments
  - Blue-Green Deployments
- ✅ **Automated Analysis** with Prometheus metrics
- ✅ **Spring Boot Actuator** integration
- ✅ **Horizontal Pod Autoscaling** (HPA)
- ✅ **Pod Disruption Budget** for high availability
- ✅ **Security Best Practices** (non-root, read-only filesystem)
- ✅ **Fully Configurable** - No hardcoded values

## Prerequisites

1. **Kubernetes Cluster** (1.19+)
2. **Argo Rollouts Controller** installed
3. **Helm 3.x**
4. Optional: **Prometheus** (for automated analysis)
5. Optional: **Nginx Ingress Controller** (for traffic splitting)

## Installation

### 1. Install Argo Rollouts Controller

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### 2. Install kubectl Argo Rollouts Plugin

```bash
# Mac
brew install argoproj/tap/kubectl-argo-rollouts

# Linux
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

### 3. Deploy the Helm Chart

#### Default Canary Deployment

```bash
helm install my-java-app ./java-app-rollout \
  --set image.repository=your-registry.azurecr.io/java-app \
  --set image.tag=v1.0.0 \
  --namespace default
```

#### Blue-Green Deployment

```bash
helm install my-java-app ./java-app-rollout \
  --values ./java-app-rollout/examples/values-bluegreen.yaml \
  --set image.repository=your-registry.azurecr.io/java-app \
  --set image.tag=v1.0.0 \
  --namespace default
```

## Configuration

### Key Configuration Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `strategy.type` | Deployment strategy: `canary` or `blueGreen` | `canary` |
| `replicaCount` | Number of pod replicas | `3` |
| `image.repository` | Container image repository | `your-registry.azurecr.io/java-app` |
| `image.tag` | Container image tag | `1.0.0` |
| `java.javaOpts` | JVM options | `-Xms512m -Xmx1024m -XX:+UseG1GC` |
| `java.port` | Application port | `8080` |
| `java.managementPort` | Actuator/management port | `8081` |
| `resources.requests.cpu` | CPU request | `500m` |
| `resources.requests.memory` | Memory request | `1Gi` |
| `resources.limits.cpu` | CPU limit | `2000m` |
| `resources.limits.memory` | Memory limit | `2Gi` |

### Canary Deployment Configuration

```yaml
strategy:
  type: canary

canary:
  maxSurge: "25%"
  maxUnavailable: 0
  steps:
    - setWeight: 20
    - pause: {duration: 30s}
    - setWeight: 40
    - pause: {duration: 30s}
    - setWeight: 60
    - pause: {duration: 30s}
    - setWeight: 80
    - pause: {duration: 30s}
  analysis:
    enabled: true
    startingStep: 2
    prometheus:
      address: "http://prometheus-server.monitoring.svc.cluster.local"
    metrics:
      successRate:
        enabled: true
        interval: 30s
        threshold: 0.95
        failureLimit: 3
      avgResponseTime:
        enabled: true
        interval: 30s
        threshold: 500
        failureLimit: 3
```

### Blue-Green Deployment Configuration

```yaml
strategy:
  type: blueGreen

blueGreen:
  autoPromotionEnabled: false
  autoPromotionSeconds: 30
  scaleDownDelaySeconds: 30
  previewReplicaCount: 1
  analysis:
    enabled: true
    prometheus:
      address: "http://prometheus-server.monitoring.svc.cluster.local"
    metrics:
      successRate:
        enabled: true
        interval: 30s
        threshold: 0.95
        failureLimit: 3
```

## Usage

### Deploy a New Version

```bash
# Update the image tag
helm upgrade my-java-app ./java-app-rollout \
  --set image.tag=v2.0.0 \
  --reuse-values
```

### Monitor Rollout Progress

```bash
# Watch the rollout
kubectl argo rollouts get rollout my-java-app --watch

# Check rollout status
kubectl argo rollouts status my-java-app
```

### Canary Deployment Commands

```bash
# Promote to next step
kubectl argo rollouts promote my-java-app

# Abort the rollout
kubectl argo rollouts abort my-java-app

# Full rollback
kubectl argo rollouts undo my-java-app
```

### Blue-Green Deployment Commands

```bash
# Promote preview to active
kubectl argo rollouts promote my-java-app

# Abort and rollback
kubectl argo rollouts abort my-java-app
```

### Access the Argo Rollouts Dashboard

```bash
kubectl argo rollouts dashboard
```

Then open: http://localhost:3100/rollouts

## Example Application (Spring Boot)

Your Java application should expose Spring Boot Actuator endpoints:

### Maven Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### application.properties

```properties
management.server.port=8081
management.endpoints.web.exposure.include=health,info,prometheus,metrics
management.endpoint.health.probes.enabled=true
management.health.livenessState.enabled=true
management.health.readinessState.enabled=true
```

## Dockerfile Example

```dockerfile
FROM eclipse-temurin:17-jre-alpine

# Create non-root user
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

# Create necessary directories
RUN mkdir -p /app /tmp /app/logs && \
    chown -R appuser:appuser /app /tmp /app/logs

# Switch to non-root user
USER appuser

WORKDIR /app

# Copy the JAR file
COPY --chown=appuser:appuser target/*.jar app.jar

# Expose ports
EXPOSE 8080 8081

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## Advanced Features

### Enable Autoscaling

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

### Configure Ingress

```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: java-app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: java-app-tls
      hosts:
        - java-app.example.com
```

### Add ConfigMap and Secrets

```yaml
configMap:
  enabled: true
  data:
    application.properties: |
      spring.application.name=my-java-app
      logging.level.root=INFO

secret:
  enabled: true
  data:
    database-password: cGFzc3dvcmQ=  # base64 encoded

envFrom:
  - configMapRef:
      name: my-java-app
  - secretRef:
      name: my-java-app
```

## Troubleshooting

### Check Rollout Events

```bash
kubectl describe rollout my-java-app
```

### View Analysis Runs

```bash
kubectl get analysisrun
kubectl describe analysisrun <analysis-run-name>
```

### Check Pod Logs

```bash
kubectl logs -l app.kubernetes.io/name=java-app-rollout --tail=100 -f
```

### Verify Health Endpoints

```bash
# Port-forward to pod
kubectl port-forward deployment/my-java-app 8081:8081

# Check health
curl http://localhost:8081/actuator/health
curl http://localhost:8081/actuator/health/liveness
curl http://localhost:8081/actuator/health/readiness
```

## Uninstall

```bash
helm uninstall my-java-app
```

## Support

For issues and feature requests, please contact the DevOps team.

## License

Copyright © 2025 DevOps Team

