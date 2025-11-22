
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

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev-env
resources:
- ../../base
```

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

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: test-env
patches:   # [1]
- patch: |-
    - op: replace  # [2]
      path: /metadata/name
      value: frontend-test
  target:  # [3]
    kind: Deployment
    name: frontend
- patch: |-  # [4]
    - op: replace
      path: /spec/replicas
      value: 15
  target:
    kind: Deployment
    name: frontend
resources:  # [5]
- ../../base
commonLabels:  # [6]
  env: test
```

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
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod-env
patches:  # [1]
- path: patch.yaml  # [2]
  target: # [3]
    kind: Deployment
    name: frontend
  options:
    allowNameChange: true # [4]
resources:  # [5]
- ../../base
commonLabels:  # [6]
  env: prod
```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_overlays-CO9-1)|The `patches` field lists the patches that are applied by using a production kustomization file.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_overlays-CO9-2)|The `path` field specifies the name of the patching YAML file.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_overlays-CO9-3)|The `target` field specifies the kind and name of the resource to apply the patch. In this example, you are targeting the `frontend` deployment.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_overlays-CO9-4)|The `allowNameChange` field enables kustomization to update the name by using a patch YAML file.|
|[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_overlays-CO9-5)|The `frontend-app/overlay/production/kustomization.yaml` file uses the base kustomization file at `../../base` to create an application.|
|[![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)](https://rol.redhat.com/rol/app/#_overlays-CO9-6)|The `commonLabels` field adds an `env: prod` label to all resources.|

The `patch.yaml` file has the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-prod ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
spec:
  replicas: 5 ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_overlays-CO10-1)|The `metadata.name` field in the patch file updates the `frontend` deployment name to `frontend-prod` if the `allowNameChange` field is set to `true` in the kustomization YAML file.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_overlays-CO10-2)|The `spec/replicas` field in the patch file updates the number of replicas of the `frontend-prod` deployment.|

### View and Deploy Resources by Using Kustomize

Run the ``kubectl kustomize _`kustomization-directory`_`` command to render the manifests without applying them to the cluster.

```sh
[user@host frontend-app]$ kubectl kustomize overlay/production
...output omitted...
kind: Deployment
metadata:
  labels:
    app: frontend
    `env: prod`
  name: `frontend-prod`
...output omitted...
spec:
  `replicas: 5`
  selector:
    matchLabels:
      app: frontend
      `env: prod`
...output omitted...
```

The `kubectl apply` command applies configurations to the resources in the cluster. If resources are not available, then the `kubectl apply` command creates resources. The `kubectl apply` command applies a kustomization with the `-k` flag.

```sh
[user@host frontend-app]$ kubectl apply -k overlay/production
deployment.apps/frontend-prod created
...output omitted...
```

### Delete Resources by Using Kustomize

Run the ``oc delete -k _`kustomization-directory`_`` command to delete the resources that were deployed by using Kustomize.

```yaml
[user@host frontend-app]$ oc delete -k overlay/production
configmap "database" deleted
secret "database" deleted
service "database" deleted
deployment.apps "database" deleted
```

### Kustomize Generators

Configuration maps hold non-confidential data by using a key-value pair. Secrets are similar to configuration maps, but secrets hold confidential information such as usernames and passwords. Kustomize has `configMapGenerator` and `secretGenerator` fields that generate configuration map and secret resources.

The configuration map and secret generators can include content from external files in the generated resources. By keeping the content of the generated resources outside the resource definitions, you can use files that other tools generated, or that are stored in different systems. Generators help to manage the content of configuration maps and secrets, by taking care of encoding and including content from other sources.

#### Configuration Map Generator

Kustomize provides a `configMapGenerator` field to create a configuration map. The configuration map that a `configMapGenerator` field creates behaves differently from configuration maps that are created without a Kustomize file. With generated configuration maps, Kustomize appends a hash to the name, and any change in the configuration map triggers a rolling update.

The following example adds a configuration map by using the `configMapGenerator` field in the staging kustomization file. The `hello` application deployment has two environment variables to refer to the `hello-app-configmap` configuration map.

The `kustomization.yaml` file has the following content:

```yaml
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
```

The `deployment.yaml` file has the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  labels:
    app: hello
    name: hello
spec:
...output omitted...
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
```

You can view and deploy all resources and customizations that the kustomization YAML file defines, in the development directory.

```sh
[user@host hello-app]$ kubectl kustomize overlays/staging
apiVersion: v1
data:
  enable: "true"
  msg: Welcome!
kind: ConfigMap
metadata:
  name: hello-app-configmap-9tcmf95d77
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
...output omitted...
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
...output omitted...

[user@host hello-app]$ kubectl apply -k overlays/staging
`configmap/hello-app-configmap-9tcmf95d77 created`
deployment.apps/hello created

[user@host hello-app]$ oc get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/hello-75dc9cfc87-jh62k   1/1     Running   0          97s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello   1/1     1            1           97s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-75dc9cfc87   1         1         1       97s
```
The `kubectl apply -k` command creates a `hello-app-configmap-9tcmf95d77` configuration map and a `hello` deployment. Update the `kustomization.yaml` file with the configuration map values.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: hello-stage
resources:
- ../../base
configMapGenerator:
- name: hello-app-configmap
  literals:
    - msg="Welcome Back!"
    - enable="true"`
```

Then, apply the overlay with the `kubectl apply` command.

```sh
[user@host hello-app]$ kubectl apply -k overlays/staging
configmap/hello-app-configmap-696dm8h728 created
deployment.apps/hello configured

[user@host hello-app]$ oc get all
NAME                        READY   STATUS    RESTARTS   AGE
`pod/hello-55bc55ff9-hrszh`   1/1     Running   0          3s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello   1/1     1            1           5m5s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-55bc55ff9    1         1         1       3s
replicaset.apps/hello-75dc9cfc87   0         0         0       5m5s
```

The `kubectl apply -k` command applies kustomization. Kustomize appends a new hash to the configuration map name, which creates a `hello-app-configmap-696dm8h728` configuration map. The new configuration map triggers the generation of a new `hello-55bc55ff9-hrszh` pod.

You can generate a configuration map by using the `files` key from the `.properties` file or from the `.env` file by using the `envs` key with the file name as the value. You can also create a configuration map from a literal key-value pair by using the `literals` key.

The following example shows a `kustomization.yaml` file with the `configMapGenerator` field.

```yaml
...output omitted...
configMapGenerator:
- name: configmap-1  # [1]
  files:
    - application.properties
- name: configmap-2  #[2]
  envs:
    - configmap-2.env
- name: configmap-3  # [3]
  literals:
    - name="configmap-3"
    - description="literal key-value pair"
```

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

```sh
[user@host base]$ kubectl kustomize .
apiVersion: v1
data:
  application.properties: |
    Day=Monday
    Enable=True
kind: ConfigMap
metadata:
  name: configmap-1-5g2mh569b5 # [1]
---
apiVersion: v1
data:
  Enable: "True"
  Greet: Welcome
kind: ConfigMap
metadata:
  name: configmap-2-92m84tg9kt  # [2]
---
apiVersion: v1
data:
  description: literal key-value pair
  name: configmap-3
kind: ConfigMap
metadata:
  name: configmap-3-k7g7d5bffd  # [3]
---
_...output omitted..._
```

