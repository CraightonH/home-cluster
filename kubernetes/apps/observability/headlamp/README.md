# Headlamp - Mobile-Friendly Kubernetes Dashboard

Headlamp is a modern, mobile-responsive Kubernetes dashboard.

## Access

URL: `https://headlamp.${SECRET_DOMAIN}`

## Authentication

Headlamp uses Kubernetes service account tokens for authentication.

### Read-Only Access (Default)

A read-only service account is deployed with cluster-wide view permissions.

**Get the token:**
```bash
kubectl get secret headlamp-readonly-token -n observability -o jsonpath='{.data.token}' | base64 -d
```

Copy this token and paste it into the Headlamp login screen.

**Permissions:** View-only access to all namespaces (uses built-in `view` ClusterRole)

### Admin Access

For full cluster management, use the headlamp service account (created by Helm chart):

**Get the token:**
```bash
kubectl create token headlamp -n observability --duration=8760h
```

**Permissions:** Cluster admin (full read/write access)

### Custom Access

Create your own ServiceAccount with specific RBAC permissions:

```bash
# Create service account
kubectl create serviceaccount myuser -n default

# Bind to a role (examples)
kubectl create clusterrolebinding myuser-view --clusterrole=view --serviceaccount=default:myuser
# OR for namespace-specific access:
kubectl create rolebinding myuser-admin -n my-namespace --clusterrole=admin --serviceaccount=default:myuser

# Create token secret
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: myuser-token
  namespace: default
  annotations:
    kubernetes.io/service-account.name: myuser
type: kubernetes.io/service-account-token
EOF

# Get token
kubectl get secret myuser-token -n default -o jsonpath='{.data.token}' | base64 -d
```

## Features

- Mobile-responsive web interface optimized for touch
- View pods, logs, events, and cluster resources
- Execute into containers
- Real-time updates via WebSocket
- Multi-cluster support (if configured)
- Plugin system for extensibility

## Documentation

- Official docs: https://headlamp.dev/docs/
- GitHub: https://github.com/headlamp-k8s/headlamp
