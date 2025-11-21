
# Chapter 1.  Declarative Resource Management

## Resource Manifests

### Objectives

- Deploy and update applications from resource manifests that are stored as YAML files.
    

An application in a Kubernetes cluster often consists of multiple resources that work together. Each resource has a definition and a configuration. Many of the resource configurations share common attributes that must match to operate correctly. Imperative commands configure each resource, one at time. However, using imperative commands has some issues:

- Impaired reproducibility
    
- Lacking version control
    
- Lacking support for GitOps
    

Rather than imperative commands, declarative commands are instead the preferred way to manage resources, by using resource manifests. A resource manifest is a file, in JSON or YAML format, with resource definition and configuration information. Resource manifests simplify the management of Kubernetes resources, by encapsulating all the attributes of an application in a file or a set of related files. Kubernetes uses declarative commands to read the resource manifests and to apply changes to the cluster to meet the state that the resource manifest defines.

The resource manifests are in YAML or JSON format, and thus can be version-controlled. Version control of resource manifests enables tracing of configuration changes. As such, adverse changes can be rolled back to an earlier version to support recoverability.

Resource manifests ensure that applications can be precisely reproduced, typically with a single command to deploy many resources. The reproducibility from resource manifests supports the automation of the GitOps practices of continuous integration and continuous delivery (CI/CD).

#### Imperative Versus Declarative Workflows

The Kubernetes CLI uses both imperative and declarative commands. Imperative commands perform an action that is based on a command, and use command names that closely reflect the action. In contrast, declarative commands use a resource manifest file to declare the intended state of a resource.

A Kubernetes manifest is a YAML- or JSON-formatted file with declaration statements for Kubernetes resources such as deployments, pods, or services. Instead of using imperative commands to create Kubernetes resources, manifest files provide all the details for the resource in a single file. Working with manifest files enables the use of more reproducible processes. Instead of reproducing sequences of imperative commands, manifest files contain the entire definition of resources and can be applied in a single step. Using manifest files is also useful for tracking system configuration changes in a source code management system.

Given a new or updated resource manifest, Kubernetes provides commands that compare the intended state that is specified in the resource manifest to the current state of the resource. These commands then apply transformations to the current state to match the intended state.

#### Imperative Workflow

An imperative workflow is useful for developing and testing. The following example uses the `kubectl create deployment` imperative command, to create a deployment for a MYSQL database.

[user@host ~]$ **`kubectl create deployment db-pod --port 3306 \   --image registry.ocp4.example.com:8443/rhel8/mysql-80`**
deployment.apps/db-pod created

In addition to using verbs that reflect the action of the command, imperative commands use options to provide the details. The example command uses the `--port` and the `--image` options to provide the required details to create the deployment.

The use of imperative commands affects applying changes to live resources. For example, the pod from the previous deployment would fail to start due to missing environment variables. The following `kubectl set env deployment` imperative command resolves the problem by adding the required environment variables to the deployment:

[user@host ~]$ **`kubectl set env deployment/db-pod \   MYSQL_USER='user1' \   MYSQL_PASSWORD='mypa55w0rd' \   MYSQL_DATABASE='items'`**
deployment.apps/db-pod updated

Executing this `kubectl set env deployment` command changes the deployment resource named `db-pod`, and provides the extra needed variables to start the container. A developer can continue building out the application, by using imperative commands to add components, such as services, routes, volume mounts, and persistent volume claims. With the addition of each component, the developer can run tests to ensure that the component correctly executes the intended function.

Imperative commands are useful for developing and experimenting. With imperative commands, a developer can build up an application one component at a time. When a component is added, the Kubernetes cluster provides error messages that are specific to the component. The process is analogous to using a debugger to step through code execution one line at a time. Using imperative commands usually provides clearer error messages, because an error occurs after adding a specific component.

However, long command lines and a fragmented application deployment are not ideal for deploying an application in production. With imperative commands, changes are a sequence of commands that must be maintained to reflect the intended state of the resources. The sequence of commands must be tracked and kept up to date.

#### Using Declarative Commands

Instead of tracking a sequence of commands, a manifest file captures the intended state of the sequence. In contrast to using imperative commands, declarative commands use a manifest file, or a set of manifest files, to combine all the details for creating those components into YAML files that can be applied in a single command. Future changes to the manifest files require only reapplying the manifests. Instead of tracking a sequence of complex commands, version control systems can track changes to the manifest file.

Although manifest files can also use the JSON syntax, YAML is generally preferred and is more popular. To continue the debugging analogy, debugging an application that is deployed from manifests is similar to trying to debug a full, completed running application. It can take more effort to find the source of the error, especially when the error is not a result of manifest errors.

#### Creating Kubernetes Manifests

Creating manifest files from scratch can take time. You can use the following techniques to provide a starting point for your manifest files:

- Use the YAML view of a resource from the web console.
    
- Use imperative commands with the `--dry-run=client` option to generate manifests that correspond to the imperative command.
    

The `kubectl explain` command provides the details for any field in the manifest. For example, use the `kubectl explain deployment.spec.template.spec` command to view field descriptions that specify a pod object within a deployment manifest.

To create a starter deployment manifest, use the `kubectl create deployment` command to generate a manifest by using the `--dry-run=client` option:

[user@host ~]$ **`kubectl create deployment hello-openshift -o yaml \   --image registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:v1.0 \   --save-config \ ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)   --dry-run=client \ ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)   > ~/my-app/example-deployment.yaml`**

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_creating_kubernetes_manifests-CO1-1)|The `--save-config` option adds configuration attributes that declarative commands use. For `deployments` resources, this option saves the resource configuration in an `kubectl.kubernetes.io/last-applied-configuration` annotation.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_creating_kubernetes_manifests-CO1-2)|The `--dry-run=client` option prevents the command from creating resources in the cluster.|

The following example shows a minimal deployment manifest file, not production-ready, for the `hello-openshift` deployment:

apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
      _...output omitted..._
  creationTimestamp: null
  labels:
    app: hello-openshift
  name: hello-openshift
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-openshift
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: hello-openshift
    spec:
      containers:
      - image: quay.io/redhattraining/hello-world-nginx:v1.0
        name: hello-world-nginx
        resources: {}
status: {}

When using imperative commands to create manifests, the resulting manifests might contain fields that are not necessary for creating a resource. For example, the following example changes the manifest by removing the empty and null fields. Removing unnecessary fields can significantly reduce the length of the manifests, and in turn reduce the overhead to work with them.

Additionally, you might need to further customize the manifests. For example, in a deployment, you might customize the number of replicas, or declare the ports that the deployment provides. The following notes explain the additional changes:

apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: `resource-manifests` ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
  labels:
    app: hello-openshift
  name: hello-openshift
spec:
  `replicas: 2` ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
  selector:
    matchLabels:
      app: hello-openshift
  template:
    metadata:
      labels:
        app: hello-openshift
    spec:
      containers:
      - image: quay.io/redhattraining/hello-world-nginx:v1.0
        name: hello-world-nginx
        ports:
        - `containerPort: 8080` ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
          protocol: TCP

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_creating_kubernetes_manifests-CO2-1)|Add a namespace attribute to prevent deployment to the wrong project.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_creating_kubernetes_manifests-CO2-2)|Requires two replicas instead of one.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_creating_kubernetes_manifests-CO2-3)|Specifies the container port for the service to use.|

