# Couchbase backed  by Portworx on Kubernetes

## Pre-requisites
```
1. A working Openshift cluster
2. Storage for Portworx
```

## Install Portworx

### Pre-requisites
```
https://docs.portworx.com/start-here-installation/
```

### Install Portworx
```
https://docs.portworx.com/portworx-install-with-kubernetes/on-premise/openshift/
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

### (Optional) To create encrypted PVCs
Portworx supports encrypting the volumes. In order to encrypt, Portworx uses the libgcrypt library to interface with the dm-crypt module for creating, accessing and managing encrypted devices. Portworx uses the LUKS format of dm-crypt and AES-256 as the cipher with xts-plain64 as the cipher mode.

All encrypted volumes are protected by a passphrase. Portworx uses this passphrase to encrypt the volume data at rest as well as in transit. It is recommended to store these passphrases in a secure secret store. More information [here](https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-pvcs/create-encrypted-pvcs/#volume-encryption)

This guide shall use Kubernetes as the secret store, i.e. the passphrase shall be stored in a kubernetes secret in `portworx` namespace. 

#### Create permissions to access secret(s)
```
cat <<EOF | oc apply -f -
# Namespace to store credentials
apiVersion: v1
kind: Namespace
metadata:
  name: portworx
---
# Role to access secrets under portworx namespace only
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: px-role
  namespace: portworx
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "create", "update", "patch"]
---
# Allow portworx service account to access the secrets under the portworx namespace
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: px-role-binding
  namespace: portworx
subjects:
- kind: ServiceAccount
  name: px-account
  namespace: kube-system
roleRef:
  kind: Role
  name: px-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

*NOTE*: Make sure that `"-secret_type"`, `"k8s"` arguments are present in `portworx` container arguments in the daemonset. If not add it. It should look like this:
```
containers:
  - args:
    - -c
    - testclusterid
    - -s
    - /dev/sdb
    - -x
    - kubernetes
    - -secret_type
    - k8s
    name: portworx
```

#### Set the cluster wide secret key
A cluster wide secret key is a common encryption key/ passphrase that shall be used to encrypt all PVCs. Portworx also supports having an individual encryption key/passphrase per PVC. Refer to [this doc](https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-pvcs/create-encrypted-pvcs/#volume-encryption) for more details. For this guide, we shall use cluster-wide secret key.

Create a kubernetes secret to store the encryption key/ passphrase:
```
oc -n portworx create secret generic px-vol-encryption \
  --from-literal=cluster-wide-secret-key=<value>
```

Set the cluster wide secret key in Portworx:
```
PX_POD=$(oc get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')
oc exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl secrets set-cluster-key \
  --secret cluster-wide-secret-key
```

#### Enable encryption in the storageClass
Configure the storageClass to turn ON encryption by adding `secure: true` in the storageClass parameter. So, the storageClass from the section above shall now look like this:
```
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
 name: px-db-rf2-secure-sc
provisioner: kubernetes.io/portworx-volume
allowVolumeExpansion: true
parameters:
 repl: "2"
 priority_io: "high"
 io_profile: "db"
 secure: "true"
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
https://packages.couchbase.com/kubernetes/1.2.1/couchbase-autonomous-operator-openshift_1.2.1-linux-x86_64.tar.gz
```

Once the above package is downloaded:
```
tar -xzvf couchbase-autonomous-operator-openshift_1.2.1-linux-x86_64.tar.gz
cd couchbase-autonomous-operator-openshift_1.2.1-linux-x86_64/
```

### Create the dynamic admission controller
```
oc create -f admission.yaml
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
oc get pods
NAME                                               READY   STATUS    RESTARTS   AGE
couchbase-operator-admission-59b9bb8fc6-xz2ld      1/1     Running   0          2m20s
```
### Deploy the operator
```
oc create -f crd.yaml
oc create -f operator-service-account.yaml
oc create -f operator-role.yaml
oc create -f operator-role-binding.yaml
oc create -f operator-deployment.yaml
```

Verify that the operator is running
```
oc get pods
NAME                                  READY   STATUS    RESTARTS   AGE
couchbase-operator-577fbfdbcd-6sgz9   1/1     Running   0          66s
```

### Install the couchbase cluster
Create a secret containing the Auth Credentials
```
oc create -f secret.yaml
```

Now we can deploy the couchbase persistent cluster
```
oc create -f https://raw.githubusercontent.com/satchpx/couchbase-px/master/openshift/couchbase-persistent-cluster.yaml
```

*NOTE*: The couchbase cluster may take a while to start. To follow progress check the operator logs:
```
oc logs -f <couchbase-operator-pod>
```

One the cluster is up, you should see something like this in the logs:
```
time="2019-11-13T17:25:46Z" level=info cluster-name=cb-openshift-cluster module=cluster
time="2019-11-13T17:25:54Z" level=info msg="reconcile finished" cluster-name=cb-openshift-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="Cluster status: balanced" cluster-name=cb-openshift-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="Node status:" cluster-name=cb-openshift-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="┌───────────────────────────┬──────────────────┬────────┬────────────────┐" cluster-name=cb-openshift-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="│ Server                    │ Version          │ Class  │ Status         │" cluster-name=cb-openshift-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="├───────────────────────────┼──────────────────┼────────┼────────────────┤" cluster-name=cb-openshift-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="│ cb-openshift-cluster-0000 │ enterprise-6.0.1 │ data   │ managed+active │" cluster-name=cb-openshift-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="│ cb-openshift-cluster-0001 │ enterprise-6.0.1 │ index  │ managed+active │" cluster-name=cb-openshift-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="│ cb-openshift-cluster-0002 │ enterprise-6.0.1 │ data   │ managed+active │" cluster-name=cb-openshift-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="│ cb-openshift-cluster-0003 │ enterprise-6.0.1 │ data   │ managed+active │" cluster-name=cb-openshift-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="│ cb-openshift-cluster-0004 │ enterprise-6.0.1 │ search │ managed+active │" cluster-name=cb-openshift-cluster module=cluster
time="2019-11-13T17:26:23Z" level=info msg="└───────────────────────────┴──────────────────┴────────┴────────────────┘" cluster-name=cb-openshift-cluster module=cluster
```


*NOTE*: The persistent cluster configuration includes 3 data nodes, 1 search and 1 index node in the couchbase cluster.
```
# oc get pods
NAME                                               READY   STATUS    RESTARTS   AGE
cb-openshift-cluster-0000                          1/1     Running   0          17h
cb-openshift-cluster-0001                          1/1     Running   0          17h
cb-openshift-cluster-0002                          1/1     Running   0          17h
cb-openshift-cluster-0003                          1/1     Running   0          17h
cb-openshift-cluster-0004                          1/1     Running   0          17h
couchbase-operator-admission-59b9bb8fc6-xz2ld      1/1     Running   0          17h
couchbase-operator-b9c9d4977-gz9l4                 1/1     Running   5          17h
```

### Pillowfight
```
TBD
```

### Using stork-scheduler
```
@TODO
```