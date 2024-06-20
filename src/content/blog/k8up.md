---
title: 'Backup Kubernetes PVC with Restic, and ketchup(k8up)'
description: 'Discover how to implement a robust disaster recovery strategy using Kubernetes. This comprehensive guide covers the steps to set up reliable backup and restore processes with K8up and Restic, ensuring the security and availability of your data. Enhance your data protection and resilience with this step-by-step tutorial.'
pubDate: 'Jun 20 2024'
heroImage: '../../assets/blog/k8up/k8up.webp'
---

I am using Bitwarden as a password manager with Vaultwarden as the server implementation. When migrating this valuable data into my homelab Kubernetes setup, I decided to implement proper disaster recovery. As part of the migration, I planned to use a backup/restore procedure to facilitate data movement and validate the recovery process.

Before diving into the implementation details, let's review our goals and briefly describe the tooling.

Vaultwarden is running as a pod in a Kubernetes cluster. ArgoCD is responsible for provisioning all Kubernetes resources from a single source of truth. Data is stored within a Persistent Volume Claim (PVC). I'm running this homelab on a Virtual Private Cloud (VPC), so it lacks all the "cloud" features like persistence on EBS or similar services. To ensure peace of mind, I need to be sure that all my passwords are safe in case of an emergency. Therefore, I decided to go with a slightly modified version of the 3-2-1 backup schema: one backup to a local MinIO deployment, and another backup to remote S3-compatible storage.

