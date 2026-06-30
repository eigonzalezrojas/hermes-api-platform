# TASK-004 — Kubernetes manifests and EKS deployment

## Status

Planned

## Parent story

`US-006-production-ready.md`

## Summary

Create the Kubernetes manifests for `hermes-gateway` and `hermes-admin` using Kustomize base/overlay structure. Manifests define Deployments with IRSA service accounts, readiness and liveness probes, resource requests and limits, HorizontalPodAutoscaler, and PodDisruptionBudget. Production secrets (DB credentials, JWT key) are injected via External Secrets Operator from AWS Secrets Manager.

## Technical scope

This task includes:

- `infra/k8s/base/` — namespace, service accounts, ConfigMaps, base Deployments and Services for both services.
- `infra/k8s/overlays/prod/` — production replica counts, resource limits, image tags (updated by CI), ingress annotations.
- `infra/k8s/overlays/dev/` — reduced replicas, lower resource limits for cost savings.
- `HorizontalPodAutoscaler` for `hermes-gateway`: scale on CPU utilization (target 70%), min 2 / max 10 replicas.
- `PodDisruptionBudget` for both services: `minAvailable: 1` during node maintenance.
- `ExternalSecret` resources pulling DB credentials and JWT public key from AWS Secrets Manager.
- `Ingress` resource with ALB Ingress Controller annotations for the gateway.
- Readiness probe: `GET /actuator/health/readiness` — pod receives traffic only when this returns 200.
- Liveness probe: `GET /actuator/health/liveness` — pod is restarted when this fails.

## Out of scope

This task does not include:

- Network policies (deferred — requires CNI plugin supporting network policies, e.g., Calico).
- Pod Security Admission policies (deferred).
- Persistent volumes (neither service writes to local disk — RDS and ElastiCache are managed).
- Helm chart conversion (Kustomize is sufficient for v1.0.0; Helm is deferred if multi-team templating is needed).
- Argo CD or Flux CD (GitOps controller — deferred; `kubectl apply -k` from GitHub Actions is sufficient for v1.0.0).

## Implementation notes

### Directory structure

```text
infra/k8s/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── hermes-gateway/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── hpa.yaml
│   │   ├── pdb.yaml
│   │   └── externalsecret.yaml
│   └── hermes-admin/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── serviceaccount.yaml
│       ├── pdb.yaml
│       └── externalsecret.yaml
└── overlays/
    ├── prod/
    │   ├── kustomization.yaml
    │   └── ingress.yaml
    └── dev/
        ├── kustomization.yaml
        └── replicas-patch.yaml
```

### namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hermes
```

### hermes-gateway/deployment.yaml (base)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hermes-gateway
  namespace: hermes
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hermes-gateway
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0       # zero-downtime rolling update
  template:
    metadata:
      labels:
        app: hermes-gateway
    spec:
      serviceAccountName: hermes-gateway
      terminationGracePeriodSeconds: 30
      containers:
        - name: hermes-gateway
          image: hermes-gateway:latest   # overridden by Kustomize in CI
          ports:
            - containerPort: 8080
          envFrom:
            - secretRef:
                name: hermes-gateway-secrets   # populated by ExternalSecret
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 15
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]  # allow LB to drain connections
```

### serviceaccount.yaml (hermes-gateway)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: hermes-gateway
  namespace: hermes
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::{ACCOUNT_ID}:role/hermes-gateway-role
```

The `role-arn` annotation is the IRSA binding. The role ARN is overridden per environment via Kustomize patch or substitution.

### hpa.yaml (hermes-gateway)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hermes-gateway
  namespace: hermes
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hermes-gateway
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### pdb.yaml (hermes-gateway)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: hermes-gateway-pdb
  namespace: hermes
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: hermes-gateway
```

### externalsecret.yaml (hermes-gateway)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: hermes-gateway-secrets
  namespace: hermes
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: hermes-gateway-secrets
    creationPolicy: Owner
  data:
    - secretKey: SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_PUBLIC_KEY_LOCATION_CONTENT
      remoteRef:
        key: hermes/prod/jwt-public-key
    - secretKey: SPRING_DATA_REDIS_HOST
      remoteRef:
        key: hermes/prod/redis-endpoint
```

### overlays/prod/ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hermes-gateway
  namespace: hermes
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:{REGION}:{ACCOUNT_ID}:certificate/{CERT_ID}
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:{REGION}:{ACCOUNT_ID}:regional/webacl/hermes-waf/{ID}
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/actions.ssl-redirect: >
      {"type":"redirect","redirectConfig":{"port":"443","protocol":"HTTPS","statusCode":"HTTP_301"}}
spec:
  rules:
    - host: api.{DOMAIN}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hermes-gateway
                port:
                  number: 8080
```