You can create a manifest file for each resource that you manage. Alternatively, add each of the manifests to a single multi-part YAML file, and use a `---` line to separate the manifests.

---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: resource-manifests
  annotations:
  _...output omitted..._
---
apiVersion: v1
kind: Service
metadata:
  namespace: resource-manifests
  labels:
    app: hello-openshift
  name: hello-openshift
spec:
  _...output omitted..._

Using a single file with multiple manifests versus using manifests that are defined in multiple manifest files is a matter of organizational preference. The single file approach has the advantage of keeping together related manifests. With the single file approach, it can be more convenient to change a resource that must be reflected across multiple manifests. In contrast, keeping manifests in multiple files can be more convenient for sharing resource definitions with others.

After creating manifests, you can test them in a non-production environment, or proceed to deploy the manifests. Validate the resource manifests before deploying applications in the production environment.

#### Declarative Workflows

Declarative commands use a resource manifest instead of adding the details to many options on the command line. To create a resource, use the ``kubectl create -f _`resource.yaml`_`` command. Instead of a file name, you can pass a directory to the command to process all the resource files in a directory. Add the `--recursive=true` or `-R` option to recursively process resource files that are provided in multiple subdirectories.

The following example creates the resources from the manifests in the `my-app` directory. In this example, the `my-app` directory contains the `example-deployment.yaml` and `service/example-service.yaml` files from previously.

[user@host ~]$ **`tree my-app`**
my-app
├── example_deployment.yaml
└── service
    └── example_service.yaml

[user@host ~]$ **`kubectl create -R -f ~/my-app`**
deployment.apps/hello-openshift created
service/hello-openshift created

The command also accepts a URL:

[user@host ~]$ **`kubectl create -f \   https://example.com/example-apps/deployment.yaml`**
deployment.apps/hello-openshift created

#### Updating Resources

The `kubectl apply` command can also create resources with the same `-f` option that is illustrated with the `kubectl create` command. However, the `kubectl apply` command can also update a resource.

Updating resources is more complex than creating resources. The `kubectl apply` command implements several techniques to apply the updates without causing issues.

The `kubectl apply` command writes the contents of the configuration file to the `kubectl.kubernetes.io/last-applied-configuration` annotation. The `kubectl create` command can also generate this annotation by using the `--save-config` option. The `kubectl apply` command uses the `last-applied-configuration` annotation to identify fields that are removed from the configuration file and that must be cleared from the live configuration.

Although the `kubectl create -f` command can create resources from a manifest, the command is imperative and thus does not account for the current state of a live resource. Executing `kubectl create -f` against a manifest for a live resource gives an error. In contrast, the `kubectl apply -f` command is declarative, and considers the difference between the current resource state in the cluster and the intended resource state that is expressed in the manifest.

For example, to update the container's image from version `v1.0` to `latest`, first update the YAML resource manifest to specify the new tag on the image. Then, use the `kubectl apply` command to instruct Kubernetes to create a version of the deployment resource by using the updated image version that is specified in the manifest.

#### YAML Validation

Before applying the changes to the resource, use the `--dry-run=server` and the `--validate=true` flags to inspect the file for errors.

- The `--dry-run=server` option submits a server-side request without persisting the resource.
    
- The `--validate=true` option uses a schema to validate the input and fails the request if it is invalid.
    

Any syntax errors in the YAML are included in the output. Most importantly, the `--dry-run=server` option prevents applying any changes to the Kubernetes runtime.

[user@host ~]$ **`kubectl apply -f ~/my-app/example-deployment.yaml \   --dry-run=server --validate=true`**
deployment.apps/hello-openshift created (server dry-run) ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_yaml_validation-CO3-1)|The output line that ends in `(server dry-run)` provides the action that the resource file would perform if applied.|

> Note
The `--dry-run=client` option prints only the object that would be sent to the server. The cluster resource controllers can refuse a manifest even if the syntax is valid YAML. In contrast, the `--dry-run=server` option sends the request to the server to confirm that the manifest conforms to current server policies, without creating resources on the server.

#### Comparing Resources

Use the `kubectl diff` command to review differences between live objects and manifests. When updating resource manifests, you can track differences in the changed files. However, many manifest changes, when applied, do not change the state of the cluster resources. A text-based diff tool would show all such differences, and result in a noisy output.

In contrast, using the `kubectl diff` command might be more convenient to preview changes. The `kubectl diff` command emphasizes the significant changes for the Kubernetes cluster. Review the differences to validate that manifest changes have the intended effect.

[user@host ~]$ **`kubectl diff -f example-deployment.yaml`**
_...output omitted..._
diff -u -N /tmp/LIVE-2647853521/apps.v1.Deployment.resource...
--- /tmp/LIVE-2647853521/apps.v1.Deployment.resource-manife...
+++ /tmp/MERGED-2640652736/apps.v1.Deployment.resource-mani...
@@ -6,7 +6,7 @@
     kubectl.kubernetes.io/last-applied-configuration: |
       _...output omitted..._
   creationTimestamp: "2023-04-27T16:07:47Z"
-  generation: 1 ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
+  `generation: 2`
   labels:
     app: hello-openshift
   name: hello-openshift
@@ -32,7 +32,7 @@
         app: hello-openshift
     spec:
       containers:
-      - image: registry.ocp4.example.com:8443/.../hello-world-nginx:v1.0 ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
+      - `image: registry.ocp4.example.com:8443/.../hello-world-nginx:latest`
         imagePullPolicy: IfNotPresent
         name: hello-openshift
         ports:

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_comparing_resources-CO4-1)|The line that starts with the `-` character shows that the current deployment is on generation 1. The following line, which starts with the `+` character, shows that the generation changes to 2 when the manifest file is applied.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_comparing_resources-CO4-2)|The image line, which starts with the `-` character, shows that the current image uses the `v1.0` version. The following line, which starts with the `+` character, shows a version change to `latest` when the manifest file is applied.|

Kubernetes resource controllers automatically add annotations and attributes to the live resource that make the output of other text-based diff tools misleading, by reporting many differences that have no impact on the resource configuration. Extracting manifests from live resources and making comparisons with tools such as the `diff` command reports many differences of no value. Using the `kubectl diff` command confirms that a live resource matches a resource configuration that a manifest provides. GitOps tools depend on the `kubectl diff` command to determine whether anyone changed resources outside the GitOps workflow. Because the tools themselves cannot know all details about how any controllers might change a resource, the tools defer to the cluster to determine whether a change is meaningful.

#### Update Considerations

When using the `oc diff` command, recognize when applying a manifest change does not generate new pods. For example, if an updated manifest changes only values in secret or a configuration map, then applying the updated manifest does not generate new pods that use those values. Because pods read secret and configuration maps at startup, in this case applying the updated manifest leaves the pods in a vulnerable state, with stale values that are not synchronized with the updated secret or with the configuration map.

