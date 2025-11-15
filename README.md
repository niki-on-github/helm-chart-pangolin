# Pangolin Helm Chart

> [!WARNING]  
> This project is a work in progress and is not yet ready for production use.

This Helm chart deploys Pangolin, a self-hosted tunnel and reverse proxy solution, on Kubernetes.

## Overview

Pangolin consists of three main components:

- **Pangolin**: The main application server (API, frontend, and WebSocket server)
- **Gerbil**: WireGuard tunnel server
- **Traefik**: Reverse proxy for routing traffic (runs as sidecar with Gerbil)

## Prerequisites

- Kubernetes cluster (1.24+)
- Helm 3.0+
- A domain name pointing to your cluster's LoadBalancer IP
- Storage provisioner for PersistentVolumes

## Installation

> **Note**: This chart is published to GitHub Pages. Make sure GitHub Pages is enabled in your repository settings (Settings → Pages → Source: GitHub Actions).

### Add the Helm Repository

```bash
helm repo add pangolin https://christian-deleon.github.io/helm-chart-pangolin
helm repo update
```

### Install the Chart

1. **Configure Values**

   Create a custom values file (e.g., `values.prod.yaml`):

   ```yaml
   pangolin:
     domain: "pangolin.example.com"
     email: "admin@example.com"
     
     config:
       server:
         secret: "your-secure-random-secret-here"

   gerbil:
     service:
       type: LoadBalancer
       annotations:
         # Add cloud provider specific annotations here
         # Example for AWS:
         # service.beta.kubernetes.io/aws-load-balancer-type: "nlb"

   storage:
     config:
       storageClass: "your-storage-class"
       size: 5Gi
     letsencrypt:
       storageClass: "your-storage-class"

   traefik:
     enabled: true
     additionalArguments:
       - "--certificatesresolvers.letsencrypt.acme.email=admin@example.com"
   ```

2. **Install the Chart**

   ```bash
   helm install pangolin pangolin/pangolin -f values.prod.yaml --namespace pangolin --create-namespace
   ```

   Or for development:

   ```bash
   helm install pangolin pangolin/pangolin -f values.dev.yaml --namespace pangolin --create-namespace
   ```

### Install from Local Chart

Alternatively, you can install directly from the repository:

```bash
git clone https://github.com/christian-deleon/helm-chart-pangolin.git
cd helm-chart-pangolin
helm install pangolin . -f values.prod.yaml --namespace pangolin --create-namespace
```

## Development with k3d

