# Polar Labs Standard Helm Deployment #

This repo provides a standardized and automated way to deploy applications to Kubernetes. It includes a reusable library chart for applications and a pre-configured umbrella chart to bootstrap a full environment.

# How to Deploy Everything

The easiest way to get started is to use the root of this repository as your deployment chart. This will deploy both the necessary cluster infrastructure (like `ingress-nginx` and `cert-manager`) and a sample application in a single command, using the included charts.

### 1. Configure Your Deployment

All configuration is handled in a single file: `pl-cluster-setup-chart/values.yaml` and `pl-workload-chart/values.yaml`. These files are divided into two main sections:

-   `pl-cluster-setup-chart`: Contains all the settings for the base infrastructure. Here you will configure `cert-manager`, `external-dns`, and `ingress-nginx`.
-   `pl-workload-chart`: Contains all the settings for your application, such as the image to deploy, resource requests, and ingress rules.

Open these files and edit them to match your environment (e.g., set your domain names, cloud provider settings, etc.).

### 2. Install the Charts

Once your values files are configured, navigate to the root of this repository and run the following commands.

First, update the local chart dependencies:
```bash
helm dependency update ./pl-cluster-setup-chart
helm dependency update ./pl-workload-chart
```

Then, run the installation. This command will deploy the stack (in two steps):
```bash
helm install cluster-setup ./pl-cluster-setup-chart --namespace cluster-setup --create-namespace
helm install my-app ./pl-workload-chart --namespace my-app --create-namespace
```

This will bootstrap your cluster and deploy your application, all managed through version-controlled `values.yaml` files.

---

# Chart Details

## `pl-cluster-setup-chart`

This chart installs and configures the base services required for a modern Kubernetes environment. It manages:

-   **ingress-nginx:** An Ingress controller.
-   **cert-manager:** For automated TLS certificate management.
-   **external-dns:** To automatically manage DNS records.

It is designed to be deployed directly or as a dependency of a top-level chart.

## `pl-workload-chart`

This is a reusable library chart for deploying your applications. It provides templates for creating Deployments, Services, Ingresses, and other standard Kubernetes resources.

To use it in your own projects, you would add it as a dependency to your application's Helm chart and configure it via your `values.yaml`, similar to the example above.

---

# How the pipeline works

We are using [Chartmuseum](https://github.com/helm/chartmuseum) to host our charts which we deploy using pl-standard-deployment charts into our own kube cluster. The repo is located currently at https://helm.polarlabs.ca. If you would like to host your own, we include the chart for Chartmuseum in this repo and you can deploy to your own cluster.

This repo automates deployment of updates to the standard deployment charts with a simple script in a bitbucket pipeline. It simply packages the file and uploads to Chartmuseum via curl, and can be easily extended to update many charts to Chartmuseum.


# NOTES #

- If you use a persistent storage claim with Digital Ocean you can only have 1 replica because DO volumes only support ReadWriteOnce. You must also manually delete PVCs after their creation, they don't go down with the deployment. They should go down with`helm uninstall` though.

 # Considerations: #

- The test-chart from previous commits is removed for now. The chart capabilities can be tested from the root chart which triggers execution of 2 sub charts

- Cloud Infra level tasks are expected to be done manually e.g: creation of serviceAccounts in GCP or other clouds (which is reasonable because helm is not supposed to be provisioning cloud resources)