As a solution, use the ``oc rollout restart deployment _`deployment-name`_`` command to force a restart of the pods that are associated with the deployment. The forced restart generates pods that use the new values from the updated secret or configuration map.

In deployments with a single replica, you can also resolve the problem by deleting the pod. Kubernetes responds by automatically creating a pod to replace the deleted pod. However, for multiple replicas, using the `oc rollout` command to restart the pods is preferred, because the pods are stopped and replaced in a smart manner that minimizes downtime.

This course covers other resource management mechanisms that can automate or eliminate some of these challenges.

#### Applying Changes

The `kubectl create` command attempts to create the specified resources in the manifest file. Using the `kubectl create` command generates an error if the targeted resources are already live in the cluster. In contrast, the `kubectl apply` command compares three sources to determine how to process the request and to apply changes.

1. The manifest file
    
2. The live configuration of the resource in the cluster
    
3. The configuration that is stored in the `last-applied-configuration` annotation
    

If the specified resource in the manifest file does not exist, then the `kubectl apply` command creates the resource. If any fields in the `last-applied-configuration` annotation of the live resource are not present in the manifest, then the command removes those fields from the live configuration. After applying changes to the live resource, the `kubectl apply` command updates the `last-applied-configuration` annotation of the live resource to account for the change.

When creating a resource, the `--save-config` option of the `kubectl create` command produces the required annotations for future `kubectl apply` commands to operate.

### Patching Kubernetes Resources

You can modify objects in OpenShift in a repeatable way with the `oc patch` command. The `oc patch` command updates or adds fields in an existing object from a provided JSON or YAML snippet or file. A software developer might distribute a patch file or snippet to fix problems before a full update is available.

To patch an object from a snippet, use the `oc patch` command with the `-p` option and the snippet. The following example updates the `hello` deployment to have a CPU resource request of `100m` with a JSON snippet:

[user@host ~]$ **`oc patch deployment hello -p \`**
    **`'{"spec":{"template":{"spec":{"containers":[{"name": \`**
    **`"hello-rhel7","resources": {"requests": {"cpu": "100m"}}}]}}}}'`**
deployment/hello patched

To patch an object from a patch file, use the `oc patch` command with the `--patch-file` option and the location of the patch file. The following example updates the `hello` deployment to include the content of the `~/volume-mount.yaml` patch file:

[user@host ~]$ **`oc patch deployment hello --patch-file ~/volume-mount.yaml`**
deployment.apps/hello patched

The contents of the patch file describe mounting a persistent volume claim as a volume:

spec:
  template:
    spec:
      containers:
        - name: hello
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html/
      volumes:
        - name: www
          persistentVolumeClaim:
            claimName: nginx-www

This patch results in the following manifest for the `hello` deployment:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  _...output omitted..._
spec:
  _...output omitted..._
  template:
    _...output omitted..._
    spec:
      containers:
        _...output omitted..._
        name: server
        _...output omitted..._
        volumeMounts:
        - `mountPath: /usr/share/nginx/html/           name: www`
        - mountPath: /etc/nginx/conf.d/
          name: tls-conf
      _...output omitted..._
      volumes:
      - configMap:
          defaultMode: 420
          name: tls-conf
        name: tls-conf
      - `persistentVolumeClaim:           claimName: nginx-www         name: www`
_...output omitted..._

The patch applies to the `hello` deployment regardless of whether the `www` volume mount exists. The `oc patch` command modifies existing fields in the object that are specified in the patch:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  _...output omitted..._
spec:
  _...output omitted..._
  template:
    _...output omitted..._
    spec:
      containers:
        _...output omitted..._
        name: server
        _...output omitted..._
        volumeMounts:
        - mountPath: _`/usr/share/nginx/www/`_ ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
          name: www
        - mountPath: /etc/nginx/conf.d/
          name: tls-conf
      _...output omitted..._
      volumes:
      - configMap:
          defaultMode: 420
          name: tls-conf
        name: tls-conf
      - persistentVolumeClaim:
          claimName: _`deprecated-www`_ ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
        name: www
_...output omitted..._

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_patching_kubernetes_resources-CO5-1) [![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_patching_kubernetes_resources-CO5-2)|The `www` volume already exists. The patch replaces the existing data with the new data.|


## Kustomize Overlays

### Objectives

- Deploy and update applications from resource manifests that are augmented by Kustomize.
    

### Kustomize

When using Kubernetes, multiple teams use multiple environments, such as development, staging, testing, and production, to deploy applications. These environments use applications with minor configuration changes.

Many organizations deploy a single application to multiple data centers for multiple teams and regions. Depending on the load, the organization needs a different number of replicas for every region. The organization might need various configurations that are specific to a data center or team.

All these use cases require a single set of manifests with multiple customizations at multiple levels. Kustomize can support such use cases.

Kustomize is a configuration management tool to make declarative changes to application configurations and components and preserve the original base YAML files. You group in a directory the Kubernetes resources that constitute your application, and then use Kustomize to copy and adapt these resource files to your environments and clusters. Both the `kubectl` and `oc` commands integrate the Kustomization tool.

### Kustomize File Structure

Kustomize works on directories that contain a `kustomization.yaml` file at the root. Kustomize supports compositions and customization of different resources such as deployment, service, and secret. You can use patches to apply customization to different resources. Kustomize has a concept of _base_ and _overlays_.

#### Base

A base directory contains a `kustomization.yaml` file. The `kustomization.yaml` file has a list resource field to include all resource files. As the name implies, all resources in the base directory are a common resource set. You can create a base application by composing all common resources from the base directory.

The following diagram shows the structure of a base directory:

base
├── configmap.yaml
├── deployment.yaml
├── secret.yaml
├── service.yaml
├── route.yaml
└── kustomization.yaml

The base directory has YAML files to create configuration map, deployment, service, secret, and route resources. The base directory also has a `kustomization.yaml` file, such as the following example:

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- configmap.yaml
- deployment.yaml
- secret.yaml
- service.yaml
- route.yaml

The `kustomization.yaml` file lists all resource files.

#### Overlays

Kustomize overlays declarative YAML artifacts, or patches, that override the general settings without modifying the original files. The overlay directory contains a `kustomization.yaml` file. The `kustomization.yaml` file can refer to one or more directories as bases. Multiple overlays can use a common base kustomization directory.

The following diagram shows the structure of all Kustomize directories:

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do280-4.14/images/declarative/kustomize/assets/kustomize-directory-structure.svg)|

Figure 1.4: Kustomize file structure

The following example shows the directory structure of the `frontend-app` directory containing the `base` and `overlay` directories:

```
[user@host frontend-app]$ 
```

The following example shows a `kustomization.yaml` file in the `overlays/development` directory:

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev-env
resources:
- ../../base

The `frontend-app/overlay/development/kustomization.yaml` file uses the base kustomization file at `../../base` to create all the application resources in the `dev-env` namespace.

Kustomize provides fields to set values for all resources in the kustomization file:

|Field|Description|
|:--|:--|
|`namespace`|Set a specific namespace for all resources.|
|`namePrefix`|Add a prefix to the name of all resources.|
|`nameSuffix`|Add a suffix to the name of all resources.|
|`commonLabels`|Add labels to all resources and selectors.|
|`commonAnnotations`|Add annotations to all resources and selectors.|

You can customize for multiple environments by using overlays and patching. The `patches` mechanism has two elements: `patch` and `target`.

Previously, Kustomize used the `PatchesJson6902` and `PatchesStrategicMerge` keys to add resource patches. These keys are deprecated in Kustomize version 5 and are replaced with a single key. However, the content of the patches key continues to use the same patch formats.

You can use JSON Patch and strategic merge patches. See the references section for further information about both patch formats.

The following is an example of a `kustomization.yaml` file in the `overlays/testing` directory:

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: test-env
patches: ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
- patch: |-
    - op: replace ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
      path: /metadata/name
      value: frontend-test
  target: ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
    kind: Deployment
    name: frontend
- patch: |- ![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
    - op: replace
      path: /spec/replicas
      value: 15
  target:
    kind: Deployment
    name: frontend
resources: ![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)
- ../../base
commonLabels: ![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)
  env: test

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_overlays-CO8-1)|The `patches` field contains a list of patches.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_overlays-CO8-2)|The `patch` field defines operation, path, and value keys. In this example, the name changes to `frontend-test`.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_overlays-CO8-3)|The `target` field specifies the kind and name of the resource to apply the patch. In this example, you are changing the `frontend` deployment name to `frontend-test`.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_overlays-CO8-4)|This patch updates the number of replicas of the `frontend` deployment.|
|[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_overlays-CO8-5)|The `frontend-app/overlay/testing/kustomization.yaml` file uses the base kustomization file at `../../base` to create an application.|
|[![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)](https://rol.redhat.com/rol/app/#_overlays-CO8-6)|The `commonLabels` field adds the `env: test` label to all resources.|

The `patches` mechanism also provides an option to include patches from a separate YAML file by using the `path` key.

The following example shows a `kustomization.yaml` file that uses a `patch.yaml` file:

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod-env
patches: ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
- path: patch.yaml ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
  target: ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
    kind: Deployment
    name: frontend
  options:
    allowNameChange: true ![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
resources: ![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)
- ../../base
commonLabels: ![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)
  env: prod

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_overlays-CO9-1)|The `patches` field lists the patches that are applied by using a production kustomization file.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_overlays-CO9-2)|The `path` field specifies the name of the patching YAML file.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_overlays-CO9-3)|The `target` field specifies the kind and name of the resource to apply the patch. In this example, you are targeting the `frontend` deployment.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_overlays-CO9-4)|The `allowNameChange` field enables kustomization to update the name by using a patch YAML file.|
|[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_overlays-CO9-5)|The `frontend-app/overlay/production/kustomization.yaml` file uses the base kustomization file at `../../base` to create an application.|
|[![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)](https://rol.redhat.com/rol/app/#_overlays-CO9-6)|The `commonLabels` field adds an `env: prod` label to all resources.|

The `patch.yaml` file has the following content:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-prod ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
spec:
  replicas: 5 ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_overlays-CO10-1)|The `metadata.name` field in the patch file updates the `frontend` deployment name to `frontend-prod` if the `allowNameChange` field is set to `true` in the kustomization YAML file.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_overlays-CO10-2)|The `spec/replicas` field in the patch file updates the number of replicas of the `frontend-prod` deployment.|

### View and Deploy Resources by Using Kustomize

Run the ``kubectl kustomize _`kustomization-directory`_`` command to render the manifests without applying them to the cluster.

[user@host frontend-app]$ **`kubectl kustomize overlay/production`**
_...output omitted..._
kind: Deployment
metadata:
  labels:
    app: frontend
    `env: prod`
  name: `frontend-prod`
_...output omitted..._
spec:
  `replicas: 5`
  selector:
    matchLabels:
      app: frontend
      `env: prod`
_...output omitted..._

The `kubectl apply` command applies configurations to the resources in the cluster. If resources are not available, then the `kubectl apply` command creates resources. The `kubectl apply` command applies a kustomization with the `-k` flag.

[user@host frontend-app]$ **`kubectl apply -k overlay/production`**
deployment.apps/frontend-prod created
_...output omitted..._

### Delete Resources by Using Kustomize

Run the ``oc delete -k _`kustomization-directory`_`` command to delete the resources that were deployed by using Kustomize.

[user@host frontend-app]$ **`oc delete -k overlay/production`**
configmap "database" deleted
secret "database" deleted
service "database" deleted
deployment.apps "database" deleted

### Kustomize Generators

Configuration maps hold non-confidential data by using a key-value pair. Secrets are similar to configuration maps, but secrets hold confidential information such as usernames and passwords. Kustomize has `configMapGenerator` and `secretGenerator` fields that generate configuration map and secret resources.

The configuration map and secret generators can include content from external files in the generated resources. By keeping the content of the generated resources outside the resource definitions, you can use files that other tools generated, or that are stored in different systems. Generators help to manage the content of configuration maps and secrets, by taking care of encoding and including content from other sources.

#### Configuration Map Generator

Kustomize provides a `configMapGenerator` field to create a configuration map. The configuration map that a `configMapGenerator` field creates behaves differently from configuration maps that are created without a Kustomize file. With generated configuration maps, Kustomize appends a hash to the name, and any change in the configuration map triggers a rolling update.

The following example adds a configuration map by using the `configMapGenerator` field in the staging kustomization file. The `hello` application deployment has two environment variables to refer to the `hello-app-configmap` configuration map.

The `kustomization.yaml` file has the following content:

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: hello-stage
resources:
- ../../base
configMapGenerator:
- name: hello-app-configmap
  literals:
    - msg="Welcome!"
    - enable="true"

The `deployment.yaml` file has the following content:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  labels:
    app: hello
    name: hello
spec:
_...output omitted..._
    spec:
      containers:
      - name: hello
        image: quay.io/hello-app:v1.0
        env:
        - name: MY_MESSAGE
          valueFrom:
            configMapKeyRef:
              name: hello-app-configmap
              key: msg
        - name: MSG_ENABLE
          valueFrom:
            configMapKeyRef:
              name: hello-app-configmap
              key: enable

You can view and deploy all resources and customizations that the kustomization YAML file defines, in the development directory.

[user@host hello-app]$ **`kubectl kustomize overlays/staging`**
apiVersion: v1
data:
  enable: "true"
  msg: Welcome!
kind: ConfigMap
metadata:
  name: `hello-app-configmap-9tcmf95d77`
  namespace: hello-stage
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello
    name: hello
  name: hello
  namespace: hello-stage
spec:
_...output omitted..._
    spec:
      containers:
      - env:
        - name: MY_MESSAGE
          valueFrom:
            configMapKeyRef:
              key: msg
              name: `hello-app-configmap-9tcmf95d77`
        - name: MSG_ENABLE
          valueFrom:
            configMapKeyRef:
              key: enable
              name: `hello-app-configmap-9tcmf95d77`
_...output omitted..._

[user@host hello-app]$ **`kubectl apply -k overlays/staging`**
`configmap/hello-app-configmap-9tcmf95d77 created`
deployment.apps/hello created