For local development and testing, you can use [k3d](https://k3d.io/) to run a local Kubernetes cluster. This is useful for testing Helm chart configurations and deployments, but has limitations for testing all Pangolin features.

### Development Prerequisites

- [k3d](https://k3d.io/) installed
- [just](https://github.com/casey/just) (optional, for convenience commands)

### Setup

1. **Create the k3d cluster:**

   ```bash
   just create-cluster
   ```

   Or manually:

   ```bash
   k3d cluster create pangolin --config k3d.yaml
   ```

   This creates a cluster with ports 80 and 443 mapped to your localhost.

2. **Install Pangolin:**

   First, add the Helm repository (if not already added):

   ```bash
   helm repo add pangolin https://christian-deleon.github.io/helm-chart-pangolin
   helm repo update
   ```

   Then install using justfile:

   ```bash
   just install
   ```

   Or manually:

   ```bash
   helm install pangolin pangolin/pangolin \
     --namespace pangolin \
     --create-namespace \
     --values values.dev.yaml \
     --kube-context k3d-pangolin
   ```

   Or install from local chart:

   ```bash
   git clone https://github.com/christian-deleon/helm-chart-pangolin.git
   cd helm-chart-pangolin
   helm install pangolin . \
     --namespace pangolin \
     --create-namespace \
     --values values.yaml \
     --values values.dev.yaml \
     --kube-context k3d-pangolin
   ```

3. **Access Pangolin:**

   Since `k3d.yaml` maps ports 80 and 443 to localhost, and most systems (including macOS) resolve `*.localhost` to `127.0.0.1`, you can access Pangolin at:

   ```text
   http://pangolin.localhost/auth/initial-setup
   ```

   The domain `localhost` is already configured in `values.dev.yaml` for this purpose.

   See the [Accessing Pangolin](#accessing-pangolin) section below for instructions on getting the setup token and completing the initial configuration.

### Limitations

⚠️ **Important**: Running Pangolin with k3d has limitations:

- **WireGuard tunnels cannot be tested** - The UDP ports (51820, 21820) for WireGuard are not exposed through k3d's port mapping. The service will be created, but external WireGuard clients cannot connect.

- **This setup is primarily for:**
  - Testing Helm chart configuration
  - Verifying deployment processes
  - Validating Traefik routing
  - Testing the web interface and API

- **For full feature testing**, deploy to a real Kubernetes cluster with proper LoadBalancer support and exposed UDP ports.

### Cleanup

To remove the local cluster:

```bash
just delete-cluster
```

Or manually:

```bash
k3d cluster delete pangolin
```

## Configuration

### Key Configuration Options

| Parameter | Description | Default |
|-----------|-------------|---------|
| `pangolin.domain` | Domain for accessing Pangolin | `pangolin.local` |
| `pangolin.email` | Admin email address | `admin@pangolin.local` |
| `pangolin.image.tag` | Pangolin image tag | `1.12.2` |
| `pangolin.config.server.secret` | Secret key for server | `changeme-generate-random-secret` |
| `gerbil.service.type` | Service type for Gerbil | `LoadBalancer` |
| `storage.config.size` | Size of config PVC | `1Gi` |
| `traefik.enabled` | Enable Traefik sidecar | `true` |

### Pangolin Configuration

The Pangolin configuration supports various options. See the [official documentation](https://docs.pangolin.net/) for details.

Key configuration sections in `values.yaml`:

```yaml
pangolin:
  config:
    gerbil:
      start_port: 51820
    app:
      log_level: "info"
    server:
      secret: "your-secret"
      cors:
        methods: ["GET", "POST", "PUT", "DELETE", "PATCH"]
    flags:
      require_email_verification: false
      disable_signup_without_invite: true
```

## Architecture

### Network Architecture

In the Docker Compose version, Traefik shares the network namespace with Gerbil using `network_mode: service:gerbil`. In Kubernetes, this is achieved by running Traefik as a sidecar container in the Gerbil pod, allowing them to share the same network namespace.

```text
┌─────────────────┐
│   LoadBalancer  │
│   (Gerbil Svc)  │
└────────┬────────┘
         │
         ├─ Port 80/443 (HTTP/HTTPS)
         ├─ Port 51820/UDP (WireGuard)
         └─ Port 21820/UDP (WireGuard)
         │
         ▼
┌────────────────────────┐
│   Gerbil Pod           │
│  ┌──────────────────┐  │
│  │ Gerbil Container │  │
│  └──────────────────┘  │
│  ┌──────────────────┐  │
│  │ Traefik Sidecar  │  │
│  └──────────────────┘  │
└────────┬───────────────┘
         │
         ▼
┌────────────────────┐
│   Pangolin Pod     │
│  ┌──────────────┐  │
│  │  Pangolin    │  │
│  │  - API:3000  │  │
│  │  - Int:3001  │  │
│  │  - Web:3002  │  │
│  └──────────────┘  │
└────────────────────┘
```

### Services

- **pangolin-pangolin**: ClusterIP service exposing Pangolin's three ports
  - Port 3000: API/WebSocket server
  - Port 3001: Internal API (used by Gerbil and Traefik)
  - Port 3002: Frontend (Next.js)

- **pangolin-gerbil**: LoadBalancer service exposing:
  - Port 80/443: HTTP/HTTPS traffic (handled by Traefik sidecar)
  - Port 51820/21820 UDP: WireGuard tunnels

### Storage

The chart creates two PersistentVolumeClaims:

1. **config**: Shared between Pangolin and Gerbil for:
   - Database storage (`/app/config/db`)
   - WireGuard keys (`/var/config/key`)

2. **letsencrypt**: Used by Traefik for:
   - SSL certificate storage (`/letsencrypt/acme.json`)

## Accessing Pangolin

After installation:

1. Wait for the LoadBalancer to get an external IP:

   ```bash
   kubectl get svc pangolin-gerbil -n pangolin
   ```

2. Update your DNS to point your domain to the LoadBalancer IP

3. Get the setup token from the Pangolin pod logs:

   ```bash
   kubectl logs -l app.kubernetes.io/component=pangolin -n pangolin | grep -A 3 "SETUP TOKEN"
   ```

   The token will be displayed in the logs when Pangolin starts for the first time. Look for output like:

   ```text
   === SETUP TOKEN GENERATED ===
   Token: <your-token-here>
   Use this token on the initial setup page
   ================================
   ```

4. Access the dashboard at:

   ```text
   https://pangolin.your-domain.com/auth/initial-setup
   ```

   Enter the setup token from step 3 to complete the initial configuration.

## Troubleshooting

### Check Pod Status

Using justfile commands:

```bash
just pods
just describe
just logs
```

Or manually:

```bash
kubectl get pods -n pangolin
kubectl describe pod pangolin-gerbil-xxx -n pangolin
kubectl logs pangolin-gerbil-xxx -c gerbil -n pangolin
kubectl logs pangolin-gerbil-xxx -c traefik -n pangolin
kubectl logs pangolin-pangolin-xxx -n pangolin
```

### Check Services

```bash
just svc
```

Or manually:

```bash
kubectl get svc -n pangolin
```

### Check PVCs

```bash
just pvc
```

Or manually:

```bash
kubectl get pvc -n pangolin
```

### View Logs

Convenient justfile commands for viewing logs:

```bash
just setup-token    # Get the setup token
just logs-pangolin  # View Pangolin logs
just logs-gerbil    # View Gerbil logs
just logs-traefik   # View Traefik logs
just logs           # View all logs
```

### Common Issues

1. **Traefik can't reach Pangolin**: Ensure the init container completed successfully and Pangolin is ready
2. **SSL certificate issues**: Check that your domain DNS is properly configured
3. **Storage issues**: Verify your storage class is available: `kubectl get storageclass`

## Upgrading

```bash
helm repo update
helm upgrade pangolin pangolin/pangolin -f values.prod.yaml --namespace pangolin
```

Or if installed from local chart:

```bash
helm upgrade pangolin . -f values.prod.yaml --namespace pangolin
```

## Uninstalling

```bash
helm uninstall pangolin --namespace pangolin
```

Note: PersistentVolumeClaims are not automatically deleted. To delete them:

```bash
kubectl delete pvc pangolin-config pangolin-letsencrypt -n pangolin
```

## License

This chart is provided as-is. Pangolin and its components are subject to their respective licenses.

## Links

- [Pangolin Documentation](https://docs.pangolin.net/)
- [Pangolin GitHub](https://github.com/fosrl/pangolin)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
