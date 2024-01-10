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

Goal: we need to add a specific k8s resource (as an example it whould be tls secret) to the ArgoCD application when source of the application managed by third-party. And do not manage fork of that software.

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

### Hydrate chart with Kustomize

It's possibe to render Helm Chart by Kustomize. This approach allow additional and vary tunable way to handle Helm Charts. This feature doesn't enabled by default thougs required custom configuration. You can check more details on that matter [here](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/#kustomizing-helm-charts) and [here](https://kubectl.docs.kubernetes.io/references/kustomize/builtins/#_helmchartinflationgenerator_)

### Outline

- Multiple Sources for an Application
-

### Multiple Sources for an Application

Recent version (2.6) ArgoCD support `spec.sources` (plural) on Application instead of `spec.source`. This allow to specify multiple sources and if they produce the same resource (same group, kind, name, and namespace), the last source to produce the resource will take precedence.

Because of the fact that this feature is a part of ArgoCD Application definition. It's support all avaliable for ArgoCD application sources out of the box. On a cone side is that only whole resource overwrite are possible.

Here is example of using Helm Chart from upstream with our values.yaml + resources on top of that
```
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: authentik
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
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


NOTE: such vast combinatoric options - is a bless and a curs of k8s/ArgoCD. You need to have a clear vision or you can stuck with all that complexity and spend time and effort on many avaliable options with small return.