[user@host hello-app]$ **`oc get all`**
NAME                         READY   STATUS    RESTARTS   AGE
`pod/hello-75dc9cfc87-jh62k`   1/1     Running   0          97s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello   1/1     1            1           97s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-75dc9cfc87   1         1         1       97s

The `kubectl apply -k` command creates a `hello-app-configmap-9tcmf95d77` configuration map and a `hello` deployment. Update the `kustomization.yaml` file with the configuration map values.

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: hello-stage
resources:
- ../../base
configMapGenerator:
- name: hello-app-configmap
  literals:
    - msg="Welcome Back!"
    - enable="true"

Then, apply the overlay with the `kubectl apply` command.

[user@host hello-app]$ **`kubectl apply -k overlays/staging`**
configmap/hello-app-configmap-696dm8h728 created
deployment.apps/hello configured

[user@host hello-app]$ **`oc get all`**
NAME                        READY   STATUS    RESTARTS   AGE
`pod/hello-55bc55ff9-hrszh`   1/1     Running   0          3s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello   1/1     1            1           5m5s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-55bc55ff9    1         1         1       3s
replicaset.apps/hello-75dc9cfc87   0         0         0       5m5s

The `kubectl apply -k` command applies kustomization. Kustomize appends a new hash to the configuration map name, which creates a `hello-app-configmap-696dm8h728` configuration map. The new configuration map triggers the generation of a new `hello-55bc55ff9-hrszh` pod.

You can generate a configuration map by using the `files` key from the `.properties` file or from the `.env` file by using the `envs` key with the file name as the value. You can also create a configuration map from a literal key-value pair by using the `literals` key.

The following example shows a `kustomization.yaml` file with the `configMapGenerator` field.

_...output omitted..._
`configMapGenerator:`
- name: `configmap-1`  ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
  `files:`
    - application.properties
- name: `configmap-2`  ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
  `envs:`
    - configmap-2.env
- name: `configmap-3`  ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
  `literals:`
    - name="configmap-3"
    - description="literal key-value pair"

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_configuration_map_generator-CO11-1)|The `configmap-1` key is using the `application.properties` file.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_configuration_map_generator-CO11-2)|The `configmap-2` key is using the `configmap-2.env` file.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_configuration_map_generator-CO11-3)|The `configmap-3` key is using a literal key-value pair.|

The following example shows the `application.properties` file that is referenced in the `configmap-1` key.

Day=Monday
Enable=True

The following example shows the `configmap-2.env` file that is referenced in the `configmap-2` key.

Greet=Welcome
Enable=True

Run the `kubectl kustomize` command to view details of resources and customizations that the kustomization YAML file defines:

[user@host base]$ **`kubectl kustomize .`**
apiVersion: v1
data:
  application.properties: |
    Day=Monday
    Enable=True
kind: ConfigMap
metadata:
  name: configmap-1-5g2mh569b5 ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
---
apiVersion: v1
data:
  Enable: "True"
  Greet: Welcome
kind: ConfigMap
metadata:
  name: configmap-2-92m84tg9kt ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
---
apiVersion: v1
data:
  description: literal key-value pair
  name: configmap-3
kind: ConfigMap
metadata:
  name: configmap-3-k7g7d5bffd ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
---
_...output omitted..._

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_configuration_map_generator-CO12-1)|The `configMapGenerator` field appends a hash to all ConfigMap resources. The `configmap-1-5g2mh569b5` configuration map is generated from the `application.properties` file, and the data field has a single key with the `application.properties` value.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_configuration_map_generator-CO12-2)|The `configmap-2-92m84tg9kt` configuration map is generated from the `configmap-2.env` file, and the data field has separate keys for each listed variable in the `configmap-2.env` file.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_configuration_map_generator-CO12-3)|The `configmap-3-k7g7d5bffd` configuration map is generated from a literal key-value pair.|

#### Secret Generator

A secret resource has sensitive data such as a username and a password. You can generate the secret by using the `secretGenerator` field. The `secretGenerator` field works similarly to the `configMapGenerator` field. However, the `secretGenerator` field also performs the base64 encoding that secret resources require.

The following example shows a `kustomization.yaml` file with the `secretGenerator` field:

_...output omitted..._
`secretGenerator:`
- name: `secret-1`  ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
  `files:`
    - password.txt
- name: `secret-2`  ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
  `envs:`
    - secret-mysql.env
- name: `secret-3`  ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
  `literals:`
    - MYSQL_DB=mysql
    - MYSQL_PASS=root

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_secret_generator-CO13-1)|The `secret-1` key is using the `password.txt` file.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_secret_generator-CO13-2)|The `secret-2` key is using the `secret-mysql.env` file.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_secret_generator-CO13-3)|The `secret-3` key is using literal key-value pairs.|

#### Generator Options

Kustomize provides a `generatorOptions` field to alter the default behavior of Kustomize generators. The `configMapGenerator` and `secretGenerator` fields append a hash suffix to the name of the generated resources.

Workload resources such as deployments do not detect any content changes to configuration maps and secrets. Any changes to a configuration map or secret do not apply automatically.

Because the generators append a hash, when you update the configuration map or secret, the resource name changes. This change triggers a rollout.

In some cases, the hash is not needed. Some operators observe the contents of the configuration maps and secrets that they use, and apply changes immediately. For example, the OpenShift OAuth operator applies changes to htpasswd secrets automatically. You can disable this feature with the `generatorOptions` field.

You can also add labels and annotations to the generated resources by using the `generatorOptions` field.

The following example shows the use of the `generatorOptions` field.

_...output omitted..._
`configMapGenerator:`
- name: `my-configmap`
  `literals:`
    - name="configmap-3"
    - description="literal key-value pair"
`generatorOptions:`
  `disableNameSuffixHash: true`
  labels:
    `type: generated-disabled-suffix`
  annotations:
    `note: generated-disabled-suffix`

You can use the `kubectl kustomize` command to render the changes to verify their effect.

[user@host base]$ **`kubectl kustomize .`**
apiVersion: v1
data:
  description: literal key-value pair
  name: configmap-3
kind: ConfigMap
metadata:
  `annotations:`
    note: generated-disabled-suffix
  `labels:`
    type: generated-disabled-suffix
  `name: my-configmap`

The `my-configmap` configuration map is without a hash suffix, and has a label and annotations that are defined in the kustomization file.


# Chapter 2.  Deploy Packaged Applications

## OpenShift Templates

### Objectives

- Deploy and update applications from resource manifests that are packaged as OpenShift templates.

### OpenShift Templates

A template is a Kubernetes custom resource that describes a set of Kubernetes resource configurations. Templates can have parameters. You can create a set of related Kubernetes resources from a template by processing the template, and providing values for the parameters. Templates have varied use cases, and can create any Kubernetes resource. You can create a list of resources from a template by using the CLI or, if a template is uploaded to your project or to the global template library, by using the web console.

The template resource is a Kubernetes extension that Red Hat OpenShift provides. The Cluster Samples Operator populates templates (and image streams) in the `openshift` namespace. You can opt out of adding templates during installation, and you can restrict the list of templates that the operator populates.

