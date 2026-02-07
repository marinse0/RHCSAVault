# Enterprise Kubernetes Storage with Red Hat OpenShift Data Foundation

## Deploying OpenShift Data Foundation from the Command Line

### Objectives

- Deploy Red Hat OpenShift Data Foundation on premise by using the CLI.
    

### Deploying OpenShift Data Foundation from the Command-line Interface

For automation or infrastructure as code (IaC) purposes, administrators can install OpenShift Data Foundation from the command line by using the `oc` or `kubectl` tools.

As previously explained, the installation of OpenShift Data Foundation consists of installing the OpenShift Data Foundation operator and creating the storage cluster object. Additionally, if local volumes are attached to the nodes, then you must install the local storage operator.

The Kubernetes operator model enables administrators to install an operator in Red Hat OpenShift Container Platform (RHOCP) by creating the corresponding operator group and subscription.

The operator group object selects target namespaces for generating the required role-based access control (RBAC) access for its member operators.

The subscription object defines the name and namespace of the operator, the catalog that includes the operator data, and the channel that determines the operator stream.

>Note
For more details about Kubernetes operators, refer to the _DO280: Red Hat OpenShift Administration II: Operating a Production Kubernetes Cluster_ training course.

#### Preparing the Nodes

OpenShift Data Foundation requires a minimum of three nodes with the same number of disks, size, type, and performance capabilities.

You can save on subscription costs by deploying OpenShift Data Foundation on infrastructure nodes. Counts of Red Hat subscription virtual CPU (vCPU) omit any vCPU that is reported by a node with a label of `node-role.kubernetes.io/infra: ""`.

The following diagram provides an RHOCP cluster overview with OpenShift Data Foundation deployed on infrastructure nodes.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do370-4.16/images/internal/cli/assets/odf-infra.svg)|

Figure 1.24: OpenShift Data Foundation deployment on infrastructure nodes

Infrastructure nodes use taints to prevent a pod from being scheduled unless that pod can tolerate the taint.

The following YAML example shows the taint in an infrastructure node specification:

```
...output omitted...
spec:
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/infra
    value: reserved
...output omitted...
```

The following YAML example shows the toleration in a pod specification:
```
...output omitted...
spec:
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/infra
    value: reserved
...output omitted...
```

Applying a taint to the infrastructure nodes and a toleration for that taint to all infrastructure components ensures that only those resources are scheduled on the infrastructure nodes.

>Note
For more details about taints and tolerations, refer to the _DO380: Red Hat OpenShift Administration III: Scaling Deployments in the Enterprise_ training course.

You can dedicate infrastructure nodes to OpenShift Data Foundation by adding an OpenShift Data Foundation taint with a `NoSchedule` effect. RHOCP then schedules only OpenShift Data Foundation resources on those nodes.

You can add the OpenShift Data Foundation taint by using the following command:

```
[user@host ~]$ oc adm taint node `nodename` \   node.ocs.openshift.io/storage=true:NoSchedule
```

>Note
OpenShift Data Foundation components tolerate the OpenShift Data Foundation taint by default. You do not need to configure the toleration in the StorageCluster resource.

The local storage operator, on the other hand, requires this toleration to detect disks on the dedicated nodes.

Add the OpenShift Data Foundation label to each node with storage devices. The OpenShift Data Foundation operator determines from this label which nodes can schedule targets for OpenShift Data Foundation components.

You can add the OpenShift Data Foundation label to the nodes by using the following command:

```
[user@host ~]$ oc label node `nodename` \   cluster.ocs.openshift.io/openshift-storage=''
```

You can view the OpenShift Data Foundation labeled nodes by running the following command:

```
[user@host ~]$ oc get nodes -l \   cluster.ocs.openshift.io/openshift-storage=""
NAME       STATUS   ROLES ...
worker01   Ready    worker   ...
worker02   Ready    worker   ...
worker03   Ready    worker   ...
```

#### Installing the Local Storage Operator

Install the local storage operator when deploying OpenShift Data Foundation on RHOCP by using local storage devices.

The following YAML examples are of the `OperatorGroup` and `Subscription` objects, and assume that the `openshift-local-storage` namespace exists:

apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-local-storage
  namespace: openshift-local-storage
spec:
  targetNamespaces:
    - openshift-local-storage

apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: local-storage-operator
  namespace: openshift-local-storage
spec:
  channel: stable
  name: local-storage-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace

After you create the `OperatorGroup` and `Subscription` objects, you must create the following custom resources (CR) to finish installing the local storage operator:

- The `LocalVolumeDiscovery` CR, to discover a list of potentially usable disks on the chosen set of nodes.
    
- The `LocalVolumeSet` CR, to filter a set of storage volumes on the selected nodes, group them, and create a dedicated storage class to consume storage from the set of volumes.
    

The following YAML examples are of the `LocalVolumeDiscovery` and `LocalVolumeSet` objects:

apiVersion: local.storage.openshift.io/v1alpha1
kind: LocalVolumeDiscovery
metadata:
  name: auto-discover-devices
  namespace: openshift-local-storage
