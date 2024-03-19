---
title: 'Kubernetes, PVC, Restic, and ketchup(k8up)'
description: '???'
pubDate: 'Mar 19 2024'
heroImage: '../../assets/blog/unfork-with-argocd/unfork-with-argocd.png'
---

I'm using Bitwarden as password manager with Vaultwarden as server implementation. And when I migrate this small but hugely valuable piece of data into homelab Kubernetes I decide to cover it with a proper disater recovery. As part of migration I decide to backup/restore procedure. This should help with data movement as well as validation of recovery procedure.

But before we jump into implementation detail let's look up on what we want to archive and brifely describe tooling.

Vaultwarden running as a pod in Kubernetes cluster. ArgoCD responsible for provisioning all k8s resources from single source of thougth. Data stored within PVC. I'm running this homelab on VPC so as a result it's lack all "cloud" goodies like persistence on EBS or smt similar. In order to have good sleep I need to be sure that all my password are safe in case of emergency. So I decide to go with sligthely modifed version of 3-2-1 backup schema: one copy to local minio deployment, other copy to remote storage.

# k8up 101

K8up it's Kubernetes Backup Operator. It's CNCF Sandbox Open Source project wich is using another briliant open source software such Restic for handeling backups and working with remote storage.