You can also create templates from scratch, or copy and customize a template to suit the needs of your project.

#### Discovering Templates

The templates that the Cluster Samples Operator provides are in the `openshift` namespace. Use the following `oc get` command to view a list of these templates:

[user@host ~]$ **`oc get templates -n openshift`**
NAME                      DESCRIPTION            PARAMETERS        OBJECTS
cache-service             Red Hat Data Grid...    8 (1 blank)      4
cakephp-mysql-example     An example CakePHP...  21 (4 blank)      8
cakephp-mysql-persistent  An example CakePHP...  22 (4 blank)      9
_...output omitted..._

To evaluate any template, use the ``oc describe template _`template-name`_ -n openshift`` command to view more details about the template, including the description, the labels that the template uses, the template parameters, and the resources that the template generates.

The following example shows the details of the `cache-service` template:

```yaml
[user@host ~]$ **`oc describe template cache-service -n openshift`**
Name: cache-service
Namespace: openshift
Created: 2 months ago
Labels: samples.operator.openshift.io/managed=true
template=cache-service
Description: Red Hat Data Grid is an in-memory, distributed key/value store.  # [1]
Annotations: iconClass=icon-datagrid
_...output omitted..._

Parameters:  # [2]
    Name: APPLICATION_NAME
    Display Name: Application Name
    Description: Specifies a name for the application.
    Required: true
    Value: cache-service  # [3]

 ...output omitted...
    Name: APPLICATION_PASSWORD
    Display Name: Client Password
    Description: Sets a password to authenticate client applications.
    Required: false
    Generated: expression  # [4]
    From: [a-zA-Z0-9]{16}

Object Labels: template=cache-service  # [5]

Message: <none>

Objects:  # [6]
    Secret ${APPLICATION_NAME}
    Service ${APPLICATION_NAME}-ping
    Service ${APPLICATION_NAME}
    StatefulSet.apps ${APPLICATION_NAME}
```

|---|---|
| 1 | Use the description to determine the purpose of the template.|
| 2 | The parameters provide deployment flexibility.|
| 3 |The `value` field provides a default value that you can override.|
| 4 | The `Generated` and `From` fields also generate default values.|
| 5 |The object labels are applied to all resources that the template creates.|
| 6 |The objects section lists the resources that the template creates.|

In addition to using the `oc describe` command to view information about a template, the `oc process` command provides a `--parameters` option to view only the parameters that a template uses. For example, use the following command to view the parameters that the `cache-service` template uses:

```yaml
[user@host ~]$ **`oc process --parameters cache-service -n openshift`**
NAME                   ...   GENERATOR    VALUE
APPLICATION_NAME       ...                cache-service
IMAGE                  ...                registry.redhat.io/jboss-datagrid-7/...
NUMBER_OF_INSTANCES    ...                1
REPLICATION_FACTOR     ...                1
EVICTION_POLICY        ...                evict
TOTAL_CONTAINER_MEM    ...                512
APPLICATION_USER       ...
APPLICATION_PASSWORD   ...   expression   [a-zA-Z0-9]{16}
```

Use the `-f` option to view the parameters of a template that are defined in a file:

[user@host ~]$ **`oc process --parameters -f  my-cache-service.yaml`**

Use the ``oc get template _`template-name`_ -o yaml -n _`namespace`_`` command to view the manifest for the template. The following example retrieves the template manifest for the `cache-service` template:

```yaml
[user@host ~]$ **`oc get template cache-service -o yaml -n openshift`**
apiVersion: template.openshift.io/v1
kind: `Template`
labels:
  template: cache-service
metadata:
 _...output omitted..._
- apiVersion: v1
  kind: `Secret`
  metadata:
 _...output omitted..._
- apiVersion: v1
  kind: `Service`
  metadata:
 _...output omitted..._
- apiVersion: v1
  kind: `Service`
  metadata:
 _...output omitted..._
- apiVersion: apps/v1
  kind: `StatefulSet`
  metadata:
 _...output omitted..._
`parameters`:
- description: Specifies a name for the application.
  displayName: Application Name
  name: APPLICATION_NAME
  required: true
  value: cache-service
- description: Sets an image to bootstrap the service.
  name: IMAGE
 _...output omitted..._
```

In the template manifest, examine how the template creates resources. The manifest is also a good resource for learning how to create your own templates.

#### Using Templates

The `oc new-app` command has a `--template` option that can deploy the template resources directly from the `openshift` project. The following example deploys the resources that are defined in the `cache-service` template from the `openshift` project:

[user@host ~]$ **`oc new-app --template=cache-service -p APPLICATION_USER=my-user`**

Using the `oc new-app` command to deploy the template resources is convenient for development and testing. However, for production usage, consume templates in a manner that helps resource and configuration tracking. For example, the `oc new-app` command can only create new resources, not update existing resources.

You can use the `oc process` command to apply parameters to a template, to produce manifests to deploy the templates with a set of parameters. The `oc process` command can process both templates that are stored in files locally, and templates that are stored in the cluster. However, to process templates in a namespace, you must have write permissions on the template namespace. For example, to run `oc process` on the templates in the `openshift` namespace, you must have write permissions on this namespace.

>Note
Unprivileged users can read the templates in the `openshift` namespace by default. Those users can extract the template from the `openshift` namespace and create a copy in a project where they have wider permissions. By copying a template to a project, they can use the `oc process` command on the template.

#### Deploying Applications from Templates

The `oc process` command uses parameter values to transform a template into a set of related Kubernetes resource manifests. For example, the following command creates a set of resource manifests for the `my-cache-service` template. When you use the `-o yaml` option, the resulting manifests are in the YAML format. The example writes the manifests to a `my-cache-service-manifest.yaml` file:

[user@host ~]$ **`oc process my-cache-service \   -p APPLICATION_USER=user1 -o yaml > my-cache-service-manifest.yaml`**

The previous example uses the `-p` option to provide a parameter value to the only required parameter without a default value.

Use the `-f` option with the `oc process` command to process a template that is defined in a file:

[user@host ~]$ **`oc process -f my-cache-service.yaml \   -p APPLICATION_USER=user1 -o yaml > my-cache-service-manifest.yaml`**

Use the `-p` option with ``_`key`_=_`value`_`` pairs with the `oc process` command to use parameter values that override the default values. The following example passes three parameter values to the `my-cache-service` template, and overrides the default values of the specified parameters:

[user@host ~]$ **`oc process my-cache-service -o yaml \   -p TOTAL_CONTAINER_MEM=1024 \   -p APPLICATION_USER='cache-user' \   -p APPLICATION_PASSWORD='my-secret-password' \   > my-cache-service-manifest.yaml`**

Instead of specifying parameters on the command line, place the parameters in a file. This option cleans up the command line when many parameter values are required. Save the parameters file in a version control system to keep records of the parameters that are used in production deployments.

For example, instead of using the command-line options in the previous examples, place the key-value pairs in a `my-cache-service-params.env` file. Add the key-value pairs to the file, with each pair on a separate line:

TOTAL_CONTAINER_MEM=1024
APPLICATION_USER='cache-user'
APPLICATION_PASSWORD='my-secret-password'

The corresponding `oc process` command uses the `--param-file` option to pass the parameters as follows:

[user@host ~]$ **`oc process my-cache-service -o yaml \   --param-file=my-cache-service-params.env > my-cache-service-manifest.yaml`**

Generating a manifest file is not required to use templates. Instead, pipe the output of the `oc process` command directly to the input for the `oc apply -f -` command. The `oc apply` command creates live resources on the Kubernetes cluster.

[user@host ~]$ **`oc process my-cache-service \   --param-file=my-cache-service-params.env | oc apply -f -`**

Because templates are flexible, you can use the same template to create different resources by changing the input parameters.

#### Updating Apps from Templates

Because you use the `oc apply` command, after deploying a set of manifests from a template, you can process the template again and use `oc apply` for updates. This procedure can make simple changes to deployed templates, such as changing a parameter. However, many workload updates are not possible with this mechanism. To manage more complex applications, consider using other mechanisms such as Helm charts, which are described elsewhere in this course.

To compare the results of applying a different parameters file to a template against the live resources, pipe the manifest to the `oc diff -f -` command. For example, given a second parameter file named `my-cache-service-params-2.env`, use the following command:

```yaml
[user@host ~]$ **`oc process my-cache-service -o yaml \   --param-file=my-cache-service-params-2.env | oc diff -f -`**
 _...output omitted..._
-  generation: 1
+  generation: 2
   labels:
     application: cache-service
     template: cache-service
@@ -86,10 +86,10 @@
           timeoutSeconds: 10
         resources:
           limits:
-            memory: 1Gi
+            memory: 2Gi
           requests:
             cpu: 500m
-            memory: 1Gi
+            memory: 2Gi
         terminationMessagePath: /dev/termination-log
         terminationMessagePolicy: File
         volumeMounts:
```
In this case, the configuration change increases the memory usage of the application. The output shows that the second generation uses `2Gi` of memory instead of `1Gi`.

After verifying that the changes are what you intend, you can pipe the output of the `oc process` to the `oc apply -f -` command.

#### Managing Templates

For production usage, make a customized copy of the template, to change the default values of the template to suitable values for the target project. To copy a template into your project, use the `oc get template` command with the `-o yaml` option to copy the template YAML to a file.

The following example copies the `cache-service` template from the `openshift` project to a YAML file named `my-cache-service.yaml`:

[user@host ~]$ **`oc get template cache-service -o yaml \   -n openshift > my-cache-service.yaml`**

After creating a YAML file for a template, consider making the following changes to the template:

- Give the template a new name that is specific to the target use of the template resources.
    
- Apply appropriate changes to the parameter default values at the end of the file.
    
- Remove the `namespace` field of the `template` resource.
    

You can process templates in other namespaces, if you can create the processed template resource in those namespaces. Processing the template in a different project without changing the template namespace to match the target namespace gives an error. Optionally, you can also delete the `namespace` field from the `metadata` field of the `template` resource.

After you have a YAML file for a template, use the `oc create -f` command to upload the template to the current project. In this case, the `oc create` command is not creating the resources that the template defines. Instead, the command is creating a `template` resource in the project. Using a template that is uploaded to a project clarifies which template provides the resource definitions of a project. After uploading, the template is available to anyone with access to the project.

The following example uploads a customized template that is defined in the `my-cache-service.yaml` file to the current project:

[user@host ~]$ **`oc create -f my-cache-service.yaml`**

Use the ``-n _`namespace`_`` option to upload the template to a different project. The following example uploads the template that is defined in the `my-cache-service.yaml` file to the `shared-templates` project:

[user@host ~]$ **`oc create -f my-cache-service.yaml -n shared-templates`**

Use the `oc get templates` command to view a list of available templates in the project:

[user@host ~]$ **`oc get templates -n shared-templates`**
NAME                      DESCRIPTION            PARAMETERS        OBJECTS
my-cache-service          Red Hat Data Grid...    8 (1 blank)      4


## Helm Charts

### Objectives

- Deploy and update applications from resource manifests that are packaged as Helm charts.
    

### Helm

Helm is an open source application that helps to manage the lifecycle of Kubernetes applications.

Helm introduces the concept of _charts_. A chart is a package that describes a set of Kubernetes resources that you can deploy. Helm charts define values that you can customize when deploying an application. Helm includes functions to distribute charts and updates.

Many organizations distribute Helm charts to deploy applications. Often, Helm is the supported mechanism to deploy a specific application.

However, Helm does not cover all needs to manage certain kinds of applications. Operators have a more complete model that can handle the lifecycle of more complex applications. For more details about operators, refer to [the section called “ Kubernetes Operators and the Operator Lifecycle Manager ”](https://rol.redhat.com/rol/app/courses/do280-4.14/pages/ch07).

### Helm Charts

A Helm chart defines Kubernetes resources that you can deploy. A chart is a collection of files with a defined structure. These files include chart metadata (such as the chart name or version), resource definitions, and supporting material.

Chart authors can use the template feature of the Go language for the resource definitions. For example, instead of specifying the image for a deployment, charts can use user-provided values for the image. By using values to choose an image, cluster administrators can replace a default public image with an image from a private repository.

The following diagram shows the structure of a minimal Helm chart:

sample/
├── Chart.yaml ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
├── templates ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
|   |── example.yaml
└── values.yaml ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_helm_charts-CO19-1)|The `Chart.yaml` file contains chart metadata, such as the name and version of the chart.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_helm_charts-CO19-2)|The `templates` directory contains files that define application resources such as deployments.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_helm_charts-CO19-3)|The `values.yaml` file contains default values for the chart.|

Helm charts can contain hooks that Helm executes at different points during installations and upgrades. Hooks can automate tasks for installations and upgrades. With hooks, Helm charts can manage more complex applications than purely manifest-based processes. Review the chart documentation to learn about the chart hooks and their implications.

### Using Helm Charts

Helm is a command-line application. The `helm` command interacts with the following entities:

Charts

Charts are the packaged applications that the `helm` command deploys.

Releases

A _release_ is the result of deploying a chart. You can deploy a chart many times to the same cluster. Each deployment is a different release.

Versions

A Helm chart can have many versions. Chart authors can release updates to charts, to adapt to later application versions, introduce new features, or fix issues.

You can use and refer to charts in various ways. For example, if your local file system contains a chart, then you can refer to that chart by using the path to the chart directory. You can also use a path or a URL that contains a chart that is packaged in a tar archive with `gzip` compression.

#### Inspecting Helm Charts

Use the `helm show` command to display information about a chart. The `show chart` subcommand displays general information, such as the maintainers, or the source URL.

[user@host ~]$ **``helm show chart _`chart-reference`_``**
apiVersion: v1
description: A Helm chart for Kubernetes
name: examplechart
version: 0.1.0
maintainers:
- email: dev@example.com
  name: Developer
sources:
- https://git.example.com/examplechart

The `show values` subcommand displays the default values for the chart. The output is in YAML format and comes from the `values.yaml` file in the chart.

