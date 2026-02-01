## Affinity

The following example is a `Pod` spec with a rule that requires the pod be placed on a node with a label whose key is `e2e-az-NorthSouth` and whose value is either `e2e-az-North` or `e2e-az-South`:

**Example pod configuration file with a node affinity required rule**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  affinity:
    nodeAffinity: # 1 
      requiredDuringSchedulingIgnoredDuringExecution: # 2 
        nodeSelectorTerms:
        - matchExpressions:
          - key: e2e-az-NorthSouth # 3 
            operator: In # 4 
            values:
            - e2e-az-North # 5 
            - e2e-az-South # 6 
  containers:
  - name: with-node-affinity
    image: docker.io/ocpqe/hello-pod
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: [ALL]
# ...
```

[1] The stanza to configure node affinity.
[2] Defines a required rule.
[3, 5, 6] The key/value pair (label) that must be matched to apply the rule.
[4] The operator represents the relationship between the label on the node and the set of values in the `matchExpression` parameters in the `Pod` spec. This value can be `In`, `NotIn`, `Exists`, or `DoesNotExist`, `Lt`, or `Gt`.

There is no explicit _node anti-affinity_ concept, but using the `NotIn` or `DoesNotExist` operator replicates that behavior.

> Note
 If you are using node affinity and node selectors in the same pod configuration, note the following:
 -If you configure both `nodeSelector` and `nodeAffinity`, both conditions must be satisfied for the pod to be scheduled onto a -candidate node.
  -If you specify multiple `nodeSelectorTerms` associated with `nodeAffinity` types, then the pod can be scheduled onto a node if one of the `nodeSelectorTerms` is satisfied.
  -If you specify multiple `matchExpressions` associated with `nodeSelectorTerms`, then the pod can be scheduled onto a node only if all `matchExpressions` are satisfied.

## Taints and tolerations
A _taint_ allows a node to refuse a pod to be scheduled unless that pod has a matching _toleration_.

You apply taints to a node through the `Node` specification (`NodeSpec`) and apply tolerations to a pod through the `Pod` specification (`PodSpec`). When you apply a taint to a node, the scheduler cannot place a pod on that node unless the pod can tolerate the taint.

**Example taint in a node specification**
```yaml
apiVersion: v1
kind: Node
metadata:
  name: my-node
#...
spec:
  taints:
  - effect: NoExecute
    key: key1
    value: value1
#...
```

**Example toleration in a `Pod` spec**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
#...
spec:
  tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoExecute"
    tolerationSeconds: 3600
#...
```

A toleration matches a taint:

- If the `operator` parameter is set to `Equal`:
    - the `key` parameters are the same;
    - the `value` parameters are the same;
    - the `effect` parameters are the same.
    
- If the `operator` parameter is set to `Exists`:
    - the `key` parameters are the same;
    - the `effect` parameters are the same.

## Node selectors
A _node selector_ specifies a map of key/value pairs that are defined using custom labels on nodes and selectors specified in pods.

For the pod to be eligible to run on a node, the pod must have the same key/value node selector as the label on the node.

You can use a node selector to:
- place specific pods on specific nodes, 
- cluster-wide node selectors to place new pods on specific nodes anywhere in the cluster, 
- project node selectors to place new pods in a project on specific nodes.

>Note
You cannot add a node selector directly to an existing scheduled pod. You must label the object that controls the pod, such as deployment config.

For example, the following `Node` object has the `region: east` label:

### Node selectors on specific pods and nodes

**Sample `Node` object with a label**
```yaml
kind: Node
apiVersion: v1
metadata:
  name: ip-10-0-131-14.ec2.internal
  selfLink: /api/v1/nodes/ip-10-0-131-14.ec2.internal
  uid: 7bc2580a-8b8e-11e9-8e01-021ab4174c74
  resourceVersion: '478704'
  creationTimestamp: '2019-06-10T14:46:08Z'
  labels:
    kubernetes.io/os: linux
    ...
    region: east # 1 
    type: user-node
#...
```

[1] Labels to match the pod node selector.

A pod has the `type: user-node,region: east` node selector:

**Sample `Pod` object with node selectors**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: s1
#...
spec:
  nodeSelector: # 1 
    region: east
    type: user-node
