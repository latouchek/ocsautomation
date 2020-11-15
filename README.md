# How to deploy OpenShift Container Storage 4 in OCP 4.6 with no GUI involved



## Introduction

So I wanted to deploy OCS in my Lab and after reading the excellent blog post from [Muhammad Aizuddin Zali](https://www.techbeatly.com/2020/01/red-hat-openshift-container-storage-4-2-installation.html#.XxKBJmMzaZT), I decided to add some automation to get rid of GUI actions during deployment.

 The purpose of this post is to show how to deploy and configure in an automated manner, OCS in an Openshift 4.6 lab. In this setup OCS operator will create an hyperconverged Ceph Cluster using Workers local disks. In order to do so we will also need to deploy the Local Storage Operator.



## Lab Overview
* KVM host 256Go of RAM + 500Go SSD
* UPI baremetal OCP 4.6 deployment
* All oc commands are issued from a CentOS 8 bastion node
* 3 Masters (16Go of RAM) and 3 Workers (32Go of RAM) VMs
```bash
[root@ocp4-bastion ~] oc get nodes
NAME                             STATUS   ROLES    AGE   VERSION
ocp4-master1.mycluster.lab.local   Ready    master   16h   v1.18.3+6025c28
ocp4-master2.mycluster.lab.local   Ready    master   16h   v1.18.3+6025c28
ocp4-master3.mycluster.lab.local   Ready    master   16h   v1.18.3+6025c28
ocp4-worker1.mycluster.lab.local   Ready    worker   16h   v1.18.3+6025c28
ocp4-worker2.mycluster.lab.local   Ready    worker   16h   v1.18.3+6025c28
ocp4-worker3.mycluster.lab.local   Ready    worker   16h   v1.18.3+6025c28
```
* Each worker has 4 disks. OCS will use vdb, vdc and vdd for monitors and OSDs
 ```bash
 [core@ocp4-worker* ~]$ lsblk
 NAME                         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
 vda                          252:0    0  23.6G  0 disk
 ├─vda1                       252:1    0   384M  0 part /boot
 ├─vda2                       252:2    0   127M  0 part /boot/efi
 ├─vda3                       252:3    0     1M  0 part
 └─vda4                       252:4    0  23.1G  0 part
   └─coreos-luks-root-nocrypt 253:0    0  23.1G  0 dm   /sysroot
 vdb                          252:16   0    14G  0 disk
 vdc                          252:32   0 102.5G  0 disk
 vdd                          252:48   0 102.5G  0 disk

 ```

## Quickstart
* Deploy Local Storage Operator
  ```bash
  oc apply -f ocs/local-storage-operator.yaml
  ```
* Wait for the operator to be ready
```bash
    [root@ocp4-bastion ~] oc get pod -n local-storage
    NAME                                      READY   STATUS    RESTARTS   AGE
    local-storage-operator-5cc58555f6-6smlv   1/1     Running   0          14s
```
* Create LocalVolume to be used as Monitors and OSDs by OCS

* Make sure to modify Workers FQDN in  **local-storage.yaml** so it fits to your environment
 ```yaml
  spec:
   nodeSelector:
     nodeSelectorTerms:
     - matchExpressions:
         - key: kubernetes.io/hostname
           operator: In
           values:
             - ocp4-worker1.mycluster.lab.local  #change Workers FQDN
             - ocp4-worker2.mycluster.lab.local
             - ocp4-worker3.mycluster.lab.local
```
* Apply changes
 ```bash
   oc apply -f ocs/local-storage.yaml
   ```
* Check Pods, SC and PVs have been properly created
```bash
[root@ocp4-bastion ~] oc get pods -n local-storage
NAME                                      READY   STATUS    RESTARTS   AGE
local-disks-mon-local-diskmaker-fcst8     1/1     Running   0          58s
local-disks-mon-local-diskmaker-kvzsp     1/1     Running   0          58s
.
.
.
.
local-storage-operator-5cc58555f6-mtcjt   1/1     Running   0          16m
[root@ocp4-bastion ~] oc get sc
NAME           PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-sc-mon   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  5m42s
local-sc-osd   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  5m41s
[root@ocp4-bastion ~]oc get pv
NAME                    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                             STORAGECLASS   REASON   AGE
local-pv-110b1b1b       13Gi       RWO            Delete           Available                                                     local-sc-mon            5m49s
local-pv-38624ea6       102Gi      RWO            Delete           Available                                                     local-sc-osd            5m49s
local-pv-74bab7b9       13Gi       RWO            Delete           Available                                                     local-sc-mon            5m49s
local-pv-810ed429       102Gi      RWO            Delete           Available                                                     local-sc-osd            5m50s
local-pv-816280fe       13Gi       RWO            Delete           Available                                                     
.
.
.
.
```
We are now ready to deploy OCS operator

* Label Workers so they can be used as storage cluster members
```bash
oc label nodes ocp4-worker1.mycluster.lab.local  cluster.ocs.openshift.io/openshift-storage=''
oc label nodes ocp4-worker2.mycluster.lab.local  cluster.ocs.openshift.io/openshift-storage=''
oc label nodes ocp4-worker3.mycluster.lab.local  cluster.ocs.openshift.io/openshift-storage=''
```
* Install OCS Operator and wait pods are up and running
```bash
[root@ocp4-bastion ~] oc apply -f ocs/ocs-storage-operator.yaml
```
```bash
[root@ocp4-bastion ~] oc get pods -n openshift-storage
NAME                                  READY   STATUS    RESTARTS   AGE
aws-s3-provisioner-5d8f48bb55-tpd4b   1/1     Running   0          3m1s
noobaa-operator-74d7d65b-tgqhw        1/1     Running   0          3m27s
ocs-operator-6c948ccd9-xffq5          1/1     Running   0          3m27s
rook-ceph-operator-5f7797979b-2pr6f   1/1     Running   0          3m27s
```
* Create the storage cluster and wait until it's ready
```bash
oc apply -f ocs/ocs-cluster.yaml
```
```bash
[root@ocp4-bastion ~] oc get pod -n openshift-storage
NAME                                            READY   STATUS              RESTARTS   AGE
aws-s3-provisioner-5d8f48bb55-hg48f             1/1     Running             0          5m26s
csi-cephfsplugin-2n6v2                          0/3     ContainerCreating   0          35s
csi-cephfsplugin-f7njt                          0/3     ContainerCreating   0          35s
csi-cephfsplugin-lxl7s                          3/3     Running             0          35s
csi-cephfsplugin-provisioner-7dccf64b5b-6nnsv   0/5     ContainerCreating   0          33s
csi-cephfsplugin-provisioner-7dccf64b5b-tjgql   0/5     ContainerCreating   0          33s
csi-rbdplugin-brbrr                             3/3     Running             0          35s
csi-rbdplugin-dnskp                             0/3     ContainerCreating   0          35s
csi-rbdplugin-mwtqq                             0/3     ContainerCreating   0          35s
csi-rbdplugin-provisioner-7cdfdc587-mpbq6       0/5     ContainerCreating   0          35s
csi-rbdplugin-provisioner-7cdfdc587-p6v7q       0/5     ContainerCreating   0          34s
lib-bucket-provisioner-6485cf48bd-247zs         1/1     Running             0          5m55s
noobaa-operator-74d7d65b-pkn5w                  1/1     Running             0          6m38s
ocs-operator-6c948ccd9-fd9jq                    0/1     Running             0          6m38s
rook-ceph-detect-version-jllnf                  0/1     Init:0/1            0          28s
rook-ceph-operator-5f7797979b-ss7bx             1/1     Running             0          6m38s
```
```bash
[root@ocp4-bastion ~] oc get storageclusters -A                                                                                                                                                                                                     
NAMESPACE           NAME                 AGE     PHASE   CREATED AT             VERSION
openshift-storage   ocs-storagecluster   8m12s   Ready   2020-07-19T05:55:31Z   4.4.0
```
* Check new storage classes are available
 ```bash
[root@ocp4-bastion ~] oc get sc
NAME                          PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-sc-mon                  kubernetes.io/no-provisioner            Delete          WaitForFirstConsumer   false                  14m
local-sc-osd                  kubernetes.io/no-provisioner            Delete          WaitForFirstConsumer   false                  14m
ocs-storagecluster-ceph-rbd   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              false                  7m
ocs-storagecluster-cephfs     openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              false                  7m
openshift-storage.noobaa.io   openshift-storage.noobaa.io/obc         Delete          Immediate              false                  4m
  ```
* Check the cluster health:
  * Enable Ceph tools
   ```bash
   [root@ocp4-bastion ~] oc patch OCSInitialization ocsinit -n openshift-storage --type json --patch  '[{ "op": "replace", "path": "/spec/enableCephTools", "value": true }]'
    ```
  * Find out Ceph tools pod and rsh into it
    ```bash
    [root@ocp4-bastion ~] oc get pod|grep rook-ceph-tools
    rook-ceph-tools-7f5877799b-mgj5n                                  1/1     Running     0          22m
    ```
    ```bash
    [root@ocp4-bastion ~] oc rsh rook-ceph-tools-7f5877799b-mgj5n
    ```
  * Once logged in you can use ceph commands
    ```bash
    sh-4.4 ceph -s
    cluster:
      id:     da46c3ea-ad0e-45ea-afb6-9cd7e9e49df0
      health: HEALTH_OK
    services:
      mon: 3 daemons, quorum a,b,c (age 41m)
      mgr: a(active, since 41m)
      mds: ocs-storagecluster-cephfilesystem:1 {0=ocs-storagecluster-cephfilesystem-a=up:active} 1 up:standby-replay
      osd: 3 osds: 3 up (since 40m), 3 in (since 40m)
      rgw: 1 daemon active (ocs.storagecluster.cephobjectstore.a)
    task status:
      scrub status:
          mds.ocs-storagecluster-cephfilesystem-a: idle
          mds.ocs-storagecluster-cephfilesystem-b: idle
    data:
      pools:   10 pools, 176 pgs
      objects: 332 objects, 87 MiB
      usage:   3.2 GiB used, 304 GiB / 307 GiB avail
      pgs:     176 active+clean
    io:
      client:   1.2 KiB/s rd, 10 KiB/s wr, 2 op/s rd, 1 op/s wr
    ```
  Our cluster is now ready to be used
## How did we do it?

It takes 3 steps to deploy an Operator from cli

* Create the namespace where we want to deploy the Operator
* Create an OperatorGroup to insure the Operator will be deployed in the previously created namespace
* Create a Subscription object for the desired Operator

  * Let's get into details and take a close look at **local-storage-operator.yaml**
   1. We created the local-storage namespace
    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: local-storage
    ```

    2.  To deploy the Local Storage Operator in the previously created namespace we create an OperatorGroup object.

      An OperatorGroup is an OLM resource that provides multitenant configuration to OLM-installed Operators. An OperatorGroup selects target namespaces in which to generate required RBAC access for its member Operators.

     ```yaml
     ---
    apiVersion: operators.coreos.com/v1
    kind: OperatorGroup
    metadata:
      name: local-storage-operator
      namespace: local-storage
    spec:
      targetNamespaces:
      - local-storage
     ```
    3.  Finally we create a Subscription object in the local-storage namespace.

      ```yaml
      ---
      apiVersion: operators.coreos.com/v1alpha1
      kind: Subscription
      metadata:
        name: local-storage-operator
        namespace: local-storage
      spec:
        channel: "4.4"
        name: local-storage-operator
        source: redhat-operators
        sourceNamespace: openshift-marketplace
      ```
    4. Check the deployment went well
    ```bash
    [root@ocp4-bastion ~] oc get pod -n local-storage
    NAME                                      READY   STATUS    RESTARTS   AGE
    local-storage-operator-5cc58555f6-6smlv   1/1     Running   0          14s
    ```
    5. Exact same approach can be observed to deploy OCS operator in **ocs-storage-operator.yaml** file