[user@host ~]$ **``helm show values _`chart-reference`_``**
image:
  repository: "sample"
  tag: "1.8.10"
  pullPolicy: IfNotPresent
_...output omitted..._

Chart resources use the values from the `values.yaml` file by default. You can override these default values. You can use the output of the `show values` command to discover customizable values.

#### Installing Helm Charts

After inspecting the chart, you can deploy the resources in the chart by using the `helm install` command. In Helm, _install_ refers to deploying the resources in a chart to create a release.

Always refer to the documentation of the chart before installation to learn about prerequisites, extra installation steps, and other information.

To install a chart, you must decide on the following parameters:

- The deployment target namespace
    
- The values to override
    
- The release name
    

Helm charts can contain Kubernetes resources of any kind. These resources can be namespaced or non-namespaced. Like normal resource definitions, namespaced resources in charts can define or omit a namespace declaration.

Most Helm charts that deploy applications do not create a namespace, and namespaced resources in the chart omit a namespace declaration. Typically, when deploying a chart that follows this structure, you create a namespace for the deployment, and Helm creates namespaced resources in this namespace.

After deciding the target namespace, you can design the values to use. Inspect the documentation and the output of the `helm show values` command to decide which values to override.

You can define values by writing a YAML file that contains them. This file can follow the structure from the output of the `helm show values` command, which contains the default values. Specify only the values to override.

Consider the following output from the `helm show values` command for an example chart:

image:
  repository: "sample"
  tag: "1.8.10"
  pullPolicy: IfNotPresent

Create a `values.yaml` file without the `image` key if you do not want to override any image parameters. Omit the `pullPolicy` key to override the `tag` key but not the pull policy. For example, the following YAML file would override only the image tag:

image:
  tag: "1.8.10-patched"

Besides the YAML file, you can override specific values by using command-line arguments.

The final element to prepare a chart deployment is choosing a release name. You can deploy a chart many times to a cluster. Each chart deployment must have a unique release name for identification purposes. Many Helm charts use the release name to construct the name of the created resources.

With the namespace, values, and release name, you can start the deployment process. The `helm install` command creates a release in a namespace, with a set of values.

#### Rendering Manifests from a Chart

You can use the `--dry-run` option to preview the effects of installing a chart.

[user@host ~]$ **``helm install _`release-name`_ _`chart-reference`_  --dry-run \   --values values.yaml``**
NAME: release-name ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
LAST DEPLOYED: Tue May 30 13:14:57 2023
NAMESPACE: current-namespace
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: chart/templates/serviceaccount.yaml ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-release-sa
  labels:
_...output omitted..._

NOTES: ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
The application can be accessed via port 1234.
_...output omitted..._

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_rendering_manifests_from_a_chart-CO20-1)|General information about the new release|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_rendering_manifests_from_a_chart-CO20-2)|A list of the resources that the `helm install` command would create|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_rendering_manifests_from_a_chart-CO20-3)|Additional information|

### Note

You define values to use for the installation with the `--values values.yaml` option. In this file, you override the default values from the chart that are defined in the `values.yaml` file that the chart contains.

Often, chart resource names include the release name. In the example output of the `helm install` command, the service account is a combination of the release name and the `-sa` text.

Chart authors can provide installation notes that use the chart values. In the same example, the port number in the notes reflects a value from the `values.yaml` file.

If the preview looks correct, then you can run the same command without the `--dry-run` option to deploy the resources and create the release.

### Releases

When the `helm install` command runs successfully, besides creating the resources, Helm creates a release. Helm stores information about the release as a secret of the `helm.sh/release.v1` type.

#### Inspecting Releases

Use the `helm list` command to inspect releases on a cluster.

[user@host ~]$ **`helm list`**
NAME         NAMESPACE   REVISION  ...  STATUS     CHART            APP VERSION
my-release   example     1         ...  deployed   example-4.12.1   1.8.10

Similarly to `kubectl` commands, many `helm` commands have the `--all-namespaces` and `--namespace` options. The `helm list` command without options lists releases in the current namespace. If you use the `--all-namespaces` option, then it lists releases in all namespaces. If you use the `--namespace` option, then it lists releases in a single namespace.

### Warning

Do not manipulate the release secret. If you remove the secret, then Helm cannot operate with the release.

#### Upgrading Releases

The `helm upgrade` command can apply changes to existing releases, such as updating values or the chart version.

### Important

By default, this command automatically updates releases to use the latest version of the chart.

The `helm upgrade` command uses similar arguments and options to the `helm install` command. However, the `helm upgrade` command interacts with existing resources in the cluster instead of creating resources from a blank state. Therefore, the `helm upgrade` command can have more complex effects, such as conflicting changes. Always review the chart documentation when using a later version of a chart, and when changing values. You can use the `--dry-run` option to preview the manifests that the `helm upgrade` command uses, and compare them to the running resources.

#### Rolling Back Helm Upgrades

Helm keeps a log of release upgrades, to review changes and roll back to previous releases.

You can review this log by using the `helm history` command:

[user@host ~]$ **``helm history _`release_name`_``**
REVISION  UPDATED        STATUS      CHART        APP VERSION  DESCRIPTION
1         Wed May 31...  superseded  chart-0.0.6  latest       Install complete
2         Wed May 31...  deployed    chart-0.0.7  latest       Upgrade complete

You can use the `helm rollback` command to revert to an earlier revision:

[user@host ~]$ **``helm rollback _`release_name`_ _`revision`_``**
Rollback was a success! Happy Helming!

Rolling back can have greater implications than upgrading, because upgrades might not be reversible. If you keep a test environment with the same upgrades as a production environment, then you can test rollbacks before performing them in the production environment to find potential issues.

### Helm Repositories

Charts can be distributed as files, archives, or container images, or by using chart repositories.

The `helm repo` command provides the following subcommands to work with chart repositories.

|Subcommand|Description|
|:--|:--|
|``add _`NAME`_ _`REPOSITORY_URL`_``|Add a Helm chart repository.|
|`list`|List Helm chart repositories.|
|`update`|Update Helm chart repositories.|
|``remove _`REPOSITORY1_NAME`_ _`REPOSITORY2_NAME`_ …​``|Remove Helm chart repositories.|

The following command adds a repository:

[user@host ~]$ **`helm repo add \   openshift-helm-charts https://charts.openshift.io/`**
"openshift-helm-charts" has been added to your repositories

This command and other repository commands change only local configuration, and do not affect any cluster resources. The `helm repo add` command updates the `~/.config/helm/repositories.yaml` configuration file, which keeps the list of configured repositories.

When repositories are configured, other commands can use the list of repositories to perform actions. For example, the `helm search repo` command lists all available charts in the configured repositories:

[user@host ~]$ **`helm search repo`**
NAME          CHART VERSION    APP VERSION    DESCRIPTION
repo/chart    0.0.7            latest         A sample chart
_...output omitted..._

By default, the `helm search repo` command shows only the latest version of a chart. Use the `--versions` option to list all available versions. By default, the `install` and `upgrade` commands use the latest version of the chart in the repository. You can use the `--version` option to install specific versions.

