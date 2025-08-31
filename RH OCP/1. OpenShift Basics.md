# OpenShift
## The RHOCP CLI

Command-line utilities (CLI) provide an alternative for interacting with your RHOCP cluster. Developers familiar with Kubernetes can use the `kubectl` utility to manage an RHOCP cluster. This course uses the `oc` command-line utility, which is designed to take advantage of additional RHOCP features.

The CLI utilities provide developers with a range of commands that are useful for managing the RHOCP cluster and its applications. Each command is translated into an API call, and the response is displayed in the command line.

The following is a list of the most common `oc` commands.

**oc login**

Before you can interact with your RHOCP cluster, you must authenticate your requests. Use the `login` command to authenticate your requests.

For example, in this course, you can use the following command:

```
[user@host ~]$ oc login https://api.ocp4.example.com:6443
Username: developer
Password: developer
Login successful.
```

You don't have any projects. You can try to create a new project, by running

```
  $ oc new-project <projectname>

Welcome to OpenShift! See 'oc help' to get started.
```

**oc get**

Use the `get` command to retrieve a list of selected resources in the selected project.

You must specify the resource type to list.

For example, the following command returns the list of the `pod` resources in the current project.

```
[user@host ~]$ oc get pod
NAME                          READY   STATUS    RESTARTS   AGE
quotes-api-6c9f758574-nk8kd   1/1     Running   0          39m
quotes-ui-d7d457674-rbkl7     1/1     Running   0          67s
```

**oc create**

Use the `create` command to create an RHOCP resource. Developers commonly use the `-f` flag to indicate the file that contains the JSON or YAML representation of an RHOCP resource.

For example, to create resources from the `pod.yaml` file, use the following command:

```
[user@host ~]$ oc create -f pod.yaml
pod/quotes-pod created
```

RHOCP resources in YAML format are discussed later.

**oc delete**

Use the `delete` command to delete an existing RHOCP resource. You must specify the resource type and the resource name.

For example, to delete the `quotes-ui` pod, use the following command:

```
[user@host ~]$ oc delete pod quotes-ui
pod/quotes-ui deleted
```

**oc logs**

Use the `logs` command to print the standard output of a pod. This command requires a pod name as an argument. You can print only logs of a container in a pod, which means the resource type is omitted.

For example, to print the logs from the `react-ui` pod, use the following command:

```
[user@host ~]$ oc logs react-ui
Compiled successfully!

You can now view ts-page in the browser.

  Local:            http://localhost:3000
  On Your Network:  http://10.0.1.23:3000
_...output omitted..._
```

The `oc` utility provides equal functionality to the web console. For a full list of commands, execute `oc --help`. Additionally, you can execute ``oc _`command`_ --help``, for example `oc logs --help` to view the `oc logs` command documentation.

## RHOCP Resources

Developers configure RHOCP by using a set of Kubernetes and RHOCP-specific objects. When you create or modify an object, you make a persistent record of the intended state. RHOCP reads the object and modifies the current state accordingly.

### Shared Resource Fields

All RHOCP and Kubernetes objects can be represented as a JSON or YAML structure with common fields. Consider the following pod object in the YAML format:

```
kind: Pod ![1]
apiVersion: v1 ![2]
metadata: ![3]
  name: example-pod
  namespace: example-namespace
spec: ![4]
_...definition omitted..._
status: ![5]
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2022-08-19T12:59:22Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: cri-o://e37c....f5c2
    image: quay.io/example/awesome-container:latest
    lastState: {}
    name: podman-quotes-ui
    ready: true
_...object omitted..._
```

|     |                                                                                                                                                      |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Schema identifier. In this example, the object conforms to the pod schema.                                                                           |
| 2   | Identifier of the object schema version.                                                                                                             |
| 3   | Metadata for a given resource, such as annotations, labels, name, namespace, and others. The `.metadata.namespace` field refers to an RHOCP project. |
| 4   | The desired state of the object.                                                                                                                     |
| 5   | Current state of the object. This field is provided by RHOCP, and lists information such as runtime status, readiness, container image, and others.  |

Every RHOCP resource contains the `.kind`, `.apiVersion`, `.spec` and `.status` fields. However, when you create an object definition, you do not need to provide the `.status` field. The `.status` field is useful, for example, for troubleshooting the reason for a pod scheduling error.

