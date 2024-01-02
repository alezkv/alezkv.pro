---
title: 'Unfork with ArgoCD'
description: 'How to add or change k8s resources using ArgoCD avoiding forking of third-party software.'
pubDate: 'Jan 3 2024'
heroImage: '../../assets/blog/container-is-a-process/container-is-a-process.png'
---

With a help of existed freely avaliable software we can build personal or company software stack standing on a holders of a giants, constantly not reinventing most of the parts of our systems. But somethimes existed of the shelf solutions doesn't provide enought customization to archive reuired goals. In this case we face a dillema: fork or not to fork. There is a reasons for either of them. But today we will explore "unfork" approach.

### Assumption about environment and the goal

- Kubernetes
- ArgoCD
- Any third-party off the shelf software

Goal: we need to add a specific k8s resource (as an example it whould be tls secret) to the ArgoCD application when source of the application managed by third-party. And do not namage fork of that software.

### Flaivours of software distribution

- plain kubernetes manifests
- kustomize
- helm chart



TODO: note about best paractive for updating software with a help of renovate
