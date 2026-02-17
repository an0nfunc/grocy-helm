# Grocy Helm Chart

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

A Helm chart for deploying [Grocy](https://grocy.info) — a self-hosted grocery and household management solution — on Kubernetes.

Uses the [LinuxServer.io](https://docs.linuxserver.io/images/docker-grocy/) container image (`lscr.io/linuxserver/grocy`).

## Prerequisites

- Kubernetes 1.26+
- Helm 3
- Gateway API CRDs installed (only if using `httpRoute`)

## Installing

### From OCI registry

```bash
helm install grocy oci://ghcr.io/harting/charts/grocy
```

### From source

```bash
git clone https://github.com/harting/grocy-helm.git
cd grocy-helm
helm install grocy .
```

## Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `imagePullSecrets` | Image pull secrets | `[]` |
| `serviceAccount.create` | Create a ServiceAccount | `true` |
| `serviceAccount.name` | ServiceAccount name (auto-generated if empty) | `""` |
| `serviceAccount.annotations` | ServiceAccount annotations | `{}` |
| `serviceAccount.automountServiceAccountToken` | Automount API credentials | `true` |
| `image.repository` | Container image repository | `lscr.io/linuxserver/grocy` |
| `image.tag` | Image tag (defaults to appVersion) | `""` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `env` | Map of environment variables | `{PUID: "1000", PGID: "1000", TZ: "Etc/UTC"}` |
| `extraEnv` | List-form env vars for valueFrom/secret refs | `[]` |
| `envFrom` | Bulk secret/configmap env mounting | `[]` |
| `podAnnotations` | Annotations added to pods only | `{}` |
| `podLabels` | Labels added to pods only | `{}` |
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Service port | `80` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `persistence.storageClass` | StorageClass (empty = cluster default) | `""` |
| `persistence.accessMode` | PVC access mode | `ReadWriteOnce` |
| `persistence.size` | PVC size | `1Gi` |
| `persistence.existingClaim` | Use an existing PVC | `""` |
| `extraVolumes` | Additional volumes | `[]` |
| `extraVolumeMounts` | Additional volume mounts | `[]` |
| `extraInitContainers` | Additional init containers | `[]` |
| `extraContainers` | Additional sidecar containers | `[]` |
| `ingress.enabled` | Enable Ingress | `false` |
| `ingress.className` | Ingress class name | `""` |
| `ingress.annotations` | Ingress annotations | `{}` |
| `ingress.hosts` | Ingress host rules | see values.yaml |
| `ingress.tls` | Ingress TLS config | `[]` |
| `httpRoute.enabled` | Enable Gateway API HTTPRoute | `false` |
| `httpRoute.annotations` | HTTPRoute annotations | `{}` |
| `httpRoute.parentRefs` | Gateway parent references | `[]` |
| `httpRoute.hostnames` | HTTPRoute hostnames | `[]` |
| `httpRoute.httpRedirect.enabled` | Enable HTTP-to-HTTPS redirect route | `false` |
| `httpRoute.httpRedirect.sectionName` | HTTP listener section name | `http` |
| `resources.requests.cpu` | CPU request | `50m` |
| `resources.requests.memory` | Memory request | `128Mi` |
| `resources.limits.memory` | Memory limit | `512Mi` |
| `livenessProbe` | Liveness probe configuration | HTTP GET `/` |
| `readinessProbe` | Readiness probe configuration | HTTP GET `/` |
| `startupProbe.enabled` | Enable startup probe (for slow first-run migrations) | `false` |
| `startupProbe.failureThreshold` | Startup probe failure threshold | `30` |
| `nodeSelector` | Node selector | `{}` |
| `tolerations` | Tolerations | `[]` |
| `affinity` | Affinity rules | `{}` |
| `topologySpreadConstraints` | Topology spread constraints | `[]` |
| `priorityClassName` | Pod priority class | `""` |
| `dnsConfig` | Pod DNS config | `{}` |
| `dnsPolicy` | Pod DNS policy | `""` |
| `podSecurityContext` | Pod security context | `{}` |
| `securityContext` | Container security context | `{}` |
| `commonLabels` | Labels added to all resources | `{}` |
| `commonAnnotations` | Annotations added to all resources | `{}` |

## Examples

See the [`examples/`](examples/) directory for complete values files:

### Minimal (port-forward)

```yaml
# Just use defaults — persistence enabled, no ingress
# Access via: kubectl port-forward svc/grocy 8080:80
```

### Traditional Ingress (nginx + cert-manager)

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: grocy.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: grocy-tls
      hosts:
        - grocy.example.com
```

### Gateway API HTTPRoute

```yaml
httpRoute:
  enabled: true
  hostnames:
    - grocy.example.com
  parentRefs:
    - name: default
      namespace: nginx-gateway
      sectionName: https
  httpRedirect:
    enabled: true
```

## Persistence

Grocy uses SQLite for its database, stored under `/config` in the container. The chart creates a PVC to persist this data across pod restarts.

To use an existing PVC:

```yaml
persistence:
  existingClaim: my-grocy-pvc
```

To disable persistence entirely (data lost on pod restart):

```yaml
persistence:
  enabled: false
```

**Backup**: Since Grocy uses SQLite, you can back up by copying the `/config` directory contents from the PVC. Consider using a CronJob or Velero for automated backups.

## Upgrading

The chart uses `strategy: Recreate` because SQLite cannot handle concurrent access from multiple pods. During upgrades the pod will be terminated before the new one starts, causing brief downtime.

After upgrading to a new Grocy version, the application may run database migrations automatically on first access.

## Uninstalling

```bash
helm uninstall grocy
```

**Note**: The PVC is not deleted automatically. To remove it:

```bash
kubectl delete pvc grocy
```