Use the `oc explain` command to get information about valid fields for an object. For example, execute `oc explain pod` to get information about possible Pod object fields. You can use the YAML path to get information about a particular field, for example:

```
[user@host ~]$ oc explain pod.metadata.name
KIND:     Pod
VERSION:  v1

FIELD:    name <string>

DESCRIPTION:
_...output omitted..._
```

###  Label Kubernetes Objects

Labels are key-value pairs that you define in the `.metadata.labels` object, for example:

```
kind: Pod
apiVersion: v1
metadata:
  name: example-pod
  labels:
    app: example-pod
    group: developers
_...object omitted..._
```
The preceding example contains the `app=example-pod` and `group=developers` labels. Developers often use labels to target a set of objects by using the `-l` or the equivalent `--selector` option. For example, the following `oc get` command lists pods that contain the `group=developers` label:

```
[user@host ~]$ oc get pod --selector group=developers
NAME                          READY   STATUS    RESTARTS   AGE
example-pod-6c9f758574-7fhg   1/1     Running   5          11d
```

## Deploy Applications as Pods to RHOCP

A pod represents a group of one or multiple containers that share resources, such as a network interface or file system.

Grouping containers into pods is useful for implementing multi-container design patterns, such as using an initialization container to load application data when a pod starts for the first time. Multi-container design patterns are out of scope for this course.

You can use pods locally or on a cluster. On a local system, Podman can manage pods by using the `podman pod` command. This is useful for developing and testing applications that use a multi-container design pattern. On Kubernetes or RHOCP, pods are the atomic unit for deploying and managing your applications. For example, you can deploy a single-container application by creating a pod that contains a single container.

Pods do not expose advanced functionality, such as application scaling or zero-downtime updates. Kubernetes and RHOCP implement such functionality with objects that control pod sets, such as the _Deployment_ object. Controller objects are covered in a later section.

#### Create Pods Declaratively

Storing a pod definition in a file is called the _declarative approach_ for creating RHOCP objects, because you declare the intended pod state outside of RHOCP. This means you can store the pod definition in a version control system, such as Git, and incrementally change your definition as your application evolves.

The following YAML object demonstrates the important fields of a pod object:

```
kind: Pod ![1]
apiVersion: v1
metadata:
  name: example-pod ![2]
  namespace: example-project ![3]
spec: ![4]
  containers: ![5]
  - name: example-container ![6]
    image: quay.io/example/awesome-container ![7]
    ports: ![8]
    - containerPort: 8080
    env: ![9]
    - name: GREETING
      value: "Hello from the awesome container"

```

|     |                                                                                                                                                                                          |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | This YAML object defines a pod.                                                                                                                                                          |
| 2   | The `.metadata.name` field defines the name of the pod.                                                                                                                                  |
| 3   | The project in which you create the pod. If this project does not exist, then the pod creation fails. If you do not specify a project, then RHOCP uses the currently configured project. |
| 4   | The `.spec` field contains the pod object configuration.                                                                                                                                 |
| 5   | The `.spec.containers` field defines containers in a pod. In this example, the `example-pod` contains one container.                                                                     |
| 6   | The name of the container inside of a pod. Container names are important for `oc` commands when a pod contains multiple containers.                                                      |
| 7   | The image for the container.                                                                                                                                                             |
| 8   | The port metadata specifies what ports the container uses. This property is similar to the `EXPOSE` Containerfile property.                                                              |
| 9   | The `env` property defines environment variables.                                                                                                                                        |

You can create the pod object by saving the YAML definition into a file and then using the `oc` command, for example:

```
[user@host ~]$ oc create -f pod.yaml
pod/example-pod created
```

#### Create Pods Imperatively

RHOCP also provides the _imperative approach_ to create RHOCP objects. The imperative approach uses the `oc run` command to create a pod without a definition.

The following command creates the `example-pod` pod:

```
[user@host ~]$ oc run example-pod \ ![1]
--image=quay.io/example/awesome-container \ ![2]
--env GREETING='Hello from the awesome container' \ ![3]
--port=8080 ![4]
pod/example-pod created
```