#...
```

[1] Node selectors to match the node label. The node must have a label for each node selector.

### Default cluster-wide node selectors

With default cluster-wide node selectors, when you create a pod in that cluster, OpenShift Container Platform adds the default node selectors to the pod and schedules the pod on nodes with matching labels.

For example, the following `Scheduler` object has the default cluster-wide `region=east` and `type=user-node` node selectors:

**Example Scheduler Operator Custom Resource**

```yaml
apiVersion: config.openshift.io/v1
kind: Scheduler
metadata:
  name: cluster
#...
spec:
  defaultNodeSelector: type=user-node,region=east
#...
```

A node in that cluster has the `type=user-node,region=east` labels:

**Example `Node` object**
```yaml
apiVersion: v1
kind: Node
metadata:
  name: ci-ln-qg1il3k-f76d1-hlmhl-worker-b-df2s4
#...
  labels:
    region: east
    type: user-node
#...
```

**Example `Pod` object with a node selector**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: s1
#...
spec:
  nodeSelector:
    region: east
#...
```

When you create the pod using the example pod spec in the example cluster, the pod is created with the cluster-wide node selector and is scheduled on the labeled

### Project node selectors

With project node selectors, when you create a pod in this project, OpenShift Container Platform adds the node selectors to the pod and schedules the pods on a node with matching labels. 
If there is a cluster-wide default node selector, a project node selector takes preference.

For example, the following project has the `region=east` node selector:
**Example `Namespace` object**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: east-region
  annotations:
    openshift.io/node-selector: "region=east"
#...
```

The following node has the `type=user-node,region=east` labels:
**Example `Node` object**
```yaml
apiVersion: v1
kind: Node
metadata:
  name: ci-ln-qg1il3k-f76d1-hlmhl-worker-b-df2s4
#...
  labels:
    region: east
    type: user-node
#...
```

When you create the pod using the example pod spec in this example project, the pod is created with the project node selectors and is scheduled on the labeled node:
**Example `Pod` object**

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: east-region
#...
spec:
  nodeSelector:
    region: east
    type: user-node
#...
```

Show more

**Example pod list with the pod on the labeled node**

```shell-session
NAME     READY   STATUS    RESTARTS   AGE   IP           NODE                                       NOMINATED NODE   READINESS GATES
pod-s1   1/1     Running   0          20s   10.131.2.6   ci-ln-qg1il3k-f76d1-hlmhl-worker-b-df2s4   <none>           <none>
```

A pod in the project is not created or scheduled if the pod contains different node selectors.

## Using 'explain' learn about resources

List all available resources (names, short names, api versions...)
```bash
[install@ssttocp99 ~]$ oc api-resources | head
NAME                                  SHORTNAMES                                                                             APIVERSION                                     NAMESPACED   KIND
bindings                                                                                                                     v1                                             true         Binding
componentstatuses                     cs                                                                                     v1                                             false        ComponentStatus
configmaps                            cm                                                                                     v1                                             true         ConfigMap
endpoints                             ep                                                                                     v1                                             true         Endpoints
events                                ev                                                                                     v1                                             true         Event
limitranges                           limits                                                                                 v1                                             true         LimitRange
namespaces                            ns                                                                                     v1                                             false        Namespace
nodes                                 no                                                                                     v1                                             false        Node
persistentvolumeclaims                pvc                                                                                    v1                                             true         PersistentVolumeClaim
```

Details about specific resource
```bash
oc explain localvolumes  # only top level
oc explain localvolumes.spec.storageClassDevices # more specific
```

List all files as in yaml file
```bash
oc explain localvolumes --recursive
```

  
## Resize elasticsearch PVC

