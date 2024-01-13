---
title: 'Unfork with ArgoCD'
description: 'Post provides strategies for customizing third-party software in Kubernetes without maintaining a fork, focusing on ArgoCD integration and management techniques'
pubDate: 'Jan 13 2024'
heroImage: '../../assets/blog/unfork-with-argocd/unfork-with-argocd.png'
---

With the help of existing freely available software, we can build a personal or company software stack without starting from scratch, but rather by standing on the shoulders of giants. This eliminates the need to constantly reinvent most parts of our systems. However, sometimes existing off-the-shelf solutions don't provide enough customization to achieve the required goals. In such cases, we face a dilemma: to fork or not to fork. There are reasons for either choice, but today we will explore the "unfork" approach. We will investigate available options with examples and compare them afterward.

All examples are available in [companion repo](https://github.com/alezkv/unfork-with-argocd/).

### Assumptions regarding the environment and the goal

- Kubernetes - extensible API server as universal control plane
- ArgoCD - GitOPS controller which monitoring Git repos and apply objects to k8s
- Any third-party off-the-shelf software

Goal: We need to add a specific Kubernetes (k8s) resource to the ArgoCD application when the source of the application is managed by a third party, without managing a fork of that software.

In all the described cases, you can either add or overwrite the entire resource. More granular patching is not always available; I'll note this later.

### Flavors of software distribution

Here is a list of ways to distribute software that occur in the wild, along with corresponding examples.

- plain kubernetes manifests: [ArgoCD](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#multi-tenant)
- kustomize [Kube Prometheus](https://github.com/prometheus-operator/kube-prometheus/blob/main/kustomization.yaml)
- helm chart [Traefik Ingress](https://github.com/traefik/traefik-helm-chart)

[ArgoCD has its own set of supported sources for applications](https://argo-cd.readthedocs.io/en/stable/user-guide/application_sources/) as such:
- Kustomize applications
- Helm charts
- A directory of manifests

### How does Renovate fit in here

It is a good practice to keep software up to date. To track changes in upstream software, we can utilize automatic dependency tracking systems such as [Dependabot](https://github.com/dependabot) or [Renovate](https://github.com/renovatebot/renovate). This is a broad topic and requires a separate article to be covered. If you would like to read about it, please vote in the comments section below.

### Outline

- Multiple Sources for an Application
- Umbrella chart
- Kustomize them all

### Multiple Sources for an Application

Recent version (2.6) of ArgoCD supports spec.sources (plural) on an Application instead of spec.source. This allows you to specify multiple sources, and if they produce the same resource (same group, kind, name, and namespace), the last source to produce the resource will take precedence.

Pros:
- Because this feature is part of the ArgoCD Application definition, it supports all available ArgoCD application sources out of the box

Cons:
- Only complete resource overwrites are possible
- This is a beta feature. The UI and CLI still generally behave as if only the first source is specified
- [Rollback is not supported for applications with multiple sources](https://github.com/argoproj/argo-cd/blob/cd4fc97c9dee7b69721bbb577a4f50ba897399c5/ui/src/app/applications/components/application-details/application-details.tsx#L802)

Here is an example of using a Helm Chart from upstream with our custom values.yaml from Git, along with resources from plain manifests overwriting on top.

`$ cat apps/_installed/multiple-sources.yaml`
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multiple-sources
  namespace: argocd
spec:
  project: default
  sources:
    - chart: authentik
      repoURL: https://charts.goauthentik.io
      targetRevision: 2023.10.5
      helm:
        valueFiles:
          - $values/apps/multiple-sources/values.yaml
    - repoURL: https://github.com/alezkv/unfork-with-argocd
      targetRevision: HEAD
      ref: values
    - repoURL: https://github.com/alezkv/unfork-with-argocd
      path: apps/multiple-sources/resources/
      targetRevision: HEAD
  destination:
    server: "https://kubernetes.default.svc"
    namespace: multiple-sources
```

You can combine any sources supported by ArgoCD. For example, you could obtain external software from Kustomize, then apply a local Helm Chart, and finally apply a couple of plain manifests on top.

### Umbrella chart

This technique utilizes the dependency feature of Helm Chart. You can use multiple charts as dependencies and also embed their configurations within an umbrella chart.

You need to have the Application and the umbrella chart.

`$ tree apps`
```
apps
├── _installed
│   └── chart-umbrella.yaml
└── chart-umbrella
    ├── Chart.yaml
    ├── templates
    │   └── configmap.yaml
    └── values.yaml
```

`$ cat apps/_installed/chart-umbrella.yaml`
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: chart-umbrella
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://github.com/alezkv/unfork-with-argocd
      path: apps/chart-umbrella/
      targetRevision: HEAD
  destination:
    server: "https://kubernetes.default.svc"
    namespace: chart-umbrella
```

`$ cat apps/chart-umbrella/Chart.yaml`
```
apiVersion: v1
version: 1.0.0
name: my-podinfo

dependencies:
  - name: podinfo
    version: 6.5.4
    repository: https://stefanprodan.github.io/podinfo
```

`$ cat apps/chart-umbrella/values.yaml`
```
podinfo:  # Values for the sub-chart must be under its dependency name key
  replicaCount: 2
```

This setup will use the upstream chart, apply any specified values to it, and, on top of that, add resources from the umbrella chart's template directory.

### Kustomize them all

Kustomize alone deserves a dedicated series; let's try to stick with the unforked approach for now. The whole nature of it is to add, remove, or update configuration options without forking.

#### Hydrate chart with Kustomize

It's possible to render a Helm Chart with Kustomize. This approach allows for additional and highly tunable ways to handle Helm Charts. However, this feature isn't enabled by default and requires custom configuration. You can find more details on this matter [here](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/#kustomizing-helm-charts) and [here](https://kubectl.docs.kubernetes.io/references/kustomize/builtins/#_helmchartinflationgenerator_)

To use Kustomize, you'll need to create a kustomization.yaml file and point the ArgoCD Application to its directory.

`$ tree apps`
```
apps
├── _installed
│   └── kustomize.yaml
└── kustomize
    ├── cloudflare-api-token.yaml
    ├── cloudflare-issuer.yaml
    └── kustomization.yaml
```

`$ cat apps/_installed/kustomize.yaml`
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kustomize
  namespace: argocd
spec:
  project: default
  source:
    path: apps/kustomize
    repoURL: https://github.com/alezkv/unfork-with-argocd
    targetRevision: HEAD
  destination:
    namespace: kustomize
    server: https://kubernetes.default.svc
```

`$ cat apps/kustomize/kustomization.yaml`
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

helmCharts:
  - name: cert-manager
    repo: https://charts.jetstack.io
    releaseName: cert-manager
    namespace: kustomize
    version: v1.13.3
    includeCRDs: true
    valuesInline:
      installCRDs: true

resources:
  - cloudflare-api-token.yaml
  - cloudflare-issuer.yaml
```

In this setup, you can add or replace resources in an existing chart using resources:. This will work the same way as with multiple application sources, overwriting the 'same' resources. However, you can also utilize the full transformation capabilities of Kustomize for more precise resource manipulation. This could include [replacement](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/replacements/), [patches](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/), or [others methods](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/)

Don't forget that the source for resources in Kustomize could be remote Git repositories with their own Kustomize or plain Kubernetes manifests

### What to pick

Multiple sources in ArgoCD are great for merging separate configurations, best for complete resource overwrites. Umbrella charts, using Helm's dependencies, offer structured management of complex deployments, ideal for hierarchical configuration integration. Kustomize, with its detailed customization capabilities, excels in precise resource manipulation for nuanced adjustments.

### Final Thoughts

Kubernetes and ArgoCD offer tremendous flexibility and power for managing containerized applications, but this comes with the responsibility of having a well-defined strategy and vision. Without a clear direction, the complexity of these tools can become a hindrance rather than an advantage. Therefore, it's essential to strike a balance between leveraging the flexibility they provide and maintaining a clear and purposeful approach to application deployment and management