|     |                                                                                  |
| --- | -------------------------------------------------------------------------------- |
| 1   | The pod `.metadata.name` definition.                                             |
| 2   | The image used for the single container in this pod.                             |
| 3   | The `--env` option injects an environment variable in the container of this pod. |
| 4   | The port metadata definition.                                                    |

The imperative commands are a faster way of creating pods, because such commands do not require a pod object definition. However, developers lose the ability to version and incrementally change the pod definition.

Generally, developers test the deployment by using imperative commands, and then use the imperative commands to generate the pod object definition. Use the `--dry-run=client` option to avoid creating the object in RHOCP. Additionally, use the `-o yaml` or `-o json` option to configure the definition format.

The following command is an example of generating the YAML definition for the `example-pod` pod.

```
[user@host ~]$ oc run example-pod \
--image=quay.io/example/awesome-container \
--env GREETING='Hello from the awesome container' \
--port=8080 \
--dry-run=client -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: example-pod
  name: example-pod
spec:
  containers:
_...output omitted..._
```

### Application Networking in RHOCP

Developers configure internal pod-to-pod network communication in RHOCP by using the _Service_ object. Applications send requests to the service name and port. RHOCP provides a virtual network which reroutes such requests to the pods that the service targets by using labels.

#### Create Services Declaratively

The following YAML object demonstrates a service object:

```
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  ports:
  - port: 8080 ![1]
    protocol: TCP
    targetPort: 8080 ![2]
  selector: ![3]
    app: backend-app
```

|     |                                                                                                                                                   |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Service port. This is the port on which the service listens.                                                                                      |
| 2   | Target port. This is the pod port to which the service routes requests. This port corresponds to the `containerPort` value in the pod definition. |
| 3   | The selector configures which pods to target. In this case, the service routes to any pods that contain the `app=backend-app` label.              |

The preceding example defines the `backend` service. This means that applications can send requests to the `http://backend:8080` URL. Such requests are routed to the pods with the `app=backend-app` label on port `8080`.

In RHOCP, the default service type is `ClusterIP`, which means the service is used for pod-to-pod routing within the RHOCP cluster. Other service types, such as the `LoadBalancer` service, are outside of the scope of this course.

#### Create Services Imperatively

Similarly to pods, you can create services imperatively by using the `oc expose` command. The following example creates a service for the `backend-app` pod:

```
[user@host ~]$ **`oc expose pod backend-app \   --port=8080 \ ![1]
--targetPort=8080 \ ![2]
--name=backend-app ![3]
service/backend-app exposed
```

|     |                                    |
| --- | ---------------------------------- |
| 1   | Port on which the service listens. |
| 2   | Target container port.             |
| 3   | Service name.                      |

You can also use the `--dry-run=client` and `-o` options to generate a service definition, for example:

```
[user@host ~]$ oc expose pod backend-app \
--port=8080 \
--targetPort=8080 \
--name=backend-app \
--dry-run=client -o yaml

apiVersion: v1
kind: Service
metadata:
  name: backend-app
spec:
_...output omitted..._
```

## Multi-pod Applications

### Manage Pods with Controllers

Kubernetes encourages developers to use a declarative deployment style, which means that the developer declares the desired application state and Kubernetes reaches the declared state. Kubernetes uses the _controller pattern_ to implement the declarative deployment style.

In the controller pattern, a controller continuously watches the current state of a system. If the system state does not match the desired state, then the controller sends signals to the system until the system state matches the desired state. This is called the _control loop_.

Kubernetes uses a number of controllers to deal with constant change. Consider a single-container _bare pod_ application, which means an application that is deployed by using the `Pod` object. When a bare pod application fails, for example due to a memory leak, Kubernetes can handle the failure by restarting the containers according to the pod's `restartPolicy`. However, in case of a node failure, Kubernetes does not reschedule a bare pod, which is why developers rarely use bare pods for application deployment.

You can use a number of controller objects provided by Kubernetes, such as `Deployment`, `ReplicaSet`, `StatefulSet`, and others.

### Deploy Pods with Deployments

**The `Deployment` object is a Kubernetes controller that manages pods.** Developers use deployments for the following use cases:

**Managing Pod Scaling**

You can configure the number of pod replicas for a given deployment. If the actual replica count decreases, for example due to node failure or a network partition, then the deployment schedules new pods until the declared replica count is reached.

**Managing Application Changes**

Most of the `Pod` object fields are immutable, such as the name or environment variable configuration. To change those fields with bare pods, developers must delete the pod and recreate it with the new configuration.

When you manage pods with a deployment, you declare the new configuration in the deployment object. The deployment controller then deletes the original pods and recreates the new pods that contain the new configuration.

**Managing Application Updates**

Deployments implement the `RollingUpdate` strategy for gradually updating application pods when you declare a new application image. This ensures zero-downtime application updates.

Additionally, you can _roll back_ to the previous application version in case the update does not work correctly.

#### Create Deployments Declaratively

The following YAML object demonstrates a Kubernetes deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:   #![1]
  labels:
    app: deployment-label
  name: example-deployment
spec:
  replicas: 3   #![2]
  selector:   #![3]
    matchLabels:
      app: example-deployment
  strategy: RollingUpdate   #![4]
  template:   #![5]
    metadata:
      labels:
        app: example-deployment
    spec:   #![6]
      containers:
      - image: quay.io/example/awesome-container
        name: awesome-pod
```

