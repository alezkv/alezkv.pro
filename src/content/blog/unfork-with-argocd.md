---
title: 'Unfork with ArgoCD'
description: 'How to add or change k8s resources using ArgoCD avoiding forking of third-party software.'
pubDate: 'Jan 8 2024'
heroImage: '../../assets/blog/container-is-a-process/container-is-a-process.png'
---

With a help of existed freely avaliable software we can build personal or company software stack not from groudn up. But get standing on a holders of a giants. This eliminate constantly reinventing most of the parts of our systems. But somethimes existed of the shelf solutions doesn't provide enought customization to archive reqired goals. In this case we face a dillema: fork or not to fork. There is a reasons for either of them. But today we will explore "unfork" approach. We will explore avaliable options with examples and compare them afterwards.

### Assumption about environment and the goal

- Kubernetes
- ArgoCD
- Any third-party off the shelf software

Goal: we need to add a specific k8s resource to the ArgoCD application when source of the application managed by third-party. And do not manage a fork of that software.

In all described cases you can add or overwrite whole resource. More granular patching is not always avaliable, I'll note about this later.

### Flaivours of software distribution

Here is the list of a way to distribute software that occur in the wild. With corresponding examples.

- plain kubernetes manifests: [ArgoCD](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#multi-tenant)
- kustomize [Kube Prometheus](https://github.com/prometheus-operator/kube-prometheus/blob/main/kustomization.yaml)
- helm chart [Traefik Ingress](https://github.com/traefik/traefik-helm-chart)

[ArgoCD have their own set of supported sources for application](https://argo-cd.readthedocs.io/en/stable/user-guide/application_sources/) as such:

- Kustomize applications
- Helm charts
- A directory of manifests

### How Renovate fits here

It's is a good practice to get updates for used software. In order to track changes of upstream we can utilize uptomatic dependancy tracking systems such as [Dependabot](https://github.com/dependabot) or [Rennovate](https://github.com/renovatebot/renovate). This is a broad topic and it require separate article to be covered. If you want to read about that, please vote in comments section down below.

### Outline

- Multiple Sources for an Application
- Umbrella chart
- Kustomize them all

### Multiple Sources for an Application

Recent version (2.6) ArgoCD support `spec.sources` (plural) on Application instead of `spec.source`. This allow to specify multiple sources and if they produce the same resource (same group, kind, name, and namespace), the last source to produce the resource will take precedence.

Pros:
- Because of the fact that this feature is a part of ArgoCD Application definition. It's support all avaliable for ArgoCD application sources out of the box

Cons:
- Only whole resource overwrite are possible
- This is beta feature. The UI and CLI still generally behave as if only the first source is specified
- [Rollback is not supported for apps with multiple sources](https://github.com/argoproj/argo-cd/blob/cd4fc97c9dee7b69721bbb577a4f50ba897399c5/ui/src/app/applications/components/application-details/application-details.tsx#L802)

Here is example of using Helm Chart from upstream with our custom values.yaml from Git. Plus resources from plain manifests owerwrite on top.
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: authentik
  namespace: argocd
spec:
  project: default
  sources:
    - chart: authentik
      repoURL: https://charts.goauthentik.io
      targetRevision: 2023.10.5
      helm:
        valueFiles:
          - $values/apps/authentik/values.yaml
    - repoURL: https://github.com/alezkv/unfork-argocd
      targetRevision: HEAD
      ref: values
    - repoURL: https://github.com/alezkv/unfork-argocd
      path: apps/authentik/resources/
      targetRevision: HEAD
  destination:
    server: "https://kubernetes.default.svc"
    namespace: authentik
```

You can combine any suported by ArgoCD sources. So for example you could get external software from kustomize, then apply local Helm Chart and finally trow on top couple plain manifests.

### Umbrella chart

This tecknik utilyze dependency feature of Helm Cart. You can use multiple charts as a dependacyes as well as bake it's configuration within umrella chart.

You need to have `Application` and umbrella Chart.

`$ tree apps`
```
apps
├── _installed
│   └── podinfo-umbrella.yaml
└── podinfo-umbrella
    ├── Chart.yaml
    ├── templates
    │   └── configmap.yaml
    └── values.yaml
```

`$ cat apps/_installed/podinfo-umbrella.yaml`
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: podinfo-umbrella
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://github.com/alezkv/c.alezkv.net
      path: apps/podinfo-umbrella/
      targetRevision: HEAD
  destination:
    server: "https://kubernetes.default.svc"
    namespace: podinfo-umbrella
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    automated:
      prune: true
      selfHeal: true
```

`$ cat apps/podinfo-umbrella/Chart.yaml`
```
apiVersion: v1
version: 1.0.0
name: my-podinfo

dependencies:
  - name: podinfo
    version: 6.5.4
    repository: https://stefanprodan.github.io/podinfo
```

`$ cat apps/podinfo-umbrella/values.yaml`
```
podinfo: # values for sub chart must be under it's name key
  replicaCount: 2
```

This set up will ... 

### Kustomize them all

Kustomize alone deserves dedicated series, let's try to stick with unfork approach by now. Kustimize natively supported source for Application by ArgoCD. And the whole nature of it is to add, remove or update configuration options without forking.

#### Hydrate chart with Kustomize

It's possibe to render Helm Chart by Kustomize. This approach allow additional and vary tunable way to handle Helm Charts. This feature doesn't enabled by default thougs required custom configuration. You can check more details on that matter [here](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/#kustomizing-helm-charts) and [here](https://kubectl.docs.kubernetes.io/references/kustomize/builtins/#_helmchartinflationgenerator_)

In order to be able to use kustomize we will need to create `kustomization.yaml` and point ArgoCD Application to it's directory.

`$ tree apps`
```
apps
├── _installed
│   └── cert-manager.yaml
└── cert-manager
    ├── cloudflare-api-token.yaml
    ├── cloudflare-issuer.yaml
    └── kustomization.yaml
```

`$ cat apps/_installed/cert-manager.yaml`
```
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
spec:
  project: default
  source:
    path: apps/cert-manager
    repoURL: https://github.com/alezkv/c.alezkv.net
    targetRevision: HEAD
  destination:
    namespace: cert-manager
    server: https://kubernetes.default.svc
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    automated:
      prune: true
      selfHeal: true
```

`$ cat apps/cert-manager/kustomization.yaml`
```
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

helmCharts:
  - name: cert-manager
    repo: https://charts.jetstack.io
    releaseName: cert-manager
    namespace: cert-manager
    version: v1.13.3
    includeCRDs: true
    valuesInline:
      installCRDs: true

resources:
  - cloudflare-api-token.yaml
  - cloudflare-issuer.yaml
```

In this setup you can add/replace resources to existed chart using `resources:`. This will work the same way as with multiple application sources by owerwriting "same" resources.
But you can also utilyze whole transformation capabilityes of Kustomize, to more precise resource manipulation. This could be [replacement](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/replacements/) or [patches](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/) or [others](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/)

Don't forget that as source for resources in kustomize could be remote Git repositoryes with theris own Kustomize or plain Kubernetes manifests.


NOTE: such vast combinatoric options - is a bless and a curs of k8s/ArgoCD. You need to have a clear vision or you can stuck with all that complexity and spend time and effort on many avaliable options with small return.