spec
  `nodeSelector`: ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
    nodeSelectorTerms:
    - matchExpressions:
      - key: cluster.ocs.openshift.io/openshift-storage
        operator: Exists
  `tolerations`: ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
  - effect: NoSchedule
    key: node-role.kubernetes.io/infra
    value: reserved
  _...output omitted..._

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_installing_the_local_storage_operator-CO1-1)|Select a set of nodes based on their label.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_installing_the_local_storage_operator-CO1-2)|If you are deploying OpenShift Data Foundation on infrastructure nodes, then set the tolerations for the infrastructure node taints to discover the disks on the infrastructure nodes.|

>Note
If you are deploying OpenShift Data Foundation components on dedicated infrastructure nodes with the OpenShift Data Foundation taint, then use the following tolerations instead, in both the `LocalVolumeDiscovery` and `LocalVolumeSet` objects.

_...output omitted..._
tolerations:
- effect: NoSchedule
  key: node.ocs.openshift.io/storage
  operator: Equal
  value: "true"
_...output omitted..._

Creating the `LocalVolumeDiscovery` object generates the `LocalVolumeDiscoveryResults` object, which contains the list of disks on each selected node.

You can retrieve the `LocalVolumeDiscoveryResults` objects with the following command:

[user@host ~]$ **`oc get localvolumediscoveryresults \   -o custom-columns='NAME:metadata.name,PATH:status.discoveredDevices[*].path' \   -n openshift-local-storage`**
NAME                        PATH
discovery-result-worker01   /dev/vda1,/dev/vda2,/dev/vda3,/dev/vdb
discovery-result-worker02   /dev/vda1,/dev/vda2,/dev/vda3,/dev/vdb
discovery-result-worker03   /dev/vda1,/dev/vda2,/dev/vda3,/dev/vdb

When the discovery finishes, you can create a `LocalVolumeSet` object by using the devices that the `LocalVolumeDiscovery` object discovered.

apiVersion: local.storage.openshift.io/v1alpha1
kind: LocalVolumeSet
metadata:
  name: lso-volumeset
  namespace: openshift-local-storage