|                                                                                                                                                |                                                                                                                                                                                                                                                            |
| ---------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_configuration_map_generator-CO12-1) | The `configMapGenerator` field appends a hash to all ConfigMap resources. The `configmap-1-5g2mh569b5` configuration map is generated from the `application.properties` file, and the data field has a single key with the `application.properties` value. |
| [![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_configuration_map_generator-CO12-2) | The `configmap-2-92m84tg9kt` configuration map is generated from the `configmap-2.env` file, and the data field has separate keys for each listed variable in the `configmap-2.env` file.                                                                  |
| [![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_configuration_map_generator-CO12-3) | The `configmap-3-k7g7d5bffd` configuration map is generated from a literal key-value pair.                                                                                                                                                                 |

#### Secret Generator

A secret resource has sensitive data such as a username and a password. You can generate the secret by using the `secretGenerator` field. The `secretGenerator` field works similarly to the `configMapGenerator` field. However, the `secretGenerator` field also performs the base64 encoding that secret resources require.

The following example shows a `kustomization.yaml` file with the `secretGenerator` field:

```yaml
...output omitted..._
secretGenerator:
- name: secret-1  # [1]
  files:
    - password.txt
- name: secret-2   # [2]
  envs:
    - secret-mysql.env
- name: secret-3  # [3]
  literals:
    - MYSQL_DB=mysql
    - MYSQL_PASS=root
```

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

```yaml
...output omitted...
configMapGenerator:
- name: my-configmap
  literals:
    - name="configmap-3"
    - description="literal key-value pair"
generatorOptions:
  disableNameSuffixHash: true
  labels:
    type: generated-disabled-suffix
  annotations:
    note: generated-disabled-suffix
```

You can use the `kubectl kustomize` command to render the changes to verify their effect.

```sh
[user@host base]$ kubectl kustomize .
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
```
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

```sh
[user@host ~]$ oc get templates -n openshift
NAME                      DESCRIPTION            PARAMETERS        OBJECTS
cache-service             Red Hat Data Grid...    8 (1 blank)      4
cakephp-mysql-example     An example CakePHP...  21 (4 blank)      8
cakephp-mysql-persistent  An example CakePHP...  22 (4 blank)      9
...output omitted...
```

To evaluate any template, use the ``oc describe template _`template-name`_ -n openshift`` command to view more details about the template, including the description, the labels that the template uses, the template parameters, and the resources that the template generates.

The following example shows the details of the `cache-service` template:

```sh
[user@host ~]$ oc describe template cache-service -n openshift
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

```sh
[user@host ~]$ oc process --parameters cache-service -n openshift
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

```sh
[user@host ~]$ oc get template cache-service -o yaml -n openshift
apiVersion: template.openshift.io/v1
kind: `Template`
labels:
  template: cache-service
metadata:
 ...output omitted...
- apiVersion: v1
  kind: `Secret`
  metadata:
 ...output omitted...
- apiVersion: v1
  kind: `Service`
  metadata:
 ...output omitted...
- apiVersion: v1
  kind: `Service`
  metadata:
 ...output omitted...
- apiVersion: apps/v1
  kind: `StatefulSet`
  metadata:
 ...output omitted...
`parameters`:
- description: Specifies a name for the application.
  displayName: Application Name
  name: APPLICATION_NAME
  required: true
  value: cache-service
- description: Sets an image to bootstrap the service.
  name: IMAGE
 ...output omitted...
```

In the template manifest, examine how the template creates resources. The manifest is also a good resource for learning how to create your own templates.

#### Using Templates

The `oc new-app` command has a `--template` option that can deploy the template resources directly from the `openshift` project. The following example deploys the resources that are defined in the `cache-service` template from the `openshift` project:

```sh
[user@host ~]$ oc new-app --template=cache-service -p APPLICATION_USER=my-user
```

Using the `oc new-app` command to deploy the template resources is convenient for development and testing. However, for production usage, consume templates in a manner that helps resource and configuration tracking. For example, the `oc new-app` command can only create new resources, not update existing resources.

You can use the `oc process` command to apply parameters to a template, to produce manifests to deploy the templates with a set of parameters. The `oc process` command can process both templates that are stored in files locally, and templates that are stored in the cluster. However, to process templates in a namespace, you must have write permissions on the template namespace. For example, to run `oc process` on the templates in the `openshift` namespace, you must have write permissions on this namespace.

>Note
Unprivileged users can read the templates in the `openshift` namespace by default. Those users can extract the template from the `openshift` namespace and create a copy in a project where they have wider permissions. By copying a template to a project, they can use the `oc process` command on the template.

#### Deploying Applications from Templates

The `oc process` command uses parameter values to transform a template into a set of related Kubernetes resource manifests. For example, the following command creates a set of resource manifests for the `my-cache-service` template. When you use the `-o yaml` option, the resulting manifests are in the YAML format. The example writes the manifests to a `my-cache-service-manifest.yaml` file:
```sh
[user@host ~]$ oc process my-cache-service \   -p APPLICATION_USER=user1 -o yaml > my-cache-service-manifest.yaml
```
The previous example uses the `-p` option to provide a parameter value to the only required parameter without a default value.

Use the `-f` option with the `oc process` command to process a template that is defined in a file:

```sh
[user@host ~]$ oc process -f my-cache-service.yaml \   -p APPLICATION_USER=user1 -o yaml > my-cache-service-manifest.yaml
```

Use the `-p` option with ``_`key`_=_`value`_`` pairs with the `oc process` command to use parameter values that override the default values. The following example passes three parameter values to the `my-cache-service` template, and overrides the default values of the specified parameters:

```sh
[user@host ~]$ oc process my-cache-service -o yaml \   -p TOTAL_CONTAINER_MEM=1024 \   -p APPLICATION_USER='cache-user' \   -p APPLICATION_PASSWORD='my-secret-password' \   > my-cache-service-manifest.yaml
```

Instead of specifying parameters on the command line, place the parameters in a file. This option cleans up the command line when many parameter values are required. Save the parameters file in a version control system to keep records of the parameters that are used in production deployments.

For example, instead of using the command-line options in the previous examples, place the key-value pairs in a `my-cache-service-params.env` file. Add the key-value pairs to the file, with each pair on a separate line:

TOTAL_CONTAINER_MEM=1024
APPLICATION_USER='cache-user'
APPLICATION_PASSWORD='my-secret-password'

The corresponding `oc process` command uses the `--param-file` option to pass the parameters as follows:

```sh
[user@host ~]$ oc process my-cache-service -o yaml \   --param-file=my-cache-service-params.env > my-cache-service-manifest.yaml
```

Generating a manifest file is not required to use templates. Instead, pipe the output of the `oc process` command directly to the input for the `oc apply -f -` command. The `oc apply` command creates live resources on the Kubernetes cluster.

```sh
[user@host ~]$ oc process my-cache-service \   --param-file=my-cache-service-params.env | oc apply -f -
```

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

```sh
[user@host ~]$ oc create -f my-cache-service.yaml
```

Use the ``-n _`namespace`_`` option to upload the template to a different project. The following example uploads the template that is defined in the `my-cache-service.yaml` file to the `shared-templates` project:

```sh
[user@host ~]$ oc create -f my-cache-service.yaml -n shared-templates
```

Use the `oc get templates` command to view a list of available templates in the project:

```sh
[user@host ~]$ oc get templates -n shared-templates
NAME                      DESCRIPTION            PARAMETERS        OBJECTS
my-cache-service          Red Hat Data Grid...    8 (1 blank)      4
```

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

|     |                                                                                                 |
| --- | ----------------------------------------------------------------------------------------------- |
| [1] | The `Chart.yaml` file contains chart metadata, such as the name and version of the chart.       |
| [2] | The `templates` directory contains files that define application resources such as deployments. |
| [3] | The `values.yaml` file contains default values for the chart.                                   |

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

```sh
[user@host ~]$ helm show chart chart-reference
apiVersion: v1
description: A Helm chart for Kubernetes
name: examplechart
version: 0.1.0
maintainers:
- email: dev@example.com
  name: Developer
sources:
- https://git.example.com/examplechart
```

The `show values` subcommand displays the default values for the chart. The output is in YAML format and comes from the `values.yaml` file in the chart.

```sh
[user@host ~]$ helm show values chart-reference
image:
  repository: "sample"
  tag: "1.8.10"
  pullPolicy: IfNotPresent
...output omitted...
```

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

```
image:
  repository: "sample"
  tag: "1.8.10"
  pullPolicy: IfNotPresent
```

Create a `values.yaml` file without the `image` key if you do not want to override any image parameters. Omit the `pullPolicy` key to override the `tag` key but not the pull policy. For example, the following YAML file would override only the image tag:

```
image:
  tag: "1.8.10-patched"
```

Besides the YAML file, you can override specific values by using command-line arguments.

The final element to prepare a chart deployment is choosing a release name. You can deploy a chart many times to a cluster. Each chart deployment must have a unique release name for identification purposes. Many Helm charts use the release name to construct the name of the created resources.

With the namespace, values, and release name, you can start the deployment process. The `helm install` command creates a release in a namespace, with a set of values.

#### Rendering Manifests from a Chart

You can use the `--dry-run` option to preview the effects of installing a chart.

```sh
[user@host ~]$ helm install release-name chart-reference  --dry-run \   --values values.yaml
NAME: release-name  # [1]
LAST DEPLOYED: Tue May 30 13:14:57 2023
NAMESPACE: current-namespace
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: chart/templates/serviceaccount.yaml # [2]
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-release-sa
  labels:
...output omitted...

NOTES: ![3]
The application can be accessed via port 1234.
...output omitted...
```

|     |                                                                      |
| --- | -------------------------------------------------------------------- |
| [1] | General information about the new release                            |
| [2] | A list of the resources that the `helm install` command would create |
| [3] | Additional information                                               |

>Note
You define values to use for the installation with the `--values values.yaml` option. In this file, you override the default values from the chart that are defined in the `values.yaml` file that the chart contains.

Often, chart resource names include the release name. In the example output of the `helm install` command, the service account is a combination of the release name and the `-sa` text.

Chart authors can provide installation notes that use the chart values. In the same example, the port number in the notes reflects a value from the `values.yaml` file.

If the preview looks correct, then you can run the same command without the `--dry-run` option to deploy the resources and create the release.

### Releases

When the `helm install` command runs successfully, besides creating the resources, Helm creates a release. Helm stores information about the release as a secret of the `helm.sh/release.v1` type.

#### Inspecting Releases

Use the `helm list` command to inspect releases on a cluster.

```sh
[user@host ~]$ helm list
NAME         NAMESPACE   REVISION  ...  STATUS     CHART            APP VERSION
my-release   example     1         ...  deployed   example-4.12.1   1.8.10
```

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

```
[user@host ~]$ helm history release_name
REVISION  UPDATED        STATUS      CHART        APP VERSION  DESCRIPTION
1         Wed May 31...  superseded  chart-0.0.6  latest       Install complete
2         Wed May 31...  deployed    chart-0.0.7  latest       Upgrade complete
```

You can use the `helm rollback` command to revert to an earlier revision:

```sh
[user@host ~]$ helm rollback release_name revision
Rollback was a success! Happy Helming!
```

Rolling back can have greater implications than upgrading, because upgrades might not be reversible. If you keep a test environment with the same upgrades as a production environment, then you can test rollbacks before performing them in the production environment to find potential issues.

### Helm Repositories

Charts can be distributed as files, archives, or container images, or by using chart repositories.

The `helm repo` command provides the following subcommands to work with chart repositories.

| Subcommand                                              | Description                     |
| :------------------------------------------------------ | :------------------------------ |
| ``add _`NAME`_ _`REPOSITORY_URL`_``                     | Add a Helm chart repository.    |
| `list`                                                  | List Helm chart repositories.   |
| `update`                                                | Update Helm chart repositories. |
| ``remove _`REPOSITORY1_NAME`_ _`REPOSITORY2_NAME`_ …​`` | Remove Helm chart repositories. |

The following command adds a repository:

```sh
[user@host ~]$ helm repo add \   openshift-helm-charts https://charts.openshift.io/
"openshift-helm-charts" has been added to your repositories
```

This command and other repository commands change only local configuration, and do not affect any cluster resources. The `helm repo add` command updates the `~/.config/helm/repositories.yaml` configuration file, which keeps the list of configured repositories.

When repositories are configured, other commands can use the list of repositories to perform actions. For example, the `helm search repo` command lists all available charts in the configured repositories:

```sh
[user@host ~]$ helm search repo
NAME          CHART VERSION    APP VERSION    DESCRIPTION
repo/chart    0.0.7            latest         A sample chart
...output omitted...
```
By default, the `helm search repo` command shows only the latest version of a chart. Use the `--versions` option to list all available versions. By default, the `install` and `upgrade` commands use the latest version of the chart in the repository. You can use the `--version` option to install specific versions.


# Chapter 3.  Authentication and Authorization

**Abstract**

|   |   |
|---|---|
|**Goal**|Configure authentication with the HTPasswd identity provider and assign roles to users and groups.|
|**Objectives**|- Configure the HTPasswd identity provider for OpenShift authentication.<br>    <br>- Define role-based access controls and apply permissions to users.|
|**Sections**|- Configure Identity Providers (and Guided Exercise)<br>    <br>- Define and Apply Permissions with RBAC (and Guided Exercise)|
|**Lab**|- Authentication and Authorization|

## Configure Identity Providers

### Objectives

- Configure the HTPasswd identity provider for OpenShift authentication.

### OpenShift Users and Groups

Several OpenShift resources relate to authentication and authorization. The following list shows the primary resource types and their definitions:

- User

In the OpenShift Container Platform architecture, users are entities that interact with the API server. The user resource represents an actor within the system. Assign permissions by adding roles to the user directly or to the groups that the user is a member of.

- Identity

The identity resource keeps a record of successful authentication attempts from a specific user and identity provider. Any data about the source of the authentication is stored on the identity.

- Service Account

In OpenShift, applications can communicate with the API independently when user credentials cannot be acquired. To preserve the integrity of the credentials for a regular user, credentials are not shared and service accounts are used instead. With service accounts, you can control API access without the need to borrow a regular user's credentials.

- Group

Groups represent a specific set of users. Users are assigned to groups. Authorization policies use groups to assign permissions to multiple users at the same time. For example, to grant 20 users access to objects within a project, it is better to use a group instead of granting access to each user individually. OpenShift Container Platform also provides system groups or virtual groups that are provisioned automatically by the cluster.

- Role

A role defines the API operations that a user has permissions to perform on specified resource types. You grant permissions to users, groups, and service accounts by assigning roles to them.

User and identity resources are usually not created in advance. OpenShift usually creates these resources automatically after a successful interactive login with OAuth.

### Authenticating API Requests

The authentication and authorization security layers enable user interaction with the cluster. When a user makes a request to the API, the API associates the user with the request. The authentication layer authenticates the user. On successful authentication, the authorization layer either accepts or rejects the API request. The authorization layer uses role-based access control (RBAC) policies to determine user privileges.

The OpenShift API has two methods for authenticating requests:

- OAuth access tokens
- X.509 client certificates

If the request does not present an access token or certificate, then the authentication layer assigns it the `system:anonymous` virtual user and the `system:unauthenticated` virtual group.

#### The Authentication Operator

The OpenShift Container Platform provides the Authentication operator, which runs an OAuth server. The OAuth server provides OAuth access tokens to users when they attempt to authenticate to the API. An identity provider must be configured and available to the OAuth server. The OAuth server uses an identity provider to validate the identity of the requester. The server reconciles the user with the identity and creates the OAuth access token for the user. OpenShift automatically creates identity and user resources after a successful login.

#### Identity Providers

The OpenShift OAuth server can be configured to use many identity providers. The following lists includes the more common identity providers:

HTPasswd

Validates usernames and passwords against a secret that stores credentials that are generated by using the `htpasswd` command.

Keystone

Enables shared authentication with an OpenStack Keystone v3 server.

LDAP

Configures the LDAP identity provider to validate usernames and passwords against an LDAPv3 server, by using simple bind authentication.

GitHub or GitHub Enterprise

Configures a GitHub identity provider to validate usernames and passwords against GitHub or the GitHub Enterprise OAuth authentication server.

OpenID Connect

Integrates with an OpenID Connect identity provider by using an Authorization Code Flow.

The OAuth custom resource must be updated with your chosen identity provider. You can define multiple identity providers, of the same or different kinds, on the same OAuth custom resource.

### Authenticating as a Cluster Administrator

Before you can configure an identity provider and manage users, you must access your OpenShift cluster as a cluster administrator. A newly installed OpenShift cluster provides two ways to authenticate API requests with cluster administrator privileges. One way is to use the `kubeconfig` file, which embeds an X.509 client certificate that never expires. Another way is to authenticate as the `kubeadmin` virtual user. Successful authentication grants an OAuth access token.

To create additional users and grant them different access levels, you must configure an identity provider and assign roles to your users.

#### Authenticating with the X.509 Certificate

During installation, the OpenShift installer creates a unique `kubeconfig` file in the `auth` directory. The `kubeconfig` file contains specific details and parameters for the CLI to connect a client to the correct API server, including an X.509 certificate.

The installation logs provide the location of the `kubeconfig` file:

INFO Run 'export KUBECONFIG=/root/auth/kubeconfig' to manage the cluster with 'oc'.

### Note

In the classroom environment, the `utility` machine stores the `kubeconfig` file at `/home/lab/ocp4/auth/kubeconfig`. Use the `ssh lab@utility` command from the `workstation` machine to access the `utility` machine.

To use the `kubeconfig` file to authenticate `oc` commands, you must copy the file to your workstation and set the absolute or relative path to the `KUBECONFIG` environment variable. Then, you can run any `oc` command that requires cluster administrator privileges without logging in to OpenShift.

[user@host ~]$ **`export KUBECONFIG=/home/user/auth/kubeconfig`**
[user@host ~]$ **`oc get nodes`**

As an alternative, you can use the `--kubeconfig` option of the `oc` command.

[user@host ~]$ **`oc --kubeconfig /home/user/auth/kubeconfig get nodes`**

#### Authenticating with the `kubeadmin` Virtual User

After installation completes, OpenShift creates the `kubeadmin` virtual user. The `kubeadmin` secret in the `kube-system` namespace contains the hashed password for the `kubeadmin` user. The `kubeadmin` user has cluster administrator privileges.

The OpenShift installer dynamically generates a unique `kubeadmin` password for the cluster. The installation logs provide the `kubeadmin` credentials to log in to the cluster. The cluster installation logs also provide the login, password, and the URL for console access.

_...output omitted..._
INFO The cluster is ready when '`oc login -u kubeadmin -p shdU_trbi_6ucX_edbu_aqop'`
_...output omitted..._
INFO Access the OpenShift web-console here:
     https://console-openshift-console.apps.ocp4.example.com
INFO Login to the console with user: kubeadmin, password: shdU_trbi_6ucX_edbu_aqop

### Note

In the classroom environment, the `utility` machine stores the password for the `kubeadmin` user in the `/home/lab/ocp4/auth/kubeadmin-password` file.

#### Deleting the Virtual User

After you define an identity provider, create a user, and assign that user the `cluster-admin` role, you can remove the `kubeadmin` user credentials to improve cluster security.

[user@host ~]$ **`oc delete secret kubeadmin -n kube-system`**

### Warning

If you delete the `kubeadmin` secret before you configure another user with cluster admin privileges, then you can administer your cluster only by using the `kubeconfig` file. If you do not have a copy of this file in a safe location, then you cannot recover administrative access to your cluster. The only alternative is to destroy and reinstall your cluster.

### Warning

**Do not** delete the `kubeadmin` user at any time during this course. The `kubeadmin` user is essential to the course lab architecture. If you deleted this user, you would have to delete the lab environment and re-create it.

### Managing Users with the HTPasswd Identity Provider

Managing user credentials with the HTPasswd Identity Provider requires creating a temporary `htpasswd` file, changing the file, and applying these changes to the secret.

#### Creating an HTPasswd File

The `httpd-tools` package provides the `htpasswd` utility, which must be installed and available on your system.

Create the `htpasswd` file.

[user@host ~]$ **`htpasswd -c -B -b /tmp/htpasswd student redhat123`**

### Important

Use the `-c` option only when creating a file. The `-c` option replaces all file content if the file already exists.

Add or update credentials.

[user@host ~]$ **`htpasswd -b /tmp/htpasswd student redhat1234`**

Delete credentials.

[user@host ~]$ **`htpasswd -D /tmp/htpasswd student`**

#### Creating the HTPasswd Secret

To use the HTPasswd provider, you must create a secret that contains the `htpasswd` file data. The following example uses a secret named `htpasswd-secret`.

[user@host ~]$ **`oc create secret generic htpasswd-secret \   --from-file htpasswd=/tmp/htpasswd -n openshift-config`**

### Important

A secret that the HTPasswd identity provider uses requires adding the `htpasswd=` prefix before specifying the path to the file.

#### Extracting Secret Data

When adding or removing users, use the `oc extract` command to retrieve the secret. Extracting the secret ensures that you work on the current set of users.

By default, the `oc extract` command saves each key within a configuration map or secret as a separate file. Alternatively, you can then redirect all data to a file or display it as standard output. To extract data from the `htpasswd-secret` secret to the `/tmp/` directory, use the following command. The `--confirm` option replaces the file if it exists.

[user@host ~]$ **`oc extract secret/htpasswd-secret -n openshift-config \   --to /tmp/ --confirm`**
/tmp/htpasswd

#### Updating the HTPasswd Secret

The secret must be updated after adding, changing, or deleting users. Use the `oc set data secret` command to update a secret. Unless the file name is `htpasswd`, you must specify `htpasswd=` to update the `htpasswd` key within the secret.

The following command updates the `htpasswd-secret` secret in the `openshift-config` namespace by using the content of the `/tmp/htpasswd` file.

[user@host ~]$ **`oc set data secret/htpasswd-secret \   --from-file htpasswd=/tmp/htpasswd -n openshift-config`**

After updating the secret, the OAuth operator redeploys pods in the `openshift-authentication` namespace. Monitor the redeployment of the new OAuth pods by running the following command:

[user@host ~]$ **`watch oc get pods -n openshift-authentication`**

Test additions, changes, or deletions to the secret after the new pods finish deploying.

### Configuring the HTPasswd Identity Provider

The HTPasswd identity provider validates users against a secret that contains usernames and passwords that are generated with the `htpasswd` command from the Apache HTTP Server project. Only a cluster administrator can change the data inside the HTPasswd secret. Regular users cannot change their own passwords.

Managing users with the HTPasswd identity provider might suffice for a proof-of-concept environment with a small set of users. However, most production environments require a more powerful identity provider that integrates with the organization's identity management system.

#### Configuring the OAuth Custom Resource

To use the HTPasswd identity provider, the OAuth custom resource must be edited to add an entry to the `.spec.identityProviders` array:

apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
`spec`:
  `identityProviders`:
  - name: `my_htpasswd_provider` ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
    mappingMethod: `claim` ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
    type: HTPasswd
    htpasswd:
      fileData:
        name: `htpasswd-secret` ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_configuring_the_oauth_custom_resource-CO22-1)|This provider name is prefixed to provider user names to form an identity name.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_configuring_the_oauth_custom_resource-CO22-2)|Controls how mappings are established between provider identities and user objects. With the default `claim` value, you cannot log in with different identity providers.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_configuring_the_oauth_custom_resource-CO22-3)|An existing secret that contains data that is generated by using the `htpasswd` command.|

#### Updating the OAuth Custom Resource

To update the OAuth custom resource, use the `oc get` command to export the existing OAuth cluster resource to a file in YAML format.

[user@host ~]$ **`oc get oauth cluster -o yaml > oauth.yaml`**

Then, open the resulting file in a text editor and make the needed changes to the embedded identity provider settings.

After completing modifications and saving the file, you must apply the new custom resource by using the `oc replace` command.

[user@host ~]$ **`oc replace -f oauth.yaml`**

### Deleting Users and Identities

When a scenario occurs that requires you to delete a user, it is not sufficient to delete the user from the identity provider. The user and identity resources must also be deleted.

You must remove the password from the htpasswd secret, remove the user from the local htpasswd file, and then update the secret.

To delete the user from `htpasswd`, run the following command:

[user@host ~]$ **`htpasswd -D /tmp/htpasswd manager`**

Update the secret to remove all remnants of the user's password.

[user@host ~]$ **`oc set data secret/htpasswd-secret \   --from-file htpasswd=/tmp/htpasswd -n openshift-config`**

Remove the user resource with the following command:

[user@host ~]$ **`oc delete user manager`**
user.user.openshift.io "manager" deleted

Identity resources include the name of the identity provider. To delete the identity resource for the `manager` user, find the resource and then delete it.

[user@host ~]$ **`oc get identities | grep manager`**
my_htpasswd_provider:manager   my_htpasswd_provider   manager       manager   ...

[user@host ~]$ **`oc delete identity my_htpasswd_provider:manager`**
identity.user.openshift.io "my_htpasswd_provider:manager" deleted

### Assigning Administrative Privileges

The cluster-wide `cluster-admin` role grants cluster administration privileges to users and groups. With this role, the user can perform any action on any resources within the cluster. The following example assigns the `cluster-admin` role to the `student` user.

[user@host ~]$ **`oc adm policy add-cluster-role-to-user cluster-admin student`**

### References

For more information about identity providers, refer to the _Understanding Identity Provider Configuration_ chapter in the Red Hat OpenShift Container Platform 4.14 _Authentication and Authorization_ documentation at [https://docs.redhat.com/en/documentation/openshift_container_platform/4.14/html-single/authentication_and_authorization/index#understanding-identity-provider](https://docs.redhat.com/en/documentation/openshift_container_platform/4.14/html-single/authentication_and_authorization/index#understanding-identity-provider)

**Revision:** do280-4.14-unknown

Configure Identity Providers

16/71

![Red Hat Training + Certification logo](data:image/svg+xml;base64,PHN2ZyBpZD0iTGF5ZXJfMSIgZGF0YS1uYW1lPSJMYXllciAxIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCA4MDYgMzUwIj48ZGVmcz48c3R5bGU+LmNscy0xe2ZpbGw6I2UwMDt9LmNscy0ye2ZpbGw6I2ZmZjt9PC9zdHlsZT48L2RlZnM+PHRpdGxlPkxvZ28tUmVkX0hhdC1UcmFpbmluZ19hbmRfQ2VydGlmaWNhdGlvbi1CLVJldmVyc2UtUkdCPC90aXRsZT48cGF0aCBjbGFzcz0iY2xzLTEiIGQ9Ik0xMjgsODRjMTIuNTEsMCwzMC42MS0yLjU4LDMwLjYxLTE3LjQ2YTE0LDE0LDAsMCwwLS4zMS0zLjQybC03LjQ1LTMyLjM2Yy0xLjcyLTcuMTItMy4yMy0xMC4zNS0xNS43My0xNi42QzEyNS4zOSw5LjE5LDEwNC4yNiwxLDk4LDFjLTUuODIsMC03LjU1LDcuNTQtMTQuNDUsNy41NC02LjY4LDAtMTEuNjQtNS42LTE3Ljg5LTUuNi02LDAtOS45MSw0LjA5LTEyLjkzLDEyLjUsMCwwLTguNDEsMjMuNzItOS40OSwyNy4xNkE2LjQzLDYuNDMsMCwwLDAsNDMsNDQuNTRDNDMsNTMuNzYsNzkuMzMsODQsMTI4LDg0bTMyLjU1LTExLjQyYzEuNzMsOC4xOSwxLjczLDkuMDUsMS43MywxMC4xMywwLDE0LTE1Ljc0LDIxLjc3LTM2LjQzLDIxLjc3Qzc5LDEwNC40NywzOC4wOCw3Ny4xLDM4LjA4LDU5YTE4LjQ1LDE4LjQ1LDAsMCwxLDEuNTEtNy4zM0MyMi43Nyw1Mi41MiwxLDU1LjU0LDEsNzQuNzIsMSwxMDYuMiw3NS41OSwxNDUsMTM0LjY1LDE0NWM0NS4yOCwwLDU2LjctMjAuNDgsNTYuNy0zNi42NSwwLTEyLjcyLTExLTI3LjE2LTMwLjgzLTM1Ljc4Ii8+PHBhdGggZD0iTTE2MC41Miw3Mi41N2MxLjczLDguMTksMS43Myw5LjA1LDEuNzMsMTAuMTMsMCwxNC0xNS43NCwyMS43Ny0zNi40MywyMS43N0M3OSwxMDQuNDcsMzguMDgsNzcuMSwzOC4wOCw1OWExOC40NSwxOC40NSwwLDAsMSwxLjUxLTcuMzNsMy42Ni05LjA2QTYuNDMsNi40MywwLDAsMCw0Myw0NC41NEM0Myw1My43Niw3OS4zMyw4NCwxMjgsODRjMTIuNTEsMCwzMC42MS0yLjU4LDMwLjYxLTE3LjQ2YTE0LDE0LDAsMCwwLS4zMS0zLjQyWiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTIyMiwxNDUuNzNoNjIuMDl2OS42N0gyNTguNTR2NjMuMTRIMjQ3LjYyVjE1NS40SDIyMloiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik0yODcsMTY1LjZoMTAuNHY2Ljc2QTE2LjU3LDE2LjU3LDAsMCwxLDMxMiwxNjQuNDVhMTMuMTMsMTMuMTMsMCwwLDEsNS4zMS44NHY5LjM2YTE3LjgxLDE3LjgxLDAsMCwwLTYuMTQtMS4xNWMtNi4xMywwLTExLDMuMzMtMTMuNzMsOS42N3YzNS4zN0gyODdaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNMzIwLjczLDIwMy4zNWMwLTEwLDguMTEtMTYuMTIsMjEuNDItMTYuMTJhMzcuNDEsMzcuNDEsMCwwLDEsMTQuNDYsMi45MXYtNS42MWMwLTcuNDktNC40Ny0xMS4yNC0xMi45LTExLjI0LTQuODgsMC05Ljg4LDEuMzYtMTYuNDMsNC40OEwzMjMuNDMsMTcwYzcuOTEtMy43NCwxNC43Ny01LjQsMjEuNzQtNS40LDEzLjczLDAsMjEuNjMsNi43NiwyMS42MywxOC45M3YzNUgzNTYuNjFWMjE0YTI1LjEyLDI1LjEyLDAsMCwxLTE2LjQzLDUuNTFDMzI4LjYzLDIxOS40NywzMjAuNzMsMjEyLjkyLDMyMC43MywyMDMuMzVabTIxLjg0LDguNDNhMjAuMzQsMjAuMzQsMCwwLDAsMTQtNXYtOS4xNUEyNy41NCwyNy41NCwwLDAsMCwzNDMsMTk0LjQxYy03LjYsMC0xMi4yOCwzLjQzLTEyLjI4LDguNzNDMzMwLjcxLDIwOC4yNCwzMzUuNSwyMTEuNzgsMzQyLjU3LDIxMS43OFoiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik0zNzcuNjIsMTUwLjYyYTYuMzksNi4zOSwwLDAsMSw2LjM0LTYuNDUsNi40NSw2LjQ1LDAsMSwxLDAsMTIuOUE2LjM5LDYuMzksMCwwLDEsMzc3LjYyLDE1MC42MlptMTEuNTQsNjcuOTJoLTEwLjRWMTY1LjZoMTAuNFoiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik00MDEuNDMsMTY1LjZoMTAuNHY1LjNhMjEuOCwyMS44LDAsMCwxLDE1LjkxLTYuMzRjMTIuMTcsMCwyMC43LDguNDIsMjAuNywyMC42OXYzMy4yOUg0MzhWMTg3YzAtOC4zMi01LjEtMTMuNDEtMTMuMS0xMy40MWExNSwxNSwwLDAsMC0xMy4xMSw2Ljg2djM4LjA3aC0xMC40WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTQ1OS4yNSwxNTAuNjJhNi4zOSw2LjM5LDAsMCwxLDYuMzUtNi40NSw2LjQ1LDYuNDUsMCwwLDEsMCwxMi45QTYuMzksNi4zOSwwLDAsMSw0NTkuMjUsMTUwLjYyWm0xMS41NSw2Ny45Mkg0NjAuNFYxNjUuNmgxMC40WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTQ4My4wNywxNjUuNmgxMC40djUuM2EyMS44LDIxLjgsMCwwLDEsMTUuOTEtNi4zNGMxMi4xNywwLDIwLjcsOC40MiwyMC43LDIwLjY5djMzLjI5aC0xMC40VjE4N2MwLTguMzItNS4xLTEzLjQxLTEzLjExLTEzLjQxYTE1LDE1LDAsMCwwLTEzLjEsNi44NnYzOC4wN2gtMTAuNFoiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik01MzkuNjQsMTkxLjgxYTI2LjcxLDI2LjcxLDAsMCwxLDI2Ljg0LTI3LjE1LDI2LDI2LDAsMCwxLDE1LjcsNS4zVjE2NS42aDEwLjN2NTMuNjZjMCwxMy43My04LjY0LDIxLjQzLTI0LjI0LDIxLjQzYTQ0LjkyLDQ0LjkyLDAsMCwxLTIxLjMyLTUuMmw0LjA2LTguMTFjNiwzLjEyLDExLjM0LDQuNTcsMTYuOTUsNC41Nyw5LjI2LDAsMTQuMTUtNC4zNywxNC4xNS0xMi42OXYtNS45M0EyNC45MiwyNC45MiwwLDAsMSw1NjYuMjcsMjE5QzU1MS4zOSwyMTksNTM5LjY0LDIwNyw1MzkuNjQsMTkxLjgxWm0yNy44OCwxOC4wOWM2LDAsMTEuMTItMi4xOCwxNC41Ni02LjEzVjE3OS44NWExOS4xLDE5LjEsMCwwLDAtMTQuNTYtNi4xNGMtMTAsMC0xNy42OSw3LjktMTcuNjksMTguMVM1NTcuNTMsMjA5LjksNTY3LjUyLDIwOS45WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTYyMy43OCwyMDMuMzVjMC0xMCw4LjExLTE2LjEyLDIxLjQyLTE2LjEyYTM3LjQxLDM3LjQxLDAsMCwxLDE0LjQ2LDIuOTF2LTUuNjFjMC03LjQ5LTQuNDctMTEuMjQtMTIuOS0xMS4yNC00Ljg4LDAtOS44OCwxLjM2LTE2LjQzLDQuNDhMNjI2LjQ4LDE3MGM3LjkxLTMuNzQsMTQuNzctNS40LDIxLjc0LTUuNCwxMy43MywwLDIxLjYzLDYuNzYsMjEuNjMsMTguOTN2MzVINjU5LjY2VjIxNGEyNS4xMiwyNS4xMiwwLDAsMS0xNi40Myw1LjUxQzYzMS42OCwyMTkuNDcsNjIzLjc4LDIxMi45Miw2MjMuNzgsMjAzLjM1Wm0yMS44NCw4LjQzYTIwLjM0LDIwLjM0LDAsMCwwLDE0LTV2LTkuMTVBMjcuNTQsMjcuNTQsMCwwLDAsNjQ2LDE5NC40MWMtNy42LDAtMTIuMjgsMy40My0xMi4yOCw4LjczQzYzMy43NiwyMDguMjQsNjM4LjU1LDIxMS43OCw2NDUuNjIsMjExLjc4WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTY4MS44MSwxNjUuNmgxMC40djUuM2EyMS44LDIxLjgsMCwwLDEsMTUuOTEtNi4zNGMxMi4xNywwLDIwLjcsOC40MiwyMC43LDIwLjY5djMzLjI5aC0xMC40VjE4N2MwLTguMzItNS4xLTEzLjQxLTEzLjExLTEzLjQxYTE1LDE1LDAsMCwwLTEzLjEsNi44NnYzOC4wN2gtMTAuNFoiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik03ODEuMzQsMjEzLjU0YTI1LDI1LDAsMCwxLTE2LjIzLDUuODNjLTE1LDAtMjYuNzMtMTItMjYuNzMtMjcuMzZzMTEuNzYtMjcuMjQsMjYuOTQtMjcuMjRBMjcsMjcsMCwwLDEsNzgxLjIzLDE3MFYxNDUuNzNsMTAuNC0yLjI5djc1LjFINzgxLjM0Wm0tMTQuNzctMy4yMkExOSwxOSwwLDAsMCw3ODEuMjMsMjA0VjE4MGExOS4zNSwxOS4zNSwwLDAsMC0xNC42Ni02LjI0Yy0xMC4xOSwwLTE4LDcuOC0xOCwxOC4yUzc1Ni4zOCwyMTAuMzIsNzY2LjU3LDIxMC4zMloiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik0yODQuNjQsMzA3LjgxbDcuMTgsNy4yOGE0MC4wNyw0MC4wNywwLDAsMS0yOS4xMiwxMi41OWMtMjEuNTMsMC0zOC4xNy0xNi41NC0zOC4xNy0zNy41NXMxNi42NC0zNy41NCwzOC4xNy0zNy41NGMxMS40NCwwLDIyLjQ2LDQuNzgsMjkuMzMsMTIuNThsLTcuMjgsNy40OWEyOC45NCwyOC45NCwwLDAsMC0yMi4wNS0xMGMtMTUuMzksMC0yNywxMS44NS0yNywyNy40NXMxMS43NSwyNy40NiwyNy4zNSwyNy40NkEyOC4yNSwyOC4yNSwwLDAsMCwyODQuNjQsMzA3LjgxWiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTMyNC41OCwzMjcuNDdjLTE1LjYsMC0yNy43Ny0xMi0yNy43Ny0yNy40NiwwLTE1LjI4LDExLjY1LTI3LjI0LDI2LjUyLTI3LjI0LDE0LjU2LDAsMjUuNTgsMTIuMDYsMjUuNTgsMjcuNjZ2M2gtNDEuOEExNy44NCwxNy44NCwwLDAsMCwzMjUsMzE4LjYzLDIwLjYyLDIwLjYyLDAsMCwwLDMzOC40MSwzMTRsNi42Niw2LjU1QTMxLjU2LDMxLjU2LDAsMCwxLDMyNC41OCwzMjcuNDdabS0xNy4zNy0zMS44MmgzMS40MWMtMS41Ni04LjEyLTcuOC0xNC4xNS0xNS41LTE0LjE1QzMxNS4xMSwyODEuNSwzMDguNzcsMjg3LjIyLDMwNy4yMSwyOTUuNjVaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNMzU4Ljc5LDI3My42aDEwLjR2Ni43NmExNi41NywxNi41NywwLDAsMSwxNC41Ni03LjkxLDEzLjEzLDEzLjEzLDAsMCwxLDUuMzEuODR2OS4zNmExNy44MSwxNy44MSwwLDAsMC02LjE0LTEuMTVjLTYuMTMsMC0xMSwzLjMzLTEzLjczLDkuNjd2MzUuMzdoLTEwLjRaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNNDA0LjY1LDI4Mi4zM0gzOTMuNDJWMjczLjZoMTEuMjNWMjYwLjA4bDEwLjMtMi41djE2aDE1LjZ2OC43M0g0MTVWMzExYzAsNS40MSwyLjE4LDcuMzgsNy44LDcuMzhhMjAuMzUsMjAuMzUsMCwwLDAsNy41OS0xLjI1djguNzRhMzQuMjYsMzQuMjYsMCwwLDEtOS44OCwxLjU2Yy0xMC4yOSwwLTE1LjgxLTQuODktMTUuODEtMTRaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNNDM3LjEsMjU4LjYyYTYuMzksNi4zOSwwLDAsMSw2LjM1LTYuNDUsNi40NSw2LjQ1LDAsMCwxLDAsMTIuOUE2LjM5LDYuMzksMCwwLDEsNDM3LjEsMjU4LjYyWm0xMS41NSw2Ny45MmgtMTAuNFYyNzMuNmgxMC40WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTQ2OC44MiwyNzMuNnYtOGMwLTExLDYuMzUtMTcuMDYsMTgtMTcuMDZhMjcuMDUsMjcuMDUsMCwwLDEsNy40OS45NHY5YTIwLjczLDIwLjczLDAsMCwwLTYuNTUtLjk0Yy01LjcyLDAtOC41MywyLjYtOC41Myw4LjIydjcuOEg0OTQuM3Y4LjczSDQ3OS4yMnY0NC4yMWgtMTAuNFYyODIuMzNINDU2LjU1VjI3My42WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTUwMS4wNiwyNTguNjJhNi4zOSw2LjM5LDAsMCwxLDYuMzQtNi40NSw2LjQ1LDYuNDUsMCwxLDEsMCwxMi45QTYuMzksNi4zOSwwLDAsMSw1MDEuMDYsMjU4LjYyWm0xMS41NCw2Ny45Mkg1MDIuMlYyNzMuNmgxMC40WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTU2NC40LDMxMS44N2w2LjI0LDYuNzZhMjkuMjcsMjkuMjcsMCwwLDEtMjAuOTEsOC44NCwyNy40NiwyNy40NiwwLDAsMSwwLTU0LjkxLDMwLDMwLDAsMCwxLDIxLjMyLDguODRsLTYuNTUsNy4wN2ExOS42OCwxOS42OCwwLDAsMC0xNC41Ni02LjY2Yy05LjY3LDAtMTcuMTYsOC0xNy4xNiwxOC4yLDAsMTAuNCw3LjU5LDE4LjMxLDE3LjM3LDE4LjMxQzU1NS40NSwzMTguMzIsNTYwLjEzLDMxNi4yNCw1NjQuNCwzMTEuODdaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNNTc2LjM1LDMxMS4zNWMwLTEwLDguMTItMTYuMTIsMjEuNDMtMTYuMTJhMzcuNDUsMzcuNDUsMCwwLDEsMTQuNDYsMi45MXYtNS42MWMwLTcuNDktNC40OC0xMS4yNC0xMi45LTExLjI0LTQuODksMC05Ljg4LDEuMzYtMTYuNDMsNC40OEw1NzkuMDYsMjc4YzcuOS0zLjc0LDE0Ljc3LTUuNCwyMS43My01LjQsMTMuNzMsMCwyMS42NCw2Ljc2LDIxLjY0LDE4LjkzdjM1LjA1SDYxMi4yNFYzMjJhMjUuMTcsMjUuMTcsMCwwLDEtMTYuNDQsNS41MUM1ODQuMjYsMzI3LjQ3LDU3Ni4zNSwzMjAuOTIsNTc2LjM1LDMxMS4zNVptMjEuODUsOC40M2EyMC4zNywyMC4zNywwLDAsMCwxNC01di05LjE1YTI3LjU1LDI3LjU1LDAsMCwwLTEzLjYzLTMuMjJjLTcuNTksMC0xMi4yNywzLjQzLTEyLjI3LDguNzNDNTg2LjM0LDMxNi4yNCw1OTEuMTIsMzE5Ljc4LDU5OC4yLDMxOS43OFoiLz48cGF0aCBjbGFzcz0iY2xzLTIiIGQ9Ik02MzkuMTcsMjgyLjMzSDYyNy45NFYyNzMuNmgxMS4yM1YyNjAuMDhsMTAuMy0yLjV2MTZoMTUuNnY4LjczaC0xNS42VjMxMWMwLDUuNDEsMi4xOCw3LjM4LDcuOCw3LjM4YTIwLjM1LDIwLjM1LDAsMCwwLDcuNTktMS4yNXY4Ljc0YTM0LjMyLDM0LjMyLDAsMCwxLTkuODgsMS41NmMtMTAuMywwLTE1LjgxLTQuODktMTUuODEtMTRaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNNjcxLjYyLDI1OC42MmE2LjM5LDYuMzksMCwwLDEsNi4zNC02LjQ1LDYuNDUsNi40NSwwLDAsMSwwLDEyLjlBNi4zOSw2LjM5LDAsMCwxLDY3MS42MiwyNTguNjJabTExLjU0LDY3LjkyaC0xMC40VjI3My42aDEwLjRaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNNzIwLjYsMjcyLjU2QTI3LjUxLDI3LjUxLDAsMSwxLDY5MywzMDAsMjcuMTcsMjcuMTcsMCwwLDEsNzIwLjYsMjcyLjU2Wk03MzcuODcsMzAwYzAtMTAuMjktNy43LTE4LjMtMTcuMjctMTguM3MtMTcuMzcsOC0xNy4zNywxOC4zLDcuNTksMTguNDEsMTcuMzcsMTguNDFDNzMwLjE3LDMxOC40Miw3MzcuODcsMzEwLjMxLDczNy44NywzMDBaIi8+PHBhdGggY2xhc3M9ImNscy0yIiBkPSJNNzU3Ljk0LDI3My42aDEwLjR2NS4zYTIxLjc4LDIxLjc4LDAsMCwxLDE1LjkxLTYuMzRjMTIuMTcsMCwyMC43LDguNDIsMjAuNywyMC42OXYzMy4yOUg3OTQuNTRWMjk1YzAtOC4zMi01LjA5LTEzLjQxLTEzLjEtMTMuNDFhMTUsMTUsMCwwLDAtMTMuMSw2Ljg2djM4LjA3aC0xMC40WiIvPjxwYXRoIGNsYXNzPSJjbHMtMiIgZD0iTTU4MC4yNCw5My4zYzAsMTEuODksNy4xNSwxNy42NywyMC4xOSwxNy42N2E1Mi4xMSw1Mi4xMSwwLDAsMCwxMS44OS0xLjY4Vjk1LjUxYTI0Ljg0LDI0Ljg0LDAsMCwxLTcuNjgsMS4xNmMtNS4zNywwLTcuMzYtMS42OC03LjM2LTYuNzNWNjguOGgxNS41NlY1NC42SDU5Ny4yOHYtMThsLTE3LDMuNjhWNTQuNkg1NjlWNjguOGgxMS4yNVptLTUzLC4zMmMwLTMuNjgsMy42OS01LjQ3LDkuMjYtNS40N2E0My4xMiw0My4xMiwwLDAsMSwxMC4xLDEuMjZ2Ny4xNUEyMS41MSwyMS41MSwwLDAsMSw1MzYsOTkuMTljLTUuNDYsMC04LjczLTIuMS04LjczLTUuNTdtNS4yLDE3LjU2YzYsMCwxMC44NC0xLjI2LDE1LjM2LTQuMzF2My4zN2gxNi44MlY3NC41OGMwLTEzLjU2LTkuMTQtMjEtMjQuMzktMjEtOC41MiwwLTE2Ljk0LDItMjYsNi4xbDYuMSwxMi41MmM2LjUyLTIuNzQsMTItNC40MiwxNi44My00LjQyLDcsMCwxMC42MiwyLjczLDEwLjYyLDguMzF2Mi43M2E0OS41Myw0OS41MywwLDAsMC0xMi42Mi0xLjU4Yy0xNC4zMSwwLTIyLjkzLDYtMjIuOTMsMTYuNzMsMCw5Ljc4LDcuNzgsMTcuMjQsMjAuMTksMTcuMjRtLTkyLjQ0LS45NGgxOC4wOVY4MS40MmgzMC4yOXYyOC44MmgxOC4wOVYzNi42Mkg0ODguNDNWNjQuOTFINDU4LjE0VjM2LjYySDQ0MC4wNVpNMzcxLjEyLDgyLjM3YzAtOCw2LjMxLTE0LjEsMTQuNjItMTQuMWExNy4yMiwxNy4yMiwwLDAsMSwxMS43OCw0LjMyVjkyYTE2LjM2LDE2LjM2LDAsMCwxLTExLjc4LDQuNDJjLTguMiwwLTE0LjYyLTYuMS0xNC42Mi0xNC4wOW0yNi42MSwyNy44N2gxNi44M1YzMi45NGwtMTcsMy42OFY1Ny41NWEyOC4zLDI4LjMsMCwwLDAtMTQuMi0zLjY4Yy0xNi4xOSwwLTI4LjkyLDEyLjUxLTI4LjkyLDI4LjVBMjguMjUsMjguMjUsMCwwLDAsMzgyLjgsMTExYTI1LjEyLDI1LjEyLDAsMCwwLDE0LjkzLTQuODNabS03Ny4xOS00Mi43YzUuMzYsMCw5Ljg4LDMuNDcsMTEuNjcsOC44M0gzMDljMS42OC01LjU3LDUuODktOC44MywxMS41Ny04LjgzTTI5MS44Myw4Mi40N2MwLDE2LjIsMTMuMjUsMjguODIsMzAuMjgsMjguODIsOS4zNiwwLDE2LjItMi41MywyMy4yNS04LjQybC0xMS4yNi0xMGMtMi42MywyLjc0LTYuNTIsNC4yMS0xMS4xNCw0LjIxYTE0LjM5LDE0LjM5LDAsMCwxLTEzLjY4LTguODNoMzkuNjVWODQuMDVjMC0xNy42Ny0xMS44OC0zMC4zOS0yOC4wOC0zMC4zOWEyOC41NywyOC41NywwLDAsMC0yOSwyOC44MU0yNjIuNDksNTIuMDhjNiwwLDkuMzYsMy43OCw5LjM2LDguMzFzLTMuMzcsOC4zMS05LjM2LDguMzFIMjQ0LjYxVjUyLjA4Wm0tMzYsNTguMTZoMTguMDlWODMuNDJoMTMuNzdsMTMuODksMjYuODJoMjAuMTlsLTE2LjItMjkuNDVhMjIuMjcsMjIuMjcsMCwwLDAsMTMuODgtMjAuNzJjMC0xMy4yNS0xMC40MS0yMy40NS0yNi0yMy40NUgyMjYuNTJaIi8+PC9zdmc+)

- [Privacy Policy](https://www.redhat.com/en/about/privacy-policy?extIdCarryOver=true&sc_cid=701f2000001D8QoAAK)
- [Red Hat Training Policies](https://www.redhat.com/en/about/red-hat-training-policies)
- [Terms of Use](https://www.redhat.com/en/about/terms-use)
- [All policies and guidelines](https://www.redhat.com/en/about/all-policies-guidelines)
- [Release Notes](https://learn.redhat.com/t5/Red-Hat-Learning-Subscription/bg-p/RedHatLearningSubscriptionGroupblog-board)