|     |                                                                                                                                                  |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | Deployment object metadata. This deployment uses the `app=deployment-label` label.                                                               |
| 2   | Number of pod replicas. This deployment maintains `3` identical containers in `3` pods.                                                          |
| 3   | Selector for pods that the deployment manages. This deployment manages pods that use the `app=example-deployment` label.                         |
| 4   | Strategy for updating pods. The default `RollingUpdate` strategy ensures gradual, zero-downtime pod rollout when you modify the container image. |
| 5   | The template configuration for pods that the deployment creates and manages.                                                                     |
| 6   | The `.spec.template.spec` field corresponds to the `.spec` field of the `Pod` object.                                                            |

#### Create Deployments Imperatively

You can use the `oc create deployment` command to create a deployment:

```bash
[user@host ~]$ oc create deployment example-deployment \ #![1]
--image=quay.io/example/awesome-container \ #![2]
--replicas=3  #![3]
deployment/example-deployment created
```

|     |                                       |
| --- | ------------------------------------- |
| 1   | Set the name to `example-deployment`. |
| 2   | Set the image.                        |
| 3   | Create and maintain `3` replica pods. |

You can also use the `--dry-run=client` and `-o` options to generate a deployment definition, for example:

```bash
[user@host ~]$ oc create deployment example-deployment \
--image=quay.io/example/awesome-container \
--replicas=3 \
--dry-run=client -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
_...output omitted..._
```

#### Deployment Pod Selection

Controllers create and manage the lifecycle of pods that are specified in the controller configuration. Consequently, a controller must have an ownership relationship with pods, and one pod can be owned by at most one controller.

The deployment controller uses labels to target dependent pods. For example, consider the following deployment configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
_...configuration omitted..._
spec:
  selector:
    matchLabels:   #![1]
      app: example-deployment
  template:
    metadata:
      labels:   #![2]
        app: example-deployment
    spec:
_...configuration omitted..._

```

|     |                                             |
| --- | ------------------------------------------- |
| 1   | The `.spec.selector.matchLabels` field.     |
| 2   | The `.spec.template.metadata.labels` field. |

The `.spec.template.metadata.labels` field determines the set of labels applied to the pods created or managed by this deployment. Consequently, the `.spec.selector.matchLabels` field must be a subset of labels of the `.spec.template.metadata.labels` field.

A deployment that targets pod labels missing from the pod template is considered invalid.

### Note

The deployment controller targets a `ReplicaSet` object, which in turn targets pods. Additionally, pod ownership uses object references so that multiple pods can use the same labels and be managed by different controllers.

See the references section for complete information about object ownership and the deployment controller.

### Expose Applications for External Access

Developers commonly expose applications for access from outside of the Kubernetes cluster. 

The default `Service` object provides a stable, internal IP address and a DNS record for pods. However, applications outside of the Kubernetes cluster cannot connect to the IP address or resolve the DNS record.

You can use the `Route` object to expose applications. Route is a Red Hat OpenShift Container Platform (RHOCP) object that connects a public-facing IP address and DNS hostname to an internal-facing IP address. Routes use information from the service object to route requests to pods.

RHOCP exposes an HAProxy router pod that listens on a RHOCP node public IP address. The router serves as an ingress entrypoint to the internal RHOCP traffic.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do188-4.14/images/openshift/multipod/assets/images/openshift/ocp_route.svg)|

Figure 8.5: RHOCP network architecture

Routes are RHOCP-specific objects. If you require full compatibility with other Kubernetes distributions, you can use the `Ingress` object to create and manage routes, which is out of scope of this course.

#### Create Routes Declaratively

The following YAML object demonstrates a RHOCP route:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: app-ui
  name: app-ui
  namespace: awesome-app
spec:
  port:
    targetPort: 8080   #![1]
  host: ""
  to:   #![2]
    kind: "Service"
    name: "app-ui"
```