spec:
  deviceInclusionSpec:
    `deviceTypes`: ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
    - disk
    - part
    `minSize`: 1Gi ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
  `nodeSelector`: ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
    nodeSelectorTerms:
    - matchExpressions:
      - key: cluster.ocs.openshift.io/openshift-storage
        operator: Exists
  `tolerations`: ![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
  - effect: NoSchedule
    key: node-role.kubernetes.io/infra
    value: reserved
  _...output omitted..._
  `storageClassName`: lso-volumeset ![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)
  `maxDeviceCount`: 1 ![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)
  volumeMode: Block

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_installing_the_local_storage_operator-CO2-1)|Select the type of device.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_installing_the_local_storage_operator-CO2-2)|Set the minimum size of the disk.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_installing_the_local_storage_operator-CO2-3)|Match only the selected nodes.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_installing_the_local_storage_operator-CO2-4)|If you are deploying OpenShift Data Foundation on infrastructure nodes, then set the tolerations for the infrastructure node taints to get the disks on the infrastructure nodes.|
|[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_installing_the_local_storage_operator-CO2-5)|The name of the storage class to create.|
|[![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)](https://rol.redhat.com/rol/app/#_installing_the_local_storage_operator-CO2-6)|The maximum number of devices per node to use; if it is not set, then all the disks are used.|

#### Installing the Red Hat OpenShift Data Foundation Operator

To install the Red Hat OpenShift Data Foundation operator, first create the `openshift-storage` namespace, and then create the `OperatorGroup` and `Subscription` objects.

Label the `openshift-storage` namespace to use the available cluster monitoring solution. You can label the namespace with the following command:

[user@host ~]$ **`oc label namespace/openshift-storage \   openshift.io/cluster-monitoring=true`**

The following YAML examples are of the `OperatorGroup` and `Subscription` objects:

apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-storage
  namespace: openshift-storage
spec:
  targetNamespaces:
    - openshift-storage

apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: odf-operator
  namespace: openshift-storage
spec:
  channel: "stable-4.16"
  installPlanApproval: Automatic
  name: odf-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace

After creating the operator group and the subscription for the operator, create the OpenShift Data Foundation storage cluster.

If you are deploying OpenShift Data Foundation on infrastructure nodes, then add tolerations for the infrastructure taints to the OpenShift Data Foundation components within the `StorageCluster` object specification.

The following list is of the OpenShift Data Foundation components:

- `mds`
    
- `noobaa-core`
    
- `rgw`
    
- `csi-provisioner`
    
- `csi-plugin`
    
- `metrics-exporter`
    
- `nfs`
    
- `toolbox`
    

Assuming an internal mode installation, the following YAML example creates the `StorageCluster` custom resource:

apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:
  name: ocs-storagecluster
  namespace: openshift-storage
spec:
  monDataDirHostPath: /var/lib/rook
  storageDeviceSets:
  - `count`: 1 ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
    dataPVCTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            `storage`: "1" ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
        `storageClassName`: lso-volumeset ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
        `volumeMode`: Block ![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
    name: ocs-deviceset-lso-volumeset
    replica: 3
  `flexibleScaling`: false ![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)
  `placement`: ![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)
    all:
      tolerations:
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoSchedule
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoExecute
    mds:
      tolerations:
      _...output omitted..._
    noobaa-core:
      _...output omitted..._

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_installing_the_redhat_openshift_data_foundation_operator-CO3-1)|For each set of 3 disks, increment the count by 1.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_installing_the_redhat_openshift_data_foundation_operator-CO3-2)|Set the minimum required storage to 1 B, with a maximum value of 4 TiB.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_installing_the_redhat_openshift_data_foundation_operator-CO3-3)|The required name of the storage class for the claim that the local storage operator created.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_installing_the_redhat_openshift_data_foundation_operator-CO3-4)|The type of volume that the claim requires.|
|[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_installing_the_redhat_openshift_data_foundation_operator-CO3-5)|When true, set the failure domain to the host domain, to enable devices to be distributed evenly across all nodes, regardless of distribution in zones or racks.|
|[![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)](https://rol.redhat.com/rol/app/#_installing_the_redhat_openshift_data_foundation_operator-CO3-6)|Set the tolerations to the OpenShift Data Foundation components for the infrastructure node taints when deploying OpenShift Data Foundation on infrastructure nodes.|

For the deployments where the nodes are spread across fewer than three availability zones, you can enable the flex scaling feature to use individual hosts as failure domains.

Flex scaling enables you to scale up by adding any number of disks or nodes to the cluster.

If three or more availability zones are set by using the Kubernetes topology labels, then OpenShift Data Foundation uses them to configure the failure domains. Scaling the cluster up then requires adding the same number of disks or nodes in each availability zone.

You can get the node in a specific rack with the following command:

[user@host ~]$ **`oc get no -l topology.rook.io/rack=rack0`**
NAME       STATUS   ROLES    AGE    VERSION
worker01   Ready    worker   2d7h   v1.29.6+aba1e8d

### Note

For more details about Kubernetes topology and availaibility zones, refer to the _DO380: Red Hat OpenShift Administration III: Scaling Deployments in the Enterprise_ training course.

#### Verifying the Red Hat OpenShift Data Foundation Installation

After the operator installation is complete and the storage cluster is created, cluster administrators can verify that OpenShift Data Foundation is deployed correctly by using the command line.

To verify that the `StorageCluster` object is ready, use the following command:

[user@host ~]$ **`oc get storagecluster -n openshift-storage`**
NAME                 AGE   PHASE   EXTERNAL   CREATED AT             VERSION
ocs-storagecluster   40h   `Ready`              2024-08-01T20:47:04Z   4.16.0

Check that all the pods in the `openshift-storage` namespace are either in the `Running` or the `Completed` state by running the following command:

[user@host ~]$ **`oc get pods -n openshift-storage`**
NAME                                                READY   STATUS    ...
pod/csi-cephfsplugin-k59p6                          3/3     Running   ...
pod/csi-cephfsplugin-mlrr8                          3/3     Running   ...
pod/csi-cephfsplugin-provisioner-584644bb58-lz6wd   6/6     Running   ...
_...output omitted..._

### Note

For more information about the expected number of pods for each component, and how it varies depending on the number of nodes, refer to [https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.16/html-single/deploying_openshift_data_foundation_using_bare_metal_infrastructure/index#verifying-the-state-of-the-pods_local-bare-metal](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.16/html-single/deploying_openshift_data_foundation_using_bare_metal_infrastructure/index#verifying-the-state-of-the-pods_local-bare-metal)

Verify that the `cephCluster` object is ready and healthy by running the following command:

[user@host ~]$ **`oc get cephcluster -n openshift-storage`**
NAME                             ...   PHASE   ...   HEALTH   ...
ocs-storagecluster-cephcluster   ...   `Ready`   ...   `HEALTH_OK`  ...

Verify that the `noobaa` object is ready by running the following command:

[user@host ~]$ **`oc get noobaa -n openshift-storage`**
NAME     ...   PHASE   AGE
noobaa   ...   `Ready`   57m

Verify that the following storage classes are created with the OpenShift Data Foundation cluster creation:

- `ocs-storagecluster-ceph-rbd`
    
- `ocs-storagecluster-cephfs`
    
- `openshift-storage.noobaa.io`
    
- `ocs-storagecluster-ceph-rgw`
    

You can get the storage classes with the following command:

[user@host ~]$ **`oc get storageclasses -o name`**
_...output omitted..._
storageclass.storage.k8s.io/`ocs-storagecluster-ceph-rbd`
storageclass.storage.k8s.io/`ocs-storagecluster-ceph-rgw`
storageclass.storage.k8s.io/`ocs-storagecluster-cephfs`
storageclass.storage.k8s.io/`openshift-storage.noobaa.io`

### References

For more information about OpenShift Data Foundation deployment by using local storage devices, refer to the _Deploy OpenShift Data Foundation Using Local Storage Devices_ chapter in the Red Hat OpenShift Data Foundation 4.16 _Deploying OpenShift Data Foundation Using Bare Metal Infrastructure_ documentation at [https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.16/html-single/deploying_openshift_data_foundation_using_bare_metal_infrastructure/index#deploy-using-local-storage-devices-bm](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.16/html-single/deploying_openshift_data_foundation_using_bare_metal_infrastructure/index#deploy-using-local-storage-devices-bm)

For more information about taints and tolerations, refer to the _Controlling Pod Placement Using Node Taints_ section in the _Controlling Pod Placement onto Nodes (Scheduling)_ chapter in the Red Hat OpenShift Container Platform 4.16 _Nodes_ documentation at [https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html-single/nodes/index#nodes-scheduler-taints-tolerations-about_nodes-scheduler-taints-tolerations](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html-single/nodes/index#nodes-scheduler-taints-tolerations-about_nodes-scheduler-taints-tolerations)

For more information about infrastructure nodes, refer to the _Creating Infrastructure Nodes_ section in the _Working with Nodes_ chapter in the Red Hat OpenShift Container Platform 4.16 _Nodes_ documentation at [https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html-single/nodes/index#nodes-nodes-creating-infrastructure-nodes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html-single/nodes/index#nodes-nodes-creating-infrastructure-nodes)

**Revision:** do370-4.16-unknown

Deploying OpenShift Data Foundation from the Command Line

Previous

8/72

Next

![Red Hat Training + Certification logo](data:image/svg+xml;base64,PHN2ZyBpZD0iTGF5ZXJfMSIgZGF0YS1uYW1lPSJMYXllciAxIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCA4MDYgMzUwIj48ZGVmcz48c3R5bGU+LmNscy0xe2ZpbGw6I2UwMDt9LmNscy0ye2ZpbGw6I2ZmZjt9PC9zdHlsZT48L2RlZnM+PHRpdGxlPkxvZ28tUmVkX0hhdC1UcmFpbmluZ19hbmRfQ2VydGlmaWNhdGlvbi1CLVJldmVyc2UtUkdCPC90aXRsZT48cGF0aCBjbGFzcz0iY2xzLTEiIGQ9Ik0xMjgsODRjMTIuNTEsMCwzMC42MS0yLjU4LDMwLjYxLTE3LjQ2YTE0LDE0LDAsMCwwLS4zMS0zLjQybC03LjQ1LTMyLjM2Yy0xLjcyLTcuMTItMy4yMy0xMC4zNS0xNS43My0xNi42QzEyNS4zOSw5LjE5LDEwNC4yNiwxLDk4LDFjLTUuODIsMC03LjU1LDcuNTQtMTQuNDUsNy41NC02LjY4LDAtMTEuNjQtNS42LTE3Ljg5LTUuNi02LDAtOS45MSw0LjA5LTEyLjkzLDEyLjUsMCwwLTguNDEsMjMuNzItOS40OSwyNy4xNkE2LjQzLDYuNDMsMCwwLDAsNDMsNDQuNTRDNDMsNTMuNzYsNzkuMzMsODQsMTI4LDg0bTMyLjU1LTExLjQyYzEuNzMsOC4xOSwxLjczLDkuMDUsMS43MywxMC4xMywwLDE0LTE1Ljc0LDIxLjc3LTM2LjQzLDIxLjc3Qzc5LDEwNC40NywzOC4wOCw3Ny4xLDM4LjA4LDU5YTE4LjQ1LDE4LjQ1LDAsMCwxLDEuNTEtNy4zM0MyMi43Nyw1Mi41MiwxLDU1LjU0LDEsNzQuNzIsMSwxMDYuMiw3NS41OSwxNDUsMTM0LjY1LDE0NWM0NS4yOCwwLDU2LjctMjAuNDgsNTYuNy0zNi42NSwwLTEyLjcyLTExLTI3LjE2LTMwLjgzLTM1Ljc4Ii8+PHBhdGggZD0iTTE2MC41Miw3Mi41N2MxLjczLDguMTksMS43Myw5LjA1LDEuNzMsMTAuMTMsMCwxNC0xNS43NCwyMS43Ny0zNi40MywyMS43N0M3OSwxMDQuNDcsMzguMDgsNzcuMSwzOC4wOCw1OWExOC40NSwxOC40NSwwLDAsMSwxLjUxLTcuMzNsMy42Ni05LjA2QTYuNDMsNi40MywwLDAsMCw0Myw0NC41NEM0Myw1My43Niw3OS4zMyw4NCwxMjgsODRjMTIuNTEsMCwzMC42MS0yLjU4LDMwLjYxLTE3LjQ2YTE0LDE0LDAsMCwwLS4zMS0zLjQyWiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTIyMiwxNDUuNzNoNjIuMDl2OS42N0gyNTguNTR2NjMuMTRIMjQ3LjYyVjE1NS40SDIyMloiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik0yODcsMTY1LjZoMTAuNHY2Ljc2QTE2LjU3LDE2LjU3LDAsMCwxLDMxMiwxNjQuNDVhMTMuMTMsMTMuMTMsMCwwLDEsNS4zMS44NHY5LjM2YTE3LjgxLDE3LjgxLDAsMCwwLTYuMTQtMS4xNWMtNi4xMywwLTExLDMuMzMtMTMuNzMsOS42N3YzNS4zN0gyODdaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNMzIwLjczLDIwMy4zNWMwLTEwLDguMTEtMTYuMTIsMjEuNDItMTYuMTJhMzcuNDEsMzcuNDEsMCwwLDEsMTQuNDYsMi45MXYtNS42MWMwLTcuNDktNC40Ny0xMS4yNC0xMi45LTExLjI0LTQuODgsMC05Ljg4LDEuMzYtMTYuNDMsNC40OEwzMjMuNDMsMTcwYzcuOTEtMy43NCwxNC43Ny01LjQsMjEuNzQtNS40LDEzLjczLDAsMjEuNjMsNi43NiwyMS42MywxOC45M3YzNUgzNTYuNjFWMjE0YTI1LjEyLDI1LjEyLDAsMCwxLTE2LjQzLDUuNTFDMzI4LjYzLDIxOS40NywzMjAuNzMsMjEyLjkyLDMyMC43MywyMDMuMzVabTIxLjg0LDguNDNhMjAuMzQsMjAuMzQsMCwwLDAsMTQtNXYtOS4xNUEyNy41NCwyNy41NCwwLDAsMCwzNDMsMTk0LjQxYy03LjYsMC0xMi4yOCwzLjQzLTEyLjI4LDguNzNDMzMwLjcxLDIwOC4yNCwzMzUuNSwyMTEuNzgsMzQyLjU3LDIxMS43OFoiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik0zNzcuNjIsMTUwLjYyYTYuMzksNi4zOSwwLDAsMSw2LjM0LTYuNDUsNi40NSw2LjQ1LDAsMSwxLDAsMTIuOUE2LjM5LDYuMzksMCwwLDEsMzc3LjYyLDE1MC42MlptMTEuNTQsNjcuOTJoLTEwLjRWMTY1LjZoMTAuNFoiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik00MDEuNDMsMTY1LjZoMTAuNHY1LjNhMjEuOCwyMS44LDAsMCwxLDE1LjkxLTYuMzRjMTIuMTcsMCwyMC43LDguNDIsMjAuNywyMC42OXYzMy4yOUg0MzhWMTg3YzAtOC4zMi01LjEtMTMuNDEtMTMuMS0xMy40MWExNSwxNSwwLDAsMC0xMy4xMSw2Ljg2djM4LjA3aC0xMC40WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTQ1OS4yNSwxNTAuNjJhNi4zOSw2LjM5LDAsMCwxLDYuMzUtNi40NSw2LjQ1LDYuNDUsMCwwLDEsMCwxMi45QTYuMzksNi4zOSwwLDAsMSw0NTkuMjUsMTUwLjYyWm0xMS41NSw2Ny45Mkg0NjAuNFYxNjUuNmgxMC40WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTQ4My4wNywxNjUuNmgxMC40djUuM2EyMS44LDIxLjgsMCwwLDEsMTUuOTEtNi4zNGMxMi4xNywwLDIwLjcsOC40MiwyMC43LDIwLjY5djMzLjI5aC0xMC40VjE4N2MwLTguMzItNS4xLTEzLjQxLTEzLjExLTEzLjQxYTE1LDE1LDAsMCwwLTEzLjEsNi44NnYzOC4wN2gtMTAuNFoiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik01MzkuNjQsMTkxLjgxYTI2LjcxLDI2LjcxLDAsMCwxLDI2Ljg0LTI3LjE1LDI2LDI2LDAsMCwxLDE1LjcsNS4zVjE2NS42aDEwLjN2NTMuNjZjMCwxMy43My04LjY0LDIxLjQzLTI0LjI0LDIxLjQzYTQ0LjkyLDQ0LjkyLDAsMCwxLTIxLjMyLTUuMmw0LjA2LTguMTFjNiwzLjEyLDExLjM0LDQuNTcsMTYuOTUsNC41Nyw5LjI2LDAsMTQuMTUtNC4zNywxNC4xNS0xMi42OXYtNS45M0EyNC45MiwyNC45MiwwLDAsMSw1NjYuMjcsMjE5QzU1MS4zOSwyMTksNTM5LjY0LDIwNyw1MzkuNjQsMTkxLjgxWm0yNy44OCwxOC4wOWM2LDAsMTEuMTItMi4xOCwxNC41Ni02LjEzVjE3OS44NWExOS4xLDE5LjEsMCwwLDAtMTQuNTYtNi4xNGMtMTAsMC0xNy42OSw3LjktMTcuNjksMTguMVM1NTcuNTMsMjA5LjksNTY3LjUyLDIwOS45WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTYyMy43OCwyMDMuMzVjMC0xMCw4LjExLTE2LjEyLDIxLjQyLTE2LjEyYTM3LjQxLDM3LjQxLDAsMCwxLDE0LjQ2LDIuOTF2LTUuNjFjMC03LjQ5LTQuNDctMTEuMjQtMTIuOS0xMS4yNC00Ljg4LDAtOS44OCwxLjM2LTE2LjQzLDQuNDhMNjI2LjQ4LDE3MGM3LjkxLTMuNzQsMTQuNzctNS40LDIxLjc0LTUuNCwxMy43MywwLDIxLjYzLDYuNzYsMjEuNjMsMTguOTN2MzVINjU5LjY2VjIxNGEyNS4xMiwyNS4xMiwwLDAsMS0xNi40Myw1LjUxQzYzMS42OCwyMTkuNDcsNjIzLjc4LDIxMi45Miw2MjMuNzgsMjAzLjM1Wm0yMS44NCw4LjQzYTIwLjM0LDIwLjM0LDAsMCwwLDE0LTV2LTkuMTVBMjcuNTQsMjcuNTQsMCwwLDAsNjQ2LDE5NC40MWMtNy42LDAtMTIuMjgsMy40My0xMi4yOCw4LjczQzYzMy43NiwyMDguMjQsNjM4LjU1LDIxMS43OCw2NDUuNjIsMjExLjc4WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTY4MS44MSwxNjUuNmgxMC40djUuM2EyMS44LDIxLjgsMCwwLDEsMTUuOTEtNi4zNGMxMi4xNywwLDIwLjcsOC40MiwyMC43LDIwLjY5djMzLjI5aC0xMC40VjE4N2MwLTguMzItNS4xLTEzLjQxLTEzLjExLTEzLjQxYTE1LDE1LDAsMCwwLTEzLjEsNi44NnYzOC4wN2gtMTAuNFoiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik03ODEuMzQsMjEzLjU0YTI1LDI1LDAsMCwxLTE2LjIzLDUuODNjLTE1LDAtMjYuNzMtMTItMjYuNzMtMjcuMzZzMTEuNzYtMjcuMjQsMjYuOTQtMjcuMjRBMjcsMjcsMCwwLDEsNzgxLjIzLDE3MFYxNDUuNzNsMTAuNC0yLjI5djc1LjFINzgxLjM0Wm0tMTQuNzctMy4yMkExOSwxOSwwLDAsMCw3ODEuMjMsMjA0VjE4MGExOS4zNSwxOS4zNSwwLDAsMC0xNC42Ni02LjI0Yy0xMC4xOSwwLTE4LDcuOC0xOCwxOC4yUzc1Ni4zOCwyMTAuMzIsNzY2LjU3LDIxMC4zMloiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik0yODQuNjQsMzA3LjgxbDcuMTgsNy4yOGE0MC4wNyw0MC4wNywwLDAsMS0yOS4xMiwxMi41OWMtMjEuNTMsMC0zOC4xNy0xNi41NC0zOC4xNy0zNy41NXMxNi42NC0zNy41NCwzOC4xNy0zNy41NGMxMS40NCwwLDIyLjQ2LDQuNzgsMjkuMzMsMTIuNThsLTcuMjgsNy40OWEyOC45NCwyOC45NCwwLDAsMC0yMi4wNS0xMGMtMTUuMzksMC0yNywxMS44NS0yNywyNy40NXMxMS43NSwyNy40NiwyNy4zNSwyNy40NkEyOC4yNSwyOC4yNSwwLDAsMCwyODQuNjQsMzA3LjgxWiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTMyNC41OCwzMjcuNDdjLTE1LjYsMC0yNy43Ny0xMi0yNy43Ny0yNy40NiwwLTE1LjI4LDExLjY1LTI3LjI0LDI2LjUyLTI3LjI0LDE0LjU2LDAsMjUuNTgsMTIuMDYsMjUuNTgsMjcuNjZ2M2gtNDEuOEExNy44NCwxNy44NCwwLDAsMCwzMjUsMzE4LjYzLDIwLjYyLDIwLjYyLDAsMCwwLDMzOC40MSwzMTRsNi42Niw2LjU1QTMxLjU2LDMxLjU2LDAsMCwxLDMyNC41OCwzMjcuNDdabS0xNy4zNy0zMS44MmgzMS40MWMtMS41Ni04LjEyLTcuOC0xNC4xNS0xNS41LTE0LjE1QzMxNS4xMSwyODEuNSwzMDguNzcsMjg3LjIyLDMwNy4yMSwyOTUuNjVaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNMzU4Ljc5LDI3My42aDEwLjR2Ni43NmExNi41NywxNi41NywwLDAsMSwxNC41Ni03LjkxLDEzLjEzLDEzLjEzLDAsMCwxLDUuMzEuODR2OS4zNmExNy44MSwxNy44MSwwLDAsMC02LjE0LTEuMTVjLTYuMTMsMC0xMSwzLjMzLTEzLjczLDkuNjd2MzUuMzdoLTEwLjRaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNNDA0LjY1LDI4Mi4zM0gzOTMuNDJWMjczLjZoMTEuMjNWMjYwLjA4bDEwLjMtMi41djE2aDE1LjZ2OC43M0g0MTVWMzExYzAsNS40MSwyLjE4LDcuMzgsNy44LDcuMzhhMjAuMzUsMjAuMzUsMCwwLDAsNy41OS0xLjI1djguNzRhMzQuMjYsMzQuMjYsMCwwLDEtOS44OCwxLjU2Yy0xMC4yOSwwLTE1LjgxLTQuODktMTUuODEtMTRaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNNDM3LjEsMjU4LjYyYTYuMzksNi4zOSwwLDAsMSw2LjM1LTYuNDUsNi40NSw2LjQ1LDAsMCwxLDAsMTIuOUE2LjM5LDYuMzksMCwwLDEsNDM3LjEsMjU4LjYyWm0xMS41NSw2Ny45MmgtMTAuNFYyNzMuNmgxMC40WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTQ2OC44MiwyNzMuNnYtOGMwLTExLDYuMzUtMTcuMDYsMTgtMTcuMDZhMjcuMDUsMjcuMDUsMCwwLDEsNy40OS45NHY5YTIwLjczLDIwLjczLDAsMCwwLTYuNTUtLjk0Yy01LjcyLDAtOC41MywyLjYtOC41Myw4LjIydjcuOEg0OTQuM3Y4LjczSDQ3OS4yMnY0NC4yMWgtMTAuNFYyODIuMzNINDU2LjU1VjI3My42WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTUwMS4wNiwyNTguNjJhNi4zOSw2LjM5LDAsMCwxLDYuMzQtNi40NSw2LjQ1LDYuNDUsMCwxLDEsMCwxMi45QTYuMzksNi4zOSwwLDAsMSw1MDEuMDYsMjU4LjYyWm0xMS41NCw2Ny45Mkg1MDIuMlYyNzMuNmgxMC40WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTU2NC40LDMxMS44N2w2LjI0LDYuNzZhMjkuMjcsMjkuMjcsMCwwLDEtMjAuOTEsOC44NCwyNy40NiwyNy40NiwwLDAsMSwwLTU0LjkxLDMwLDMwLDAsMCwxLDIxLjMyLDguODRsLTYuNTUsNy4wN2ExOS42OCwxOS42OCwwLDAsMC0xNC41Ni02LjY2Yy05LjY3LDAtMTcuMTYsOC0xNy4xNiwxOC4yLDAsMTAuNCw3LjU5LDE4LjMxLDE3LjM3LDE4LjMxQzU1NS40NSwzMTguMzIsNTYwLjEzLDMxNi4yNCw1NjQuNCwzMTEuODdaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNNTc2LjM1LDMxMS4zNWMwLTEwLDguMTItMTYuMTIsMjEuNDMtMTYuMTJhMzcuNDUsMzcuNDUsMCwwLDEsMTQuNDYsMi45MXYtNS42MWMwLTcuNDktNC40OC0xMS4yNC0xMi45LTExLjI0LTQuODksMC05Ljg4LDEuMzYtMTYuNDMsNC40OEw1NzkuMDYsMjc4YzcuOS0zLjc0LDE0Ljc3LTUuNCwyMS43My01LjQsMTMuNzMsMCwyMS42NCw2Ljc2LDIxLjY0LDE4LjkzdjM1LjA1SDYxMi4yNFYzMjJhMjUuMTcsMjUuMTcsMCwwLDEtMTYuNDQsNS41MUM1ODQuMjYsMzI3LjQ3LDU3Ni4zNSwzMjAuOTIsNTc2LjM1LDMxMS4zNVptMjEuODUsOC40M2EyMC4zNywyMC4zNywwLDAsMCwxNC01di05LjE1YTI3LjU1LDI3LjU1LDAsMCwwLTEzLjYzLTMuMjJjLTcuNTksMC0xMi4yNywzLjQzLTEyLjI3LDguNzNDNTg2LjM0LDMxNi4yNCw1OTEuMTIsMzE5Ljc4LDU5OC4yLDMxOS43OFoiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik02MzkuMTcsMjgyLjMzSDYyNy45NFYyNzMuNmgxMS4yM1YyNjAuMDhsMTAuMy0yLjV2MTZoMTUuNnY4LjczaC0xNS42VjMxMWMwLDUuNDEsMi4xOCw3LjM4LDcuOCw3LjM4YTIwLjM1LDIwLjM1LDAsMCwwLDcuNTktMS4yNXY4Ljc0YTM0LjMyLDM0LjMyLDAsMCwxLTkuODgsMS41NmMtMTAuMywwLTE1LjgxLTQuODktMTUuODEtMTRaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNNjcxLjYyLDI1OC42MmE2LjM5LDYuMzksMCwwLDEsNi4zNC02LjQ1LDYuNDUsNi40NSwwLDAsMSwwLDEyLjlBNi4zOSw2LjM5LDAsMCwxLDY3MS42MiwyNTguNjJabTExLjU0LDY3LjkyaC0xMC40VjI3My42aDEwLjRaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNNzIwLjYsMjcyLjU2QTI3LjUxLDI3LjUxLDAsMSwxLDY5MywzMDAsMjcuMTcsMjcuMTcsMCwwLDEsNzIwLjYsMjcyLjU2Wk03MzcuODcsMzAwYzAtMTAuMjktNy43LTE4LjMtMTcuMjctMTguM3MtMTcuMzcsOC0xNy4zNywxOC4zLDcuNTksMTguNDEsMTcuMzcsMTguNDFDNzMwLjE3LDMxOC40Miw3MzcuODcsMzEwLjMxLDczNy44NywzMDBaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNNzU3Ljk0LDI3My42aDEwLjR2NS4zYTIxLjc4LDIxLjc4LDAsMCwxLDE1LjkxLTYuMzRjMTIuMTcsMCwyMC43LDguNDIsMjAuNywyMC42OXYzMy4yOUg3OTQuNTRWMjk1YzAtOC4zMi01LjA5LTEzLjQxLTEzLjEtMTMuNDFhMTUsMTUsMCwwLDAtMTMuMSw2Ljg2djM4LjA3aC0xMC40WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTU4MC4yNCw5My4zYzAsMTEuODksNy4xNSwxNy42NywyMC4xOSwxNy42N2E1Mi4xMSw1Mi4xMSwwLDAsMCwxMS44OS0xLjY4Vjk1LjUxYTI0Ljg0LDI0Ljg0LDAsMCwxLTcuNjgsMS4xNmMtNS4zNywwLTcuMzYtMS42OC03LjM2LTYuNzNWNjguOGgxNS41NlY1NC42SDU5Ny4yOHYtMThsLTE3LDMuNjhWNTQuNkg1NjlWNjguOGgxMS4yNVptLTUzLC4zMmMwLTMuNjgsMy42OS01LjQ3LDkuMjYtNS40N2E0My4xMiw0My4xMiwwLDAsMSwxMC4xLDEuMjZ2Ny4xNUEyMS41MSwyMS41MSwwLDAsMSw1MzYsOTkuMTljLTUuNDYsMC04LjczLTIuMS04LjczLTUuNTdtNS4yLDE3LjU2YzYsMCwxMC44NC0xLjI2LDE1LjM2LTQuMzF2My4zN2gxNi44MlY3NC41OGMwLTEzLjU2LTkuMTQtMjEtMjQuMzktMjEtOC41MiwwLTE2Ljk0LDItMjYsNi4xbDYuMSwxMi41MmM2LjUyLTIuNzQsMTItNC40MiwxNi44My00LjQyLDcsMCwxMC42MiwyLjczLDEwLjYyLDguMzF2Mi43M2E0OS41Myw0OS41MywwLDAsMC0xMi42Mi0xLjU4Yy0xNC4zMSwwLTIyLjkzLDYtMjIuOTMsMTYuNzMsMCw5Ljc4LDcuNzgsMTcuMjQsMjAuMTksMTcuMjRtLTkyLjQ0LS45NGgxOC4wOVY4MS40MmgzMC4yOXYyOC44MmgxOC4wOVYzNi42Mkg0ODguNDNWNjQuOTFINDU4LjE0VjM2LjYySDQ0MC4wNVpNMzcxLjEyLDgyLjM3YzAtOCw2LjMxLTE0LjEsMTQuNjItMTQuMWExNy4yMiwxNy4yMiwwLDAsMSwxMS43OCw0LjMyVjkyYTE2LjM2LDE2LjM2LDAsMCwxLTExLjc4LDQuNDJjLTguMiwwLTE0LjYyLTYuMS0xNC42Mi0xNC4wOW0yNi42MSwyNy44N2gxNi44M1YzMi45NGwtMTcsMy42OFY1Ny41NWEyOC4zLDI4LjMsMCwwLDAtMTQuMi0zLjY4Yy0xNi4xOSwwLTI4LjkyLDEyLjUxLTI4LjkyLDI4LjVBMjguMjUsMjguMjUsMCwwLDAsMzgyLjgsMTExYTI1LjEyLDI1LjEyLDAsMCwwLDE0LjkzLTQuODNabS03Ny4xOS00Mi43YzUuMzYsMCw5Ljg4LDMuNDcsMTEuNjcsOC44M0gzMDljMS42OC01LjU3LDUuODktOC44MywxMS41Ny04LjgzTTI5MS44Myw4Mi40N2MwLDE2LjIsMTMuMjUsMjguODIsMzAuMjgsMjguODIsOS4zNiwwLDE2LjItMi41MywyMy4yNS04LjQybC0xMS4yNi0xMGMtMi42MywyLjc0LTYuNTIsNC4yMS0xMS4xNCw0LjIxYTE0LjM5LDE0LjM5LDAsMCwxLTEzLjY4LTguODNoMzkuNjVWODQuMDVjMC0xNy42Ny0xMS44OC0zMC4zOS0yOC4wOC0zMC4zOWEyOC41NywyOC41NywwLDAsMC0yOSwyOC44MU0yNjIuNDksNTIuMDhjNiwwLDkuMzYsMy43OCw5LjM2LDguMzFzLTMuMzcsOC4zMS05LjM2LDguMzFIMjQ0LjYxVjUyLjA4Wm0tMzYsNTguMTZoMTguMDlWODMuNDJoMTMuNzdsMTMuODksMjYuODJoMjAuMTlsLTE2LjItMjkuNDVhMjIuMjcsMjIuMjcsMCwwLDAsMTMuODgtMjAuNzJjMC0xMy4yNS0xMC40MS0yMy40NS0yNi0yMy40NUgyMjYuNTJaIi8+PC9zdmc+)

- [Privacy Policy](https://www.redhat.com/en/about/privacy-policy?extIdCarryOver=true&sc_cid=701f2000001D8QoAAK)
- [Red Hat Training Policies](https://www.redhat.com/en/about/red-hat-training-policies)
- [Terms of Use](https://www.redhat.com/en/about/terms-use)
- [All policies and guidelines](https://www.redhat.com/en/about/all-policies-guidelines)
- [Release Notes](https://learn.redhat.com/t5/Red-Hat-Learning-Subscription/bg-p/RedHatLearningSubscriptionGroupblog-board)