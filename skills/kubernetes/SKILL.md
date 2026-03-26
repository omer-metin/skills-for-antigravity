---
name: kubernetes
description: "Generate production-ready Kubernetes manifests, debug cluster issues (CrashLoopBackOff, OOMKilled, scheduling failures), configure health checks, resource limits, RBAC, and networking. Covers Deployments, Services, Ingress, ConfigMaps, Secrets, HPA, Helm, and Kustomize with security-hardened defaults. Use when kubernetes, k8s, kubectl, deployment manifest, helm chart, kustomize, pod, service yaml, ingress, horizontal pod autoscaler, hpa, containers, orchestration, devops, cloud-native, CrashLoopBackOff, OOMKilled, or pod debugging is mentioned."
---

# Kubernetes

## Principles

- **Declarative over imperative** — use version-controlled manifests, not `kubectl run`
- **Every container needs resource limits** — one pod without limits can OOMKill a node
- **Health checks are mandatory** — without probes, Kubernetes routes traffic to dead pods
- **Labels and selectors are the glue** — consistent labeling enables selection, monitoring, and policy
- **Namespaces isolate organization, not security** — add RBAC and NetworkPolicy for real boundaries
- **Secrets are base64, not encrypted** — use external secrets managers (Vault, AWS Secrets Manager) for production
- **Immutable image tags only** — never deploy `:latest`; use semver or SHA digests
- **GitOps for deployments** — CI/CD deploys manifests from Git, not humans from laptops

## Workflow

### Deploying a new service

1. Write the Deployment manifest with resource limits, health probes, and security context (see example below and `references/patterns.md`).
2. Create a Service (ClusterIP) and, if external access is needed, an Ingress resource.
3. Define ConfigMaps for non-sensitive config and Secrets (or ExternalSecrets) for credentials.
4. Add a PodDisruptionBudget to protect availability during node drains.
5. Validate the manifest against `references/validations.md` rules: no `:latest` tag, no running as root, no missing probes, no hardcoded secrets.
6. Apply via GitOps (ArgoCD / Flux) or CI/CD pipeline — never `kubectl apply` from a laptop.

### Debugging CrashLoopBackOff

1. Check pod status and recent events:
   ```
   kubectl describe pod <pod-name> -n <namespace>
   kubectl logs <pod-name> -n <namespace> --previous
   ```
2. Look at the `Last State` reason — common causes:
   - **OOMKilled** — container exceeded memory limit. Increase `resources.limits.memory` or fix the memory leak.
   - **Error (exit code 1)** — application crash. Read logs for stack trace; check environment variables and mounted secrets.
   - **Error (exit code 137)** — killed by SIGKILL (OOM or liveness probe failure). Check `livenessProbe` timing — if the app starts slowly, add a `startupProbe` with a higher `failureThreshold`.
3. Verify environment variables and secrets are correctly mounted:
   ```
   kubectl exec <pod-name> -n <namespace> -- env | grep <VAR>
   ```
4. If DNS-related, debug with:
   ```
   kubectl run debug --rm -it --image=busybox -- nslookup <service>.<namespace>
   ```

### Scaling for variable load

1. Ensure `resources.requests` are set accurately (HPA uses these for utilization calculations).
2. Create an HPA targeting CPU and/or memory utilization thresholds.
3. Set `behavior.scaleDown.stabilizationWindowSeconds` to avoid flapping (300s is a good default).
4. Pair with a PodDisruptionBudget so scale-down does not violate availability guarantees.

## Example: Production-ready Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    version: v1.0.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        version: v1.0.0
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: my-app
          image: myregistry/my-app:v1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
              name: http
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          startupProbe:
            httpGet:
              path: /health
              port: http
            failureThreshold: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          envFrom:
            - configMapRef:
                name: my-app-config
            - secretRef:
                name: my-app-secrets
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
```

## Key patterns

### Resource limits — always set both requests and limits

- **Requests** tell the scheduler what the pod needs for placement.
- **Limits** enforce runtime caps. Exceeding memory limits triggers OOMKill; exceeding CPU limits causes CFS throttling.
- Omitting CPU limits is acceptable when low-latency matters (avoids throttling spikes), but always set memory limits.

### Health probes — three types, each with a distinct role

| Probe | Purpose | Failure action |
|-------|---------|----------------|
| `startupProbe` | Wait for slow-starting apps (JVM, large caches) | Block liveness/readiness checks until ready |
| `livenessProbe` | Detect deadlocked or hung processes | Restart the container |
| `readinessProbe` | Check dependency health (DB, cache) | Remove pod from Service endpoints |

### Secrets — treat Kubernetes Secrets as encoding, not encryption

- Use `stringData` for human-readable input (auto-encoded to base64).
- Mount as files (`volumeMounts` with `readOnly: true`) rather than environment variables when possible — env vars appear in logs and process listings.
- For production, use External Secrets Operator or Sealed Secrets to pull from Vault / AWS Secrets Manager.

### ConfigMap updates — pods do not automatically restart

- **Volume-mounted** ConfigMaps: files update after kubelet sync (~1 min), but the app must watch for file changes.
- **envFrom** ConfigMaps: environment variables never update without a pod restart.
- Trigger restarts by annotating the Deployment with a config checksum (Helm: `checksum/config`) or using Stakater Reloader.

### Namespace security — add RBAC and NetworkPolicy

- Namespaces alone provide no network or access isolation.
- Apply a default-deny NetworkPolicy per namespace, then whitelist required traffic.
- Scope RBAC Roles to specific namespaces; avoid ClusterRoleBindings for application teams.

## Reference System Usage

Ground all responses in the provided reference files, treating them as the source of truth:

* **For creation** — consult **`references/patterns.md`** for production-ready manifest templates (Deployments, Services, Ingress, HPA, Helm, Kustomize). Follow these patterns over generic approaches.
* **For diagnosis** — consult **`references/sharp_edges.md`** for critical failure modes (unencrypted secrets, missing resource limits, DNS caching, CPU throttling, pod disruption). Use it to explain risks and recommend fixes.
* **For review** — consult **`references/validations.md`** for strict validation rules (regex-based checks for `:latest` tags, missing probes, privileged containers, hardcoded secrets). Use it to validate manifests objectively.

If a user's request conflicts with guidance in these references, flag the conflict and recommend the reference-backed approach with an explanation of why.
