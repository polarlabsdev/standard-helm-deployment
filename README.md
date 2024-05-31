# Polar Labs Standard Helm Deployment #

This repo contains 2 helm charts that we use as part of our starter kits to standardize a typical deploy. Currently the repo assumes a Digital Ocean managed Kubernetes, but the technologies used (nginx-ingress and cert-manager) can be applied to pretty much any Kubernetes Cluster (though for something like Amazon you might consider AWS Load Balancer Controller).

# Setup

[This guide from Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes) is the main guide you want to follow, but you will need to supplement with others along the way as this guide is a little old and uses raw kube templates instead of helm which makes maintenance trickier.

The first part of the guide is setting up a test container for the ingress you're going to install. That is already provided in the "test-chart" chart in this repo using our custom helm templates.

When you get to the section about installing the nginx-ingress controller, **read the entire section but don't follow those instructions**. Instead use [this other guide from Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-on-digitalocean-kubernetes-using-helm). However! don't use the install command provided in that article. As you read in the first article, there are annotations you need to work with Digital Ocean that aren't provided in the standard nginx-ingress install and for some reason aren't mentioned in the second digital ocean article. However, [this article](https://www.shebanglabs.io/set-up-nginx-ingress-proxy-protocol-on-digitalocean/) tells you how to correctly install it with helm. We have slightly modified their command to include the workaround domain mentioned in the first DO article:

```
helm install nginx-ingress ingress-nginx/ingress-nginx \
--set controller.publishService.enabled=true \
--set-string controller.config.use-forward-headers=true,controller.config.compute-full-forward-for=true,controller.config.use-proxy-protocol=true  \
--set controller.service.annotations."service\.beta\.kubernetes\.io/do-loadbalancer-enable-proxy-protocol=true" \
--set controller.service.annotations."service\.beta\.kubernetes\.io/do-loadbalancer-hostname=kube-workaround\.polarlabs\.ca"
```

Obviously you need to switch the domain to what works for you.

Next up is cert manager. Again, read the entire section but we recommend installing from [the official cert-manager docs](https://cert-manager.io/docs/installation/helm/). The rest of the installation is valid in the first DO article, and we have copies of the templates you'll need under the `_support-templates/cert-manager/` folder in this repo. Just swap the values to match what you need them to be and follow the instructions from the article.

Lastly, follow [this article](https://www.digitalocean.com/community/tutorials/how-to-automatically-manage-dns-records-from-digitalocean-kubernetes-using-externaldns) in order to install externaldns into your cluster. This part is very easy. We have included a copy of the values.yaml in `_support-templates/external-dns/` that you'll need to go with the article, but you'll have to use your own DO API token.

We have also added a couple hacks to make it so one can disable external-dns on specific ingresses by adding `disableExternalDNS: true` to your values.yaml file (see the example below). There is also a conflict between external-dns and cetr-manager where external-dns tries to create domains for ingresses created by cert-manager. To work around this, we only run external-dns on ingresses that include the cert-manager annotation - aka only ones we have edited ourselves. This does however mean, you must request a cert for an ingress to get external-dns support. This happens by default with our ingress file though.

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

You can test it with `helm install your-chart ./path/to/chart/folder --dry-run` and make sure it looks like what you expect. Then remove `--dry-run` to actually install it.

**NOTE: It is good practise to use the staging issuer until you're ready to deploy it for sure because there is a rate limit on the production issuer** 

# How the pipeline works

We are using [Chartmuseum](https://github.com/helm/chartmuseum) to host our charts which we deploying using pl-standard-deployment charts into our own kube cluster. The repo is located currently at https://helm.polarlabs.ca. If you would like to host your own, we include the chart for Chartmuseum in this repo and you can deploy to your own cluster.

This repo automates deployment of updates to the standard deployment charts with a simple script in a bitbucket pipeline. It simply packages the file and uploads to Chartmuseum via curl, and can be easily extended to update many charts to Chartmuseum.


# NOTES #

- If you use a persistent storage claim with Digital Ocean you can only have 1 replica because DO volumes only support ReadWriteOnce. You must also manually delete PVCs after their creation, they don't go down with the deployment. They should go down with`helm uninstall` though.