# Couchbase backed  by Portworx on Kubernetes

## Pre-requisites
```
1. A working Kubernetes cluster
2. Storage for Portworx
```

## Install Portworx

### Pre-requisites
```
https://docs.portworx.com/start-here-installation/
```

### Install Portworx
```
https://docs.portworx.com/portworx-install-with-kubernetes/
```

### Create the SchedulePolicy
The schedulePolicy is a CRD where a policy can be defined for backing up Portworx Volumes. These could either be [local snapshots](https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-snapshots/on-demand/snaps-local/) or [cloudsnaps](https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-snapshots/on-demand/snaps-cloud/).

We shall reference this schedulePolicy in the storageClass, so that the pvc's created will be automatically backed up. This enables Business Continuity, Disaster Recovery.

The key parameters in the Schedule Policy need to be driven by the Organization's RPO/RTO requirements.

The Schedule Policy defined below, has an RPO of <> and RTO of <>.
Apply the Schedule policy below:
```
apiVersion: stork.libopenstorage.org/v1alpha1
kind: SchedulePolicy
metadata:
  name: cbc-policy
policy:
  interval:
    intervalMinutes: 10
    retain: 4
  daily:
    time: "10:14PM"
    retain: 3
  weekly:
    day: "Thursday"
    time: "10:13PM"
    retain: 2
  monthly:
    date: 14
    time: "8:05PM"
    retain: 1
```

### Create the BackupLocation
Backup Location is another CRD that allows a user to define the S3 compliant object store where Portworx Volume backups shall be stored.

```
apiVersion: stork.libopenstorage.org/v1alpha1
kind: BackupLocation
metadata:
  name: pwx-s3-backuplocation
  namespace: kube-system
  annotations:
    stork.libopenstorage.org/skipresource: "true"
location:
  type: s3
  path: "pwx-volume-backups"
  s3Config:
    region: us-east-1
    accessKeyID: <redacted>
    secretAccessKey: <redacted>
    endpoint: "70.0.0.141:9010"
    disableSSL: true
```

### Create a VolumePlacementStrategy
@TODO

### Install the required storageClasses
Create a storageClass that incorporates the two objects defined above. This will ensure that all PVC's created from this storageClass will be backed up per the defined schedule to the defined location.

This will ensure Business Continuity and Disaster Recovery capabilities to our couchbase cluster and its data.

CouchDB shards its data across the cluster. Taking this into consideration, we shall define a storage replication factor of "2". We shall also set its `io_profile` to `db` so that the volume performance is optimized to database like workloads. For more information on Portworx io_profiles, refer [here](https://docs.portworx.com/install-with-other/operate-and-maintain/performance-and-tuning/tuning/#volume-granular-performance-tuning)

Create the storageClass defined below:
```
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
 name: px-db-rf2-sc
provisioner: kubernetes.io/portworx-volume
allowVolumeExpansion: true
parameters:
 repl: "2"
 priority_io: "high"
 io_profile: "db"
 disable_io_profile_protection: "1"
 snapshotschedule.stork.libopenstorage.org/interval-schedule: |
   schedulePolicyName: cbc-policy
   annotations:
     portworx/snapshot-type: cloud
     portworx/cloud-cred-id: k8s/kube-system/pwx-s3-backuplocation
```

## Install Couchbase using the operator

### Download the operator
```
https://packages.couchbase.com/kubernetes/1.2.1/couchbase-autonomous-operator-kubernetes_1.2.1-linux-x86_64.tar.gz
```

Once the above package is downloaded:
```
tar -xzvf couchbase-autonomous-operator-kubernetes_1.2.1-linux-x86_64.tar.gz
cd couchbase-autonomous-operator-kubernetes_1.2.1-linux-x86_64/
```

### Create the dynamic admission controller
```
kubectl create -f admission.yaml
```

You should see the following objects getting created:
```
serviceaccount/couchbase-operator-admission created
clusterrole.rbac.authorization.k8s.io/couchbase-operator-admission created
clusterrolebinding.rbac.authorization.k8s.io/couchbase-operator-admission created
secret/couchbase-operator-admission created
deployment.apps/couchbase-operator-admission created
service/couchbase-operator-admission created
mutatingwebhookconfiguration.admissionregistration.k8s.io/couchbase-operator-admission created
validatingwebhookconfiguration.admissionregistration.k8s.io/couchbase-operator-admission created
```

### Verify the admission controller
```
kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
couchbase-operator-admission-59b9bb8fc6-xz2ld      1/1     Running   0          2m20s
```
### Deploy the operator
```
kubectl create -f crd.yaml
kubectl create -f operator-service-account.yaml
kubectl create -f operator-role.yaml
kubectl create -f operator-role-binding.yaml
kubectl create -f operator-deployment.yaml
```

Verify that the operator is running
```
kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
couchbase-operator-577fbfdbcd-6sgz9   1/1     Running   0          66s
```

### Install the couchbase cluster
Create a secret containing the Auth Credentials
```
kubectl create -f secret.yaml
```

Now we can deploy the couchbase persistent cluster
```
kubectl create -f https://raw.githubusercontent.com/satchpx/couchbase-px/master/k8s/couchbase-persistent-cluster.yaml
```

*NOTE*: The couchbase cluster may take a while to start. To follow progress check the operator logs:
```
kubectl logs -f <couchbase-operator-pod>
```

One the cluster is up, you should see something like this in the logs:
```
time="2019-11-13T17:25:46Z" level=info cluster-name=cb-kubernetes-cluster module=cluster
time="2019-11-13T17:25:54Z" level=info msg="reconcile finished" cluster-name=cb-kubernetes-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="Cluster status: balanced" cluster-name=cb-kubernetes-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="Node status:" cluster-name=cb-kubernetes-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="┌───────────────────────────┬──────────────────┬────────┬────────────────┐" cluster-name=cb-kubernetes-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="│ Server                    │ Version          │ Class  │ Status         │" cluster-name=cb-kubernetes-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="├───────────────────────────┼──────────────────┼────────┼────────────────┤" cluster-name=cb-kubernetes-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="│ cb-kubernetes-cluster-0000 │ enterprise-6.0.1 │ data   │ managed+active │" cluster-name=cb-kubernetes-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="│ cb-kubernetes-cluster-0001 │ enterprise-6.0.1 │ index  │ managed+active │" cluster-name=cb-kubernetes-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="│ cb-kubernetes-cluster-0002 │ enterprise-6.0.1 │ data   │ managed+active │" cluster-name=cb-kubernetes-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="│ cb-kubernetes-cluster-0003 │ enterprise-6.0.1 │ data   │ managed+active │" cluster-name=cb-kubernetes-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="│ cb-kubernetes-cluster-0004 │ enterprise-6.0.1 │ search │ managed+active │" cluster-name=cb-kubernetes-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="└───────────────────────────┴──────────────────┴────────┴────────────────┘" cluster-name=cb-kubernetes-cluster module=cluster
```


*NOTE*: The persistent cluster configuration includes 3 data nodes, 1 search and 1 index node in the couchbase cluster.
```
# kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
cb-kubernetes-cluster-0000                          1/1     Running   0          17h
cb-kubernetes-cluster-0001                          1/1     Running   0          17h
cb-kubernetes-cluster-0002                          1/1     Running   0          17h
cb-kubernetes-cluster-0003                          1/1     Running   0          17h
cb-kubernetes-cluster-0004                          1/1     Running   0          17h
couchbase-operator-admission-59b9bb8fc6-xz2ld      1/1     Running   0          17h
couchbase-operator-b9c9d4977-gz9l4                 1/1     Running   5          17h
```

### Pillowfight
```
TBD
```