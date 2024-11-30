# Polar Labs Standard Helm Deployment #

This repo contains 2 helm charts that we use as part of our starter kits to standardize a typical deploy. Currently the repo assumes a Digital Ocean managed Kubernetes, but the technologies used (nginx-ingress and cert-manager) can be applied to pretty much any Kubernetes Cluster (though for something like Amazon you might consider AWS Load Balancer Controller).

# Setup

We have created 2 guides for this starterkit that you can follow to understand this starterkit and how to get your environment set up to use it:

1. [How our Helm StarterKit works](https://polarlabs.ca/stories/how-our-helm-starterkit-works)
2. [How to configure Digital Ocean Kubernetes with cert-manager and external-dns](https://polarlabs.ca/stories/digital-ocean-kubernetes-cert-manager-external-dns)

## Handy plugins we recommend

- https://krew.sigs.k8s.io/docs/user-guide/setup/install/
- https://github.com/corneliusweig/ketall
- https://pod-lens.guoxudong.io/

# Using the pl-standard-chart

This part is fairly easy as well. Create a chart in your application with `helm create your-chart`.

Inside that chart, clear out the templates folder and add the pl-standard-chart as a dependency like this:

```
apiVersion: v2
name: test-chart
description: A hello world type chart to test the library in this repo

type: application

version: 0.1.0
appVersion: "0.1.0"

dependencies:
  - name: pl-standard-chart
    version: 1.0.0
    repository: https://helm.polarlabs.ca

```

Then run `helm dependency update ./your-chart` and it will install a copy of the chart into the `charts/` folder of your chart (I know chart is getting redundant but I didn't design this). We recommend git-ignoring this folder unless you have your own charts in there, then maybe just gitignore the pl chart.

From there as you can see in the test repo you can make a single file (named anything you want) in your `templates/` directory that pulls in the 4 main files:

```
{{- include "pl-standard-chart.namespace" . }}

---

{{- include "pl-standard-chart.secrets" . }}

---

{{- include "pl-standard-chart.deployment" . }}

---

{{- include "pl-standard-chart.service" . }}

---

{{- include "pl-standard-chart.nginx-cert-manager-ingress" . }}

---
```

Lastly you need a `Values.yaml` file that looks like this:

```
namespace: hello-world
appName: hello-world
replicas: 1
imageLocation: hashicorp/http-echo
resources:
  limits:
    memory: "128Mi"
  requests:
    cpu: "50m"
    memory: "64Mi"
containerArgs:
  - -text=echo1
containerPorts:
  - name: web-ui
    port: 80
    targetPort: 5678
envVars:
  TEST: test
secrets:
  - name: do-creds
    type: Opaque
    data:
      TEST: test
clusterIssuerName: letsencrypt-staging
tlsSecretName: hello-world-tls
# disableExternalDNS: true
# disableContentSizeChecksNginx: true
hosts:
  - domainName: test.polarlabs.ca
    ingressPaths:
      - path: /
        type: Prefix
        servicePort: 80
  - domainName: test2.polarlabs.ca
    ingressPaths:
      - path: /
        type: Prefix
        servicePort: 80
# persistentVolumeClaims:
#   - name: test-pvc
#     storage: 5Gi
#     mountPath: /data
```

The following items are optional and have defaults:

- replicas
- restartPolicy
- imagePullPolicy
- resources

You can test it with `helm install your-chart ./path/to/chart/folder --dry-run` and make sure it looks like what you expect. Then remove `--dry-run` to actually install it.

**NOTE: It is good practise to use the staging issuer until you're ready to deploy it for sure because there is a rate limit on the production issuer** 

**NOTE: By default, we do not include resource requests and limits in the manifest if they are not specified in the `values.yaml` file. This means the container will not have any resource requests or limits set, causing Kubernetes to treat it as a low priority pod. It will be the first to be evicted or restricted in CPU usage, but it will still be scheduled. This setup is ideal for preview environments where resource starvation is acceptable, but scheduling is important. For production deployments, you must set appropriate resource values in the `values.yaml` file.**

# How the pipeline works

We are using [Chartmuseum](https://github.com/helm/chartmuseum) to host our charts which we deploying using pl-standard-deployment charts into our own kube cluster. The repo is located currently at https://helm.polarlabs.ca. If you would like to host your own, we include the chart for Chartmuseum in this repo and you can deploy to your own cluster.

This repo automates deployment of updates to the standard deployment charts with a simple script in a bitbucket pipeline. It simply packages the file and uploads to Chartmuseum via curl, and can be easily extended to update many charts to Chartmuseum.


# NOTES #

- If you use a persistent storage claim with Digital Ocean you can only have 1 replica because DO volumes only support ReadWriteOnce. You must also manually delete PVCs after their creation, they don't go down with the deployment. They should go down with`helm uninstall` though.