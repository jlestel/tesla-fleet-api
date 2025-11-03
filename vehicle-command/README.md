# Tesla Vehicle Command Helm Chart

This Helm chart deploys the Tesla Vehicle Command proxy server for interacting with Tesla Fleet API.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- A Tesla Developer account with Fleet API access

## Installation

### Option 1: Auto-Generate Secrets (Recommended for new installations)

The chart can automatically generate the required TLS certificates and Tesla private key during installation:

```bash
helm install vehicle-command ./vehicle-command
```

After installation, follow the onboarding instructions displayed in the NOTES to:

1. Retrieve your generated public key
2. Register it with Tesla Fleet API at https://developer.tesla.com/
3. Wait for Tesla approval

To retrieve the public key later:

```bash
kubectl logs -n <namespace> job/vehicle-command-generate-secrets
```

### Option 2: Provide Your Own Secrets

If you already have Tesla Fleet API credentials, disable auto-generation and provide your secrets:

1. Create a `custom-values.yaml` file:

```yaml
autoGenerateSecrets:
  enabled: false

extraSecrets:
  vehicle-command:
    tls.key: |
      -----BEGIN PRIVATE KEY-----
      Your TLS private key here
      -----END PRIVATE KEY-----
    cert.pem: |
      -----BEGIN CERTIFICATE-----
      Your TLS certificate here
      -----END CERTIFICATE-----
    private.pem: |
      -----BEGIN PRIVATE KEY-----
      Your Tesla private key here
      -----END PRIVATE KEY-----
```

2. Install the chart:

```bash
helm install vehicle-command ./vehicle-command -f custom-values.yaml
```

## Configuration

### Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `autoGenerateSecrets.enabled` | Enable automatic secret generation | `true` |
| `autoGenerateSecrets.image.repository` | Image used for secret generation | `tesla/vehicle-command` |
| `autoGenerateSecrets.image.tag` | Image tag for secret generation | `0.3.0` |
| `autoGenerateSecrets.tlsCN` | Common Name for TLS certificate | `vehicle-command.local` |
| `autoGenerateSecrets.teslaKeyName` | Tesla key name for tesla-keygen | `vehicle-command` |
| `extraSecrets` | Manual secrets configuration | `{}` |
| `service.type` | Kubernetes service type | `ClusterIP` |
| `service.port` | Service port | `4443` |
| `ingress.enabled` | Enable ingress | `false` |

### Environment Variables

The following environment variables are configured by default:

- `TESLA_VERBOSE`: Enable verbose logging
- `TESLA_HTTP_PROXY_TLS_CERT`: Path to TLS certificate
- `TESLA_HTTP_PROXY_TLS_KEY`: Path to TLS private key
- `TESLA_KEY_FILE`: Path to Tesla private key
- `TESLA_HTTP_PROXY_PORT`: Proxy server port
- `TESLA_HTTP_PROXY_HOST`: Proxy server host

## How It Works

### Auto-Generation Process

When `autoGenerateSecrets.enabled` is `true`:

1. A pre-install Kubernetes Job is created with necessary RBAC permissions
2. The job uses the `tesla/vehicle-command` container to run:
   - `openssl` to generate a self-signed TLS certificate and key
   - `tesla-keygen create` to generate a Tesla private/public key pair
3. The job creates a Kubernetes Secret named `vehicle-command` with all three files
4. The deployment mounts this secret to provide credentials to the proxy server
5. The public key is displayed in the job logs for registration with Tesla

### Secret Structure

The `vehicle-command` secret contains three files:

- `tls.key`: TLS private key for HTTPS communication
- `cert.pem`: TLS certificate for HTTPS communication
- `private.pem`: Tesla private key for signing commands to vehicles

## Upgrading

When upgrading with `autoGenerateSecrets.enabled`:

- If the `vehicle-command` secret already exists, it will NOT be regenerated
- To regenerate secrets, delete the existing secret first:

```bash
kubectl delete secret vehicle-command -n <namespace>
helm upgrade vehicle-command ./vehicle-command
```

## Uninstallation

```bash
helm uninstall vehicle-command
```

Note: The `vehicle-command` secret is NOT automatically deleted. To remove it:

```bash
kubectl delete secret vehicle-command -n <namespace>
```

## Tesla Fleet API Onboarding

After installing with auto-generated secrets:

1. **Get your public key:**
   ```bash
   kubectl logs -n <namespace> job/vehicle-command-generate-secrets | grep -A 50 "public key"
   ```

2. **Register with Tesla:**
   - Go to https://developer.tesla.com/
   - Navigate to your application
   - Add the public key in the Fleet API settings

3. **Wait for approval:**
   - Tesla will review and approve your key
   - This process may take some time

4. **Verify connectivity:**
   ```bash
   kubectl port-forward -n <namespace> svc/vehicle-command 4443:4443
   curl https://localhost:4443/api/1/vehicles
   ```

## Troubleshooting

### Secret generation job failed

Check the job logs:
```bash
kubectl logs -n <namespace> job/vehicle-command-generate-secrets
```

Common issues:
- Insufficient RBAC permissions: Verify the ServiceAccount has the correct Role
- Image pull errors: Verify the `tesla/vehicle-command` image is accessible

### Deployment can't find secrets

Verify the secret exists:
```bash
kubectl get secret vehicle-command -n <namespace>
```

If missing, either:
- Wait for the generation job to complete
- Manually create the secret with your credentials

### Can't connect to Tesla API

1. Verify your public key is registered with Tesla
2. Check that Tesla has approved your key
3. Verify the deployment is running:
   ```bash
   kubectl get pods -n <namespace>
   kubectl logs -n <namespace> <pod-name>
   ```

## Support

For issues with:
- This Helm chart: Open an issue in this repository
- Tesla Fleet API: Visit https://developer.tesla.com/
- Vehicle Command proxy: See https://github.com/teslamotors/vehicle-command
