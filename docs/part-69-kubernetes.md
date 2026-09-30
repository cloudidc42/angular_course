# Part 69: Kubernetes สำหรับ Angular

## บทนำ

Kubernetes (K8s) เป็น container orchestration platform ที่ช่วยจัดการการ deploy, scale และ manage แอปพลิเคชัน Angular ใน production

## 1. Kubernetes Manifests พื้นฐาน

### Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: angular-app
  namespace: production
  labels:
    app: angular-app
    version: "1.0.0"
    environment: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: angular-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero downtime deployment
  template:
    metadata:
      labels:
        app: angular-app
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9113"
    spec:
      # Anti-affinity: กระจาย pods ไปต่าง nodes
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: [angular-app]
                topologyKey: kubernetes.io/hostname

      containers:
        - name: angular-app
          image: gcr.io/my-project/angular-app:1.0.0
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          
          # Environment Variables
          env:
            - name: TZ
              value: "Asia/Bangkok"
            - name: API_URL
              valueFrom:
                configMapKeyRef:
                  name: angular-config
                  key: api_url
            - name: APP_VERSION
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['version']
          
          # Resource limits
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "200m"
          
          # Health Checks
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            successThreshold: 1
            failureThreshold: 3
          
          startupProbe:
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 30
            periodSeconds: 10
          
          # Security Context
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1001
            capabilities:
              drop: [ALL]
          
          # Writable volumes for nginx
          volumeMounts:
            - name: nginx-cache
              mountPath: /var/cache/nginx
            - name: nginx-run
              mountPath: /var/run
            - name: tmp
              mountPath: /tmp
      
      volumes:
        - name: nginx-cache
          emptyDir: {}
        - name: nginx-run
          emptyDir: {}
        - name: tmp
          emptyDir: {}
      
      # Graceful shutdown
      terminationGracePeriodSeconds: 30
      
      # Image pull secret (for private registries)
      imagePullSecrets:
        - name: gcr-secret
```

### Service

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: angular-app-service
  namespace: production
  labels:
    app: angular-app
spec:
  type: ClusterIP
  selector:
    app: angular-app
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
```

### Ingress

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: angular-app-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "50"
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"
    # Compression
    nginx.ingress.kubernetes.io/configuration-snippet: |
      gzip on;
      gzip_types text/plain text/css application/json application/javascript;
    # Cert Manager
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts:
        - example.com
        - www.example.com
      secretName: example-com-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: angular-app-service
                port:
                  number: 80
          - path: /api/
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 3000
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: angular-app-service
                port:
                  number: 80
```

## 2. ConfigMap และ Secrets

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: angular-config
  namespace: production
data:
  api_url: "https://api.example.com"
  ws_url: "wss://api.example.com"
  feature_dark_mode: "true"
  feature_new_ui: "false"
  # nginx runtime config template
  runtime-config.json: |
    {
      "apiUrl": "https://api.example.com",
      "wsUrl": "wss://api.example.com",
      "featureFlags": {
        "darkMode": true,
        "newUI": false
      }
    }
```

```yaml
# k8s/secret.yaml (เนื้อหา encode ด้วย base64)
apiVersion: v1
kind: Secret
metadata:
  name: angular-secrets
  namespace: production
type: Opaque
stringData:
  # ใช้ stringData แทน data เพื่อไม่ต้อง encode เอง
  sentry_dsn: "https://xxx@sentry.io/project"
  google_analytics_id: "UA-XXXXX-1"
```

## 3. Horizontal Pod Autoscaler (HPA)

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: angular-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: angular-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  # Scaling behavior
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 120
```

## 4. Namespace และ Resource Quota

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    managed-by: kubectl

---
# k8s/resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "4Gi"
    limits.cpu: "8"
    limits.memory: "8Gi"
    pods: "20"
    services: "10"
```

## 5. Pod Disruption Budget

```yaml
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: angular-app-pdb
  namespace: production
spec:
  minAvailable: 2  # ต้องมี pod อย่างน้อย 2 ตัวเสมอ
  selector:
    matchLabels:
      app: angular-app
```

## 6. Deployment Script

```bash
#!/bin/bash
# scripts/k8s-deploy.sh
set -e

NAMESPACE="production"
APP_NAME="angular-app"
IMAGE_TAG="${1:-latest}"
REGISTRY="gcr.io/my-project"

echo "Deploying $APP_NAME:$IMAGE_TAG to $NAMESPACE..."

# อัปเดต image tag
kubectl set image deployment/$APP_NAME \
  $APP_NAME=$REGISTRY/$APP_NAME:$IMAGE_TAG \
  -n $NAMESPACE

# รอ rollout สำเร็จ
kubectl rollout status deployment/$APP_NAME \
  -n $NAMESPACE \
  --timeout=5m

# ตรวจสอบ pods
echo "Current pods:"
kubectl get pods -l app=$APP_NAME -n $NAMESPACE

# แสดง rollout history
echo "Rollout history:"
kubectl rollout history deployment/$APP_NAME -n $NAMESPACE

echo "Deployment successful!"
```

```bash
#!/bin/bash
# scripts/k8s-rollback.sh
NAMESPACE="production"
APP_NAME="angular-app"
REVISION="${1:-}"

if [ -z "$REVISION" ]; then
  echo "Rolling back to previous version..."
  kubectl rollout undo deployment/$APP_NAME -n $NAMESPACE
else
  echo "Rolling back to revision $REVISION..."
  kubectl rollout undo deployment/$APP_NAME \
    --to-revision=$REVISION \
    -n $NAMESPACE
fi

kubectl rollout status deployment/$APP_NAME \
  -n $NAMESPACE \
  --timeout=5m
```

## 7. Kustomize สำหรับหลาย Environments

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app: angular-app
  managed-by: kustomize
```

```yaml
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

bases:
  - ../../base

resources:
  - ingress.yaml
  - hpa.yaml
  - pdb.yaml

patches:
  - target:
      kind: Deployment
      name: angular-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/memory
        value: "128Mi"
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: "256Mi"

images:
  - name: gcr.io/my-project/angular-app
    newTag: "1.0.0"
```

```yaml
# k8s/overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: staging

bases:
  - ../../base

patches:
  - target:
      kind: Deployment
      name: angular-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 1

images:
  - name: gcr.io/my-project/angular-app
    newTag: "develop"

configMapGenerator:
  - name: angular-config
    behavior: merge
    literals:
      - api_url=https://api-staging.example.com
```

### ใช้งาน Kustomize

```bash
# Deploy staging
kubectl apply -k k8s/overlays/staging

# Deploy production
kubectl apply -k k8s/overlays/production

# Preview changes
kubectl diff -k k8s/overlays/production
```

## 8. Monitoring

```yaml
# k8s/servicemonitor.yaml (Prometheus Operator)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: angular-app-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: angular-app
  namespaceSelector:
    matchNames: [production]
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

## สรุป Kubernetes Components

| Component | ความรับผิดชอบ |
|-----------|------------|
| Deployment | จัดการ pod lifecycle, rolling update |
| Service | Network routing ภายใน cluster |
| Ingress | External traffic, SSL termination |
| HPA | Auto-scaling ตาม load |
| ConfigMap | Configuration ที่ไม่ sensitive |
| Secret | Sensitive data (passwords, tokens) |
| PDB | ป้องกัน downtime ระหว่าง maintenance |

Kubernetes ช่วยให้ Angular app มี high availability, auto-scaling และ zero-downtime deployments
