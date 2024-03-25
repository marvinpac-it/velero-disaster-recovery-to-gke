# Provision a GKE Cluster, Install Velero, and Restore from bucket

This repo was built using the [Provision a GKE Cluster tutorial](https://developer.hashicorp.com/terraform/tutorials/kubernetes/gke), containing Terraform configuration files to provision an GKE cluster on GCP.

## Deploy GKE cluster
```
# Deploy GKE cluster with terraform
$ git clone git@github.com:marvinpac-it/terraform-gke-cluster.git
$ cd terraform-gke-cluster
$ terraform apply
> yes
# Install kubectl config for reaching the cluster (make sure you're logged on to gcloud)
$ gcloud container clusters get-credentials $(terraform output -raw kubernetes_cluster_name) --region $(terraform output -raw region)
# Verify cluster
$ k get nodes
NAME                                                  STATUS   ROLES    AGE   VERSION
gke-archives-storage-archives-storage-20cce987-49kb   Ready    <none>   18m   v1.27.8-gke.1067004
gke-archives-storage-archives-storage-2506a5d7-mmpx   Ready    <none>   18m   v1.27.8-gke.1067004
gke-archives-storage-archives-storage-9c510a3d-025l   Ready    <none>   18m   v1.27.8-gke.1067004
```
## Deploy Velero on GKE cluster (make sure you have velero credentials in home directory first)
```
$ velero install \
    --provider gcp \
    --plugins velero/velero-plugin-for-csi:v0.6.1,velero/velero-plugin-for-gcp:v1.8.1 \
    --bucket marvinpac-velero-replica \
    --secret-file ~/credentials-velero-gke \
    --use-node-agent \
    --features=EnableCSI
$ velero backup get
NAME                                            STATUS            ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
daily-backup-all-but-system-20240321030049      Completed         0        0          2024-03-21 03:00:49 +0000 UTC   28d       default            <none>
daily-backup-all-but-system-20240320030019      Completed         0        0          2024-03-20 03:00:19 +0000 UTC   27d       default            <none>
daily-backup-all-but-system-20240319030001      Completed         0        0          2024-03-19 03:00:01 +0000 UTC   26d       default            <none>
daily-backup-all-but-system-20240318030000      Completed         0        0          2024-03-18 03:00:00 +0000 UTC   25d       default            <none>
...
```
## Apply storage class, volume snapshot class, and class mapping yaml files.
```
$ k apply -n velero -f volume-snapshot-class.yaml
volumesnapshotclass.snapshot.storage.k8s.io/csi-gce-vsc created
$ k apply -n velero -f storage-class.yaml
storageclass.storage.k8s.io/pd-velero created
$ k apply -n velero -f storage-class-mapping.yaml
configmap/change-storage-class-config created
```
## Test restore (don't forget include-namespaces !!)
```
$ velero restore create corteza-from-daily-backup-all-but-system-20240321030049 --include-namespaces=corteza --from-backup=daily-backup-all-but-system-20240321030049
Restore request "corteza-from-daily-backup-all-but-system-20240321030049" submitted successfully.
Run `velero restore describe corteza-from-daily-backup-all-but-system-20240321030049` or `velero restore logs corteza-from-daily-backup-all-but-system-20240321030049` for more details.
```
