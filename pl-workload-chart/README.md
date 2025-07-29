# pl-workload-chart

A flexible, cloud-agnostic Helm chart for deploying application workloads on Kubernetes. Supports GCP, generic clusters, and is designed for easy extension to other cloud providers.

## Features
- Profile-based deployment: GCP, generic, and future cloud profiles
- Modular resources: Ingress, Gateway API, BackendConfig, ServiceAccount, etc.
- Enable/disable options for all major features
- DRY configuration: unified health check settings for both Gateway API and BackendConfig

## Quick Start
1. **Choose a profile:**
   - `profile: gcp` for Google Cloud (GKE)
   - `profile: generic` for standard Kubernetes clusters
   - Future profiles can be added for other clouds

2. **Configure values:**
   - Set your app name, namespace, image, ports, and other deployment details in `values.yaml` or `values.template.yaml`.
   - Use the `gcp` block for GCP-specific features (Ingress, BackendConfig, Gateway API, ServiceAccount annotations, etc.)

3. **Enable/disable features:**
   - Toggle resources like Ingress, BackendConfig, Gateway API (HTTPRoute, ReferenceGrant, HealthCheckPolicy) using the `enabled` flags in the values file.
   - Example:
     ```yaml
     gcp:
       ingress:
         enabled: false
       backendConfig:
         enabled: false
       gatewayapi:
         enabled: true
     ```

4. **Unified health checks:**
   - Configure health check settings once under `gcp.gatewayapi.healthCheckPolicy` and they will be used for both Gateway API and BackendConfig resources.

## Usage
- Render templates with Helm:
  ```bash
  helm template -f values.yaml ./pl-workload-chart
  ```
- Install to your cluster:
  ```bash
  helm install my-app ./pl-workload-chart -f values.yaml
  ```

## File Structure
- `templates/` — All Kubernetes manifests, organized by feature and cloud profile
- `values.yaml` / `values.template.yaml` — Main configuration files
- `README.md` — This guide


## Considerations
- For the time being, the gateway api resource is not added for generic profile.

- Gateway and it’s namespace creation are currently not added here - because gateway is not meant to be created everytime we run this helm chart (but it is needed for broader opensource perspective - so definitely in the plans)