|     |                                                                                                                                                                |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | The target port on pods selected by the service this route points to. If you use a string, then the route uses a named port in the target endpoints port list. |
| 2   | The route target. Currently, only the `Service` target is allowed.                                                                                             |

The preceding route routes requests to the `app-ui` service endpoints on port `8080`. Because the `app-ui` route does not specify the hostname in the `.spec.host` field, the hostname is generated in the following format:

route-name-project-name.default-domain

In the preceding example, the hostname in the classroom RHOCP cluster is `app-ui-awesome-app.apps.ocp4.example.com`. Consequently, RHOCP routes external requests from the `http://app-ui-awesome-app.apps.ocp4.example.com` domain to the `app-ui:8080` internal RHOCP service.

Administrators configure the default domain during RHOCP deployment.

#### Create Routes Imperatively

You can use the `oc expose service` command to create a route:

```bash
[user@host ~]$ oc expose service app-ui
route.route.openshift.io/app-ui exposed
```

You can also use the `--dry-run=client` and `-o` options to generate a route definition, for example:

```bash
[user@host ~]$ oc expose service app-ui \   --dry-run=client -o yaml
apiVersion: apps/v1
kind: Route
metadata:
  creationTimestamp: null
_...output omitted..._
```

Note that you can use the `oc expose` imperative command in the following forms:

- ``oc expose pod _`POD_NAME`_``: create a service for a specific pod.
- ``oc expose deployment _`DEPLOYMENT_NAME`_``: create a service for all pods managed by a controller, in this case a deployment controller.
- ``oc expose service _`SERVICE_NAME`_``: create a route that targets the specified service.

### References