Setting up MinIO is beyond the scope of this article, but you can check [this Gist](https://gist.github.com/alezkv/ac2280dcae300f24495ebb54d44d6d98)

# k8up 101

K8up is a Kubernetes Backup Operator. It's a CNCF Sandbox Open Source project that uses other brilliant open-source software like Restic for handling backups and working with remote storage. It uses custom resources to define backup, restore, and scheduling tasks.

K8up scans namespaces for matching Persistent Volume Claims (PVCs), creates backup jobs, and mounts the PVCs for Restic to back up to the configured endpoint.

To create a backup with [k8up](k8up.io), you define a Backup object in YAML, specifying details such as the backend storage and credentials. This configuration is then applied to your Kubernetes cluster using kubectl apply. For regular backups, you create a Schedule object, which outlines the frequency and other parameters for backup, prune, and check jobs.

Installation instruction can be found on the [official site](https://docs.k8up.io/k8up/how-tos/installation.html)

# Steps

1. Create an empty deployment: This will produce dummy data for the initial backup.
1. Create a Backup object: This will produce a Restic repository on the backend of your choice.
1. Backup existing data with Restic: This will create a new Restic snapshot and place all data into the backup store.
1. Clean up the deployment before restoring (optional: you can restore into a new PVC and point the deployment to it).
1. Create a Restore object: This will move data from the backup snapshot into the production PVC.
1. Create a Schedule object: This will automate the regular creation of backups.

## Create an empty deployment

I'll skip this section because your use case will probably be different from mine.

## Create a Backup object

K8up Backup object and Secret object with credentials for the backend.

```YAML
apiVersion: v1
kind: Secret
metadata:
  name: local-backup-repo
type: Opaque
stringData:
# password: change_to_strong_password_or_passhprase
```

```YAML
apiVersion: v1
kind: Secret
metadata:
  name: local-backup-creds
type: Opaque
stringData:
  username: minio
  password: minio123
```

```YAML
apiVersion: k8up.io/v1
kind: Backup
metadata:
  name: backup-dummy
spec:
    repoPasswordSecretRef:                  # (1)
      name: local-backup-repo
      key: password
    s3:                                     # (2)
      endpoint: http://minio.minio.svc:9000 # (3)
      bucket: backups/vaultwarden           # (4)
      accessKeyIDSecretRef:                 # (5)
        name: local-backup-creds
        key: username
      secretAccessKeySecretRef:             # (5)
        name: local-backup-creds
        key: password
```

Key parts of the manifest with environment variables used by Restic:
1. Restic repository password reference used to encrypt the backup: `RESTIC_PASSWORD`
2. Restic storage backend type.
3. Storage backend endpoint.
4. Backup path within the storage backend.
5. Credentials reference for the backend.

`RESTIC_REPOSITORY` could be constracted from (2),(3),(4) as such: `s3:http://minio.minio.svc:9000/backups/vaultwarden`

Applying these manifests will trigger the Backup procedure by the K8up controller. You have several means to [check the backup status](https://docs.k8up.io/k8up/how-tos/check-status.html). You should definitely do so because of a current [issue #910](https://github.com/k8up-io/k8up/issues/910) with K8up which incorrectly reports status into the Backup object itself.

Also, a Snapshot object in Kubernetes will be created as a mirror of the Restic snapshot.

After the issue is fixed, this will be the proper way to check the Backup status.

```bash
kubectl -n vaultwarden describe backups.k8up.io backup-dummy
```
```bash
Name:         backup-dummy
Namespace:    vaultwarden
...
Status:
  Conditions:
    Last Transition Time:  2024-06-19T08:21:37Z
    Message:               no container definitions found
    Reason:                NoPreBackupPodsFound
    Status:                True
    Type:                  PreBackupPodReady
    Last Transition Time:  2024-06-19T08:21:47Z
    Message:               job 'backup_backup-dummy' completed successfully
    Reason:                Finished
    Status:                False
    Type:                  Progressing
    Last Transition Time:  2024-06-19T08:21:47Z
    Message:               "backup_backup-dummy" has 1 succeeded, 0 failed, and 0 started jobs
    Reason:                Succeeded
    Status:                True
    Type:                  Completed
    Last Transition Time:  2024-06-19T08:21:47Z
    Message:               Deleted 2 resources
    Reason:                Succeeded
    Status:                True
    Type:                  Scrubbed
  Finished:                true
  Started:                 true
```

But for now, you must also check and parse Restic logs from the appropriate pod.

```bash
kubectl -n vaultwarden get po | grep dummy
```
```bash
kubectl -n vaultwarden logs <pod_name>
```

You can go deeper and list actual files within the snapshot with the Restic CLI. In this example, you need to get access to the backup storage backend.

1. Port-forward the MinIO port:
```bash
kubectl -n minio port-forward svc/minio 9000:9000
```

2. Prepare the environment for Restic:
```bash
export RESTIC_PASSWORD='<password@local-backup-repo>'
export RESTIC_REPOSITORY='s3:http://127.0.0.1:9000/backups/vaultwarden'
export AWS_ACCESS_KEY_ID=minio
export AWS_SECRET_ACCESS_KEY=minio123
```

3. Get a list of all snapshots:
```bash
restic snapshots
```

4. List files in the backup storage:
```bash
restic ls -l <snapshot_id>
```

The last step will show you how files are stored within the backup. We need this information to mimic K8up backup with the Restic CLI in the following steps. We need to know what common path prefix the files in the backup have. It should be constructed as such: `/data/<PVC_NAME>`. In my case, it was /data/vaultwarden.

## Backup existing data with Restic

The most tricky part of this step is to produce a correct Restic snapshot that can be restored by K8up. It requires properly forged paths for backed-up files and access to the storage backend.

Initially, I was trying to achieve this with Docker. But Restic failed with strange IO errors on backup. So, I switched to Podman. Let's assume that files are stored in `~/backup/vaultwarden` on my host. Here are the steps for creating a Restic snapshot with a file structure suited for K8up from local files.

1. Get access to the storage backend:
```bash
kubectl -n minio port-forward svc/minio 9000:9000
```

2. Set the environment for Restic as described earlier, altering the repository:
```bash
export RESTIC_PASSWORD='<password@local-backup-repo>'
export RESTIC_REPOSITORY='s3:http://host.containers.internal:9000/backups/vaultwarden'
export AWS_ACCESS_KEY_ID=minio
export AWS_SECRET_ACCESS_KEY=minio123
```

The trick here is to use `host.containers.internal` as the hostname. This name is provided for each Podman container and points to the host IP address. Docker has another name for that purpose: `host.docker.internal`.

3. Run the container with the proper mount and environments:
```bash
cd ~/backup/vaultwarden
podman run -it --rm --env-file <(env | egrep '^AWS_|RESTIC_') -v "$(pwd)":/data/vaultwarden --entrypoint=sh restic/restic
```

4. Backup files using the Restic CLI:
```bash
restic backup /data/vaultwarden
```

5. Get the snapshot ID

In output of previous command you should get created snapshot id. Or you can get all snapshot with:
```bash
restic snapshots
```

The output should have two snapshots. One with dummy data created by K8up and another one with actual data created with the help of Restic by you just now.

## Create a Restore object

By now, we have managed to put data into the backup storage. Let's create a Restore object to get this data into the PVC for production use.

Restore object mirrors backup and additionally specifies the restore method, which could be either a folder or S3.

```YAML
apiVersion: k8up.io/v1
kind: Restore
metadata:
  name: restore
spec:
  snapshot: <SNAPTHOST_ID>
  restoreMethod:
    folder:
      claimName: vaultwarden
  backend:
    repoPasswordSecretRef:
      name: local-backup-repo
      key: password
    s3:
      endpoint: http://minio.minio.svc:9000
      bucket: backups/vaultwarden
      accessKeyIDSecretRef:
        name: local-backup-creds
        key: username
      secretAccessKeySecretRef:
        name: local-backup-creds
        key: password
```

## Create a Schedule object

The final step in our journey will be setting up scheduling, which will combine and automate the following actions:
- backing up data frequently
- checking the integrity of backup storage
- maintaining an appropriate number of backup versions over time


```YAML
apiVersion: k8up.io/v1
kind: Schedule
metadata:
  name: local
spec:
  backend:
    s3:
      endpoint: http://minio.minio.svc:9000
      bucket: backups/vaultwarden
      accessKeyIDSecretRef:
        name: local-backup-creds
        key: username
      secretAccessKeySecretRef:
        name: local-backup-creds
        key: password
    repoPasswordSecretRef:
      name: local-backup-repo
      key: password
  backup:
    schedule: '3 * * * *'
    failedJobsHistoryLimit: 2
    successfulJobsHistoryLimit: 2
  check:
    schedule: '33 2 * * 1'
  prune:
    schedule: '33 2 * * *'
    retention:
      keepHourly: 14
      keepDaily: 14
      keepMonthly: 3
```

In conclusion, implementing a robust backup and recovery strategy for Vaultwarden in a Kubernetes setup is crucial for ensuring the security and availability of your password data. By leveraging the powerful capabilities of K8up and Restic, we can create a reliable and automated backup process that mitigates the risks associated with data loss.