[Resize ElasticSearch PersistentVolumeClaim in RHOCP4 - Red Hat Customer Portal](https://access.redhat.com/solutions/6075191)

Some command to check status
```bash
# Section 1 : Identify the Environment and capture initial data
oc get deployment -l component=elasticsearch #  List down the `ElasticSearch deployment`
oc get pod -l component=elasticsearch # List down the current running `ElasticSearch pods`.
oc get pvc -n openshift-logging # Find out the `ElasticSearch pvc`.

# Section 2 : Perform the actual Operation for ElasticSearch storage resize.
oc edit clusterloggings.logging.openshift.io instance -n openshift-logging # Edit the `clusterlogging` and update the size of `ElasticSearch` storage.
oc scale deployment elasticsearch-cdm-xcahid2f-1  --replicas=0 -n openshift-logging #  Scale down first `ElasticSearch deployment` instance:
oc delete pvc elasticsearch-elasticsearch-cdm-xcahid2f-1  -n openshift-logging # Delete the corresponding first `PVC`.
oc get pvc -n openshift-logging # Check if the volume is spawned and bound by the operator with the updated size
oc scale deployment elasticsearch-cdm-xcahid2f-1  --replicas=1 -n openshift-logging # Scale up the `ElasticSearch` instance again:
watch oc exec -c elasticsearch elasticsearch-cdm-xcahid2f-1-76bbd48d8c-xxxx -- es_util --query=_cat/health?v # Watch the progress of data resync, until "active_shards_percent" reaches 100% again and status ids green!!! use name of teh newly created pod
# Repeat the steps from `2` to `7` again for `elasticsearch-deployment-2` and `elasticsearch-deployment-3`.
for i in $(oc get po -l component=elasticsearch -o name); do oc exec -c elasticsearch $i -- es_util --query=_cat/health?v ;done # Verify that all `ElasticSearch pods` are running and in a healthy state.

oc get cronjob
```

## Elasticsearch manage indexes
Login
```
oc login --username=rmarinsek --server=https://api.bdigital-gbx.ocp.bpo.ssg:6443
```

Get pods
```
oc get pods
```

Check disk usage on specific pod
```
oc rsh -c elasticsearch elasticsearch-cdm-bnpfqeda-1-66d5794dbb-tlc8c
```

List, get indexes sizes
```
oc exec -c elasticsearch elasticsearch-cdm-bnpfqeda-1-66d5794dbb-tlc8c -- es_util --query=_cat/indices?h=i,creation.date,creation.date.string,store.size,pri.store.size | grep 11-27
```

Delete index
```
oc exec -c elasticsearch elasticsearch-cdm-ov096pjg-3-7dc4c547cb-vcxnc  -- es_util --query=app-bpb-json-001441,app-bpb-json-001444 -XDELETE
```
## Installing OC operators
Work log  
- create a YAML file with configuration of machineset, watch for labels and roles
- created machineset `oc create -f GBX_PRD_new-infra-node.yaml`  
- add taint `oc adm taint nodes my-node-name node-role.kubernetes.io/infra:NoSchedule`
- check with `oc describe nodes bdigital-gbx-smjs7-infra-qhf7w`

source: [Creating infrastructure machine sets | Machine management | OpenShift Container Platform 4.9](https://docs.openshift.com/container-platform/4.9/machine_management/creating-infrastructure-machinesets.html)  


## Deploy Grafana operator

GitHub
[grafana-operator/deploy/examples/oauth at master · grafana-operator/grafana-operator · GitHub](https://github.com/grafana-operator/grafana-operator/tree/master/deploy/examples/oauth)

Steps taken to install Grafana operator in OpenShift environment
- request token and login
- oc login token….
- crate a namespace
   `oc adm new-project grafana`

- From inside of the git clone folder of      run the following:  
kustomization
    `oc create -f crds.yaml -f deployment.yaml -f rbac.yaml`
   
- Create a session ticket
   `oc create -f session-secret.yaml -n grafana`

- Create the additional [cluster role](https://github.com/grafana-operator/grafana-operator/blob/master/deploy/examples/oauth/cluster_role.yaml) and [binding](https://github.com/grafana-operator/grafana-operator/blob/master/deploy/examples/oauth/cluster_role_binding.yaml).
    `oc create -n grafana -f cluster_role.yaml -f cluster_role_binding.yaml`

- Create config map  
    `oc create -n grafana -f ocp-injected-certs.yml`

- Create Grafan:
    `oc create -n grafana -f Grafana.yaml`

Some troubleshooting commans  
```
oc describe grafana -n grafana
oc get pv,pvc -n grafana pesisten volume
````
## Deploy new MachineSet
source: [Creating a machine set on vSphere - Creating machine sets | Machine management | OpenShift Container Platform 4.9](https://docs.openshift.com/container-platform/4.9/machine_management/creating_machinesets/creating-machineset-vsphere.html#creating-machineset-vsphere)

## Creating infrastructure machine sets
[Creating infrastructure machine sets | Machine management | OpenShift Container Platform 4.9](https://docs.openshift.com/container-platform/4.9/machine_management/creating-infrastructure-machinesets.html)

## Working with pods, nodes
In order to replace a node with the new image, the following steps should be done in written order.
1. Mark the node as unschedulable: 
```bash
oc get nodes # list nodes
oc adm cordon <node1>
```
2. Evacuate the pods:
```bash
oc adm drain <node1> <node2> --ignore-daemonsets=true --force=true
```
After node is drained, it can be deleted, if it is part of MachineSet a new one will be deployed automaticlt
3. Mark the node as schedulable when done.
```bash
oc adm uncordon <node1>
```

Some commands for pods:
```bash
oc get pods
oc get pod -o wide 
oc describe pod <pod> | grep -i control
oc delete pod -n openshift-monitoring <pod> # -n define namespace of pod

```
## Delete particular machine/node
To delete specific node/machine, you have to taint it correctly using command below
```bash
    oc annotate machine/<machine-name> -n openshift-machine-api machine.openshift.io/cluster-api-delete-machine="true"
```

once machine is anotated follow **cordon, drain*** procedures

If it is machine in machineset, scale it appropriately after with:
```bash
oc describe machineset <machineset-name> # see current machineset config
oc scale --replicas=<number-of-replicas> machineset/<machineset-name> -n openshift-machine-api
```

## Creating ConfiMap from CLI
The `ConfigMap` object provides mechanisms to inject containers with configuration data while keeping containers agnostic of OpenShift Container Platform. A config map can be used to store fine-grained information like individual properties or coarse-grained information like entire configuration files or JSON blobs.
**`ConfigMap` objects reside in a project.**
They can only be referenced by pods in the same project.
```bash
oc create configmap <configmap_name> [options]
```
Create a config map by specifying a specific file:
```bash
oc create configmap game-config-2 \
  --from-file=example-files/game.properties \
  --from-file=example-files/ui.properties
```
Verify the results:
```bash
oc get configmaps game-config-2 -o yaml
```

## Elasticsearch troubleshoot disk flooding

[Troubleshooting for Critical Alerts - Troubleshooting Logging | Logging | OpenShift Container Platform 4.13](https://docs.openshift.com/container-platform/4.13/logging/troubleshooting/cluster-logging-troubleshooting-for-critical-alerts.html#Elasticsearch-Cluster-Health-is-Red)

## Certificate renew - clusteroperator

[How to renew/rotate the certificate for cluster operator operator-lifecycle-manager-packageserver in RHOCP4 - Red Hat Customer Portal](https://access.redhat.com/solutions/6999798)


```
oc login --token=sha256~sDKady3GJjKwH7tHF-Z5U9kHp230Z1fcLEcIxPZxJFA --server=https://api.bdigital-dta.ocp.bpo.ssg:6443
oc get -o yaml clusteroperator operator-lifecycle-manager-packageserver
oc describe operator-lifecycle-manager-packageserver
oc describe clusteroperator operator-lifecycle-manager-packageserver
oc get pod -l 'app in (catalog-operator, olm-operator, packageserver, package-server-manager)' -n openshift-operator-lifecycle-manager
oc project  openshift-kube-apiserver
oc logs kube-apiserver-example.com -c kube-apiserver
oc get clusterversion
oc get co operator-lifecycle-manager-packageserver -o  yaml
oc get pods -n openshift-operator-lifecycle-manager
oc get apiservice v1.packages.operators.coreos.com
oc get apiservice v1.packages.operators.coreos.com -o jsonpath='{.spec.caBundle}' | base64 -d | openssl x509 -noout -text
oc delete secret catalog-operator-serving-cert olm-operator-serving-cert packageserver-service-cert -n openshift-operator-lifecycle-manager
oc delete pod -l 'app in (catalog-operator, olm-operator, packageserver, package-server-manager)' -n openshift-operator-lifecycle-manager
oc get pods -n openshift-operator-lifecycle-manager
oc delete apiservice v1.packages.operators.coreos.com
oc get apiservice v1.packages.operators.coreos.com -o jsonpath='{.spec.caBundle}' | base64 -d | openssl x509 -noout -text
```