[Controllers | Kubernetes](https://kubernetes.io/docs/concepts/architecture/controller/)

[Deployments | Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

[Garbage Collection | Kubernetes](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#owners-dependents)

[Workloads | Kubernetes](https://kubernetes.io/docs/concepts/workloads/)

For more information about RHOCP networking, refer to the Red Hat OpenShift Container Platform 4.14 _Networking_ documentation at [https://access.redhat.com/documentation/en-us/openshift_container_platform/4.14/html-single/networking/index](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.14/html-single/networking/index)

### Guided Exercise: Multi-pod Applications

Deploy a multi-pod application into Red Hat OpenShift Container Platform (RHOCP).

**Outcomes**

You should be able to use the `oc` command-line utility to:

- Create RHOCP deployments.
- Configure networking.
- Expose applications for external access.

**Instructions**

 1. Log in to the cluster as the `developer` user, and ensure that you use the `ocp-multipod` project.

 2. Log in to the cluster as the `developer` user.
   [student@workstation ~]$ **`oc login -u developer -p developer \ https://api.ocp4.example.com:6443`**
   Login successful.
   
   _...output omitted..._
  
 3. Ensure that you use the `ocp-multipod` project.
   [student@workstation ~]$ **`oc project ocp-multipod`**
   Already on project "ocp-multipod" on server "https://api.ocp4.example.com:6443".
  
4. Create a `Deployment` resource for the Gitea application.
  
5. Use the `oc create deployment` command to create the `Gitea` deployment.
  Configure the deployment port `3030` and use the `registry.ocp4.example.com:8443/redhattraining/podman-gitea:latest` container image.
   
   [student@workstation ~]$ **`oc create deployment gitea --port 3030 \   --image=registry.ocp4.example.com:8443/redhattraining/podman-gitea:latest`**
   Warning: would violate PodSecurity "restricted:v1.24" ...
   _...output omitted..._
   deployment.apps/gitea created
   
### Note
 You can ignore pod security warnings for exercises in this course. Red Hat OpenShift uses the Security Context Constraints controller to provide safe defaults for pod security.
 1. Verify the pod status.
  [student@workstation ~]$ **`oc get po`**
  NAME                     READY   STATUS    RESTARTS   AGE
  gitea-6d446c7b48-b89hz   `1/1     Running`   0          75s

  If the pod is in the `ContainerCreating` status, repeat the preceding command after a few seconds.
  The preceding command uses the `po` short name to refer to the `pod` resource. In the preceding command, the names `po`, `pod`, and `pods` are equivalent.

2. Create a `Deployment` resource for a PostgreSQL database called `gitea-postgres`. The Gitea application uses the database to store data.
  
  You can use the completed file in the `~/DO188/solutions/openshift-multipod` directory.
 
 3. Use the `oc create deployment` command to create the `gitea-postgres` database.
   
  Configure the deployment port `5432` and use the `registry.ocp4.example.com:8443/rhel9/postgresql-13:1` container image.
  
  Use the `--dry-run=client` and `-o yaml` options to generate a YAML file, and redirect the output to the `postgres.yaml` file.

 [student@workstation ~]$ oc create deployment gitea-postgres --port 5432 -o yaml \   --image=registry.ocp4.example.com:8443/rhel9/postgresql-13:1 \   --dry-run=client > postgres.yaml
   
 The preceding command uses output redirection (`>`) to create the `postgres.yaml` file.
  
 1. Open the `postgres.yaml` file in an editor, such as `gedit`, and add the following environment variables:
   _...file omitted..._
   containers:
   - image: registry.ocp4.example.com:8443/rhel9/postgresql-13:1
   name: postgresql-13
   ports:
   - containerPort: 5432
   env:
   - name: POSTGRESQL_USER
   - value: gitea 
   - name: POSTGRESQL_PASSWORD
   - value: gitea 
   - name: POSTGRESQL_DATABASE
   - value: gitea

o get more information about the `env` property, execute the `oc explain deployment.spec.template.spec.containers.env` command.
### Warning
The YAML format is white-space sensitive. You must use correct indentation.
In the preceding example, the `env:` object indentation is 8 spaces.
Use the `oc create` command to create the deployment.
       
        [student@workstation ~]$ **`oc create -f postgres.yaml`**
        Warning: would violate PodSecurity "restricted:v1.24"...
        _...output omitted..._
        deployment.apps/gitea-postgres created
        
        If you receive an error, compare your `postgres.yaml` file to the complete `~/DO188/solutions/openshift-multipod/postgres.yaml` file.
        
    4. Verify the pod status.
        
        [student@workstation ~]$ **`oc get po`**
        NAME                              READY   STATUS    RESTARTS   AGE
        gitea-6d446c7b48-rf578            1/1     Running   0          6m18s
        `gitea-postgres-86d47f494d-7rl5g   1/1     Running`   0          18s
        
2. Configure networking for the database.
    
    1. Expose the PostgreSQL database on the `gitea-postgres` hostname.
        
        [student@workstation ~]$ **`oc expose deployment gitea-postgres`**
        service/gitea-postgres exposed
        
    2. Verify that the PostgreSQL service uses the `gitea-postgres` name and the `5432` port.
        
        [student@workstation ~]$ **`oc get svc`**
        NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
        `gitea-postgres`   ClusterIP   172.30.124.174   <none>        `5432/TCP`   50s
        
3. Configure networking for the Gitea application.
    
    1. Expose the application within the RHOCP cluster.
        
        [student@workstation ~]$ **`oc expose deployment gitea`**
        service/gitea exposed
        
    2. Expose the `gitea` service for external access.
        
        [student@workstation ~]$ **`oc expose service gitea`**
        route.route.openshift.io/gitea exposed
        
4. Test your application functionality.
    
    1. Verify the external URL for your application.
        
        [student@workstation ~]$ **`oc get route`**
        NAME    HOST/PORT                                  SERVICES   PORT   WILDCARD
        gitea   `gitea-ocp-multipod.apps.ocp4.example.com`   gitea      3030   None
        
    2. In a web browser, open the `gitea-ocp-multipod.apps.ocp4.example.com` URL and use the following configuration:
        
        - Database type: `PostgreSQL`
            
        - Host: `gitea-postgres:5432`
            
        - Username: `gitea`
            
        - Password: `gitea`
            
        - Database name: `gitea`
            
        - Server Domain: `gitea-ocp-multipod.apps.ocp4.example.com`
            
        - Gitea Base URL: `http://gitea-ocp-multipod.apps.ocp4.example.com`
            
        
        Then, click Install Gitea.
        
        This step is successful if you see the login page.
        
  5. Optionally, click Register to create a user and log in.

## Lab: Container Orchestration with Kubernetes and OpenShift

Debug and deploy a multi-container application to the Red Hat OpenShift Container Platform (RHOCP).

**Outcomes**

You should be able to:

- Verify and correct the configuration of the `Service` and `Deployment` RHOCP objects.
    
- Deploy RHOCP objects.
    

In this exercise, your task is to deploy the `quotes` application to RHOCP.

The `quotes` application uses the `quotes-api` and `quotes-ui` containerized microservices.

Your colleague managed to deploy the first tier of the application, the `quotes-ui` container, to RHOCP. However, the pod crashes and does not respond to requests. Additionally, the colleague faces difficulties when trying to deploy the `quotes-api` container to RHOCP. You, the RHOCP expert in the company, are tasked with helping your colleague.

As the `student` user on the `workstation` machine, use the `lab` command to:

- Create the `ocp-lab` project.
    
- Deploy the `quotes-ui` microservice.
    

[student@workstation ~]$ **`lab start openshift-lab`**

The lab script continuously evaluates the objectives of this lab. Keep the script running in a terminal window and complete the objectives of this lab from a new terminal window.

**Instructions**

1. Log in to the cluster as the `developer` user, and ensure that you use the `ocp-lab` project.
    
    1. Log in to the cluster as the `developer` user.
        
        [student@workstation ~]$ **`oc login -u developer -p developer \ https://api.ocp4.example.com:6443`**
        Login successful.
        
        _...output omitted..._
        
    2. Ensure that you use the `ocp-lab` project.
        
        [student@workstation ~]$ **`oc project ocp-lab`**
        Already on project "ocp-lab" on server "https://api.ocp4.example.com:6443".
        
2. Change into the `~/DO188/labs/openshift-lab/` directory.
    
    This directory contains the `quotes-api` YAML files that your colleague created. Be aware that the YAML files might contain mistakes.
    
    [student@workstation ~]$ **`cd ~/DO188/labs/openshift-lab/`**
    
3. Use the `deployment.yaml` file to deploy the `quotes-api` container in the `ocp-lab` RHOCP project.
    
    1. Try to create the deployment by using the `deployment.yaml` file.
        
        [student@workstation openshift-lab]$ **`oc create -f deployment.yaml`**
        The Deployment "quotes-api" is invalid: spec.template.metadata.labels: Invalid value: map[string]string{"app":"quotes-api"}: `selector` does not match template `labels`
        
        The deployment defines an application pod with the `app=quotes-api` label. However, the `spec.selector.matchLabels` field uses a different label.
        
    2. Open the `deployment.yaml` file in a text editor, such as `gedit`, and modify the `spec.selector.matchLabels` field to use the same label as the `spec.template.metadata.labels` field.
        
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          labels:
            app: quotes-api
          name: quotes-api
        spec:
          replicas: 1
          selector:
            `matchLabels:       app: quotes-api`
          template:
            metadata:
              `labels:         app: quotes-api`
            spec:
              containers:
              - image: registry.ocp4.example.com:8443/redhattraining/podman-quotes-api:openshift
                name: podman-quotes-api
        
    3. Create the deployment by using the `deployment.yaml` file.
        
        [student@workstation openshift-lab]$ **`oc create -f deployment.yaml`**
        Warning: would violate PodSecurity _...output omitted..._
        deployment.apps/quotes-api created
        
        ### Note
        
        You can ignore pod security warnings for exercises in this course. Red Hat OpenShift uses the Security Context Constraints controller to provide safe defaults for pod security.
        
    4. Verify that the `quotes-api` application pod is in the `RUNNING` state.
        
        [student@workstation openshift-lab]$ **`oc get po`**
        NAME                          READY   STATUS             RESTARTS        AGE
        `quotes-api-6c9f758574-nk8kd   1/1     Running`            0               5s
        quotes-ui-d7d457674-mljrb     0/1     CrashLoopBackOff   15 (3m9s ago)   55m
        
        If the application pod is in the `ContainerCreating` state, then execute the previous command again after a few seconds.
        
    
    [Hide Solution](https://rol.redhat.com/rol/app/#)
    
4. Use the `service.yaml` file to configure the `quotes-ui` container networking in the `ocp-lab` project.
    
    Configure the `service.yaml` file to conform to the following requirements:
    
    - The `quotes-ui` container must reach the `quotes-api` container at the `http://quotes-api:8080` URL.
        
    - The `quotes-api` container listens on port `8080` by default.
        
    - Deploy the `quotes-ui` container after the `quotes-api` container becomes available on the `quotes-api` host. The application architect advised you to restart the `quotes-ui` application if it is deployed in the incorrect order.
        
    
    ### Note
    
    If you make a mistake, delete and recreate the `Service` object.
    
    For example, you can use the `oc delete -f service.yaml` command to delete the `Service` object.
    
    1. Open the `service.yaml` file in a text editor, such as `gedit`. Then, configure the service to serve on port `8080`.
        
        _...file omitted..._
        spec:
          ports:
          - `port: 8080`
            protocol: TCP
            targetPort: 3000
          selector:
            app: quotes
        
    2. Configure the service to send requests to port `8080`.
        
        _...file omitted..._
        spec:
          ports:
          - port: 8080
            protocol: TCP
            `targetPort: 8080`
          selector:
            app: quotes
        
    3. Configure the service to send requests to pods with the `quotes-api` label.
        
        _...file omitted..._
        spec:
          ports:
          - port: 8080
            protocol: TCP
            targetPort: 8080
          selector:
            `app: quotes-api`
        
    4. Configure the service to be available on the `quotes-api` hostname.
        
        apiVersion: v1
        kind: Service
        metadata:
          labels:
            app: quotes
          `name: quotes-api`
        _...file omitted..._
        
    5. Create the service by using the `service.yaml` file.
        
        [student@workstation openshift-lab]$ **`oc create -f service.yaml`**
        service/quotes-api created
        
    6. Verify the service configuration.
        
        The endpoint IP address might differ in your output.
        
        [student@workstation openshift-lab]$ **`oc describe service quotes-api`**
        `Name:              quotes-api`
        Namespace:         ocp-lab
        Labels:            app=quotes
        Annotations:       <none>
        `Selector:          app=quotes-api`
        _...output omitted..._
        `Port:              <unset>  8080/TCP TargetPort:        8080/TCP`
        `Endpoints:`         10.8.0.102:`8080`
        _...output omitted..._
        
        If your output differs from the highlighted output of the previous command, return to the previous steps and ensure you configured your service correctly.
        
    7. Verify that the `quotes-ui` container is still failing.
        
        [student@workstation openshift-lab]$ **`oc get po`**
        NAME                          READY   STATUS             RESTARTS        AGE
        quotes-api-6c9f758574-nk8kd   1/1     Running            0               20m
        quotes-ui-d7d457674-mljrb     0/1     `CrashLoopBackOff`   15 (3m9s ago)   55m
        
    8. Restart the `quotes-ui` container.
        
        You can delete containers that contain the `app=quotes-ui` label, and let the `quotes-ui` deployment recreate the container.
        
        [student@workstation openshift-lab]$ **`oc delete pod -l app=quotes-ui`**
        pod "quotes-ui-d7d457674-9cw7l" deleted
        
        Then, verify that the `quotes-ui` deployment created a new container.
        
        [student@workstation openshift-lab]$ **`oc get po`**
        NAME                          READY   STATUS    RESTARTS   AGE
        quotes-api-6c9f758574-nk8kd   1/1     Running   0          39m
        quotes-ui-d7d457674-rbkl7     1/1     Running   0          `67s`
        
    
    [Hide Solution](https://rol.redhat.com/rol/app/#)
    
5. In a web browser, navigate to `http://quotes-ui-ocp-lab.apps.ocp4.example.com` and verify that the application works.