### overlays/prod/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - ingress.yaml

patches:
  - target:
      kind: Deployment
      name: hermes-gateway
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3

images:
  - name: hermes-gateway
    newName: {ECR_REGISTRY}/hermes-gateway
    newTag: latest    # overridden by `kustomize edit set image` in CI
  - name: hermes-admin
    newName: {ECR_REGISTRY}/hermes-admin
    newTag: latest
```

## Expected files

```text
infra/k8s/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── hermes-gateway/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── hpa.yaml
│   │   ├── pdb.yaml
│   │   └── externalsecret.yaml
│   └── hermes-admin/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── serviceaccount.yaml
│       ├── pdb.yaml
│       └── externalsecret.yaml
└── overlays/
    ├── prod/
    │   ├── kustomization.yaml
    │   └── ingress.yaml
    └── dev/
        ├── kustomization.yaml
        └── replicas-patch.yaml
```

## Acceptance criteria

- [ ] `kubectl apply -k infra/k8s/overlays/prod/` applies all resources without errors.
- [ ] `kubectl get pods -n hermes` shows `hermes-gateway` and `hermes-admin` pods in `Running` state.
- [ ] Readiness probe passes: `kubectl exec -n hermes {pod} -- wget -qO- localhost:8080/actuator/health/readiness`.
- [ ] Liveness probe passes: `kubectl exec -n hermes {pod} -- wget -qO- localhost:8080/actuator/health/liveness`.
- [ ] `GET https://api.{DOMAIN}/actuator/health` returns `200` via the ALB (DNS resolved, TLS valid).
- [ ] HPA is active: `kubectl get hpa -n hermes` shows `hermes-gateway` with min 2 / max 10.
- [ ] PDB is active: `kubectl get pdb -n hermes` shows `minAvailable: 1`.
- [ ] After `kubectl delete pod {pod} -n hermes`, the pod is rescheduled and readiness is restored within 60 seconds.
- [ ] Secrets from AWS Secrets Manager are mounted correctly: `kubectl get secret hermes-gateway-secrets -n hermes` exists.

## Validation

Dry-run validation (no cluster needed):

```bash
kubectl apply -k infra/k8s/overlays/prod/ --dry-run=client
```

Live deployment:

```bash
kubectl apply -k infra/k8s/overlays/prod/
kubectl rollout status deployment/hermes-gateway -n hermes --timeout=5m
kubectl rollout status deployment/hermes-admin -n hermes --timeout=5m
```

External connectivity:

```bash
curl -s https://api.{DOMAIN}/actuator/health | jq .
```

## Commit suggestion

```text
feat: add Kubernetes manifests with Kustomize overlays, HPA, PDB, and IRSA for EKS deployment
```

## Notes

- **`preStop: sleep 5`**: when Kubernetes terminates a pod, it removes it from the Service endpoints and sends `SIGTERM` simultaneously. There is a race condition: the load balancer may continue routing traffic to the pod for a few seconds after it starts shutting down. A 5-second `preStop` sleep gives the ALB time to drain connections before the JVM begins its shutdown sequence. Without this, in-flight requests receive connection reset errors during rolling updates.
- **`maxUnavailable: 0` in rolling update**: ensures at least the desired number of pods are running throughout the update. Combined with `maxSurge: 1`, Kubernetes brings up one new pod, waits for its readiness probe to pass, then terminates one old pod. This guarantees zero-downtime rolling deployments.
- **External Secrets Operator**: the `ExternalSecret` resource requires the ESO controller to be installed in the cluster (via Helm chart). ESO watches `ExternalSecret` resources and creates Kubernetes `Secret` objects from the referenced AWS Secrets Manager values, refreshing them every `refreshInterval`. ESO installation is a one-time cluster setup step — document in `infra/k8s/README.md`.
- **IRSA in practice**: when a pod with an IRSA-annotated service account starts, the pod webhook (managed by EKS) injects `AWS_WEB_IDENTITY_TOKEN_FILE` and `AWS_ROLE_ARN` environment variables. The AWS SDK picks these up automatically — no code change is needed in `hermes-gateway` or `hermes-admin`. Spring Cloud AWS and the standard AWS SDK both support IRSA out of the box.
- **`{ECR_REGISTRY}` and `{DOMAIN}` placeholders**: these values are environment-specific and should be substituted either via Kustomize `vars`/`replacements` or via `envsubst` in the CI pipeline before `kubectl apply`. Never hardcode account IDs or domains in base manifests.
