# Chapter 1.  OpenShift Basics

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
   
 6. Verify the pod status.
  [student@workstation ~]$ **`oc get po`**
  NAME                     READY   STATUS    RESTARTS   AGE
  gitea-6d446c7b48-b89hz   `1/1     Running`   0          75s

  If the pod is in the `ContainerCreating` status, repeat the preceding command after a few seconds.
  The preceding command uses the `po` short name to refer to the `pod` resource. In the preceding command, the names `po`, `pod`, and `pods` are equivalent.

7. Create a `Deployment` resource for a PostgreSQL database called `gitea-postgres`. The Gitea application uses the database to store data.
  
  You can use the completed file in the `~/DO188/solutions/openshift-multipod` directory.
 
 8. Use the `oc create deployment` command to create the `gitea-postgres` database.
   
  Configure the deployment port `5432` and use the `registry.ocp4.example.com:8443/rhel9/postgresql-13:1` container image.
  
  Use the `--dry-run=client` and `-o yaml` options to generate a YAML file, and redirect the output to the `postgres.yaml` file.

 [student@workstation ~]$ oc create deployment gitea-postgres --port 5432 -o yaml \   --image=registry.ocp4.example.com:8443/rhel9/postgresql-13:1 \   --dry-run=client > postgres.yaml
   
 The preceding command uses output redirection (`>`) to create the `postgres.yaml` file.
  
 9. Open the `postgres.yaml` file in an editor, such as `gedit`, and add the following environment variables:
```yaml
   _...file omitted...
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
```

To get more information about the `env` property, execute the `oc explain deployment.spec.template.spec.containers.env` command.

**Warning**
The YAML format is white-space sensitive. You must use correct indentation.
In the preceding example, the `env:` object indentation is 8 spaces.

Use the `oc create` command to create the deployment.
       
```
        [student@workstation ~]$ **`oc create -f postgres.yaml`**
        Warning: would violate PodSecurity "restricted:v1.24"...
        _...output omitted..._
        deployment.apps/gitea-postgres created
        
```

 If you receive an error, compare your `postgres.yaml` file to the complete `~/DO188/solutions/openshift-multipod/postgres.yaml` file.

1. Verify the pod status.

```sh
[student@workstation ~]$ **`oc get po`**
NAME                              READY   STATUS    RESTARTS   AGE
gitea-6d446c7b48-rf578            1/1     Running   0          6m18s
gitea-postgres-86d47f494d-7rl5g   1/1     Running   0          18s
```

1. Configure networking for the database.

- Expose the PostgreSQL database on the `gitea-postgres` hostname.
```
[student@workstation ~]$ **`oc expose deployment gitea-postgres`**
service/gitea-postgres exposed
```
   
- Verify that the PostgreSQL service uses the `gitea-postgres` name and the `5432` port.
```sh
[student@workstation ~]$ oc get svc
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
gitea-postgres   ClusterIP   172.30.124.174   <none>        5432/TCP   50s
```

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

### Lab: Container Orchestration with Kubernetes and OpenShift

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
        
```
        _...file omitted..._
        spec:
          ports:
          - `port: 8080`
            protocol: TCP
            targetPort: 3000
          selector:
            app: quotes
```
        
2. Configure the service to send requests to port `8080`.
        
```
        _...file omitted..._
        spec:
          ports:
          - port: 8080
            protocol: TCP
            `targetPort: 8080`
          selector:
            app: quotes
```
        
3. Configure the service to send requests to pods with the `quotes-api` label.
        
```
        _...file omitted..._
        spec:
          ports:
          - port: 8080
            protocol: TCP
            targetPort: 8080
          selector:
            `app: quotes-api`
```
        
4. Configure the service to be available on the `quotes-api` hostname.
        
```
        apiVersion: v1
        kind: Service
        metadata:
          labels:
            app: quotes
          `name: quotes-api`
        _...file omitted..._
```
        
5. Create the service by using the `service.yaml` file.
        
```
        [student@workstation openshift-lab]$ **`oc create -f service.yaml`**
        service/quotes-api created
```
        
6. Verify the service configuration.
  The endpoint IP address might differ in your output.
   
```
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
```
   
 If your output differs from the highlighted output of the previous command, return to the previous steps and ensure you configured your service correctly.

- 7. Verify that the `quotes-ui` container is still failing.
        
```
        [student@workstation openshift-lab]$ **`oc get po`**
        NAME                          READY   STATUS             RESTARTS        AGE
        quotes-api-6c9f758574-nk8kd   1/1     Running            0               20m
        quotes-ui-d7d457674-mljrb     0/1     `CrashLoopBackOff`   15 (3m9s ago)   55m
```
        
- Restart the `quotes-ui` container.
        
You can delete containers that contain the `app=quotes-ui` label, and let the `quotes-ui` deployment recreate the container.
        
```
        [student@workstation openshift-lab]$ **`oc delete pod -l app=quotes-ui`**
        pod "quotes-ui-d7d457674-9cw7l" deleted
```
        
Then, verify that the `quotes-ui` deployment created a new container.
        
```
        [student@workstation openshift-lab]$ **`oc get po`**
        NAME                          READY   STATUS    RESTARTS   AGE
        quotes-api-6c9f758574-nk8kd   1/1     Running   0          39m
        quotes-ui-d7d457674-rbkl7     1/1     Running   0          `67s`
```

- In a web browser, navigate to `http://quotes-ui-ocp-lab.apps.ocp4.example.com` and verify that the application works.


# Chapter 2: Kubernetes and OpenShift Command-line Interfaces and APIs

## Operating a Production Cluster
### Red Hat OpenShift Key Concepts

When you navigate the OpenShift web console, it is useful to know some introductory OpenShift, Kubernetes, and container terminology. The following list includes some basic concepts that can help you to navigate the OpenShift web console.

- Pods: The smallest unit of a Kubernetes-managed containerized application. A pod consists of one or more containers.
- Deployments: The operational unit that provides granular management of a running application.
- Projects: A Kubernetes namespace with additional annotations that provide multitenancy scoping for applications.
- Routes: Networking configuration to expose your applications and services to resources outside the cluster.
- Operators: Packaged Kubernetes applications that extend cluster functions.

These concepts are covered in more detail throughout the course. You can find these concepts throughout the web console as you explore the features of an OpenShift cluster from the graphical environment.

### Accessing the OpenShift Web Console

You access the web console by any modern web browser. The web console URL is generally configurable, and you can discover the address for your cluster web console by using the `oc` command-line interface (CLI). 

From a terminal, you must first authenticate to the cluster via the CLI by using the `oc login -u <USERNAME> -p <PASSWORD> <API_ENDPOINT>:<PORT>` command:

```
[user@host ~]$ oc login -u developer -p developer https://api.ocp4.example.com:6443
Login successful.
```

Then, you execute the `oc whoami --show-console` command to retrieve the web console URL:

```
[user@host ~]$ oc whoami --show-console
https://console-openshift-console.apps.ocp4.example.com
```

Lastly, use a web browser to navigate to the URL, which displays the authentication page.

### Kubernetes Command-line Tool

The `oc` CLI installation also includes an installation of the `kubectl` CLI, which is the recommended method for installing the `kubectl` CLI for OpenShift users.

You can also install the `kubectl` CLI independently of the `oc` CLI. You must use a `kubectl` CLI version that is within one minor version difference of your cluster. For example, a `v1.26` client can communicate with `v1.25`, `v1.26`, and `v1.27` control planes. Using the latest compatible version of the `kubectl` CLI can help to avoid unforeseen issues.

To perform a manual installation of the `kubectl` binary for a Linux installation, you must first download the latest release by using the `curl` command.

```bash
user@host ~]$ curl -LO "https://dl.k8s.io/release/$(curl -L \   -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
```

Then, you must download the `kubectl` checksum file and then validate the `kubectl` binary against the checksum file.

```bash
[user@host ~]$ curl -LO "https://dl.k8s.io/$(curl -L \ -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

[user@host ~]$ echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
kubectl: OK
```

If the check fails, then the `sha256sum` command exits with nonzero status, and prints a `kubectl: FAILED` message.

You can then install the `kubectl` CLI.

```bash
[user@host ~]$ sudo install -o root -g root -m 0755 kubectl \
/usr/local/bin/kubectl
```

>Note
If you do not have `root` access on the target system, you can still install the `kubectl` CLI to the `~/.local/bin` directory. For more information, refer to [https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).

Finally, use the `kubectl version` command to verify the installed version. This command prints the client and server versions. Use the `--client` option to view the client version only.

```bash
[user@host ~]$ kubectl version --client
```

Alternatively, a distribution that is based on Red Hat Enterprise Linux (RHEL) can install the `kubectl` CLI with the following command:

```bash
[user@host ~]$ cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
[user@host ~]$ sudo yum install -y kubectl
```

To view a list of the available `kubectl` commands, use the `kubectl --help` command.

```bash
[user@host ~]$ kubectl --help
kubectl controls the Kubernetes cluster manager.


 Find more information at:
https://kubernetes.io/docs/reference/kubectl/

Basic Commands (Beginner):
  create        Create a resource from a file or from stdin
  expose        Take a replication controller, service, deployment or pod and
expose it as a new Kubernetes Service
  run           Run a particular image on the cluster
  set           Set specific features on objects

Basic Commands (Intermediate):
_...output omitted..._
```

You can also use the `--help` option on any command to view detailed information about the command, including its purpose, examples, available subcommands, and options. For example, the following command provides information about the `kubectl create` command and its usage.

```bash
[user@host ~]$ kubectl create --help
Create a resource from a file or from stdin.

 JSON and YAML formats are accepted.

Examples:
  # Create a pod using the data in pod.json
  kubectl create -f ./pod.json

  # Create a pod based on the JSON passed into stdin
  cat pod.json | kubectl create -f -

  # Edit the data in registry.yaml in JSON then create the resource using the edited data
  kubectl create -f registry.yaml --edit -o json

Available Commands:
  clusterrole              Create a cluster role
  clusterrolebinfing   	   Create a cluster role binding for a particular cluster role
_...output omitted..._
```

Kubernetes uses many resource components to support applications. The `kubectl explain` command provides detailed information about the attributes of a given resource. For example, use the following command to learn more about the attributes of a `pod` resource.

```bash
[user@host ~]$ kubectl explain pod
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.
FIELDS:
   apiVersion	<string>
     APIVersion defines the versioned schema of this representation of an
     object.
_...output omitted..._
```

Refer to [Kubernetes Documentation - Getting Started](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/) for further information about the `kubectl` commands.

### OpenShift Command-line Tool

The main method of interacting with an RHOCP cluster is by using the `oc` command.

You can download the `oc` CLI from the OpenShift web console to ensure that the CLI tools are compatible with the RHOCP cluster. From the OpenShift web console, navigate to Help → Command line tools. The Help menu is represented by a `?` icon. The web console provides several installation options for the `oc` client, such as downloads for the following operating systems:

- x86_64 Windows, Mac, and Linux systems
- ARM 64 Linux and Mac systems
- Linux for IBM Z, IBM Power, and little endian

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/cli/interfaces/assets/tools_screen.png)|

The basic usage of the `oc` command is through its subcommands in the following syntax:

```bash
[user@host ~]$ oc command
```

Because the `oc` CLI is a superset of the `kubectl` CLI, the `version`, `--help`, and `explain` commands are the same for both CLIs. However, the `oc` CLI includes additional commands that are not included in the `kubectl` CLI, such as the `oc login` and `oc new-project` commands.

### Managing Resources at the Command Line

Developers who are familiar with Kubernetes can use the `kubectl` utility to manage a RHOCP cluster. This course uses the `oc` command-line utility, to take advantage of additional RHOCP features. The `oc` commands manage resources that are exclusive to RHOCP, such as projects, deployment configurations, routes, and image streams.

Before you can interact with your RHOCP cluster, you must authenticate your requests. Use the `oc login` command to authenticate your requests. The `oc login` command provides role-based authentication and authorization that protects the RHOCP cluster from unauthorized access. The syntax to log in is shown below:

For example, in this course, you can use the following command:

```bash
[user@host ~]$ oc login https://api.ocp4.example.com:6443
Username: developer
Password: developer
Login successful.

You don't have any projects. You can try to create a new project, by running

  $ oc new-project <projectname>

Welcome to OpenShift! See 'oc help' to get started.

```

After authenticating to the RHOCP cluster, you can create a project with the `oc new-project` command. 

Projects provide isolation between your application resources. Projects are Kubernetes namespaces with additional annotations that provide multitenancy scoping for applications.

```bash
[user@host ~]$ oc new-project myapp
```

Several essential commands can manage RHOCP and Kubernetes resources, as described here. Unless otherwise specified, the following commands are compatible with both the `oc` and `kubectl` CLIs.

Some commands require a user with cluster administrator access. The following list includes several useful `oc` commands for cluster administrators.

#### oc cluster-info

The `cluster-info` command prints the address of the control plane and other cluster services. The `oc cluster-info dump` command expands the output to include helpful details for debugging cluster problems.

```bash
[user@host ~]$ oc cluster-info
Kubernetes control plane is running at https://api.ocp4.example.com:6443
_...output omitted..._
```

#### oc api-versions

The structure of cluster resources has a corresponding API version, which the `oc api-versions` command displays. The command prints the supported API versions on the server, in the form of "group/version".

In the following example, the group is `admissionregistration.k8s.io` and the version is `v1`:

```bash
[user@host ~]$ oc api-versions
admissionregistration.k8s.io/v1
_...output omitted..._
```

#### oc get clusteroperator

The cluster operators that Red Hat ships serve as the architectural foundation for RHOCP. RHOCP installs cluster operators by default. Use the `oc get clusteroperator` command to see a list of the cluster operators:

```bash
[user@host ~]$ oc get clusteroperator
NAME                     VERSION    AVAILABLE PROGRESSING DEGRADED SINCE ...
authentication           4.14.0     True      False       False    18d
baremetal                4.14.0     True      False       False    18d
_...output omitted..._
```

Other useful commands are available to both regular and administrator users:

#### oc get

Use the `get` command to retrieve information about resources in the selected project. Generally, this command shows only the most important characteristics of the resources, and omits more detailed information.

The ``oc get _`RESOURCE_TYPE`_`` command displays a summary of all resources of the specified type.

For example, the following command returns the list of the `pod` resources in the current project:

```bash
[user@host ~]$ oc get pod
NAME                          READY   STATUS    RESTARTS   AGE
quotes-api-6c9f758574-nk8kd   1/1     Running   0          39m
quotes-ui-d7d457674-rbkl7     1/1     Running   0          67s
```

You can use the `oc get RESOURCE_TYPE RESOURCE_NAME` command to export a resource definition. Typical use cases include creating a backup or modifying a definition. The `-o yaml` option prints the object representation in YAML format. You can change to JSON format by providing a `-o json` option.

#### oc get all

Use the `oc get all` command to retrieve a summary of the most important components of a cluster. This command iterates through the major resource types for the current project, and prints a summary of their information:

```bash
[user@host ~]$ oc get all
NAME       DOCKER REPO                              TAGS      UPDATED
is/nginx   172.30.1.1:5000/basic-kubernetes/nginx   latest    About an hour ago

NAME       REVISION   DESIRED   CURRENT   TRIGGERED BY
dc/nginx   1          1         1         config,image(nginx:latest)

NAME         DESIRED   CURRENT   READY     AGE
rc/nginx-1   1         1         1         1h

NAME        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
svc/nginx   172.30.72.75   <none>        80/TCP,443/TCP   1h

NAME               READY     STATUS    RESTARTS   AGE
po/nginx-1-ypp8t   1/1       Running   0          1h
```

#### oc describe

If the summaries from the `get` command are insufficient, then you can use the ``oc describe _`RESOURCE_TYPE`_ _`RESOURCE_NAME`_`` command to retrieve additional information. Unlike the `get` command, you can use the `describe` command to iterate through all the different resources by type. Although most major resources can be described, this function is not available across all resources. The following example demonstrates describing a pod resource:

```bash
[user@host ~]$ oc describe mysql-openshift-1-glgrp
Name:               mysql-openshift-1-glqrp
Namespace:          mysql-openshift
Priority:           0
Node:               cluster-worker-1/172.25.250.52
Start Time:         Fri, 15 Feb 2019 02:14:34 +0000
Labels:             app=mysql-openshift
                    deployment=mysql-openshift-1
_...output omitted..._
Status:             Running
IP:                 10.129.0.85
```

#### oc explain

To learn about the fields of an API resource object, use the `oc explain` command. This command describes the purpose and the fields that are associated with each supported API resource. You can also use this command to print the documentation of a specific field of a resource. Fields are identified via a JSONPath identifier. The following example prints the documentation for the `.spec.containers.resources` field of the `pod` resource type:

```bash
[user@host ~]$ oc explain pods.spec.containers.resources
KIND:     Pod
VERSION:  v1

FIELD: resources <ResourceRequiremnts>

DESCRIPTION:
     Compute Resources required by this container. Cannot be updated. More info:
     https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

     ResourceRequirements describes the compute resource requirements.

FIELDS:
   claims       <[]ResourceClaim>
     Claims lists the names of resources, defined in spec.resourceClaims, that
     are used by this container.

     This is an alpha field nd requires enabling the DynamicResourceAllocation
     feature gate.

     This field is immutable. It can only be set for containers.

   limits	<map[string]Quantity>
     Limits describes the maximum amount of compute resources allowed. More
     info:
     https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

   requests	<map[string]Quantity>
     Requests describes the minimum amount of compute resources required. If
     Requests is omitted for a container, it defaults to Limits if that is
     explicitly specified, otherwise to an implementation-defined value. Requests
    cannot exceed Limits. More info:
     https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

```

Add the `--recursive` flag to display all fields of a resource without descriptions. Information about each field is retrieved from the server in OpenAPI format.

#### oc create

Use the `create` command to create a RHOCP resource in the current project. This command creates resources from a resource definition. Typically, this command is paired with the `oc get RESOURCE_TYPE_ RESOURCE_NAME -o yaml` command for editing definitions. Developers commonly use the `-f` flag to indicate the file that contains the JSON or YAML representation of an RHOCP resource.

For example, to create resources from the `pod.yaml` file, use the following command:

```bash
[user@host ~]$ oc create -f pod.yaml
pod/quotes-pod created
```
RHOCP resources in the YAML format are discussed later.

#### oc status

The `oc status` command provides a high-level overview of the current project. The command shows services, deployments, build configurations, and active deployments. Information about any misconfigured components is also shown. The `--suggest` option shows additional details for any identified issues.

#### oc delete

Use the `delete` command to delete an existing RHOCP resource from the current project. You must specify the resource type and the resource name.

For example, to delete the `quotes-ui` pod, use the following command:

```bash
[user@host ~]$ oc delete pod quotes-ui
pod/quotes-ui deleted
```

A fundamental understanding of the RHOCP architecture is needed here, because deleting managed resources, such as pods, results in the automatic creation of new instances of those resources. When a project is deleted, it deletes all the resources and applications within it.

Each of these commands is executed in the current selected project. To execute commands in a different project, you must include the `--namespace` or `-n` options.

```bash
[user@host ~]$ oc get pods -n openshift-apiserver
NAME                         READY   STATUS    RESTARTS   AGE
apiserver-68c9485699-ndqlc   2/2     Running   2          18d
```

Refer to the references for a complete list of `oc` commands.

### Authentication with OAuth

For users to interact with RHOCP, they must first authenticate to the cluster. The authentication layer identifies the user that is associated with requests to the RHOCP API. After authentication, the authorization layer then uses information about the requesting user to determine whether the request is allowed.

A user in OpenShift is an entity that can make requests to the RHOCP API. An RHOCP `User` object represents an actor that can be granted permissions in the system by adding roles to the user or to the user's groups. Typically, this represents the account of a developer or an administrator.

Several types of users can exist.

Regular users

Most interactive RHOCP users are represented by this user type. An RHOCP `User` object represents a regular user.

System users

Infrastructure uses system users to interact with the API securely. Some system users are automatically created, including the cluster administrator, with access to everything. By default, unauthenticated requests use an anonymous system user.

Service accounts

`ServiceAccount` objects represent service accounts. RHOCP creates service accounts automatically when a project is created. Project administrators can create additional service accounts to define access to the contents of each project.

Each user must authenticate to access a cluster. After authentication, policy determines what the user is authorized to do.

**Note**

```
Authentication and authorization are covered in greater detail in the "DO280: Red Hat OpenShift Administration II: Operating a Production Kubernetes Cluster" course.
```

The RHOCP control plane includes a built-in OAuth server. To authenticate themselves to the API, users obtain OAuth access tokens. Token authentication is the only guaranteed method to work with any OpenShift cluster, because enterprise Single Sign-On (SSO) might replace the login form of the web console.

When a person requests a new OAuth token, the OAuth server uses the configured identity provider to determine the identity of the person who makes the request. The OAuth server then determines the user that the identity maps to; creates an access token for that user; and then returns the token for use.

To retrieve an OAuth token by using the OpenShift web console, navigate to Help → Command line tools. The Help menu is represented by a `?` icon.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/cli/interfaces/assets/select_command_line_tools.png)|

On the `Command Line Tools` page, navigate to Copy login Command. The following page requires you to log in with your OpenShift user credentials. Next, navigate to Display token. Use the command under the `Log in with this token` label to log in to the OpenShift API.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/cli/interfaces/assets/copy_login_command.png)|

Copy the command from the web console and paste it on the command line. The copied command uses the `--token` and `--server` options, similar to the following example.

```bash
[user@host ~]$ oc login --token=_`sha256-BW...rA8` \
--server=https://api.ocp4.example.com:6443
```

### Inspect Kubernetes Resources

#### Kubernetes and OpenShift Resources

Kubernetes uses API resource objects to represent the intended state of everything in the cluster. All administrative tasks require creating, viewing, and changing the API resources. Use the `oc api-resources` command to view the Kubernetes resources.

```bash
[user@host ~]$ oc api-resources
NAME                    SHORTNAMES   APIVERSION   NAMESPACED   KIND
bindings                             v1           true         Binding
componentstatuses       cs           v1           false        ComponentStatus
configmaps              cm           v1           true         ConfigMap
endpoints               ep           v1           true         Endpoints
_...output omitted..._
daemonsets              ds           apps/v1      true         DaemonSet
deployments             deploy       apps/v1      true         Deployment
replicasets             rs           apps/v1      true         ReplicaSet
statefulsets            sts          apps/v1      true         StatefulSet
_...output omitted..._
```

The `SHORTNAME` for a component helps to minimize typing long CLI commands. For example, you can use `oc get cm` instead of `oc get configmaps`.

The `APIVERSION` column divides the objects into API groups. The column uses the `<API-Group>/<API-Version>` format. The `API-Group` object is blank for Kubernetes core resource objects.

Many Kubernetes resources exist within the context of a Kubernetes namespace. Kubernetes namespaces and OpenShift projects are broadly similar. A 1:1 relationship always exists between a namespace and an OpenShift project.

The `KIND` is the formal Kubernetes resource schema type.

The `oc api-resources` command can further filter the output with options that operate on the data.

**Table 2.1. The `api-resources` Command Options**

|Option Example|Description|
|:--|:--|
|`--namespaced=true`|If false, return non-namespaced resources, otherwise return namespaced resources|
|`--api-group apps`|Limit to resources in the specified API group. Use `--api-group=''` to show core resources.|
|`--sort-by name`|If non-empty, sort list of resources using specified field. The field can be either 'name' or 'kind'.|
  
For example, use the following `oc api-resources` command to see all the `namespaced` resources in the `apps` API group, sorted by `name`.

```bash
[user@host ~]$ oc api-resources --namespaced=true --api-group apps --sort-by name
NAME                  SHORTNAMES   APIVERSION   NAMESPACED   KIND
controllerrevisions                apps/v1      true         ControllerRevision
daemonsets            ds           apps/v1      true         DaemonSet
deployments           deploy       apps/v1      true         Deployment
replicasets           rs           apps/v1      true         ReplicaSet
statefulsets          sts          apps/v1      true         StatefulSet
```

Each resource contains fields that identify the resource or that describe the intended configuration of the resource. Use the `oc explain` command to get information about valid fields for an object. For example, execute the `oc explain pod` command to get information about possible pod object fields.

```bash
[user@host ~]$ oc explain pod
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion	<string>
     APIVersion defines the versioned schema of this representation of an
     ...

   kind	<string>
     Kind is a string value representing the REST resource this object
     ...

   metadata	<ObjectMeta>
     Standard object's metadata. More info:
     ...

   spec	<PodSpec>
     Specification of the desired behavior of the pod. More info:
     _...output omitted..._
```

Every Kubernetes resource contains the `kind`, `apiVersion`, `spec`, and `status` fields. However, when you create an object definition, you do not need to provide the `status` field. Instead, Kubernetes generates the `status` field, and it lists information such as runtime status and readiness. The `status` field is useful for troubleshooting an error or for verifying the current state of a resource.

You can use the YAML path to a field and dot-notation to get information about a particular field. For example, the following `oc explain` command shows details for the `pod.spec` fields.

```bash
[user@host ~]$ oc explain pod.spec
KIND:     Pod
VERSION:  v1

FIELD: spec <PodSpec>

DESCRIPTION:
     Specification of the desired behavior of the pod. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

     PodSpec is a description of a pod.

FIELDS:
   activeDeadlineSeconds	<integer>
     Optional duration in seconds the pod may be active on the node relative to
    _...output omitted..._
```

The following Kubernetes main resource types can be created and configured by using a YAML or a JSON manifest file, or by using OpenShift management tools:

##### Pods (`pod`)

Represent a collection of containers that share resources, such as IP addresses and persistent storage volumes. It is the primary unit of work for Kubernetes.

##### Services (`svc`)

Define a single IP/port combination that provides access to a pool of pods. By default, services connect clients to pods in a round-robin fashion.

##### ReplicaSet (`rs`)

Ensure that a specified number of pod replicas are running at any given time.

##### Persistent Volumes (`pv`)

Define storage areas for Kubernetes pods to use.

##### Persistent Volume Claims (`pvc`)

Represent a request for storage by a pod. PVCs link a PV to a pod so that its containers can use the provisioned storage, usually by mounting the storage into the container's file system.

##### ConfigMaps (`cm`) and Secrets

Contain a set of keys and values that other resources can use. ConfigMaps and Secrets centralize configuration values that several resources use. Secrets differ from ConfigMaps in that the values of Secrets are always encoded (not encrypted), and their access is restricted to fewer authorized users.

##### Deployment (`deploy`)

A representation of a set of containers that are included in a pod, and the deployment strategies to use. A `deployment` object contains the configuration to apply to all containers of each pod replica, such as the base image, tags, storage definitions, and the commands to execute when the containers start. Although Kubernetes replicas can be created stand-alone in OpenShift, they are usually created by higher-level resources such as deployment controllers.

Red Hat OpenShift Container Platform (RHOCP) adds the following main resource types to Kubernetes:

##### BuildConfig (`bc`)

Defines a process to execute in the OpenShift project. The OpenShift Source-to-Image (S2I) feature uses a BuildConfig to build a container image from application source code that is stored in a Git repository. A `bc` works together with a `dc` to provide an extensible continuous integration and continuous delivery workflows.

Routes

Represent a DNS hostname that the OpenShift router recognizes as an ingress point for applications and microservices.

#### Structure of Resources

Almost every Kubernetes object includes two nested object fields that govern the object's configuration: the object `spec` and the object `status`. The `spec` object describes the intended state of the resource, and the `status` object describes the current state. You specify the `spec` section of the resource when you create the object. Kubernetes controllers continuously update the `status` of the object throughout the existence of the object. The Kubernetes control plane continuously and actively manages every object's actual state to match the desired state you supplied.

The `status` field uses a collection of `condition` resource objects with the following fields.

**Table 2.2. Condition Resource Fields**

|Field|Example|Description|
|:--|:--|:--|
|Type|ContainersReady|The type of the condition|
|Status|False|The state of the condition|
|Reason|RequirementsNotMet|An optional field to provide extra information|
|Message|2/3 containers are running|An optional textual description for the condition|
|LastTransitionTime|2023-03-07T18:05:28Z|The last time that conditions were changed|

  

For example, in Kubernetes, a `Deployment` object can represent an application that is running on your cluster. When you create a `Deployment` object, you might configure the deployment `spec` object to specify that you want three replicas of the application to be running. Kubernetes reads the deployment `spec` object and starts three instances of your chosen application, and updates the `status` field to match your `spec` object. If any of those instances fails, then Kubernetes responds to the difference between the `spec` and `status` objects by making a correction, in this case to start a replacement instance.

Other common fields provide base information in addition to the `spec` and `status` fields of a Kubernetes object.

**Table 2.3. API Resource Fields**

|Field|Description|
|:--|:--|
|`apiVersion`|Identifier of the object schema version.|
|`kind`|Schema identifier.|
|`metadata.name`|Creates a label with a `name` key that other resources in Kubernetes can use to find it.|
|`metadata.namespace`|The namespace, or the RHOCP project where the resource is.|
|`metadata.labels`|Key-value pairs that can connect identifying metadata with Kubernetes objects.|

  

Resources in Kubernetes consist of multiple objects. These objects define the intended state of a resource. When you create or modify an object, you make a persistent record of the intended state. Kubernetes reads the object and modifies the current state accordingly.

All RHOCP and Kubernetes objects can be represented as a JSON or YAML structure. Consider the following pod object in the YAML format:

apiVersion: v1 ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
kind: Pod  ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
metadata: ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
  name: wildfly  ![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
  namespace: my_app ![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)
  labels:
    name: wildfly  ![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)
spec: ![7](https://rol.redhat.com/rol/static/roc/Common_Content/images/7.svg)
  containers:
    - resources:
        limits:
          cpu: 0.5
      image: quay.io/example/todojee:v1  ![8](https://rol.redhat.com/rol/static/roc/Common_Content/images/8.svg)
      name: wildfly ![9](https://rol.redhat.com/rol/static/roc/Common_Content/images/9.svg)
      ports:
        - containerPort: 8080  ![10](https://rol.redhat.com/rol/static/roc/Common_Content/images/10.svg)
          name: wildfly
      env:  ![11](https://rol.redhat.com/rol/static/roc/Common_Content/images/11.svg)
        - name: MYSQL_DATABASE
          value: items
        - name: MYSQL_USER
          value: user1
        - name: MYSQL_PASSWORD
          value: mypa55
_...object omitted..._
status: ![12](https://rol.redhat.com/rol/static/roc/Common_Content/images/12.svg)
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2023-08-19T12:59:22Z"
    status: "True"
    type: PodScheduled

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_structure_of_resources-CO1-1)|Identifier of the object schema version.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_structure_of_resources-CO1-2)|Schema identifier. In this example, the object conforms to the pod schema.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_structure_of_resources-CO1-3)|Metadata for a given resource, such as annotations, labels, name, and namespace.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_structure_of_resources-CO1-4)|A unique name for a pod in Kubernetes that enables administrators to run commands on it.|
|[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_structure_of_resources-CO1-5)|The namespace, or the RHOCP project that the resource resides in.|
|[![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)](https://rol.redhat.com/rol/app/#_structure_of_resources-CO1-6)|Creates a label with a `name` key that other resources in Kubernetes, usually a service, can use to find it.|
|[![7](https://rol.redhat.com/rol/static/roc/Common_Content/images/7.svg)](https://rol.redhat.com/rol/app/#_structure_of_resources-CO1-7)|Defines the pod object configuration, or the intended state of the resource.|
|[![8](https://rol.redhat.com/rol/static/roc/Common_Content/images/8.svg)](https://rol.redhat.com/rol/app/#_structure_of_resources-CO1-8)|Defines the container image name.|
|[![9](https://rol.redhat.com/rol/static/roc/Common_Content/images/9.svg)](https://rol.redhat.com/rol/app/#_structure_of_resources-CO1-9)|Name of the container inside a pod. Container names are important for `oc` commands when a pod contains multiple containers.|
|[![10](https://rol.redhat.com/rol/static/roc/Common_Content/images/10.svg)](https://rol.redhat.com/rol/app/#_structure_of_resources-CO1-10)|A container-dependent attribute to identify the port that the container uses.|
|[![11](https://rol.redhat.com/rol/static/roc/Common_Content/images/11.svg)](https://rol.redhat.com/rol/app/#_structure_of_resources-CO1-11)|Defines a collection of environment variables.|
|[![12](https://rol.redhat.com/rol/static/roc/Common_Content/images/12.svg)](https://rol.redhat.com/rol/app/#_structure_of_resources-CO1-12)|Current state of the object. Kubernetes provides this field, which lists information such as runtime status, readiness, and container images.|

Labels are key-value pairs that you define in the `.metadata.labels` object path, for example:

kind: Pod
apiVersion: v1
metadata:
  name: example-pod
  labels:
    app: example-pod
    group: developers
_...object omitted..._

The preceding example contains the `app=example-pod` and `group=developers` labels. Developers often use labels to target a set of objects by using the `-l` or the `--selector` option. For example, the following `oc get` command lists pods that contain the `group=developers` label:

[user@host ~]$ **`oc get pod --selector group=developers`**
NAME                          READY   STATUS    RESTARTS   AGE
example-pod-6c9f758574-7fhg   1/1     Running   5          11d

#### Command Outputs

The `kubectl` and `oc` CLI commands provide many output formatting options. By default, many commands display a small subset of the most useful fields for the given resource type in a tabular output. Many commands support a `-o wide` option that shows additional fields.

**Table 2.4. Tabular Fields**

|`oc get pods`|`oc get pods -o wide`|Example value|
|:--|:--|:--|
|NAME|NAME|example-pod|
|READY|READY|1/1|
|STATUS|STATUS|Running|
|RESTARTS|RESTARTS|5|
|AGE|AGE|11d|
||IP|10.8.0.60|
||NODE|master01|
||NOMINATED NODE|<none>|
||READINESS GATES|<none>|

  

To view all the fields that are associated with a resource, the `describe` subcommand shows a detailed description of the selected resource and related resources. You can select a single object by name, or all objects of that type, or provide a name prefix, or a label selector.

For example, the following command first looks for an exact match on the `TYPE` object and the `NAME-PREFIX` object. If no such resource exists, then the command outputs details for every resource of that type with a name with a `NAME_PREFIX` prefix.

```
[user@host ~]$ **``oc describe _`TYPE`_ _`NAME-PREFIX`_
```

The `describe` subcommand provides detailed human-readable output. However, the format of the `describe` output might change between versions, and thus is not recommended for script development. Any scripts that rely on the output of the `describe` subcommand might break after a version update.

Kubernetes provides YAML and JSON-formatted output options that are suitable for parsing or scripting.

##### YAML Output

The `-o yaml` option provides a YAML-formatted output that is parsable and still human-readable.

```
[user@host ~]$ **`oc get pods -o yaml`**
apiVersion: v1
items:
- apiVersion: v1
  kind: Pod
  metadata:
    annotations:
_...object omitted..._
```

**Note**

The reference documentation provides a more detailed introduction to YAML.

You can use any tool that can process YAML documents to filter the YAML output for your chosen field. For example, you can use the `yq` tool at [https://mikefarah.gitbook.io/yq/](https://mikefarah.gitbook.io/yq/) to process YAML and JSON files.

The `yq` processor uses a dot notation to separate field names in a query. The following example pipes the YAML output to the `yq` command to parse the `podIP` field.

```
[user@host ~]$ **`oc get pods -o yaml | yq r - 'items[0].status.podIP'`**
10.8.0.60
```

The `[0]` in the example specifies the first index in the items array.

**Note**

The lab environment includes version 3.3.0 of the `yq` command, which these examples use. Later versions of the `yq` command introduce incompatibilities with earlier versions. The content in this course might not work with other versions.

**Note**

Another tool named `yq` is at [https://kislyuk.github.io/yq/](https://kislyuk.github.io/yq/). The two `yq` tools are not compatible; commands that are designed for one of them do not work with the other.

##### JSON Output

Kubernetes uses the JSON format internally to process resource objects. Use the `-o json` option to view a resource in the JSON format.

```
[user@host ~]$ **`oc get pods -o json`**
{
    "apiVersion": "v1",
    "items": [
        {
            "apiVersion": "v1",
            "kind": "Pod",
            "metadata": {
                "annotations": {
```

You can use other tools to process JSON documents, such as the `jq` tool at [https://jqlang.github.io/jq/](https://jqlang.github.io/jq/). Similar to the `yq` processor, use the `jq` processor and dot notation on the fields to query specific information from the JSON-formatted output.

```
[user@host ~]$ **`oc get pods -o json | jq '.items[0].status.podIP'`**
"10.8.0.60"
```

Alternatively, the example might have used `.items[].status.podIP` for the query string. The empty brackets instruct the `jq` tool to query all items.

##### Custom Output

Kubernetes provides a custom output format that combines the convenience of extracting data via `jq` styled queries with a tabular output format. Use the `-o custom-columns` option with comma-separated `column name : jq query string` pairs.

```bash
[user@host ~]$ oc get pods \
-o custom-columns=PodName:".metadata.name",\
ContainerName:"spec.containers[].name",\ 
Phase:"status.phase",\
IP:"status.podIP",\
Ports:"spec.containers[].ports[].containerPort"
PodName                  ContainerName   Phase     IP          Ports
myapp-77fb5cd997-xplhz   myapp           Running   10.8.0.60   <none>
```

Kubernetes also supports the use of JSONPath expressions. JSONPath is a query language for JSON. JSONPath expressions refer to a JSON data structure; they filter and extract formatted fields from JSON objects.

In the following example, the JSONPath expression uses the `range` operator to iterate over the list of pods to extract the name of the pod, its IP address, and the assigned ports.

```bash
[user@host ~]$ oc get pods  \
-o jsonpath='{range .items[]}{"Pod Name: "}{.metadata.name}
{"IP: "}{.status.podIP}
{"Ports: "}{.spec.containers[].ports[].containerPort}{"\n"}{end}'
Pod Name: myapp-77fb5cd997-xplhz
IP: 10.8.0.60
Ports:
```

You can customize the format of the output with Go templates, which the Go programming language uses. Use the `-o go-template` option followed by a Go template, where Go expressions are inside double braces, `{{` `}}`.

```bash
[user@host ~]$ oc get pods \
-o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'
myapp-77fb5cd997-xplhz
```

### Assess the Health of an OpenShift Cluster

#### Query Operator Conditions

Operators are important components of Red Hat OpenShift Container Platform (RHOCP). 

Operators automate the required tasks to maintain a healthy RHOCP cluster that would otherwise require human intervention. Operators are the preferred method of packaging, deploying, and managing services on the control plane.

Operators integrate with Kubernetes APIs and CLI tools such as `kubectl` and `oc` commands. Operators provide the means of monitoring applications, performing health checks, managing over-the-air (OTA) updates, and ensuring that applications remain in your specified state.

Because CRI-O and the Kubelet run on every node, almost every other cluster function can be managed on the control plane by using Operators. Components that are added to the control plane by using operators include critical networking and credential services.

Operators in RHOCP are managed by two different systems, depending on the purpose of the operator.

**Cluster Version Operator (CVO)***

Cluster operators perform cluster functions. These operators are installed by default, and the CVO manages them.

Cluster operators use a Kubernetes `kind` value of `clusteroperators`, and thus can be queried via `oc` or `kubectl` commands. As a user with the `cluster-admin` role, use the `oc get clusteroperators` command to list all the cluster operators.

```bash
[user@host ~]$ oc get clusteroperators
NAME                        VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication              4.14.0    True        False         False      3d1h
baremetal                   4.14.0    True        False         False      38d
cloud-controller-manager    4.14.0    True        False         False      38d
cloud-credential            4.14.0    True        False         False      38d
cluster-autoscaler          4.14.0    True        False         False      38d
config-operator             4.14.0    True        False         False      38d
console                     4.14.0    True        False         False      38d
_...output omitted..._
```

For more details about a cluster operator, use the ``describe clusteroperators _`operator-name`_`` command to view the field values that are associated with the operator, including the current status of the operator. The `describe` command provides a human-readable output format for a resource. As such, the output format might change with an RHOCP version update.

For an output format that is less likely to change with a version update, use one of the `-o` output options of the `get` command. For example, use the following `oc get clusteroperators` command for the YAML-formatted output details for the `dns` operator.

```bash
[user@host ~]$ oc get clusteroperators dns -o yaml
apiVersion: config.openshift.io/v1
kind: ClusterOperator
metadata:
  annotations:
_...output omitted..._
status:
  conditions:
  - lastTransitionTime: "2023-03-20T13:55:21Z"
    message: DNS "default" is available.
    reason: AsExpected
    status: "True"
    type: Available
_...output omitted..._
  relatedObjects:
  - group: ""
    name: openshift-dns-operator
    resource: namespaces
_...output omitted..._
  versions:
  - name: operator
    version: 4.14.0
_...output omitted..._
```

**Operator Lifecycle Manager (OLM) Operators

Optional add-on operators that the OLM manages can be made accessible for users to run in their applications.

As a user with the `cluster-admin` role, use the `get operators` command to list all the add-on operators.

```bash
[user@host~]$ oc get operators
NAME                                 AGE
lvms-operator.openshift-storage      34d
metallb-operator.metallb-system      34d
```

You can likewise use the `describe` and `get` commands to query details about the fields that are associated with the add-on operators.

Operators use one or more pods to provide cluster services. You can find the namespaces for these pods under the `relatedObjects` section of the detailed output for the operator. As a user with a `cluster-admin` role, use the `-n namespace` option on the `get pod` command to view the pods. For example, use the following `get pods` command to retrieve the list of pods in the `openshift-dns-operator` namespace.

```bash
[user@host~]$ oc get pods -n openshift-dns-operator
NAME                            READY   STATUS    RESTARTS   AGE
dns-operator-64688bfdd4-8zklh   2/2     Running   38         38d
```

Use the `-o yaml` or `-o json` output formats to view or analyze more details about the pods. The resource conditions, which are found in the status for the resource, track the current state of the resource object. The following example uses the `jq` processor to extract the `status` values from the JSON output details for the `dns` pod.

```bash
[user@host~]$ oc get pod -n openshift-dns-operator \
dns-operator-64688bfdd4-8zklh -o json | jq .status
{
  "conditions": [
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2023-02-09T21:24:50Z",
      "status": "True",
      "type": "Initialized"
    },
_...output omitted..._
```

In addition to listing the pods of a namespace, you can also use the `--show-labels` option of the `get` command to print the labels used by the pods. The following example retrieves the pods and their labels in the `openshift-etcd` namespace.

```bash
[user@host~]$ oc get pods -n openshift-etcd --show-labels
NAME                   READY   STATUS      RESTARTS   AGE   LABELS
etcd-master01          4/4     Running     68         35d   app=etcd,etcd=true,k8s-app=etcd,revision=3
installer-3-master01   0/1     Completed   0          35d   app=installer
```

#### Examining Cluster Metrics

Another way to gauge the health of an RHOCP cluster is to examine the compute resource usage of cluster nodes and pods. The `oc adm top` command provides this information. For example, to list the total memory and CPU usage of all pods in the cluster, you can use the `--sum` option with the command to print the sum of the resource usage.

```bash
[user@host~]$ oc adm top pods -A --sum
NAMESPACE                 NAME                      CPU(cores)   MEMORY(bytes)
metallb-system            controller-...-ddr8v      0m           57Mi
metallb-system            metallb-...-n2zsv         0m           48Mi
_...output omitted..._
openshift-storage         topolvm-node-9spzf        0m           68Mi
openshift-storage         vg-manager-z8g5k          0m           23Mi
                                                   ------       --------
                                                    428m         10933Mi
```

The `-A` option shows pods from all namespaces. Use the `-n namespace` option to filter the results to show the pods in a single namespace. Use the `--containers` option to display the resource usage of containers within a pod. For example, use the following command to list the resource usage of the containers in the `etcd-master01` pod in the `openshift-etcd` namespace.

```bash
[user@host~]$ oc adm top pods etcd-master01 -n openshift-etcd --containers
POD             NAME           CPU(cores)   MEMORY(bytes)
etcd-master01   POD            0m           0Mi
etcd-master01   etcd           71m          933Mi
etcd-master01   etcd-metrics   6m           32Mi
etcd-master01   etcd-readyz    4m           66Mi
etcd-master01   etcdctl        0m           0Mi
```

##### Viewing Cluster Metrics

The OpenShift web console incorporates graphs to visualize cluster and resource analytics. Cluster administrators and users with either the `view` or the `cluster-monitoring-view` cluster role can access the Home → Overview page. The Overview page displays a collection of cluster-wide metrics, and provides a high-level view of the overall health of the cluster.

The Overview page displays the following metrics:

- Current cluster capacity based on CPU, memory, storage, and network usage
- A time-series graph of total CPU, memory, and disk usage
- The ability to display the top consumers of CPU, memory, and storage

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/cli/health/assets/console-metrics-dash-utilization.png)|

For any of the listed resources in the Cluster Utilization section, administrators can click the link for current resource usage. The link displays a window with a breakdown of top consumers for that resource. Top consumers can be sorted by project, by pod, or by node. The list of top consumers can be useful for identifying problematic pods or nodes. For example, a pod with an unexpected memory leak might appear at the top of the list.

##### Viewing Project Metrics

The Project Details page displays metrics that provide an overview of the resources that are used within the scope of a specific project. The Utilization section displays usage information about resources, such as CPU and memory, along with the ability to display the top consumers for each resource.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/cli/health/assets/console-metrics-project-details.png)|

All metrics are pulled from Prometheus. Click any graph to navigate to the Metrics page. View the executed query, and inspect the data further.

If a resource quota is created for the project, then the current project request and limits appear on the Project Details page.

##### Viewing Resource Metrics

When troubleshooting, it is often useful to view metrics at a smaller granularity than for the entire cluster or project. The Pod Details page displays time-series graphs of the CPU, memory, and file system usage for a specific pod. A sudden change in these critical metrics, such as a CPU spike caused by high load, is visible on this page.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/cli/health/assets/console-metrics-pod-details.png)|

Figure 2.9: Time-series graphs showing various metrics for a pod

##### Performing Prometheus Queries in the Web Console

The Prometheus UI is a feature-rich tool for visualizing metrics and configuring alerts. The OpenShift web console provides an interface for executing Prometheus queries directly from the web console.

To perform a query, navigate to Observe → Metrics, enter a Prometheus Query Language expression in the text field, and click Run Queries. The results of the query are displayed as a time-series graph:

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/cli/health/assets/console-metrics-prometheus.png)|

Figure 2.10: Using a Prometheus query to display a time-series graph

>Note
The Prometheus Query Language is not discussed in detail in this course. Refer to the references section for a link to the official documentation.

#### Query Cluster Events and Alerts

Some developers consider OpenShift logs to be too low-level, thus making troubleshooting difficult. Fortunately, RHOCP provides a high-level logging and auditing facility called _events_. Kubernetes generates `event` objects in response to state changes in cluster objects, such as nodes, pods, and containers. Events signal significant actions, such as starting a container or destroying a pod.

To read events, use the `get events` command. The command lists the events for the current RHOCP project (namespace). You can display the events for a different project by adding the ``-n _`namespace`_`` option to the command. To list the events for all the projects, use the `-A` (or `--all-namespaces`) option.

>Note
To sort the events by time, add the `--sort-by .metadata.creationTimestamp` option to the `oc get events` command.

The following `get events` command prints events in the `openshift-kube-controller-manager` namespace.

```bash
[user@host~]$ oc get events -n openshift-kube-controller-manager
LAST SEEN  TYPE    REASON                 OBJECT                                MESSAGE
12m        Normal  CreatedSCCRanges       pod/kube-controller-manager-master01  created SCC...
```

You can use the ``describe pod _`pod-name`_`` command to further narrow the results to a single pod. For example, to retrieve only the events that relate to a `mysql` pod, you can refer to the `Events` field from the output of the `oc describe pod mysql` command:

```bash
[user@host~]$ oc describe pod mysql
_...output omitted..._
Events:
  FirstSeen   LastSeen    Count From         Reason          Message
  Wed, 10 ... Wed, 10 ... 1     {scheduler } scheduled       Successfully as...
_...output omitted..._
```

##### Kubernetes Alerts

RHOCP includes a monitoring stack, which is based on the Prometheus open source project. The monitoring stack is configured to monitor the core RHOCP cluster components, by default. You can optionally configure the monitoring stack also to monitor user projects.

The components of the monitoring stack are installed in the `openshift-monitoring` namespace. Use the following `get all` command to display a list of all resources, their status, and their types in the `openshift-monitoring` namespace.

```bash
[user@host~]$ oc get all -n openshift-monitoring --show-kind
NAME                                              READY  STATUS   RESTARTS AGE
pod/`alertmanager-main-0`                           6/6    Running  85       34d
pod/cluster-monitoring-operator-56b769b58f-dtmqj  2/2    Running  34       35d
pod/kube-state-metrics-75455b796c-8q28d           3/3    Running  51       35d
pod/`prometheus-k8s-0`                              6/6    Running  30       35d
_...output omitted..._
```

The Prometheus Operator in the `openshift-monitoring` namespace creates, configures, and manages platform Prometheus and Alertmanager instances. `prometheus-k8s-0` is the Prometheus pod. A cluster administrator can use the following command to get the alerts from the Prometheus API.

```bash
[user@host~]$ oc -n openshift-monitoring exec -c prometheus \ prometheus-k8s-0 -- curl -s   'http://localhost:9090/api/v1/alerts' | jq
_...output omitted..._
```

An Alertmanager pod in the `openshift-monitoring` namespace receives alerts from Prometheus. Alertmanager uses alert receivers to send alerts to external notification systems. The `alertmanager-main-0` pod is the Alertmanager for the cluster. Use the `curl` command to retrieve the fired alerts from the Alertmanager API.

[user@host~]$ **`oc -n openshift-monitoring exec -c alertmanager \ alertmanager-main-0 -- curl -s  'http://localhost:9093/api/v1/alerts' | jq`**
_...output omitted..._

#### Check Node Status

RHOCP clusters can have several components, including at least one control plane and at least one compute node. The two components can occupy a single node. The following `oc` command, or the matching `kubectl` command, can display the overall health of all cluster nodes.

```bash
[user@host~]$ oc cluster-info
```

The `oc cluster-info` output is high-level, and can verify that the cluster nodes are running. For a more detailed view into the cluster nodes, use the `get nodes` command.

```bash
[user@host~]$ oc get nodes
NAME       STATUS   ROLES                         AGE   VERSION
master01   Ready    control-plane,master,worker   35d   v1.27.6+f67aeb3
```

The example shows a single `master01` node with multiple roles. The `STATUS` value of `Ready` means that this node is healthy and can accept new pods. A `STATUS` value of `NotReady` means that a condition triggered the `NotReady` status and the node is not accepting new pods.

As with any other RHOCP resource, you can drill down into further details of the node resource with the ``describe node _`node-name`_`` command. For parsable output of the same information, use the `-o json` or the `-o yaml` output options with the ``get node _`node-name`_`` command. For more information about using and parsing these output formats, see [the section called “ Inspect Kubernetes Resources ”](https://rol.redhat.com/rol/app/courses/do180-4.14/pages/ch02s03).

The output of the ``get nodes _`node-name`_`` command with the `-o json` or `-o yaml` option is long. The following examples use the `-jsonpath` option or the `jq` processor to parse the ``get node _`node-name`_`` command output.

```bash
[user@host~]$ oc get node master01 -o jsonpath=\ 
{"Allocatable:\n"}{.status.allocatable}{"\n\n"}
{"Capacity:\n"}{.status.capacity}{"\n"}'
Allocatable:
{"cpu":"7500m","ephemeral-storage":"114396791822","hugepages-1Gi":"0",
"hugepages-2Mi":"0","memory":"19380692Ki","pods":"250"}

Capacity:
{"cpu":"8","ephemeral-storage":"125293548Ki","hugepages-1Gi":"0",
"hugepages-2Mi":"0","memory":"20531668Ki","pods":"250"}
```

The JSONPath expression in the previous command extracts the allocatable and capacity measures for the `master01` node. These measures help to understand the available resources on a node.

View the `status` object of a node to understand the current health of the node.

```bash
[user@host~]$ oc get node master01 -o json | jq '.status.conditions'
[
  {
    "lastHeartbeatTime": "2023-03-22T16:34:57Z",
    "lastTransitionTime": "2023-02-23T20:35:15Z",
    "message": "kubelet has sufficient memory available",
    "reason": "KubeletHasSufficientMemory",
    "status": "False",
    "type": "MemoryPressure"   # ![1]
  },
  {
    "lastHeartbeatTime": "2023-03-22T16:34:57Z",
    "lastTransitionTime": "2023-02-23T20:35:15Z",
    "message": "kubelet has no disk pressure",
    "reason": "KubeletHasNoDiskPressure",
    "status": "False",
    "type": "DiskPressure"   #  ![2]
  },
  {
    "lastHeartbeatTime": "2023-03-22T16:34:57Z",
    "lastTransitionTime": "2023-02-23T20:35:15Z",
    "message": "kubelet has sufficient PID available",
    "reason": "KubeletHasSufficientPID",
    "status": "False",
    "type": "PIDPressure"   #  ![3]
  },
  {
    "lastHeartbeatTime": "2023-03-22T16:34:57Z",
    "lastTransitionTime": "2023-02-23T20:35:15Z",
    "message": "kubelet is posting ready status",
    "reason": "KubeletReady",
    "status": "True",
    "type": "Ready"   #  ![4]
  }
]
```

|     |                                                                                                          |
| --- | -------------------------------------------------------------------------------------------------------- |
| 1   | If the status of the `MemoryPressure` condition is true, then the node is low on memory.                 |
| 2   | If the status of the `DiskPressure` condition is true, then the disk capacity of the node is low.        |
| 3   | If the status of the `PIDPressure` condition is true, then too many processes are running on the node.   |
| 4   | If the status of the `Ready` condition is false, then the node is not healthy and is not accepting pods. |

More conditions indicate other potential problems with a node.

**Table 2.5. Possible Node Conditions**

|Condition|Description|
|:--|:--|
|`OutOfDisk`|If true, then the node has insufficient free space on the node for adding new pods.|
|`NetworkUnavailable`|If true, then the network for the node is not correctly configured.|
|`NotReady`|If true, then one of the underlying components, such as the container runtime or network, is experiencing issues or is not yet configured.|
|`SchedulingDisabled`|Pods cannot be scheduled for placement on the node.|

  

To gain deeper insight into a given node, you can view the logs of processes that run on the node. A cluster administrator can use the `oc adm node-logs` command to view node logs. Node logs might contain sensitive output, and thus are limited to privileged node administrators. Use ``oc adm node-logs _`node-name`_`` to filter the logs to a single node.

The `oc adm node-logs` command has other options to further filter the results.

**Table 2.6. Filters for `oc adm node-logs`**

|Option Example|Description|
|:--|:--|
|`--role master`|Use the `--role` option to filter the output to nodes with a specified role.|
|`-u kubelet`|The `-u` option filters the output to a specified unit.|
|`--path=cron`|The `--path` option filters the output to a specific process under the `/var/logs` directory.|
|`--tail 1`|Use ``--tail _`x`_`` to limit output to the last `x` log entries.|

  

Use `oc adm node-logs --help` for a complete list of command options.

For example, to retrieve the most recent log entry for the `crio` service on the `master01` node, you can use the following command.

```bash
[user@host~]$ oc adm node-logs master01 -u crio --tail 1
-- Logs begin at Thu 2023-02-09 21:19:09 UTC, end at Fri 2023-03-17 15:11:43 UTC. --
Mar 17 06:16:09.519642 master01 crio[2987]: time="2023-03-17 06:16:09.519474755Z" level=info msg="Image status:
&ImageStatusResponse{Image:&Image{Id:6ef8...79ce,RepoTags:[],RepoDigests:
_...output omitted..._
```

When you create a pod with the CLI, the `oc` or `kubectl` command is sent to the `apiserver` service, which then validates the command. The `scheduler` service reads the YAML or JSON pod definition, and then assigns pods to compute nodes. Each compute node runs a `kubelet` service that converts the pod manifest to one or more containers in the CRI-O container runtime.

Each compute node must have an active `kubelet` service and an active `crio` service. To verify the health of these services, first start a debug session on the node by using the `debug` command.

```bash
[user@host~]$ oc debug node/_`node-name`_`
```
Replace the ``_`node-name`_`` value with the name of your node.

>Note
The `debug` command is covered in greater detail in a later section.

Within the debug session, change to the `/host` root directory so that you can run binaries in the host's executable path.

```bash
sh-4.4# chroot /host
```

Then, use the `systemctl is-active` calls to confirm that the services are active.

```bash
sh-4.4# for SERVICES in kubelet crio; do echo ---- $SERVICES ---- ; systemctl is-active $SERVICES ;  echo ""; done
---- kubelet ----
active

---- crio ----
active
```

For more details about the status of a service, use the `systemctl status` command.

```bash
sh-4.4# systemctl status kubelet
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─01-kubens.conf, 10-mco-default-madv.conf, 20-logging.conf, 20-nodenet.conf
   Active: active (running) since Thu 2023-03-23 14:39:11 UTC; 1h 26min ago
 Main PID: 3215 (kubelet)
    Tasks: 28 (limit: 127707)
   Memory: 391.7M
      CPU: 14min 34.568s
_...output omitted..._
```

##### Check Pod Status

With RHOCP, you can view logs in running containers and pods to ease troubleshooting. When a container starts, RHOCP redirects the container's standard output and standard error to a disk in the container's ephemeral storage. With this redirect, you can view the container logs by using `logs` commands, even after the container stops. However, the pod hosting the container must still exist.

In RHOCP, the following command returns the output for a container within a pod:

```
[user@host~]$ **``oc logs _`pod-name`_ -c _`container-name`_
```

Replace ``_`pod-name`_`` with the name of the target pod, and replace ``_`container-name`_`` with the name of the target container. The ``-c _`container-name`_`` argument is optional, if the pod has only one container. You must use the ``-c _`container-name`_`` argument to connect to a specific container in a multicontainer pod. Otherwise, the command defaults to the only running container and returns the output.

When debugging images and setup problems, it is useful to get an exact copy of a running pod configuration, and then troubleshoot it with a shell. If a pod is failing or does not include a shell, then the `rsh` and `exec` commands might not work. To resolve this issue, the `debug` command creates a copy of the specified pod and starts a shell in that pod.

By default, the `debug` command starts a shell inside the first container of the referenced pod. The debug pod is a copy of your source pod, with some additional modifications. For example, the pod labels are removed. The executed command is also changed to the '/bin/sh' command for Linux containers, or the 'cmd.exe' executable for Windows containers. Additionally, readiness and liveness probes are disabled.

A common problem for containers in pods is security policies that prohibit a container from running as a root user. You can use the `debug` command to test running a pod as a non-root user by using the `--as-user` option. You can also run a non-root pod as the root user with the `--as-root` option.

With the `debug` command, you can invoke other types of objects besides pods. For example, you can use any controller resource that creates a pod, such as a deployment, a build, or a job. The `debug` command also works with nodes, and with resources that can create pods, such as image stream tags. You can also use the `--image=IMAGE` option of the `debug` command to start a shell session by using a specified image.

If you do not include a resource type and name, then the `debug` command starts a shell session into a pod by using the OpenShift tools image.

[user@host~]$ **`oc debug`**

The next example tests running a job pod as a non-root user.

[user@host~]$ **`oc debug job/test --as-user=1000000`**

The following example creates a debug session for a node.

```bash
[user@host~]$ oc debug node/master01
Starting pod/master01-debug-wtn9r ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.50.10
If you don't see a command prompt, try pressing enter.
sh-4.4# chroot /host
sh-5.1#
```

The debug pod is deleted when the remote command completes, or when the user interrupts the shell.

#### Collect Information for Support Requests

When opening a support case, it is helpful to provide debugging information about your cluster to Red Hat Support. It is recommended that you provide the following information:

- Data gathered by using the `oc adm must-gather` command as a cluster administrator
    
- The unique cluster ID
    

The `oc adm must-gather` command collects resource definitions and service logs from your cluster that are most likely needed for debugging issues. This command creates a pod in a temporary namespace on your cluster, and the pod then gathers and downloads debugging information. By default, the `oc adm must-gather` command uses the default plug-in image, and writes into the `./must-gather.local.` directory on your local system. To write to a specific local directory, you can also use the `--dest-dir` option, such as in the following example:

[user@host~]$ **`oc adm must-gather --dest-dir /home/student/must-gather`**

Then, create a compressed archive file from the `must-gather` directory. For example, on a Linux-based system, you can run the following command:

[user@host~]$ **`tar cvaf mustgather.tar must-gather/`**

Replace `must-gather/` with the actual directory path.

Then, attach the compressed archive file to your support case in the Red Hat Customer Portal.

Similar to the `oc adm must-gather` command, the `oc adm inspect` command gathers information on a specified resource. For example, the following command collects debugging data for the `openshift-apiserver` and `kube-apiserver` cluster operators.

[user@host~]$ **`oc adm inspect clusteroperator/openshift-apiserver \ clusteroperator/kube-apiserver`**

The `oc adm inspect` command can also use the `--dest-dir` option to specify a local directory to write the gathered information. The command shows all logs by default. Use the `--since` option to filter the results to logs that are later than a relative duration, such as 5s, 2m, or 3h.

[user@host~]$ **`oc adm inspect clusteroperator/openshift-apiserver --since 10m`**


# Chapter 3: Run Applications as Containers and Pods

## Create Linux Containers and Kubernetes Pods

### Objectives

- Run containers inside pods and identify the host OS processes and namespaces that the containers use.
    

### Creating Containers and Pods

Kubernetes and OpenShift offer many ways to create containers in pods. You can use one such way, the `run` command, with the `kubectl` or `oc` CLI to create and deploy an application in a pod from a container image. A _container image_ contains immutable data that defines an application and its libraries.

**Note

Container images are discussed in more detail elsewhere in the course.

The `run` command uses the following syntax:

**``oc run _`RESOURCE/NAME`_ --image _`IMAGE`_ [options]``**

For example, the following command deploys an Apache HTTPD application in a pod named `web-server` that uses the `registry.access.redhat.com/ubi8/httpd-24` container image.

[user@host ~]$ **`kubectl run web-server --image registry.access.redhat.com/ubi8/httpd-24`**

You can use several options and flags with the `run` command. The `--command` option executes a custom command and its arguments in a container, rather than the default command that is defined in the container image. You must follow the `--command` option with a double dash (`--`) to separate the custom command and its arguments from the `run` command options. The following syntax is used with the `--command` option:

**``oc run _`RESOURCE/NAME`_ --image _`IMAGE`_ --command -- _`cmd`_ _`arg1`_ ... _`argN`_``**

You can also use the double dash option to provide custom arguments to a default command in the container image.

**``kubectl run _`RESOURCE/NAME`_ --image _`IMAGE`_ -- _`arg1`_ _`arg2`_ ... _`argN`_``**

To start an interactive session with a container in a pod, include the `-it` options before the pod name. The `-i` option tells Kubernetes to keep open the standard input (`stdin`) on the container in the pod. The `-t` option tells Kubernetes to open a TTY session for the container in the pod. You can use the `-it` options to start an interactive, remote shell in a container. From the remote shell, you can then execute additional commands in the container.

The following example starts an interactive remote shell, `/bin/bash`, in the default container in the `my-app` pod.

[user@host ~]$ **`oc run -it my-app --image registry.access.redhat.com/ubi9/ubi \ --command -- /bin/bash`**
If you don't see a command prompt, try pressing enter.
bash-5.1$

**Note

Unless you include the `--namespace` or `-n` options, the `run` command creates containers in pods in the current selected project.

You can also define a restart policy for containers in a pod by including the `--restart` option. A pod restart policy determines how the cluster should respond when containers in that pod exit. The `--restart` option has the following accepted values: `Always`, `OnFailure`, and `Never`.

`Always`

If the restart policy is set to `Always`, then the cluster continuously tries to restart a successfully exited container, for up to five minutes. The default pod restart policy is `Always`. If the `--restart` option is omitted, then the pod is configured with the `Always` policy.

`OnFailure`

Setting the pod restart policy to `OnFailure` tells the cluster to restart only failed containers in the pod, for up to five minutes.

`Never`

If the restart policy is set to `Never`, then the cluster does not try to restart exited or failed containers in a pod. Instead, the pods immediately fail and exit.

The following example command executes the `date` command in the container of the pod named `my-app`, redirects the `date` command output to the terminal, and defines `Never` as the pod restart policy.

[user@host ~]$ **`oc run -it my-app \ --image registry.access.redhat.com/ubi9/ubi \ --restart Never --command -- date`**
Mon Feb 20 22:36:55 UTC 2023

To automatically delete a pod after it exits, include the `--rm` option with the `run` command.

[user@host ~]$ **`kubectl run -it my-app --rm \ --image registry.access.redhat.com/ubi9/ubi \ --restart Never --command -- date`**
Mon Feb 20 22:38:50 UTC 2023
pod "date" deleted

For some containerized applications, you might need to specify environment variables for the application to work. To specify an environment variable and its value, include the `--env=` option with the `run` command.

[user@host ~]$ **`oc run mysql \ --image registry.redhat.io/rhel9/mysql-80 \ --env MYSQL_ROOT_PASSWORD=myP@$$123`**
pod/mysql created

### User and Group IDs Assignment

When a project is created, OpenShift adds annotations to the project that determine the user ID (UID) range and supplemental group ID (GID) for pods and their containers in the project. You can retrieve the annotations with the ``oc describe project _`project-name`_`` command.

```sh
[user@host ~]$ oc describe project my-app
Name:			my-app
_...output omitted..._
Annotations:   openshift.io/description=
			   openshift.io/display-name=
			   openshift.io/requester=developer
			   openshift.io/sa.scc.mcs=s0:c27,c4
			   **`openshift.io/sa.scc.supplemental-groups=1000710000/10000`**
			   **`openshift.io/sa.scc.uid-range=1000710000/10000`**
_...output omitted..._
```

With OpenShift default security policies, regular cluster users cannot choose the `USER` or UIDs for their containers. When a regular cluster user creates a pod, OpenShift ignores the `USER` instruction in the container image. Instead, OpenShift assigns to the user in the container a UID and a supplemental GID from the identified range in the project annotations. The GID of the user is always `0`, which means that the user belongs to the `root` group. Any files and directories that the container processes might write to must have read and write permissions by `GID=0` and have the `root` group as the owner. Although the user in the container belongs to the `root` group, the user is an unprivileged account.

In contrast, when a cluster administrator creates a pod, the `USER` instruction in the container image is processed. For example, if the `USER` instruction for the container image is set to `0`, then the user in the container is the `root` privileged account, with a `0` value for the UID. Executing a container as a privileged account is a security risk. A privileged account in a container has unrestricted access to the container's host system. Unrestricted access means that the container could modify or delete system files, install software, or otherwise compromise its host. Red Hat therefore recommends that you run containers as `rootless`, or as an unprivileged user with only the necessary privileges for the container to run.

Red Hat also recommends that you run containers from different applications with unique user IDs. Running containers from different applications with the same UID, even an unprivileged one, is a security risk. If the UID for two containers is the same, then the processes in one container could access the resources and files of the other container. By assigning a distinct range of UIDs and GIDs for each project, OpenShift ensures that applications in different projects do not run as the same UID or GID.

#### Pod Security

The Kubernetes Pod Security Admission controller issues a warning when a pod is created without a defined security context. Security contexts grant or deny OS-level privileges to pods. OpenShift uses the Security Context Constraints controller to provide safe defaults for pod security. You can ignore pod security warnings in these course exercises. Security Context Constraints (SCC) are discussed in more detail in course DO280: _Red Hat OpenShift Administration II: Operating a Production Kubernetes Cluster_.

### Execute Commands in Running Containers

To execute a command in a running container in a pod, you can use the `exec` command with the `kubectl` or `oc` CLI. The `exec` command uses the following syntax:

**``oc exec _`RESOURCE/NAME`_ -- _`COMMAND`_ [args...] [options]``**

The output of the executed command is sent to your terminal. In the following example, the `exec` command executes the `date` command in the `my-app` pod.

[user@host ~]$ **`oc exec my-app -- date`**
Tue Feb 21 20:43:53 UTC 2023

The specified command is executed in the first container of a pod. For multicontainer pods, include the `-c` or `--container=` options to specify which container is used to execute the command. The following example executes the `date` command in a container named `ruby-container` in the `my-app` pod.

[user@host ~]$ **`kubectl exec my-app -c ruby-container -- date`**
Tue Feb 21 20:46:50 UTC 2023

The `exec` command also accepts the `-i` and `-t` options to create an interactive session with a container in a pod. In the following example, Kubernetes sends `stdin` to the `bash` shell in the `ruby-container` container from the `my-app` pod, and sends `stdout` and `stderr` from the `bash` shell back to the terminal.

[user@host ~]$ **`oc exec my-app -c ruby-container -it -- bash -il`**
[1000780000@ruby-container /]$

In the previous example, a raw terminal is opened in the `ruby-container` container. From this interactive session, you can execute additional commands in the container. To terminate the interactive session, you must execute the `exit` command in the raw terminal.

[user@host ~]$ **`kubectl exec my-app -c ruby-container -it -- bash -il`**
[1000780000@ruby-container /]$ date
Tue Feb 21 21:16:00 UTC 2023
[1000780000@ruby-container] **`exit`**

### Container Logs

Container logs are the standard output (`stdout`) and standard error (`stderr`) output of a container. You can retrieve logs with the ``logs pod _`pod-name`_`` command that the `kubectl` and `oc` CLIs provide. The command includes the following options:

**`-l`** `or` **`--selector=''`**

Filter objects based on the specified `key:value` label constraint.

**`--tail=`**

Specify the number of lines of recent log files to display; the default value is `-1` with no selectors, which displays all log lines.

**`-c`** `or` **`--container=`**

Print the logs of a particular container in a multicontainer pod.

**`-f`** `or` **`--follow`**

Follow, or stream, logs for a container.

**`-p`** `or` **`--previous=true`**

Print the logs of a previous container instance in the pod, if it exists. This option is helpful for troubleshooting a pod that failed to start, because it prints the logs of the last attempt.

The following example restricts `oc logs` command output to the 10 most recent log files:

[user@host ~]$ **`oc logs postgresql-1-jw89j --tail=10`**
 done
server stopped
Starting server...
2023-01-04 22:00:16.945 UTC [1] LOG:  starting PostgreSQL 12.11 on x86_64-redhat-linux-gnu, compiled by gcc (GCC) 8.5.0 20210514 (Red Hat 8.5.0-10), 64-bit
2023-01-04 22:00:16.946 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2023-01-04 22:00:16.946 UTC [1] LOG:  listening on IPv6 address "::", port 5432
2023-01-04 22:00:16.953 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2023-01-04 22:00:16.960 UTC [1] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
2023-01-04 22:00:16.968 UTC [1] LOG:  redirecting log output to logging collector process
2023-01-04 22:00:16.968 UTC [1] HINT:  Future log output will appear in directory "log".

You can also use the ``attach _`pod-name`_ -c _`container-name`_ -it`` command to connect to and start an interactive session on a running container in a pod. The ``-c _`container-name`_`` option is required for multicontainer pods. If the container name is omitted, then Kubernetes uses the `kubectl.kubernetes.io/default-container` annotation on the pod to select the container. Otherwise, the first container in the pod is chosen. You can use the interactive session to retrieve application log files and to troubleshoot application issues.

[user@host ~]$ **`oc attach my-app -it`**
If you don't see a command prompt, try pressing enter.

bash-4.4$

You can also retrieve logs from the web console by clicking the Logs tab of any pod.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/pods/containers/assets/podlogs.png)|

If you have more than one container, then you can change between them to list the logs of each one.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/pods/containers/assets/logscontainers.png)|

### Deleting Resources

You can delete Kubernetes resources, such as pod resources, with the `delete` command. The `delete` command can delete resources by resource type and name, resource type and label, standard input (`stdin`), and with JSON- or YAML-formatted files. The command accepts only one argument type at a time.

For example, you can supply the resource type and name as a command argument.

[user@host ~]$ **`oc delete pod php-app`**

You can also delete pods from the web console by clicking Actions and then Delete Pod in the pod's principal menu.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/pods/containers/assets/podelete.png)|

To select resources based on labels, you can include the `-l` option and the `key:value` label as a command argument.

[user@host ~]$ **`kubectl delete pod -l app=my-app`**
pod "php-app" deleted
pod "mysql-db" deleted

You can also provide the resource type and a JSON- or YAML-formatted file that specifies the name of the resource. To use a file, you must include the `-f` option and provide the full path to the JSON- or YAML-formatted file.

[user@host ~]$ **`oc delete pod -f ~/php-app.json`**
pod "php-app" deleted

You can also use `stdin` and a JSON- or YAML-formatted file that includes the resource type and resource name with the `delete` command.

[user@host ~]$ **`cat ~/php-app.json | kubectl delete -f -`**
pod "php-app" deleted

Pods support graceful termination, which means that pods try to terminate their processes first before Kubernetes forcibly terminates the pods. To change the time period before a pod is forcibly terminated, you can include the `--grace-period` flag and a time period in seconds in your `delete` command. For example, to change the grace period to 10 seconds, use the following command:

[user@host ~]$ **`oc delete pod php-app --grace-period=10`**

To shut down the pod immediately, set the grace period to 1 second. You can also use the `--now` flag to set the grace period to 1 second.

[user@host ~]$ **`oc delete pod php-app --now`**

You can also forcibly delete a pod with the `--force` option. If you forcibly delete a pod, Kubernetes does not wait for a confirmation that the pod's processes ended, which can leave the pod's processes running until its node detects the deletion. Therefore, forcibly deleting a pod could result in inconsistency or data loss. Forcibly delete pods only if you are sure that the pod's processes are terminated.

[user@host ~]$ **`kubectl delete pod php-app --force`**

To delete all pods in a project, you can include the `--all` option.

[user@host ~]$ **`kubectl delete pods --all`**
pod "php-app" deleted
pod "mysql-db" deleted

Likewise, you can delete a project and its resources with the ``oc delete project _`project-name`_`` command.

[user@host ~]$ **`oc delete project my-app`**
project.project.openshift.io "my-app" deleted

### The CRI-O Container Engine

A container engine is required to run containers. Worker and control plane nodes in an OpenShift Container Platform cluster use the `CRI-O` container engine to run containers. Unlike tools such as Podman or Docker, the `CRI-O` container engine is a runtime that is designed and optimized specifically for running containers in a Kubernetes cluster. Because `CRI-O` meets the Kubernetes Container Runtime Interface (CRI) standards, the container engine can integrate with other Kubernetes and OpenShift tools, such as networking and storage plug-ins.

**Note

For more information about the Kubernetes Container Runtime Interface (CRI) standards, refer to the _CRI-API_ repository at [https://github.com/kubernetes/cri-api](https://github.com/kubernetes/cri-api).

`CRI-O` provides a command-line interface to manage containers with the `crictl` command. The `crictl` command includes several subcommands to help you to manage containers. The following subcommands are commonly used with the `crictl` command:

**`crictl pods`**

Lists all pods on a node.

**`crictl image`**

Lists all images on a node.

**`crictl inspect`**

Retrieve the status of one or more containers.

**`crictl exec`**

Run a command in a running container.

**`crictl logs`**

Retrieve the logs of a container.

**`crictl ps`**

List running containers on a node.

To manage containers with the `crictl` command, you must first identify the node that is hosting your containers.

[user@host ~]$ **`kubectl get pods -o wide`**
NAME                READY STATUS    RESTARTS AGE IP        NODE
postgresql-1-8lzf2  1/1   Running   0        20m 10.8.0.64 **`master01`**
postgresql-1-deploy 0/1   Completed 0        21m 10.8.0.63 master01

[user@host ~]$ **`oc get pod postgresql-1-8lzf2 -o jsonpath='{.spec.nodeName}{"\n"}'`**
master01

Next, you must connect to the identified node as a cluster administrator. Cluster administrators can use SSH to connect to a node or create a debug pod for the node. Regular users cannot connect to or create debug pods for cluster nodes.

As a cluster administrator, you can create a debug pod for a node with the ``oc debug node/_`node-name`_`` command. OpenShift creates the ``pod/_`node-name`_-debug`` pod in your currently selected project and automatically connects you to the pod. You must then enable access host binaries, such as the `crictl` command, with the `chroot /host` command. This command mounts the host's root file system in the `/host` directory within the debug pod shell. By changing the root directory to the `/host` directory, you can run binaries contained in the host's executable path.

[user@host ~]$ **`oc debug node/master01`**
Starting pod/master01-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.50.10
If you don't see a command prompt, try pressing enter.
sh-4.4# **`chroot /host`**

After enabling host binaries, you can use the `crictl` command to manage the containers on the node. For example, you can use the `crictl ps` and `crictl inspect` commands to retrieve the process ID (`PID`) of a running container. You can then use the PID to retrieve or enter the namespaces within a container, which is useful for troubleshooting application issues.

To find the PID of a running container, you must first determine the container's ID. You can use the `crictl ps` command with the `--name` option to filter the command output to a specific container.

sh-5.1# **`crictl ps --name postgresql`**
CONTAINER     IMAGE        CREATED STATE   NAME       ATTEMPT POD ID      POD
27943ae4f3024 image...7104 5...ago Running postgresql 0       5768...f015 postgresql-1...

The default output of the `crictl ps` command is a table. You can find the short container ID under the `CONTAINER` column. You can also use the `-o` or `--output` options to specify the format of the `crictl ps` command as JSON or YAML and then parse the output. The parsed output displays the full container ID.

sh-5.1# **`crictl ps --name postgresql -o json | jq .containers[0].id`**
"2794...29a4"

After identifying the container ID, you can use the `crictl inspect` command and the container ID to retrieve the PID of the running container. By default, the `crictl inspect` command displays verbose output. You can use the `-o` or `--output` options to format the command output as JSON, YAML, a table, or as a Go template. If you specify the JSON format, you can then parse the output with the `jq` command. Likewise, you can use the `grep` command to limit the command output.

sh-5.1# **`crictl inspect -o json 27943ae4f3024 | jq .info.pid`**
**`43453`**
sh-5.1# **`crictl inspect 27943ae4f3024 | grep pid`**
    "pid": **`43453`**,
_...output omitted..._

After determining the PID of a running container, you can use the ``lsns -p _`PID`_`` command to list the system namespaces of a container.

sh-5.1# **`lsns -p 43453`**
        NS TYPE   NPROCS   PID USER       COMMAND
4026531835 cgroup    530     1 root       /usr/lib/systemd/systemd --switched-root --system --deserialize 17
4026531837 user      530     1 root       /usr/lib/systemd/systemd --switched-root --system --deserialize 17
4026537853 uts         8 43453 1000690000 postgres
4026537854 ipc         8 43453 1000690000 postgres
4026537856 net         8 43453 1000690000 postgres
4026538013 mnt         8 43453 1000690000 postgres
4026538014 pid         8 43453 1000690000 postgres

You can also use the PID of a running container with the `nsenter` command to enter a specific namespace of a running container. For example, you can use the `nsenter` command to execute a command within a specified namespace on a running container. The following example executes the `ps -ef` command within the process namespace of a running container.

sh-5.1# **`nsenter -t 43453 -p -r ps -ef`**
UID          PID    PPID  C STIME TTY          TIME CMD
1000690+       1       0  0 18:49 ?        00:00:00 postgres
1000690+      58       1  0 18:49 ?        00:00:00 postgres: logger
1000690+      60       1  0 18:49 ?        00:00:00 postgres: checkpointer
1000690+      61       1  0 18:49 ?        00:00:00 postgres: background writer
1000690+      62       1  0 18:49 ?        00:00:00 postgres: walwriter
1000690+      63       1  0 18:49 ?        00:00:00 postgres: autovacuum launcher
1000690+      64       1  0 18:49 ?        00:00:00 postgres: stats collector
1000690+      65       1  0 18:49 ?        00:00:00 postgres: logical replication launcher
root        7414       0  0 20:14 ?        00:00:00 ps -ef

The `-t` option specifies the PID of the running container as the target PID for the `nsenter` command. The `-p` option directs the `nsenter` command to enter the process or `pid` namespace. The `-r` option sets the top-level directory of the process namespace as the root directory, thus enabling commands to execute in the context of the namespace.

You can also use the `-a` option to execute a command in all of the container's namespaces.

sh-5.1# **`nsenter -t 43453 -a ps -ef`**
UID          PID    PPID  C STIME TTY          TIME CMD
1000690+       1       0  0 18:49 ?        00:00:00 postgres
1000690+      58       1  0 18:49 ?        00:00:00 postgres: logger
1000690+      60       1  0 18:49 ?        00:00:00 postgres: checkpointer
1000690+      61       1  0 18:49 ?        00:00:00 postgres: background writer
1000690+      62       1  0 18:49 ?        00:00:00 postgres: walwriter
1000690+      63       1  0 18:49 ?        00:00:00 postgres: autovacuum launcher
1000690+      64       1  0 18:49 ?        00:00:00 postgres: stats collector
1000690+      65       1  0 18:49 ?        00:00:00 postgres: logical replication launcher
root       10058       0  0 20:45 ?        00:00:00 ps -ef

## Find and Inspect Container Images

### Objectives

- Find containerized applications in container registries and get information about the runtime parameters of supported and community container images.
    

### Container Image Overview

A container is an isolated runtime environment where applications are executed as isolated processes. The isolation of the runtime environment ensures that applications do not interfere with other containers or system processes.

A container image contains a packaged version of your application, with all the necessary dependencies for the application to run. Images can exist without containers. However, containers depend on images, because containers use container images to build a runtime environment to execute applications.

Containers can be split into two similar but distinct concepts: _container images_ and _container instances_. A _container image_ contains immutable data that defines an application and its libraries. You can use container images to create _container instances_, which are running processes that are isolated by a set of kernel namespaces.

You can use each container image many times to create many distinct container instances. These replicas can be split across multiple hosts. The application within a container is independent of the host environment.

### Container Image Registries

Image registries are services that offer container images to download. Image creators and maintainers can store and distribute container images in a controlled manner to public or private audiences. Some examples of image registries include Quay.io, Red Hat Registry, Docker Hub, and Amazon ECR.

#### Red Hat Registry

Red Hat distributes container images by using two registries: `registry.access.redhat.com` (where no authentication is required), and `registry.redhat.io` (where authentication is required). The Red Hat Ecosystem Catalog, [https://catalog.redhat.com/](https://catalog.redhat.com/), provides centralized searching utility for both registries. You can search the Red Hat Ecosystem Catalog for technical details about container images. The catalog hosts a large set of container images, including from major open source projects, such as Apache, MySQL, and Jenkins.

Because the Red Hat Ecosystem Catalog is also searched for software products other than container images, you must navigate to `[https://catalog.redhat.com/software/containers/explore](https://catalog.redhat.com/software/containers/explore)` to specifically search for container images.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/pods/images/assets/redhatcatalog.png)|

Figure 3.4: Red Hat Ecosystem Catalog

The details page of a container image gives relevant information, such as technical data, the installed packages within the image, or a security scan. You can navigate through these options by using the tabs on the website. You can also change the image version by selecting a specific tag.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/pods/images/assets/redhatcatalog2.png)|

Figure 3.5: The Red Hat Universal Base Image 9

The Red Hat internal security team vets all images in the container catalog. Red Hat rebuilds all components to avoid known security vulnerabilities.

Red Hat container images provide the following benefits:

- Trusted source: All container images use sources that Red Hat knows and trusts.
    
- Original dependencies: None of the container packages are tampered with, and include only known libraries.
    
- Vulnerability-free: Container images are free of known critical vulnerabilities in the platform components or layers.
    
- Runtime protection: All applications in container images run as non-root users, to minimize the exposure surface to malicious or faulty applications.
    
- Red Hat Enterprise Linux (RHEL) compatible: Container images are compatible with all RHEL platforms, from bare metal to cloud.
    
- Red Hat support: Red Hat commercially supports the complete stack.
    

**Note

You must log in to the `registry.redhat.io` registry with a customer portal account or a Red Hat Developer account to use the stored container images in the registry.

#### Quay.io

Although the Red Hat Registry stores only images from Red Hat and certified providers, you can store your own images with Quay.io, another public image registry that Red Hat sponsors. Although storing public images in Quay is free of charge, some options are available only for paying customers. Quay also offers an on-premise version of the product, which you can use to set up an image registry in your own servers.

Quay.io introduces features such as server-side image building, fine-grained access controls, and automatic scanning of images for known vulnerabilities.

Quay.io offers live images that creators regularly update. Quay.io users can create their namespaces, with fine-grained access control, and publish their created images to that namespace. Container Catalog users rarely or never push new images, but consume trusted images from the Red Hat team.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/pods/images/assets/quay.png)|

Figure 3.6: The Quay.io welcome page

#### Private Registries

Image creators or maintainers might want to make their images publicly available. However, other image creators might prefer to keep their images private, for the following reasons:

- Company privacy and secret protection
    
- Legal restrictions and laws
    
- Avoidance of publishing images in development
    

In some cases, private images are preferred. Private registries give image creators control over image placement, distribution, and usage. Private images are more secure than images in public registries.

#### Public Registries

Other public registries, such as Docker Hub and Amazon ECR, are also available for storing, sharing, and consuming container images. These registries can include official images that the registry owners or the registry community users create and maintain. For example, Docker Hub hosts a Docker Official Image of a WordPress container image. Although the `docker.io/library/wordpress` container image is a Docker Official Image, the container image is not supported by WordPress, Docker, or Red Hat. Instead, the Docker Community, a global group of Docker Hub users, supports and maintains the container image. Support for this container image depends on the availability and skills of the Docker Community users.

Consuming container images from public registries brings risks. For example, a container image might include malicious code or vulnerabilities, which can compromise the host system that executes the container image. A host system can also be compromised by public container images, because the images are often configured with the privileged `root` user. Additionally, the software in a container image might not be correctly licensed, or might violate licensing terms.

Before you use a container image from a public registry, review and verify the container image. Also ensure that you have the correct permissions to use the software in the container image.

### Container Image Identifiers

Several objects provide identifying information about a container image.

**`Registry`**

It is a content server, such as `registry.access.redhat.com`, that is used to store and share container images. A registry consists of one or more repositories that contain tagged container images.

**`Name`**

It identifies the container image repository; it is a string that is composed of letters, numbers, and some special characters. This component refers to the name of the directory, or the container repository, within the container registry where the container image is.

For example, consider the fully qualified domain name (FQDN) of the `registry.access.redhat.com/ubi9/httpd-24:1-233` container image. The container image is in the `ubi9/httpd-24` repository in the `registry.access.redhat.com` container registry.

**`ID/Hash`**

It is the SHA (Secure Hash Algorithm) code to pull or verify an image. The SHA image ID cannot change, and always references the same container image content. The ID/hash is the true, unique identifier of an image. For example, the `sha256:4186a1ead13fc30796f951694c494e7630b82c320b81e20c020b3b07c888985b` image ID always refers to the `registry.access.redhat.com/ubi9/httpd-24:1-233` container image.

**`Tag`**

It is a label for a container image in a repository, to distinguish from other images, for version control. The `tag` comes after the image repository name and is delimited by a colon (:).

When an image tag is omitted, the floating tag, `latest`, is used as the default tag. A floating tag is an alias to another tag. In contrast, a fixed tag points to a specific container build. For the `registry.access.redhat.com/ubi9/httpd-24:1-233.1669634588` container image, `1-233.1669634588` is the fixed tag for the image, and at the time of writing, corresponds to the floating `latest` flag.

### Container Image Components

A container image is composed of multiple components.

**`Layers`**

Container images are created from instructions. Each instruction adds a layer to the container image. Each layer consists of the differences between it and the following layer. The layers are then stacked to create a read-only container image.

**`Metadata`**

Metadata includes the instructions and documentation for a container image.

### Container Image Instructions and Metadata

Container image layers consist of instructions, or steps, and metadata for building the image. You can override instructions during container creation to adjust the container image according to your needs. Some instructions can affect the running container, and other instructions are for informational purposes only.

The following instructions affect the state of a running container:

**`ENV`**

Defines the available environment variables in the container. A container image might include multiple `ENV` instructions. Any container can recognize additional environment variables that are not listed in its metadata.

**`ARG`**

It defines build-time variables, typically to make a customizable container build. Developers commonly configure the `ENV` instructions by using the `ARG` instruction. It is useful for preserving the build-time variables for run time.

**`USER`**

Defines the active user in the container. Later instructions run as this user. It is a good practice to define a user other than `root` for security purposes. OpenShift does not honor the user in a container image, for regular cluster users. Only cluster administrators can run containers (pods) with their chosen _user ID (UIDs)_ and _group IDs (GIDs)_.

**`ENTRYPOINT`**

It defines the executable to run when the container is started.

**`CMD`**

It defines the command to execute when the container is started. This command is passed to the executable that the `ENTRYPOINT` instruction defines. Base images define a default `ENTRYPOINT` executable, which is usually a shell executable, such as Bash.

**`WORKDIR`**

It sets the current working directory within the container. Later instructions execute within this directory.

Metadata is used for documentation purposes, and does not affect the state of a running container. You can also override the metadata values during container creation.

The following metadata is for information only, and does not affect the state of the running container:

**`EXPOSE`**

It indicates the network port that the application binds to within the container. This metadata does not automatically bind the port on the host, and is used only for documentation purposes.

**`VOLUME`**

It defines where to store data outside the container. The value shows the path where your container runtime mounts the directory inside the container. More than one path can be defined to create multiple volumes.

**`LABEL`**

Adds a key-value pair to the metadata of the image for organization and image selection.

Container engines are not required to honor metadata in a container image, such as `USER` or `EXPOSE`. A container engine can also recognize additional environment variables that are not listed in the container image metadata.

### Base Images

A _base image_ is the image that your resulting container image is built on. Your chosen base image determines the Linux distribution, and any of the following components:

- Package manager
    
- `Init` system
    
- File system layout
    
- Preinstalled dependencies and runtimes
    

The base image can also influence factors such as image size, vendor support, and processor compatibility.

Red Hat provides enterprise-grade container images that are engineered to be the base operating system layer for your containerized applications. These container images are intended as a common starting point for containers, and are known as _universal base images_ (UBI). Red Hat UBI container images are _Open Container Initiative (OCI)_ compliant images that contain portions of Red Hat Enterprise Linux (RHEL). UBI container images include a subset of RHEL content. They provide a set of prebuilt runtime languages, such as Python and Node.js, and associated DNF repositories that you can use to add application dependencies. UBI-based images can be distributed without cost or restriction. They can be deployed to both Red Hat and non-Red Hat platforms, and be pushed to your chosen container registry.

A Red Hat subscription is not required to use or distribute UBI-based images. However, Red Hat provides full support only for containers that are built on UBI if the containers are deployed to a Red Hat platform, such as a Red Hat OpenShift Container Platform (RHOCP) cluster or RHEL.

Red Hat provides four UBI variants: `standard`, `init`, `minimal`, and `micro`. All UBI variants and UBI-based images use Red Hat Enterprise Linux (RHEL) at their core and are available from the Red Hat Container Catalog. The main differences are as follows:

Standard

This image is the primary UBI, which includes DNF, systemd, and utilities such as `gzip` and `tar`.

Init

This image simplifies running multiple applications within a single container by managing them with systemd.

Minimal

This image is smaller than the `init` image and provides nice-to-have features. This image uses the `microdnf` minimal package manager instead of the full-sized version of DNF.

Micro

This image is the smallest available UBI, and includes only the minimum packages. For example, this image does not include a package manager.

### Inspecting and Managing Container Images

Various tools can inspect and manage container images, including the `oc image` command and Skopeo.

#### Skopeo

Skopeo is another tool to inspect and manage remote container images. With Skopeo, you can copy and sync container images from different container registries and repositories. You can also copy an image from a remote repository and save it to a local disk. If you have the appropriate repository permissions, then you can also delete an image from container registry. You also can use Skopeo to inspect the configuration and contents of a container image, and to list the available tags for a container image. Unlike other container image tools, Skopeo can execute without a privileged account, such as `root`. Skopeo does not require a running daemon to execute various operations.

Skopeo is executed with the `skopeo` command-line utility, which you can install with various package managers, such as DNF, Brew, and APT. The `skopeo` utility might already be installed on some Linux-based distributions. You can install the `skopeo` utility on Fedora, CentOS Stream 8 and later, and Red Hat Enterprise Linux 8 and later systems by using the DNF package manager.

[user@host ~]$ **`sudo dnf -y install skopeo`**

The `skopeo` utility is currently not available as a packaged binary for Windows-based systems. However, the `skopeo` utility is available as a container image from the `quay.io/skopeo/stable` container repository. For more information about the Skopeo container image, refer to the `skopeoimage` overview guide in the Skopeo repository (`[https://github.com/containers/skopeo/blob/main/contrib/skopeoimage/README.md](https://github.com/containers/skopeo/blob/main/contrib/skopeoimage/README.md)`).

You can also build `skopeo` from source code in a container, or build it locally without using a container. Refer to the installation guide in the Skopeo repository (`[https://github.com/containers/skopeo/blob/main/install.md#container-images](https://github.com/containers/skopeo/blob/main/install.md#container-images)`) for more information about installing or building Skopeo from source code.

The `skopeo` utility provides commands to help you to manage and inspect container images and container image registries. For container registries that require authentication, you must first log in to the registry before you can execute additional `skopeo` commands.

[user@host ~]$ **`skopeo login quay.io`**

**Note

OpenShift clusters are typically configured with registry credentials. When a pod is created from a container image in a remote repository, OpenShift authenticates to the container registry with the configured registry credentials, and then pulls, or copies, the image. Because OpenShift automatically uses the registry credentials, you typically do not need to manually authenticate to a container registry when you create a pod. By contrast, the `oc image` command and the `skopeo` utility require you first to log in to a container registry.

After you log in to a container registry (if required), you can execute additional `skopeo` commands against container images in a repository. When you execute a `skopeo` command, you must specify the transport and the repository name. A _transport_ is the mechanism to transfer or move container images between locations. Two common transports are `docker` and `dir`. The `docker` transport is used for container registries, and the `dir` transport is used for local directories.

The `oc image` command and other tools default to the `docker` transport, and so you do not need to specify the transport when executing commands. However, the `skopeo` utility does not define a default transport; you must specify the transport with the container image name. Most `skopeo` commands use the ``skopeo _`command`_ _`[command options]`_ _`transport`_://_`IMAGE-NAME`_`` format. For example, the following `skopeo list-tags` command lists all available tags in a `registry.access.redhat.com/ubi9/httpd-24` container repository by using the `docker` transport:

[user@host ~]$ **`skopeo list-tags docker://registry.access.redhat.com/ubi9/httpd-24`**
{
    "Repository": "registry.access.redhat.com/ubi9/httpd-24",
    "Tags": [
        "1-229",
        "1-217.1666632462",
        "1-201",
        "1-194.1655192191-source",
        "1-229-source",
        "1-194-source",
        "1-201-source",
        "1-217.1664813224",
        "1-217.1664813224-source",
        "1-210-source",
        "1-233.1669634588",
        "1",
        "1-217.1666632462-source",
        "1-233",
        "1-194.1655192191",
        "1-217.1665076049-source",
        "1-217",
        "1-233.1669634588-source",
        "1-210",
        "1-217.1665076049",
        "1-217-source",
        "1-233-source",
        "1-194",
        "latest"
    ]
}

The `skopeo` utility includes other useful commands for container image management.

**`skopeo inspect`**

View low-level information for an image name, such as environment variables and available tags. Use the ``skopeo inspect _`[command options]`_ _`transport`_://_`IMAGE-NAME`_`` command format. You can include the `--config` flag to view the configuration, metadata, and history of a container repository. The following example retrieves the configuration information for the `registry.access.redhat.com/ubi9/httpd-24` container repository:

[user@host ~]$ **`skopeo inspect --config docker://registry.access.redhat.com/ubi9/httpd-24`**
_...output omitted..._
    "config": {
        "User": "1001",
        "ExposedPorts": {
            "8080/tcp": {},
            "8443/tcp": {}
        },
        "Env": [
_...output omitted..._
            "HTTPD_MAIN_CONF_PATH=/etc/httpd/conf",
            "HTTPD_MAIN_CONF_MODULES_D_PATH=/etc/httpd/conf.modules.d",
            "HTTPD_MAIN_CONF_D_PATH=/etc/httpd/conf.d",
            "HTTPD_TLS_CERT_PATH=/etc/httpd/tls",
            "HTTPD_VAR_RUN=/var/run/httpd",
            "HTTPD_DATA_PATH=/var/www",
            "HTTPD_DATA_ORIG_PATH=/var/www",
            "HTTPD_LOG_PATH=/var/log/httpd"
        ],
        "Entrypoint": [
            "container-entrypoint"
        ],
        "Cmd": [
            "/usr/bin/run-httpd"
        ],
        "WorkingDir": "/opt/app-root/src",
_...output omitted..._
    }
_...output omitted..._

**`skopeo copy`**

Copy an image from one location or repository to another. Use the ``skopeo copy _`transport`_://_`SOURCE-IMAGE`_ _`transport`_://_`DESTINATION-IMAGE`_`` format. For example, the following command copies the `quay.io/skopeo/stable:latest` container image to the `skopeo` repository in the `registry.example.com` container registry:

[user@host ~]$ **`skopeo copy docker://quay.io/skopeo/stable:latest \ docker://registry.example.com/skopeo:latest`**

**`skopeo delete`**

Delete a container image from a repository. You must use the ``skopeo delete _`[command options]`_ _`transport`_://_`IMAGE-NAME`_`` format. The following command deletes the `skopeo:latest` image from the `registry.example.com` container registry:

[user@host ~]$ **`skopeo delete docker://registry.example.com/skopeo:latest`**

**`skopeo sync`**

Synchronize one or more images from one location to another. Use this command to copy all container images from a source to a destination. The command uses the ``skopeo sync _`[command options]`_ --src _`transport`_ --dest _`transport`_ _`SOURCE`_ _`DESTINATION`_`` format. The following command synchronizes the `registry.access.redhat.com/ubi8/httpd-24` container repository to the `registry.example.com/httpd-24` container repository:

[user@host ~]$ **`skopeo sync --src docker --dest docker \ registry.access.redhat.com/ubi8/httpd-24 registry.example.com/httpd-24`**

#### Registry Credentials

Some registries require users to authenticate. For example, Red Hat containers that are based on RHEL typically require authenticated access:

[user@host ~]$ **`skopeo inspect docker://registry.redhat.io/rhel8/httpd-24`**
FATA[0000] Error parsing image name "docker://registry.redhat.io/rhel8/httpd-24": **`unable to retrieve auth token: invalid username/password: unauthorized: Please login to the Red Hat Registry using your Customer Portal credentials.`** Further instructions can be found here: https://access.redhat.com/RegistryAuthentication

You might choose a different image that does not require authentication, such as the UBI 8 image:

[user@host ~]$ **`skopeo inspect docker://registry.access.redhat.com/ubi8:latest`**
{
    "Name": "registry.access.redhat.com/ubi8",
    "Digest": "sha256:70fc...1173",
    "RepoTags": [
        "8.7-1054-source",
        "8.6-990-source",
        "8.6-754",
        "8.4-203.1622660121-source",
_...output omitted..._.

Alternatively, you must execute the `skopeo login` command for the registry before you can access the RHEL 8 image.

[user@host ~]$ **`skopeo login registry.redhat.io`**
Username: **`YOUR_USER`**
Password: **`YOUR_PASSWORD`**
Login Succeeded!
[user@host ~]$ **`skopeo list-tags docker://registry.redhat.io/rhel8/httpd-24`**
{
    "Repository": "registry.redhat.io/rhel8/httpd-24",
    "Tags": [
        "1-166.1645816922",
        "1-209",
        "1-160-source",
        "1-112",
_...output omitted..._

Skopeo stores the credentials in the `${XDG_RUNTIME_DIR}/containers/auth.json` file, where the `${XDG_RUNTIME_DIR}` refers to a directory that is specific to the current user. The credentials are encoded in the base64 format:

[user@host ~]$ **`cat ${XDG_RUNTIME_DIR}/containers/auth.json`**
{
	"auths": {
		"registry.redhat.io": {
			"auth": **`"dXNlcjpodW50ZXIy"`**
		}
	}
}
[user@host ~]$ **`echo -n dXNlcjpodW50ZXIy | base64 -d`**
user:hunter2

**Note

For security reasons, the `skopeo login` command does not show your password in the interactive session. Although you do not see what you are typing, Skopeo registers every key stroke. After typing your full password in the interactive session, press **Enter** to start the login.

#### The `oc image` Command

The OpenShift command-line interface provides the `oc image` command. You can use this command to inspect, configure, and retrieve information about container images.

The `oc image info` command inspects and retrieves information about a container image. You can use the `oc image info` command to identify the ID/hash SHA and to list the image layers of a container image. You can also review container image metadata, such as environment variables, network ports, and commands. If a container image repository provides a container image in multiple architectures, such as `amd64` or `arm64`, then you must include the `--filter-by-os` tag. For example, you can execute the following command to retrieve information about the `registry.access.redhat.com/ubi9/httpd-24:1-233` container image that is based on the `amd64` architecture:

[user@host ~]$ **`oc image info registry.access.redhat.com/ubi9/httpd-24:1-233 \ --filter-by-os amd64`**
Name:          registry.access.redhat.com/ubi9/httpd-24:1-233
Digest:        sha256:4186...985b
_...output omitted..._
Image Size:    130.8MB in 3 layers
Layers:        79.12MB sha256:d74e...1cad
               17.32MB sha256:dac0...a283
               34.39MB sha256:47d8...5550
OS:            linux
Arch:          amd64
Entrypoint:    container-entrypoint
Command:       /usr/bin/run-httpd
Working Dir:   /opt/app-root/src
User:          1001
Exposes Ports: 8080/tcp, 8443/tcp
Environment:   container=oci
_...output omitted..._
               HTTPD_CONTAINER_SCRIPTS_PATH=/usr/share/container-scripts/httpd/
               HTTPD_APP_ROOT=/opt/app-root
               HTTPD_CONFIGURATION_PATH=/opt/app-root/etc/httpd.d
               HTTPD_MAIN_CONF_PATH=/etc/httpd/conf
               HTTPD_MAIN_CONF_MODULES_D_PATH=/etc/httpd/conf.modules.d
               HTTPD_MAIN_CONF_D_PATH=/etc/httpd/conf.d
               HTTPD_TLS_CERT_PATH=/etc/httpd/tls
               HTTPD_VAR_RUN=/var/run/httpd
               HTTPD_DATA_PATH=/var/www
               HTTPD_DATA_ORIG_PATH=/var/www
               HTTPD_LOG_PATH=/var/log/httpd
_...output omitted..._

The `oc image` command provides more options to manage container images.

**`oc image append`**

Use this command to add layers to container images, and then push the container image to a registry.

**`oc image extract`**

You can use this command to extract or copy files from a container image to a local disk. Use this command to access the contents of a container image without first running the image as a container. A running container engine is not required.

**`oc image mirror`**

Copy or mirror container images from one container registry or repository to another. For example, you can use this command to mirror container images between public and private registries. You can also use this command to copy a container image from a registry to a disk. The command mirrors the HTTP structure of a container registry to a directory on a disk. The directory on the disk can then be served as a container registry.

### Running Containers as Root

Running containers as the root user is a security risk, because an attacker could exploit the application, access the container, and exploit further vulnerabilities to escape from the containerized environment into the host system. Attackers might escape the containerized environment by exploiting bugs and vulnerabilities that are typically in the kernel or container runtime.

Traditionally, when an attacker gains access to the container file system by using an exploit, the root user inside the container corresponds to the root user outside the container. If an attacker escapes the container isolation, then they have elevated privileges on the host system, which potentially causes more damage.

Containers that do not run as the root user have limitations that might prove unsuitable for use in your application, such as the following limitations:

Non-trivial Containerization

Some applications might require the root user. Depending on the application architecture, some applications might not be suitable for non-root containers, or might require a deeper understanding to containerize.

For example, applications such as HTTPd and Nginx start a bootstrap process and then create a process with a non-privileged user, which interacts with external users. Such applications are non-trivial to containerize for rootless use.

Red Hat provides containerized versions of HTTPd and Nginx that do not require root privileges for production usage. You can find the containers in the Red Hat container registry (https://catalog.redhat.com/software/containers/explore).

Required Use of Privileged Utilities

Non-root containers cannot bind to privileged ports, such as the `80` or `443` ports. Red Hat advises against using privileged ports, but to use port forwarding instead.

Similarly, non-root containers cannot use the `ping` utility by default, because it requires elevated privileges to establish raw sockets.

## Troubleshoot Containers and Pods

### Objectives

- Troubleshoot a pod by starting additional processes on its containers, changing their ephemeral file systems, and opening short-lived network tunnels.
    

### Container Troubleshooting Overview

Containers are designed to be immutable and ephemeral. A running container must be redeployed when changes are needed or when a new container image is available. However, you can change a running container without redeployment.

Updating a running container is best reserved for troubleshooting problematic containers. Red Hat does not generally recommend editing a running container to fix errors in a deployment. Changes to a running container are not captured in source control, but help to identify the needed corrections to the source code for the container functions. Capture these container updates in version control after you identify the necessary changes. Then, build a new container image and redeploy the application.

Custom alterations to a running container are incompatible with elegant architecture, reliability, and resilience for the environment.

### CLI Troubleshooting Tools

Administrators use various tools to interact with, inspect, and alter running containers. Administrators can use commands such as `oc get` to gather initial details for a specified resource type. Other commands are available for detailed inspection of a resource, or to update a resource in real time.

**Note

When interacting with the cluster containers, take suitable precautions with actively running components, services, and applications.

Use these tools to validate the functions and environment for a running container:

- The `kubectl` CLI provides the following commands:
    
    - `kubectl describe`: Display the details of a resource.
        
    - `kubectl edit`: Edit a resource configuration by using the system editor.
        
    - `kubectl patch`: Update a specific attribute or field for a resource.
        
    - `kubectl replace`: Deploy a new instance of the resource.
        
    - `kubectl cp`: Copy files and directories to and from containers.
        
    - `kubectl exec`: Execute a command within a specified container.
        
    - `kubectl explain`: Display documentation for a specified resource.
        
    - `kubectl port-forward`: Configure a port forwarder for a specified container.
        
    - `kubectl logs`: Retrieve the logs for a specified container.
        
    

Besides supporting the previous `kubectl` commands, the `oc` CLI adds the following commands for inspecting and troubleshooting running containers:

- The `oc` CLI provides the following commands:
    
    - `oc status`: Display the status of the containers in the selected namespace.
        
    - `oc rsync`: Synchronize files and directories to and from containers.
        
    - `oc rsh`: Start a remote shell within a specified container.
        
    

### Editing Resources

Troubleshooting and remediation often begin with a phase of inspection and data gathering. When solving issues, the `describe` command can provide helpful details about the running resource, such as the definition of a container and its purpose.

The following example demonstrates use of the ``oc describe _`RESOURCE`_ _`NAME`_`` command to retrieve information about a pod in the `openshift-dns` namespace:

[user@host ~]$ **`oc describe pod dns-default-lt13h`**
Name:               dns-default-lt13h
Namespace:          openshift-dns
Priority:           2000001000
Priority Class Name: system-node-critical
_...output omitted..._

Various CLI tools can apply a change that you determine is needed to a running container. The `edit` command opens the specified resource in the default editor for your environment. This editor is specified by setting either the `KUBE_EDITOR` or the `EDITOR` environment variable, or otherwise with the vi editor in Linux or the Notepad application in Windows.

The following example demonstrates use of the ``oc edit _`RESOURCE`_ _`NAME`_`` command to edit a running container:

[user@host ~]$ **`oc edit pod mongo-app-sw88b`**

 >Please edit the object below. Lines beginning with a '#' will be ignored,
 and an empty file will abort the edit. If an error occurs while saving this file will be
 reopened with the relevant failures.

apiVersion: v1
kind: Pod
metadata:
  annotations:
_...output omitted..._

You can also use the `patch` command to update fields of a resource.

The following example uses the `patch` command to update the container image that a pod uses:

[user@host ~]$ **`oc patch pod valid-pod --type='json' \ -p='[{"op": "replace", "path": "/spec/containers/0/image", \ "value":"http://registry.access.redhat.com/ubi8/httpd-24"}]'`**

**Note

For more information about patching resources and the different merge methods, refer to [Update API Objects in Place Using kubectl patch](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/).

### Copy Files to and from Containers

Administrators can copy files and directories to or from a container to inspect, update, or correct functionality. Adding a configuration file or retrieving an application log are common use cases.

**Note

To use the `cp` command with the `kubectl` CLI or the `oc` CLI, the `tar` binary must be present in the container. If the binary is absent, then an error message appears and the operation fails.

The following example demonstrates copying a file from a running container to a local directory by using the ``oc cp _`SOURCE`_ _`DEST`_`` command:

[user@host ~]$ **`oc cp apache-app-kc82c:/var/www/html/index.html /tmp/index.bak`**

[user@host ~]$ **`ls /tmp`**
index.bak

The following example demonstrates use of the ``oc cp _`SOURCE`_ _`DEST`_`` command to copy a file from a local directory to a directory in a running container:

[user@host ~]$ **`oc cp /tmp/index.html apache-app-kc82c:/var/www/html/`**

[user@host ~]$ **`oc exec -it apache-app-kc82c -- ls /var/www/html`**
index.html

**Note

Targeting a file path within a pod for either the ``_`SOURCE`_`` or ``_`DEST`_`` argument uses the ``_`pod_name:path`_`` format, and can include the ``-c _`container_name`_`` option to specify a container within the pod. If you omit the ``-c _`container_name`_`` option, then the command targets the first container in the pod.

Additionally, when using the `oc` CLI, file and directory synchronization is available by using the `oc rsync` command.

The following example demonstrates use of the ``oc rsync _`SOURCE_NAME`_ _`DEST`_`` command to synchronize files from a running container to a local directory.

[user@host ~]$ **`oc rsync apache-app-kc82c:/var/www/ /tmp/web_files`**

[user@host ~]$ **`ls /tmp/web_files`**
cgi-bin
html

The `oc rsync` command uses the `rsync` client on your local system to copy changed files to and from a pod container. The `rsync` binary must be available locally and within the container for this approach. If the `rsync` binary is not found, then a `tar` archive is created on the local system and is sent to the container. The container then uses the `tar` utility to extract files from the archive. Without the `rsync` and `tar` binaries, an error message occurs and the `oc rsync` command fails.

**Note

For Linux-based systems, you can install the `rsync` client and the `tar` utility on a local system by using a package manager, such as DNF. For Windows-based systems, you can install the `cwRsync` client. For more information about the `cwRysnc` client, refer to [https://www.itefix.net/cwrsync](https://www.itefix.net/cwrsync).

### Remote Container Access

Exposing a network port for a container is routine, especially for containers that provide a service. In a cluster, port forwarding connections are made through the kubelet, which maps a local port on your system to a port on a pod. Configuring port forwarding creates a request through the Kubernetes API, and creates a multiplexed stream, such as HTTP/2, with a `port` header that specifies the target port in the pod. The kubelet delivers the stream data to the target pod and port, and vice versa for egress data from the pod.

When troubleshooting an application that typically runs without a need to connect locally, you can use the port-forwarding function to expose connectivity to the pod for investigation. With this function, an administrator can connect on the new port and inspect the problematic application. After you remediate the issue, the application can be redeployed without the port-forward connection.

The following example demonstrates use of the ``oc port-forward _`RESOURCE`_ _`EXTERNAL_PORT:CONTAINER_PORT`_`` command to listen locally on port `8080` and to forward connections to port `80` on the pod:

[user@host ~]$ **`oc port-forward nginx-app-cc78k 8080:80`**
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80

### Connect to Running Containers

Administrators use CLI tools to connect to a container via a shell for forensic inspections. With this approach, you can connect to, inspect, and run any available commands within the specified container.

The following example demonstrates use of the ``oc rsh _`POD_NAME`_`` command to connect to a container via a shell:

[user@host ~]$ **`oc rsh tomcat-app-jw53r`**
sh-4.4#

If you need to connect to a specific container in a pod, then use the ``-c _`container_name`_`` option to specify the container name. If you omit this option, then the command connects to the first container in the pod.

You can also connect to running containers from the web console by clicking the Terminal tab in the pod's principal menu.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/pods/troubleshooting/assets/podterminalmenu.png)|

If you have more than one container, then you can change between them to connect to the CLI.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/pods/troubleshooting/assets/containerterminal.png)|

### Execute Commands in a Container

Passing commands to execute within a container from the CLI is another method for troubleshooting a running container. Use this method to send a command to run within the container, or to connect to the container, when further investigation is necessary.

Use the following command to pass and execute commands in a container:

**``oc exec _`POD | TYPE/NAME`_ [-c _`container_name`_] -- _`COMMAND`_ [_`arg1`_ ... _`argN`_]``**

If you omit the ``-c _`container_name`_`` option, then the command targets the first container in the pod.

The following examples demonstrate the use of the `oc exec` command to execute the `ls` command in a container to list the contents of the container's root directory:

[user@host ~]$ **`oc exec -it mariadb-lc78h -- ls /`**
bin  boot  dev	etc  help.1  home  lib	lib64 ...
_...output omitted..._

[user@host ~]$ **`oc exec mariadb-lc78h -- ls /`**
bin
boot
dev
etc
_...output omitted..._

### Note

It is common to add the `-it` flags to the `kubectl exec` or `oc exec` commands. These flags instruct the command to send `STDIN` to the container and `STDOUT`/`STDERR` back to the terminal. The format of the command output is impacted by the inclusion of the `-it` flags.

### Container Events and Logs

Reviewing the historical actions for a container can offer insights into both the lifecycle and health of the deployment. Retrieving the cluster logs provides the chronological details of the container actions. Administrators inspect this log output for information and issues that occur in the running container.

For the following commands, use the ``-c _`container_name`_`` to specify a container in the pod. If you omit this option, then the command targets the first container in the pod.

The following examples demonstrate use of the ``oc logs _`POD_NAME`_`` command to retrieve the logs for a pod:

[user@host ~]$ **`oc logs BIND9-app-rw43j`**
Defaulted container "dns" out of: dns
.:5353
[INFO] plugin/reload: Running configuration SHA512 = 7c3d...3587
CoreDNS-1.9.2
_...output omitted..._

In Kubernetes, an `event` resource is a report of an event somewhere in the cluster. You can use the `kubectl get events` and `oc get events` commands to view pod events in a namespace:

[user@host ~]$ **`oc get events`**
LAST SEEN   TYPE     REASON           OBJECT                           MESSAGE
_...output omitted..._
21m         Normal   AddedInterface   pod/php-app-5d9b84b588-kzfxd     Add eth0 [10.8.0.93/23] from ovn-kubernetes
21m         Normal   Pulled           pod/php-app-5d9b84b588-kzfxd     Container image "registry.ocp4.example.com:8443/redhattraining/php-webapp:v4" already present on machine
21m         Normal   Created          pod/php-app-5d9b84b588-kzfxd     Created container php-webapp
21m         Normal   Started          pod/php-app-5d9b84b588-kzfxd     Started container php-webapp

### Available Linux Commands in Containers

The use of Linux commands for troubleshooting applications can also help with troubleshooting containers. However, when connecting to a container, only the defined tools and applications within the container are available. You can augment the environment inside the container by adding the tools from this section or any other remedial tools to the container image.

Before you add tools to a container image, consider how the tools affect your container image.

- Additional tools increase the size of the image, which might impact container performance.
    
- Tools might require additional update packages and licensing terms, which can impact the ease of updating and distributing the container image.
    
- Hackers might exploit tools in the image.
    

### Troubleshooting from Inside the Cluster

It is routine to troubleshoot a cluster, its components, or the running applications by connecting remotely. This approach assumes that the administrator's computer contains the necessary tools for the work. When unanticipated issues arise, the necessary tools might not be available from an administrator's computer or an alternative machine.

Administrators can alternatively author and deploy a container within the cluster for investigation and remediation. By creating a container image that includes the cluster troubleshooting tools, you have a reliable environment to perform these tasks from any computer with access to the cluster. This approach ensures that an administrator always has access to the tools for reliable troubleshooting and remediation of issues.

Additionally, administrators should plan to author a container image that provides the most valuable troubleshooting tools for containerized applications. In this way, you deploy this "toolbox" container to supplement the forensic process and to provide an environment with the required commands and tools for troubleshooting problematic containers. For example, the "toolbox" container can test how resources operate inside a cluster, such as to confirm whether a pod can connect to resources outside the cluster. Regular cluster users can also create a "toolbox" container to help with application troubleshooting. For example, a regular user could run a pod with a MySQL client to connect to another pod that runs a MySQL server.

Although this approach falls outside the focus of this course, because it is more application-level remediation than container-level troubleshooting, it is important to realize that containers have such capacity.


# Chapter 4: Deploy Managed and Networked Applications on Kubernetes

## Deploy Applications from Images and Templates

### Objectives

- Identify the main resources and settings that Kubernetes uses to manage long-lived applications and demonstrate how OpenShift simplifies common application deployment workflows.
    

### Deploying Applications

Microservices and DevOps are growing trends in enterprise software. Containers and Kubernetes gained popularity alongside those trends, but have become categories of their own. Container-based infrastructures support most types of traditional and modern applications.

The term _application_ can refer to your software system or to a service within it. Given this ambiguity, it is clearer to refer directly to resources, services, and other components.

### Resources and Resource Definitions

Kubernetes manages applications, or services, as a loose collection of resources. _Resources_ are configuration pieces for components in the cluster. When you create a resource, the corresponding component is not created immediately. Instead, the cluster is responsible for eventually creating the component.

A _resource type_ represents a specific component type, such as a pod. Kubernetes ships with many default resource types, some of which overlap in function. Red Hat OpenShift Container Platform (RHOCP) includes the default Kubernetes resource types, and provides other resource types of its own. To add resource types, you can create or import custom resource definitions (CRDs).

### Managing Resources

You can add, view, and edit resources in various formats, including YAML and JSON. Traditionally, YAML is the most common format.

You can delete resources in batch by using label selectors or by deleting the entire project or namespace. For example, the following command deletes only deployments with the `app=my-app` label.

```sh
[user@host ~]$ oc delete deployment -l app=my-app
_...output omitted..._
```

Similar to creation, deleting a resource is not immediate, but is instead a request for eventual deletion.

**Note

Commands that are executed without specifying a namespace are executed in the user's current namespace.

### Common Resource Types and Their Uses

The following resources are standard Kubernetes resources unless otherwise noted.

#### Templates

Similar to projects, templates are an RHOCP addition to Kubernetes. A _template_ is a YAML manifest that contains parameterized definitions of one or more resources. RHOCP provides predefined templates in the `openshift` namespace.

Process a template into a list of resources by using the `oc process` command, which replaces values and generates resource definitions. The resulting resource definitions create or update resources in the cluster by supplying them to the `oc apply` command.

For example, the following command processes a `mysql-template.yaml` template file and generates four resource definitions.

```sh
[user@host ~]$ oc process -f mysql-template.yaml -o yaml
apiVersion: v1
items:
- apiVersion: v1
  kind: Secret
  _...output omitted..._
- apiVersion: v1
  kind: Service
  _...output omitted..._
- apiVersion: v1
  kind: PersistentVolumeClaim
  _...output omitted..._
- apiVersion: apps.openshift.io/v1
  kind: DeploymentConfig
  _...output omitted..._
```

The `--parameters` option instead displays the parameters of a template. For example, the following command lists the parameters of the `mysql-template.yaml` file.

```sh
[user@host ~]$ oc process -f mysql-template.yaml --parameters
NAME                    DESCRIPTION _...output omitted..._
_...output omitted..._
MYSQL_USER              Username for MySQL user   _...output omitted..._
MYSQL_PASSWORD          Password for the MySQL connection    _...output omitted..._
MYSQL_ROOT_PASSWORD     Password for the MySQL root user.    _...output omitted..._
MYSQL_DATABASE          Name of the MySQL database accessed. _...output omitted..._
VOLUME_CAPACITY         Volume space available for data,     _...output omitted..._
```

You can use templates with the `new-app` command from RHOCP. In the following example, the `new-app` command uses the `mysql-persistent` template to create a MySQL application and its supporting resources.

```sh
[user@host ~]$ oc new-app --template mysql-persistent
--> Deploying template "db-app/mysql-persistent" to project db-app
_...output omitted..._
     The following service(s) have been created in your project: mysql.

            Username: userQSL
            Password: pyf0yElPvFWYQQou
       Database Name: sampledb
      Connection URL: mysql://mysql:3306/
...output omitted...
     * With parameters:
        * Memory Limit=512Mi
        * Namespace=openshift
        * Database Service Name=mysql
        * MySQL Connection Username=userQSL # generated
        * MySQL Connection Password=pyf0yElPvFWYQQou # generated
        * MySQL root user Password=HHbdurqWO5gAog2m # generated
        * MySQL Database Name=sampledb
        * Volume Capacity=1Gi
        * Version of MySQL Image=8.0-el8

--> Creating resources ... # 1
    secret "mysql" created
    service "mysql" created
    persistentvolumeclaim "mysql" created
    deploymentconfig.apps.openshift.io "mysql" created
--> Success
...output omitted...
```

|     |                                                                                                                                                     |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Notice that several resources are created to meet the requirements of the deployment, including a secret, a service, and a persistent volume claim. |

**Note

You can specify environment variables to be configured in creating your application.

You can also use templates when creating applications from the web console by using the Developer Catalog. From the Developer perspective, navigate to the +Add menu and click All Services in the Developer Catalog section to open the catalog. Then, enter the application name in the filter to search for a template for your application. You can instantiate the template and change the default values, such as the Git repository, the memory limits, the application version, and much more.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/newapp/assets/templatewebconsole.png)|

You can add an application based on a template by changing to the Developer perspective, and then moving to the Topology menu and clicking Start building application or clicking the book icon.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/newapp/assets/topologymenu.png)|

#### Pod

From the RHOCP documentation, a _pod_ is defined as "the smallest compute unit that can be defined, deployed, and managed". A pod runs one or more containers that represent a single application. Containers in the pod share resources, such as networking and storage.

The following example shows a definition of a pod:

```yaml
apiVersion: v1
kind: Pod # 1 
metadata: # 2
  annotations: { ... }
  labels:
    deployment: docker-registry-1
    deploymentconfig: docker-registry
  name: registry
  namespace: pod-registries
spec: # 3
  containers:
  - env:
    - name: OPENSHIFT_CA_DATA
      value: ...
    image: openshift/origin-docker-registry:v0.6.2
    imagePullPolicy: IfNotPresent
    name: registry
    ports:
    - containerPort: 5000
      protocol: TCP
    resources: {}
    securityContext: { ... }
    volumeMounts:
    - mountPath: /registry
      name: registry-storage
  dnsPolicy: ClusterFirst
  imagePullSecrets:
  - name: default-dockercfg-at06w
  restartPolicy: Always
  serviceAccount: default
  volumes:
  - emptyDir: {}
    name: registry-storage
status: # 4
  conditions: { ... }
```

|     |                                                                                                                                                                                      |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | Resource kind set to `Pod`.                                                                                                                                                          |
| 2   | Information that describes your application, such as the name, project, attached labels, and annotations.                                                                            |
| 3   | Section where the application requirements are specified, such as the container name, the container image, environment variables, volume mounts, network configuration, and volumes. |
| 4   | Indicates the last condition of the pod, such as the last probe time, the last transition time, the status setting as `true` or `false`, and more.                                   |

#### Deployment

Similar to deployment configurations, _deployments_ define the intended state of a replica set. _Replica sets_ maintain a configurable number of pods that match a specification.

Replica sets are generally similar to and a successor to replication controllers. This difference in intermediate resources is the primary difference between deployments and deployment configurations.

Deployments and replica sets are a Kubernetes-standard replacement for deployment configurations and replication controllers, respectively. Use a deployment unless you need a feature that is specific to deployment configurations.

The following example shows the definition of a deployment:

```yaml
apiVersion: apps/v1
kind: Deployment # 1
metadata:
  name: hello-openshift # 2 
spec:
  replicas: 1 # 3
    matchLabels:
      app: hello-openshift
  template: # 4
    metadata:
      labels:
        app: hello-openshift
    spec:
      containers:
      - name: hello-openshift # 5
        image: openshift/hello-openshift:latest # 6
        ports: # 7
        - containerPort: 80
```

|     |                                                                                                   |
| --- | ------------------------------------------------------------------------------------------------- |
| 1   | Resource kind set to `Deployment`.                                                                |
| 2   | Name of the deployment resource.                                                                  |
| 3   | Number of running instances.                                                                      |
| 4   | Section to define the metadata, labels, and the container information of the deployment resource. |
| 5   | Name of the container.                                                                            |
| 6   | Resource of the image to use for creating the deployment resource.                                |
| 7   | Port configuration, such as the port number, name of the port, and the protocol.                  |

#### Deployment Configurations

_Deployment configurations_ define the specification of a pod. They manage pods by creating _replication controllers_, which manage the number of replicas of a pod. Deployment configurations and replication controllers are an RHOCP addition to Kubernetes.

**Note

As of OpenShift Container Platform 4.14, deployment configuration objects are deprecated. Deployment configurations objects are still supported, but are not recommended for new installations.

Instead, use Deployment objects or another alternative to provide declarative updates for pods.

#### Projects

RHOCP adds projects to enhance the function of Kubernetes namespaces. A _project_ is a Kubernetes namespace with additional annotations, and is the primary method for managing access to resources for regular users. Projects can be created from templates and must use Role Based Access Control (RBAC) for organization and permission management. Administrators must grant cluster users access to a project. If a cluster user is allowed to create projects, then the user automatically has access to their created projects.

Projects provide logical and organizational isolation to separate your application component resources. Resources in one project can access resources in other projects, but not by default.

The following example shows the definition of a project:

```yaml
apiVersion: project.openshift.io/v1
kind: Project # 1
metadata:
  name: test # 2
spec:
  finalizers: # 3
  - kubernetes
```

|     |                                                                                                                                           |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Resource kind set to `Project`.                                                                                                           |
| 2   | Name of the project.                                                                                                                      |
| 3   | A finalizer is a special metadata key that tells Kubernetes to wait until a specific condition is met before it fully deletes a resource. |

#### Services

You can configure internal pod-to-pod network communication in RHOCP by using the _Service_ object. Applications send requests to the service name and port. RHOCP provides a virtual network, which reroutes such requests to the pods that the service targets by using labels.

The following example shows the definition of a service:

```yaml
apiVersion: v1
kind: Service # 1
metadata:
  name: docker-registry # 2
  namespace: test # 3
spec:
  selector:
    app: MyApp # 4
  ports:
  - protocol: TCP # 5
    port: 80 # 6
    targetPort: 9376 # 7
```

|     |                                                                                                                        |
| --- | ---------------------------------------------------------------------------------------------------------------------- |
| 1   | Resource kind set to `Service`.                                                                                        |
| 2   | Name of the service.                                                                                                   |
| 3   | Project name where the service resource exists.                                                                        |
| 4   | The label selector identifies all pods with the attached `app=MyApp` label and adds the pods to the service endpoints. |
| 5   | Internet protocol set to `TCP`.                                                                                        |
| 6   | Port that the service listens on.                                                                                      |
| 7   | Port on the backing pods, which the service forwards connections to.                                                   |

#### Persistent Volume Claims

RHOCP uses the Kubernetes persistent volume (PV) framework to enable cluster administrators to provision persistent storage for a cluster. Developers can use persistent volume claims (PVCs) to request PV resources without having specific knowledge of the underlying storage infrastructure. After a PV is bound to a PVC, that PV cannot then be bound to additional PVCs. Because PVCs are namespaced objects and PVs are globally scoped objects, this binding effectively scopes a bound PV to a single namespace until the binding PVC is deleted.

The following example shows the definition of a persistent volume claim:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim # 1
metadata:
  name: mysql-pvc # 2
spec:
  accessModes:
    - ReadWriteOnce # 3
  resources:
    requests:
      storage: 1Gi # 4
  storageClassName: nfs-storage # 5
status:
  ...
```

|     |                                                                  |
| --- | ---------------------------------------------------------------- |
| 1   | Resource kind set to `PersistentVolumeClaim`.                    |
| 2   | Name of the service.                                             |
| 3   | The access mode, to define the read/write and mount permissions. |
| 4   | The amount of storage that is available to the PVC.              |
| 5   | Name of the StorageClass that the claim requires.                |

#### Secrets

Secrets provide a mechanism to hold sensitive information, such as passwords, private source repository credentials, sensitive configuration files, and credentials to an external resource, such as an SSH key or OAuth token. You can mount secrets into containers by using a volume plug-in. Kubernetes can also use secrets to perform actions, such as declaring environment variables, on a pod. Secrets can store any type of data. Kubernetes and OpenShift support different types of secrets, such as service account tokens, SSH keys, and TLS certificates.

The following example shows the definition of a secret:

```yaml
apiVersion: v1
kind: Secret # 1
metadata:
  name: example-secret # 2
  namespace: my-app # 3
type: Opaque # 4
data: # 5
  username: bXl1c2VyCg==
  password: bXlQQDU1Cg==
stringData: # 6
  hostname: myapp.mydomain.com
  secret.properties: |
    property1=valueA
    property2=valueB
```

|     |                                                 |
| --- | ----------------------------------------------- |
| 1   | Resource kind set to `Secret`.                  |
| 2   | Name of the service.                            |
| 3   | Project name where the service resource exists. |
| 4   | Specifies the type of secret.                   |
| 5   | Specifies the encoded string and data.          |
| 6   | Specifies the decoded string and data.          |

### Managing Resources from the Command Line

Kubernetes and RHOCP have many commands to create and modify cluster resources. Some commands are part of core Kubernetes, whereas others are exclusive additions to RHOCP.

The resource management commands generally fall into one of two categories: declarative or imperative. An _imperative_ command instructs what the cluster does. A _declarative_ command defines the state that the cluster attempts to match.

#### Imperative Resource Management

The `create` command is an imperative way to create resources, and is included in both of the `oc` and `kubectl` commands. For example, the following command creates a deployment called `my-app` that creates pods that are based on the specified image.

```sh
[user@host ~]$ oc create deployment my-app --image example.com/my-image:dev
deployment.apps/my-app created
```

Use the `set` command to define attributes on a resource, such as environment variables. For example, the following command adds the `TEAM=red` environment variable to the preceding deployment.

```sh
[user@host ~]$ oc set env deployment/my-app TEAM=red
deployment.apps/my-app updated
```

Another imperative approach to creating a resource is the `run` command. In the following example, the `run` command creates the `example-pod` pod.

```sh
[user@host ~]$ oc run example-pod \ # 1
--image=registry.access.redhat.com/ubi8/httpd-24 \ # 2 
--env GREETING='Hello from the awesome container' \ # 3
--port 8080 # 4
```

pod/example-pod created

|     |                                                                |
| --- | -------------------------------------------------------------- |
| 1   | The pod `.metadata.name` definition.                           |
| 2   | The image for the single container in this pod.                |
| 3   | The environment variable for the single container in this pod. |
| 4   | The port metadata definition.                                  |

The imperative commands are a faster way of creating pods, because such commands do not require a pod object definition. However, developers cannot handle versioning or incrementally change the pod definition.

Generally, developers test a deployment by using imperative commands, and then use the imperative commands to generate the pod object definition. Use the `--dry-run=client` option to avoid creating the object in RHOCP. Additionally, use the `-o yaml` or `-o json` option to configure the definition format.

The following command is an example of generating the YAML definition for the `example-pod` pod:

```sh
[user@host ~]$ oc run example-pod \ --image=registry.access.redhat.com/ubi8/httpd-24 \ --env GREETING='Hello from the awesome container' \ --port 8080 \ --dry-run=client -o yaml
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

Managing resources in this way is imperative, because you are instructing the cluster what to do rather than declaring the intended outcomes.

#### Declarative Resource Management

Using the `create` command does not guarantee that you are creating resources imperatively. For example, providing a manifest declaratively creates the resources that are defined in the YAML file, such as in the following command.

```sh
[user@host ~]$ oc create -f my-app-deployment.yaml
deployment.apps/my-app created
```

RHOCP adds the `new-app` command, which provides another declarative way to create resources. This command uses heuristics to automatically determine which types of resources to create based on the specified parameters. For example, the following command deploys the `my-app` application by creating several resources, including a deployment resource, from a YAML manifest file.

```sh
[user@host ~]$ oc new-app --file=./example/my-app.yaml
_...output omitted..._
--> Creating resources ...
    imagestream.image.openshift.io "my-app" created
    `deployment.apps "my-app" created`
    services "my-app" created
...output omitted...
```

In both of the preceding `create` and `new-app` examples, you are _declaring_ the intended state of the resources, and so they are declarative.

You can also use the `new-app` command with templates for resource management. A template describes the intended state of resources that must be created for an application to run, such as deployment configurations and services. Supplying a template to the `new-app` command is a form of declarative resource management.

The `new-app` command also includes options, such as the `--param` option, that customize an application deployment declaratively. For example, when the `new-app` is used with a template, you can include the `--param` option to override a parameter value in the template.

```sh
[user@host ~]$ *oc new-app --template mysql-persistent \ --param MYSQL_USER=operator --param MYSQL_PASSWORD=myP@55 \ --param MYSQL_DATABASE=mydata \ --param DATABASE_SERVICE_NAME=db
--> Deploying template "db-app/mysql-persistent" to project db-app
...output omitted...
     The following service(s) have been created in your project: db.

            Username: operator
            Password: myP@55
       Database Name: mydata
      Connection URL: mysql://db:3306/
...output omitted...
     * With parameters:
        * Memory Limit=512Mi
        * Namespace=openshift
        * Database Service Name=db
        * MySQL Connection Username=operator
        * MySQL Connection Password=myP@55
        * MySQL root user Password=tlH8BThuVgnIrCon # generated
        * MySQL Database Name=mydata
        * Volume Capacity=1Gi
        * Version of MySQL Image=8.0-el8

--> Creating resources ...
    secret "db" created
    service "db" created
    persistentvolumeclaim "db" created
    deploymentconfig.apps.openshift.io "db" created
--> Success
...output omitted...
```

Similar to the `create` command, you can use the `new-app` command imperatively. When using a container image with the `new-app`, you are instructing the cluster what to do, rather than declaring the intended outcomes. For example, the following command deploys the `example.com/my-app:dev` image by creating several resources, including a deployment resource.

```sh
[user@host ~]$ oc new-app --image example.com/my-app:dev
_...output omitted..._
--> Creating resources ...
    imagestream.image.openshift.io "my-app" created
    `deployment.apps "my-app" created`
_...output omitted..._
```

You can also supply a Git repository to the `new-app` command. The following command creates an application named `httpd24` by using a Git repository.

```sh
[user@host ~]$ oc new-app https://github.com/apache/httpd.git#2.4.56
_...output omitted..._
--> Creating resources ...
    imagestream.image.openshift.io "httpd24" created
    `deployment.apps "httpd24" created`
_...output omitted..._
```

### Retrieving Resource Information

You can supply an `all` argument to commands, to specify a list of common resource types. However, the `all` option does not include every available resource type. Instead, `all` is a shorthand form for a predefined subset of types. When you use this command argument, ensure that `all` includes any intended types to address.

You can view detailed information about a resource, such as the defined parameters, by using the `describe` command. For example, RHOCP provides templates in the `openshift` project to use with the `oc new-app` command. The following example command displays detailed information about the `mysql-ephemeral` template:

```sh
[user@host ~]$ oc describe template mysql-ephemeral -n openshift
Name:		mysql-ephemeral
Namespace:	openshift
...output omitted...
Parameters:
    Name:		MEMORY_LIMIT
    Display Name:	Memory Limit
    Description:	Maximum amount of memory the container can use.
    Required:		true
    Value:		512Mi

    Name:		NAMESPACE
    Display Name:	Namespace
    Description:	The OpenShift Namespace where the ImageStream resides.
    Required:		false
    Value:		openshift
...output omitted...
Objects:
    Secret				                         ${DATABASE_SERVICE_NAME}
    Service				                        ${DATABASE_SERVICE_NAME}
    DeploymentConfig.apps.openshift.io	${DATABASE_SERVICE_NAME}
```

The `describe` command cannot generate structured output, such as the YAML or JSON formats. Without a structured format, the `describe` command cannot parse or filter the output with tools such as JSONPath or Go templates. Instead, use the `get` command to generate and then to parse the structured output of a resource.

## Manage Long-lived and Short-lived Applications by Using the Kubernetes Workload API

### Objectives

- Deploy containerized applications as pods that Kubernetes workload resources manage.

### Kubernetes Workload Resources

The Kubernetes Workloads API consists of resources that represent various types of cluster tasks, jobs, and workloads. This API is composed of various Kubernetes APIs and extensions that simplify deploying and managing applications. These resource types of the Workloads API are most often used:

- Jobs
- Deployments
- Stateful sets

#### Jobs

A _job_ resource represents a one-time task to perform on the cluster. As with most cluster tasks, jobs are executed via pods. If a job's pod fails, then the cluster retries a number of times that the job specifies. The job does not run again after a specified number of successful completions.

Jobs differ from using the `kubectl run` and `oc run` commands; both of the latter create only a pod.

Common uses for jobs include the following tasks:

- Initializing or migrating a database
- Calculating one-off metrics from information within the cluster
- Creating or restoring from a data backup

The following example command creates a job that logs the date and time:

```sh
[user@host ~]$ oc create job \ # 1
date-job \ # 2
--image registry.access.redhat.com/ubi8/ubi \ # 3
-- /bin/bash -c "date" # 4
```

|     |                                                                                     |
| --- | ----------------------------------------------------------------------------------- |
| 1   | Creates a job resource.                                                             |
| 2   | Specifies a job name of `date-job`.                                                 |
| 3   | Sets `registry.access.redhat.com/ubi8/ubi` as the container image for the job pods. |
| 4   | Specifies the command to run within the pods.                                       |

You can also create a job from the web console by clicking the Workloads → Jobs menu. Click Create Job and customize the YAML manifest.

#### Cron Jobs

A _cron job_ resource builds on a regular job resource by enabling you to specify how often the job should run. Cron jobs are useful for creating periodic and recurring tasks, such as backups or report generation. Cron jobs can also schedule individual tasks for a specific time, such as to schedule a job for a low activity period. Similar to the `crontab` (cron table) file on a Linux system, the `CronJob` resource uses the `Cron` format for scheduling. A `CronJob` resource creates a `job` resource based on the configured time zone on the control plane node that runs the cron job controller.

The following example command creates a cron job named `date` that prints the system date and time every minute:

```sh
[user@host ~]$ oc create cronjob date-cronjob \ # 1
--image registry.access.redhat.com/ubi8/ubi \ # 2
--schedule "*/1 * * * *" \ # 3
-- date # 4
```

|     |                                                                                         |
| --- | --------------------------------------------------------------------------------------- |
| 1   | Creates a cron job resource with a name of `date-cronjob`.                              |
| 2   | Sets the `registry.access.redhat.com/ubi8/ubi` as the container image for the job pods. |
| 3   | Specifies the schedule for the job in `Cron` format.                                    |
| 4   | The command to execute within the pods.                                                 |

You can also create a cronjob from the web console by clicking the Workloads → CronJobs menu. Click Create CronJob and customize the YAML manifest.

#### Deployments

A _deployment_ creates a replica set to maintain pods. A _replica set_ maintains a specified number of replicas of a pod. Replica sets use selectors, such as a label, to identify pods that are part of the set. Pods are created or removed until the replicas reach the number that the deployment specifies. Replica sets are not managed directly. Instead, deployments indirectly manage replica sets.

Unlike a job, a deployment's pods are re-created after crashing or deletion. The reason is that deployments use replica sets.

The following example command creates a deployment:

```sh
[user@host ~]$ oc create deployment \ # 1
my-deployment \ # 2
--image registry.access.redhat.com/ubi8/ubi \ # 3
--replicas 3 # 4
```

|     |                                                                                 |
| --- | ------------------------------------------------------------------------------- |
| 1   | Creates a deployment resource.                                                  |
| 2   | Specifies `my-deployment` as the deployment name.                               |
| 3   | Sets `registry.access.redhat.com/ubi8/ubi` as the container image for the pods. |
| 4   | Sets the deployment to maintain three instances of the pod.                     |

You can also create a deployment from the web console by clicking the Deployments tab in the Workloads menu.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/workloads/assets/deployments.png)|

Click Create Deployment and customize the form or the YAML manifest.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/workloads/assets/formdeployment.png)|

Pods in a replica set are identical and match the pod template in the replica set definition. If the number of replicas is not met, then a new pod is created by using the template. For example, if a pod crashes or is otherwise deleted, then a new one is created to replace it.

_Labels_ are a type of resource metadata that are represented as string key-value pairs. A label indicates a common trait for resources with that label. For example, you might attach a `layer=frontend` label to resources that relate to an application's UI components.

Many `oc` and `kubectl` commands accept a label to filter affected resources. For example, the following command deletes all pods with the `environment=testing` label:

```sh
[user@host ~]$ oc delete pod -l environment=testing
```

By liberally applying labels to resources, you can cross-reference resources and craft precise selectors. A _selector_ is a query object that describes the attributes of matching resources.

Certain resources use selectors to find other resources. In the following YAML excerpt, an example replica set uses a selector to match its pods.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
_...output omitted..._
spec:
  replicas: 1
  `selector:     matchLabels:       app: httpd       pod-template-hash: 7c84fbdb57`
_...output omitted..._
```

#### Stateful Sets

Like deployments, _stateful sets_ manage a set of pods based on a container specification. However, each pod that a stateful set creates is unique. Pod uniqueness is useful when, for example, a pod needs a unique network identifier or persistent storage.

As their name implies, stateful sets are for pods that require state within the cluster. Deployments are used for stateless pods.

Stateful sets are covered further in a later chapter.

## Kubernetes Pod and Service Networks

### Objectives

- Interconnect applications pods inside the same cluster by using Kubernetes services.

### The Software-defined Network

Kubernetes implements software-defined networking (SDN) to manage the network infrastructure of the cluster and user applications. The SDN is a virtual network that encompasses all cluster nodes. The virtual network enables communication between any container or pod inside the cluster. Cluster node processes that Kubernetes pods manage can access the SDN. However, the SDN is not accessible from outside the cluster, nor to regular processes on cluster nodes. With the software-defined networking model, you can manage network services through the abstraction of several networking layers.

With the SDN, you can manage the network traffic and network resources programmatically, so that the organization teams can decide how to expose their applications. The SDN implementation creates a model that is compatible with traditional networking practices. It makes pods akin to virtual machines in terms of port allocation, IP address leasing, and reservation.

With the SDN design, you do not need to change how application components communicate with each other, which helps to containerize legacy applications. If your application is composed of many services that communicate over the TCP/UDP stack, then this approach still works, because containers in a pod use the same network stack.

The following diagram shows how all pods are connected to a shared network:

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/services/assets/network-sdn-pod-network.svg)|

Figure 4.5: How the Kubernetes SDN manages the network

Among the many features of SDN, with open standards, vendors can propose their solutions for centralized management, dynamic routing, and tenant isolation.

### Kubernetes Networking

Networking in Kubernetes provides a scalable means of communication between containers.

Kubernetes networking provides the following capabilities:

- Highly coupled container-to-container communications
- Pod-to-pod communications
- Pod-to-service communications
- External-to-service communication: covered in [the section called “ Scale and Expose Applications to External Access ”](https://rol.redhat.com/rol/app/courses/do180-4.14/pages/ch04s07)

Kubernetes automatically assigns an IP address to every pod. However, pod IP addresses are unstable, because pods are ephemeral. Pods are constantly created and destroyed across the nodes in the cluster. For example, when you deploy a new version of your application, Kubernetes destroys the existing pods and then deploys new ones.

All containers within a pod share networking resources. The IP address and MAC address that are assigned to the pod are shared among all containers in the pod. Thus, all containers within a pod can reach each other's ports through the loopback address, `localhost`. Ports that are bound to localhost are available to all containers that run within the pod, but never to containers outside it.

By default, the pods can communicate with each other even if they run on different cluster nodes or belong to different Kubernetes namespaces. Every pod is assigned an IP address in a flat shared networking namespace that has full communication with other physical computers and containers across the network. All pods are assigned a unique IP address from a Classless Inter-Domain Routing (CIDR) range of host addresses. The shared address range places all pods in the same subnet.

Because all the pods are on the same subnet, pods on all nodes can communicate with pods on any other node without the aid of Network Address Translation (NAT). Kubernetes also provides a service subnet, which links the stable IP address of a service resource to a set of specified pods. The traffic is forwarded in a transparent way to the pods; an agent (depending on the network mode that you use) manages routing rules to route traffic to the pods that match the service resource selectors. Thus, pods can be treated much like Virtual Machines (VMs) or physical hosts from the perspective of port allocation, networking, naming, service discovery, load balancing, application configuration, and migration. Kubernetes implements this infrastructure by managing the SDN.

The following illustration gives further insight into how the infrastructure components work along with the pod and service subnets to enable network access between pods inside an OpenShift instance.

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/services/assets/sdn-relation.svg)

Figure 4.6: Network access between pods in a cluster

The shared networking namespace of pods enables a straightforward communication model. However, the dynamic nature of pods presents a problem. Pods can be added on the fly to handle increased traffic. Likewise, pods can be dynamically scaled down. If a pod fails, then Kubernetes automatically replaces the pod with a new one. These events change pod IP addresses.

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/services/assets/multicontainer-design-lecture.svg)

Figure 4.7: Problem with direct access to pods

In the diagram, the `Before` side shows the `Front-end` container that is running in a pod with a `10.8.0.1` IP address. The container also refers to a `Back-end` container that is running in a pod with a `10.8.0.2` IP address. In this example, an event occurs that causes the `Back-end` container to fail. A pod can fail for many reasons. In response to the failure, Kubernetes creates a pod for the `Back-end` container that uses a new IP address of `10.8.0.4`. From the `After` side of the diagram, the `Front-end` container now has an invalid reference to the `Back-end` container because of the IP address change. Kubernetes resolves this problem with `service` resources.

#### Using Services

Containers inside Kubernetes pods must not connect directly to each other's dynamic IP address. Instead, Kubernetes assigns a stable IP address to a service resource that is linked to a set of specified pods. The service then acts as a virtual network load balancer for the pods that are linked to the service.

If the pods are restarted, replicated, or rescheduled to different nodes, then the service endpoints are updated, thus providing scalability and fault tolerance for your applications. Unlike the IP addresses of pods, the IP addresses of services do not change.

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/services/assets/multicontainer-fail-with-service.svg)

Figure 4.8: Services resolve pod failure issues

In the diagram, the `Before` side shows that the `Front-end` container now holds a reference to the stable IP address of the `Back-end` service, instead of to the IP address of the pod that is running the `Back-end` container. When the `Back-end` container fails, Kubernetes creates a pod with the `New back-end` container to replace the failed pod. In response to the change, Kubernetes removes the failed pod from the service's host list, or service endpoints, and then adds the IP address of the `New back-end` container pod to the service endpoints. With the addition of the service, requests from the `Front-end` container to the `Back-end` container continue to work, because the service is dynamically updated with the IP address change. A service provides a permanent, static IP address for a group of pods that belong to the same deployment or replica set for an application. Until you delete the service, the assigned IP address does not change, and the cluster does not reuse it.

Most real-world applications do not run as a single pod. Applications need to scale horizontally. Multiple pods run the same containers to meet a growing user demand. A _Deployment_ resource manages multiple pods that execute the same container. A service provides a single IP address for the whole set, and provides load-balancing for client requests among the member pods.

With services, containers in one pod can open network connections to containers in another pod. The pods, which the service tracks, are not required to exist on the same compute node or in the same namespace or project. Because a service provides a stable IP address for other pods to use, a pod also does not need to discover the new IP address of another pod after a restart. The service provides a stable IP address to use, no matter which compute node runs the pod after each restart.

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/services/assets/network-sdn-service-network.svg)

Figure 4.9: Service with pods on many nodes

The _SERVICE_ object provides a stable IP address for the _CLIENT_ container on _NODE X_ to send a request to any one of the _API_ containers.

Kubernetes uses labels on the pods to select the pods that are associated with a service. To include a pod in a service, the pod labels must include each of the `selector` fields of the service.

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/services/assets/network-sdn-service-selector.svg)

Figure 4.10: Service selector match to pod labels

In this example, the selector has a key-value pair of `app: myapp`. Thus, pods with a matching label of `app: myapp` are included in the set that is associated with the service. The _selector_ attribute of a service is used to identify the set of pods that form the endpoints for the service. Each pod in the set is a an endpoint for the service.

To create a service for a deployment, use the `oc expose` command:

```sh
[user@host ~]$ oc expose deployment/<deployment-name> [--selector <selector>]
[--port <port>][--target-port <target port>][--protocol <protocol>][--name <name>]
```

The `oc expose` command can use the `--selector` option to specify the label selectors to use. When the command is used without the `--selector` option, the command applies a selector to match the replication controller or replica set.

The `--port` option of the `oc expose` command specifies the port that the service listens on. This port is available only to pods within the cluster. If a port value is not provided, then the port is copied from the deployment configuration.

The `--target-port` option of the `oc expose` command specifies the name or number of the container port that the service uses to communicate with the pods.

The `--protocol` option determines the network protocol for the service. TCP is used by default.

The `--name` option of the `oc expose` command can explicitly name the service. If not specified, the service uses the same name that is provided for the deployment.

To view the selector that a service uses, use the `-o wide` option with the `oc get` command.

```sh
[user@host ~]$ oc get service db-pod -o wide
NAME    TYPE        CLUSTER-IP      EXTERNAL-IP PORT(S)     AGE     SELECTOR
db-pod  ClusterIP   172.30.108.92   <none>      3306/TCP    108s    app=db-pod
```

In this example, `db-pod` is the name of the service. Pods must use the `app=db-pod` label to be included in the host list for the `db-pod` service. To see the endpoints that a service uses, use the `oc get endpoints` command.

```sh
[user@host ~]$ oc get endpoints
NAME     ENDPOINTS                       AGE
db-pod   10.8.0.86:3306,10.8.0.88:3306   27s
```

This example illustrates a service with two pods in the host list. The `oc get endpoints` command returns the service endpoints in the current selected project. Add the name of the service to the command to show only the endpoints of a single service. Use the `--namepace` option to view the endpoints in a different namespace.

Use the `oc describe deployment <deployment name>` command to view the deployment selector.

```sh
[user@host ~]$ oc describe deployment db-pod
Name:                   db-pod
Namespace:              deploy-services
CreationTimestamp:      Wed, 18 Jan 2023 17:46:03 -0500
Labels:                 app=db-pod
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=db-pod
_...output omitted..._
```

You can view or parse the selector from the `YAML` or `JSON` output for the deployment resource from the `spec.selector.matchLabels` object. In this example, the `-o yaml` option of the `oc get` command returns the `selector` label that the deployment uses.

```sh
[user@host ~]$ oc get deployment/<deployment_name> -o yaml
..._`output ommitted`_...
  selector:
    matchLabels:
      app: db-pod
...output ommitted...
```

### Kubernetes DNS for Service Discovery

Kubernetes uses an internal Domain Name System (DNS) server that the DNS operator deploys. The DNS operator creates a default cluster DNS name, and assigns DNS names to services that you define. The DNS operator implements the DNS API from the `operator.openshift.io` API group. The operator deploys CoreDNS; creates a service resource for the CoreDNS; and then configures the `kubelet` to instruct pods to use the CoreDNS service IP for name resolution. When a service does not have a cluster IP address, the DNS operator assigns to the service a DNS record that resolves to the set of IP addresses of the pods behind the service.

The DNS server discovers a service from a pod by using the internal DNS server, which is visible only to pods. Each service is dynamically assigned a _Fully Qualified Domain Name_ (FQDN) that uses the following format:

_`SVC-NAME`_._`PROJECT-NAME`_.svc._`CLUSTER-DOMAIN`_

When a pod is created, Kubernetes provides the container with a `/etc/resolv.conf` file with similar contents to the following items:

```sh
[user@host ~]$ cat /etc/resolv.conf
search deploy-services.svc.cluster.local svc.cluster.local ...
nameserver 172.30.0.10
options ndots:5
```

In this example, `deploy-services` is the project name for the pod, and `cluster.local` is the cluster domain.

The `nameserver` directive provides the IP address of the Kubernetes internal DNS server. The `options ndots` directive specifies the number of dots that must appear in a name to qualify for an initial absolute query. Alternative hostname values are derived by appending values from the `Search` directive to the name that is sent to the DNS server.

In the `search` directive in this example, the `svc.cluster.local` entry enables any pod to communicate with another pod in the same cluster by using the service name and project name:

_`SVC-NAME`_._`PROJECT-NAME`_

The first entry in the `search` directive enables a pod to use the service name to specify another pod in the same project. In RHOCP, a project is also the namespace for the pod. The service name alone is sufficient for pods in the same RHOCP project:

_`SVC-NAME`_

### Kubernetes Networking Drivers

Container Network Interface (CNI) plug-ins provide a common interface between the network provider and the container runtime. CNI defines the specifications for plug-ins that configure network interfaces inside containers. Plug-ins that are written to the specification enable different network providers to control the RHOCP cluster network.

Red Hat provides the following CNI plug-ins for a RHOCP cluster:

- OVN-Kubernetes: The default plug-in for first-time installations of RHOCP, starting with RHOCP 4.10.
- OpenShift SDN: An earlier plug-in from RHOCP 3.x; it is incompatible with some later features of RHOCP 4.x.
- Kuryr: A plug-in for integration and performance on OpenStack deployments.

Certified CNI-plugins from other vendors are also compatible with an RHOCP cluster.

The SDN uses CNI plug-ins to create Linux namespaces to partition the usage of resources and processes on physical and virtual hosts. With this implementation, containers inside pods can share network resources, such as devices, IP stacks, firewall rules, and routing tables. The SDN allocates a unique routable IP to each pod, so that you can access the pod from any other service in the same network.

In OpenShift 4.14, OVN-Kubernetes is the default network provider.

OVN-Kubernetes uses Open Virtual Network (OVN) to manage the cluster network. A cluster that uses the OVN-Kubernetes plug-in also runs Open vSwitch (OVS) on each node. OVN configures OVS on each node to implement the declared network configuration.

#### The OpenShift Cluster Network Operator

RHOCP provides a _Cluster Network Operator_ (CNO) that configures OpenShift cluster networking. The CNO is a OpenShift cluster operator that loads and configures Container Network Interface (CNI) plug-ins. As a cluster administrator, execute the following command to observe the status of the CNO:

```bash
[user@host ~]$ **`oc get -n openshift-network-operator deployment/network-operator`**
NAME              READY   UP-TO-DATE  AVAILABLE   AGE
network-operator  1/1     1           1           41d
```

An administrator configures the cluster network operator at installation time. To see the configuration, use the following command:

```bash
[user@host ~]$ oc describe network.config/cluster
Name: cluster
_...output omitted..._
Spec:
Cluster Network:
Cidr: 10.8.0.0/14 # 1
Host Prefix: 23
External IP:
Policy:
Network Type: OVNKubernetes
Service Network:
172.30.0.0/16 # 2
_...output omitted..._
```

|     |                                                                                    |
| --- | ---------------------------------------------------------------------------------- |
| 1   | The Cluster Network CIDR defines the range of IPs for all pods in the cluster.     |
| 2   | The Service Network CIDR defines the range of IPs for all services in the cluster. |

## Kubernetes Pod and Service Networks

### Objectives

- Interconnect applications pods inside the same cluster by using Kubernetes services.

### The Software-defined Network

Kubernetes implements software-defined networking (SDN) to manage the network infrastructure of the cluster and user applications. The SDN is a virtual network that encompasses all cluster nodes. The virtual network enables communication between any container or pod inside the cluster. Cluster node processes that Kubernetes pods manage can access the SDN. However, the SDN is not accessible from outside the cluster, nor to regular processes on cluster nodes. With the software-defined networking model, you can manage network services through the abstraction of several networking layers.

With the SDN, you can manage the network traffic and network resources programmatically, so that the organization teams can decide how to expose their applications. The SDN implementation creates a model that is compatible with traditional networking practices. It makes pods akin to virtual machines in terms of port allocation, IP address leasing, and reservation.

With the SDN design, you do not need to change how application components communicate with each other, which helps to containerize legacy applications. If your application is composed of many services that communicate over the TCP/UDP stack, then this approach still works, because containers in a pod use the same network stack.

The following diagram shows how all pods are connected to a shared network:

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/services/assets/network-sdn-pod-network.svg)|

Figure 4.5: How the Kubernetes SDN manages the network

Among the many features of SDN, with open standards, vendors can propose their solutions for centralized management, dynamic routing, and tenant isolation.

### Kubernetes Networking

Networking in Kubernetes provides a scalable means of communication between containers.

Kubernetes networking provides the following capabilities:

- Highly coupled container-to-container communications
- Pod-to-pod communications
- Pod-to-service communications
- External-to-service communication: covered in [the section called “ Scale and Expose Applications to External Access ”](https://rol.redhat.com/rol/app/courses/do180-4.14/pages/ch04s07)

Kubernetes automatically assigns an IP address to every pod. However, pod IP addresses are unstable, because pods are ephemeral. Pods are constantly created and destroyed across the nodes in the cluster. For example, when you deploy a new version of your application, Kubernetes destroys the existing pods and then deploys new ones.

All containers within a pod share networking resources. The IP address and MAC address that are assigned to the pod are shared among all containers in the pod. Thus, all containers within a pod can reach each other's ports through the loopback address, `localhost`. Ports that are bound to localhost are available to all containers that run within the pod, but never to containers outside it.

By default, the pods can communicate with each other even if they run on different cluster nodes or belong to different Kubernetes namespaces. Every pod is assigned an IP address in a flat shared networking namespace that has full communication with other physical computers and containers across the network. All pods are assigned a unique IP address from a Classless Inter-Domain Routing (CIDR) range of host addresses. The shared address range places all pods in the same subnet.

Because all the pods are on the same subnet, pods on all nodes can communicate with pods on any other node without the aid of Network Address Translation (NAT). Kubernetes also provides a service subnet, which links the stable IP address of a service resource to a set of specified pods. The traffic is forwarded in a transparent way to the pods; an agent (depending on the network mode that you use) manages routing rules to route traffic to the pods that match the service resource selectors. Thus, pods can be treated much like Virtual Machines (VMs) or physical hosts from the perspective of port allocation, networking, naming, service discovery, load balancing, application configuration, and migration. Kubernetes implements this infrastructure by managing the SDN.

The following illustration gives further insight into how the infrastructure components work along with the pod and service subnets to enable network access between pods inside an OpenShift instance.

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/services/assets/sdn-relation.svg)

Figure 4.6: Network access between pods in a cluster

The shared networking namespace of pods enables a straightforward communication model. However, the dynamic nature of pods presents a problem. Pods can be added on the fly to handle increased traffic. Likewise, pods can be dynamically scaled down. If a pod fails, then Kubernetes automatically replaces the pod with a new one. These events change pod IP addresses.

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/services/assets/multicontainer-design-lecture.svg)

Figure 4.7: Problem with direct access to pods

In the diagram, the `Before` side shows the `Front-end` container that is running in a pod with a `10.8.0.1` IP address. The container also refers to a `Back-end` container that is running in a pod with a `10.8.0.2` IP address. In this example, an event occurs that causes the `Back-end` container to fail. A pod can fail for many reasons. In response to the failure, Kubernetes creates a pod for the `Back-end` container that uses a new IP address of `10.8.0.4`. From the `After` side of the diagram, the `Front-end` container now has an invalid reference to the `Back-end` container because of the IP address change. Kubernetes resolves this problem with `service` resources.

#### Using Services

Containers inside Kubernetes pods must not connect directly to each other's dynamic IP address. Instead, Kubernetes assigns a stable IP address to a service resource that is linked to a set of specified pods. The service then acts as a virtual network load balancer for the pods that are linked to the service.

If the pods are restarted, replicated, or rescheduled to different nodes, then the service endpoints are updated, thus providing scalability and fault tolerance for your applications. Unlike the IP addresses of pods, the IP addresses of services do not change.

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/services/assets/multicontainer-fail-with-service.svg)

Figure 4.8: Services resolve pod failure issues

In the diagram, the `Before` side shows that the `Front-end` container now holds a reference to the stable IP address of the `Back-end` service, instead of to the IP address of the pod that is running the `Back-end` container. When the `Back-end` container fails, Kubernetes creates a pod with the `New back-end` container to replace the failed pod. In response to the change, Kubernetes removes the failed pod from the service's host list, or service endpoints, and then adds the IP address of the `New back-end` container pod to the service endpoints. With the addition of the service, requests from the `Front-end` container to the `Back-end` container continue to work, because the service is dynamically updated with the IP address change. A service provides a permanent, static IP address for a group of pods that belong to the same deployment or replica set for an application. Until you delete the service, the assigned IP address does not change, and the cluster does not reuse it.

Most real-world applications do not run as a single pod. Applications need to scale horizontally. Multiple pods run the same containers to meet a growing user demand. A _Deployment_ resource manages multiple pods that execute the same container. A service provides a single IP address for the whole set, and provides load-balancing for client requests among the member pods.

With services, containers in one pod can open network connections to containers in another pod. The pods, which the service tracks, are not required to exist on the same compute node or in the same namespace or project. Because a service provides a stable IP address for other pods to use, a pod also does not need to discover the new IP address of another pod after a restart. The service provides a stable IP address to use, no matter which compute node runs the pod after each restart.

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/services/assets/network-sdn-service-network.svg)

Figure 4.9: Service with pods on many nodes

The _SERVICE_ object provides a stable IP address for the _CLIENT_ container on _NODE X_ to send a request to any one of the _API_ containers.

Kubernetes uses labels on the pods to select the pods that are associated with a service. To include a pod in a service, the pod labels must include each of the `selector` fields of the service.

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/services/assets/network-sdn-service-selector.svg)

Figure 4.10: Service selector match to pod labels

In this example, the selector has a key-value pair of `app: myapp`. Thus, pods with a matching label of `app: myapp` are included in the set that is associated with the service. The _selector_ attribute of a service is used to identify the set of pods that form the endpoints for the service. Each pod in the set is a an endpoint for the service.

To create a service for a deployment, use the `oc expose` command:

```sh
[user@host ~]$ oc expose deployment/<deployment-name> [--selector <selector>]
[--port <port>][--target-port <target port>][--protocol <protocol>][--name <name>]
```

The `oc expose` command can use the `--selector` option to specify the label selectors to use. When the command is used without the `--selector` option, the command applies a selector to match the replication controller or replica set.

The `--port` option of the `oc expose` command specifies the port that the service listens on. This port is available only to pods within the cluster. If a port value is not provided, then the port is copied from the deployment configuration.

The `--target-port` option of the `oc expose` command specifies the name or number of the container port that the service uses to communicate with the pods.

The `--protocol` option determines the network protocol for the service. TCP is used by default.

The `--name` option of the `oc expose` command can explicitly name the service. If not specified, the service uses the same name that is provided for the deployment.

To view the selector that a service uses, use the `-o wide` option with the `oc get` command.

```sh
[user@host ~]$ oc get service db-pod -o wide
NAME    TYPE        CLUSTER-IP      EXTERNAL-IP PORT(S)     AGE     SELECTOR
db-pod  ClusterIP   172.30.108.92   <none>      3306/TCP    108s    app=db-pod
```

In this example, `db-pod` is the name of the service. Pods must use the `app=db-pod` label to be included in the host list for the `db-pod` service. To see the endpoints that a service uses, use the `oc get endpoints` command.

```sh
[user@host ~]$ oc get endpoints
NAME     ENDPOINTS                       AGE
db-pod   10.8.0.86:3306,10.8.0.88:3306   27s
```

This example illustrates a service with two pods in the host list. The `oc get endpoints` command returns the service endpoints in the current selected project. Add the name of the service to the command to show only the endpoints of a single service. Use the `--namepace` option to view the endpoints in a different namespace.

Use the `oc describe deployment <deployment name>` command to view the deployment selector.

```sh
[user@host ~]$ oc describe deployment db-pod
Name:                   db-pod
Namespace:              deploy-services
CreationTimestamp:      Wed, 18 Jan 2023 17:46:03 -0500
Labels:                 app=db-pod
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=db-pod
...output omitted...
```

You can view or parse the selector from the `YAML` or `JSON` output for the deployment resource from the `spec.selector.matchLabels` object. In this example, the `-o yaml` option of the `oc get` command returns the `selector` label that the deployment uses.

```sh
[user@host ~]$ oc get deployment/<deployment_name> -o yaml
..._`output ommitted`_...
  selector:
    matchLabels:
      app: db-pod
...output ommitted...
```

### Kubernetes DNS for Service Discovery

Kubernetes uses an internal Domain Name System (DNS) server that the DNS operator deploys. The DNS operator creates a default cluster DNS name, and assigns DNS names to services that you define. The DNS operator implements the DNS API from the `operator.openshift.io` API group. The operator deploys CoreDNS; creates a service resource for the CoreDNS; and then configures the `kubelet` to instruct pods to use the CoreDNS service IP for name resolution. When a service does not have a cluster IP address, the DNS operator assigns to the service a DNS record that resolves to the set of IP addresses of the pods behind the service.

The DNS server discovers a service from a pod by using the internal DNS server, which is visible only to pods. Each service is dynamically assigned a _Fully Qualified Domain Name_ (FQDN) that uses the following format:

_`SVC-NAME`_._`PROJECT-NAME`_.svc._`CLUSTER-DOMAIN`_

When a pod is created, Kubernetes provides the container with a `/etc/resolv.conf` file with similar contents to the following items:

```sh
[user@host ~]$ *at /etc/resolv.conf
search deploy-services.svc.cluster.local svc.cluster.local ...
nameserver 172.30.0.10
options ndots:5
```

In this example, `deploy-services` is the project name for the pod, and `cluster.local` is the cluster domain.

The `nameserver` directive provides the IP address of the Kubernetes internal DNS server. The `options ndots` directive specifies the number of dots that must appear in a name to qualify for an initial absolute query. Alternative hostname values are derived by appending values from the `Search` directive to the name that is sent to the DNS server.

In the `search` directive in this example, the `svc.cluster.local` entry enables any pod to communicate with another pod in the same cluster by using the service name and project name:

_`SVC-NAME`_._`PROJECT-NAME`_

The first entry in the `search` directive enables a pod to use the service name to specify another pod in the same project. In RHOCP, a project is also the namespace for the pod. The service name alone is sufficient for pods in the same RHOCP project:

_`SVC-NAME`_

### Kubernetes Networking Drivers

Container Network Interface (CNI) plug-ins provide a common interface between the network provider and the container runtime. CNI defines the specifications for plug-ins that configure network interfaces inside containers. Plug-ins that are written to the specification enable different network providers to control the RHOCP cluster network.

Red Hat provides the following CNI plug-ins for a RHOCP cluster:

- OVN-Kubernetes: The default plug-in for first-time installations of RHOCP, starting with RHOCP 4.10.
- OpenShift SDN: An earlier plug-in from RHOCP 3.x; it is incompatible with some later features of RHOCP 4.x.
- Kuryr: A plug-in for integration and performance on OpenStack deployments.

Certified CNI-plugins from other vendors are also compatible with an RHOCP cluster.

The SDN uses CNI plug-ins to create Linux namespaces to partition the usage of resources and processes on physical and virtual hosts. With this implementation, containers inside pods can share network resources, such as devices, IP stacks, firewall rules, and routing tables. The SDN allocates a unique routable IP to each pod, so that you can access the pod from any other service in the same network.

In OpenShift 4.14, OVN-Kubernetes is the default network provider.

OVN-Kubernetes uses Open Virtual Network (OVN) to manage the cluster network. A cluster that uses the OVN-Kubernetes plug-in also runs Open vSwitch (OVS) on each node. OVN configures OVS on each node to implement the declared network configuration.

#### The OpenShift Cluster Network Operator

RHOCP provides a _Cluster Network Operator_ (CNO) that configures OpenShift cluster networking. The CNO is a OpenShift cluster operator that loads and configures Container Network Interface (CNI) plug-ins. As a cluster administrator, execute the following command to observe the status of the CNO:

```sh
[user@host ~]$ oc get -n openshift-network-operator deployment/network-operator
NAME              READY   UP-TO-DATE  AVAILABLE   AGE
network-operator  1/1     1           1           41d
```

An administrator configures the cluster network operator at installation time. To see the configuration, use the following command:

```sh
[user@host ~]$ oc describe network.config/cluster
Name: cluster
...output omitted...
spec:
Cluster Network:
Cidr: 10.8.0.0/14 # 1
Host Prefix: 23
External IP:
Policy:
Network Type: OVNKubernetes
Service Network:
172.30.0.0/16 # 2
...output omitted...
```

|     |                                                                                    |
| --- | ---------------------------------------------------------------------------------- |
| 1   | The Cluster Network CIDR defines the range of IPs for all pods in the cluster.     |
| 2   | The Service Network CIDR defines the range of IPs for all services in the cluster. |

## Scale and Expose Applications to External Access

### Objectives

- Expose applications to clients outside the cluster by using Kubernetes ingress and OpenShift routes.
    

### IP Addresses for Pods and Services

Most real-world applications do not run as a single pod. Because applications need to scale horizontally, many pods run the same containers from the same pod resource definition, to meet growing user demand. A service defines a single IP/port combination, and provides a single IP address to a pool of pods, and a load-balancing client request among member pods.

By default, services connect clients to pods in a round-robin fashion, and each service is assigned a unique IP address for clients to connect to. This IP address comes from an internal OpenShift virtual network, which although distinct from the pods' internal network, is visible only to pods. Each pod that matches the selector is added to the service resource as an endpoint.

Containers inside Kubernetes pods must not connect to each other's dynamic IP address directly. Services resolve this problem by linking more stable IP addresses from the SDN to the pods. If pods are restarted, replicated, or rescheduled to different nodes, then services are updated, to provide scalability and fault tolerance.

### Service Types

You can choose between several service types depending on your application needs, cluster infrastructure, and security requirements.

ClusterIP

This type is the default, unless you explicitly specify a type for a service. The ClusterIP type exposes the service on a cluster-internal IP address. If you choose this value, then the service is reachable only from within the cluster.

The ClusterIP service type is used for pod-to-pod routing within the RHOCP cluster, and enables pods to communicate with and to access each other. IP addresses for the ClusterIP services are assigned from a dedicated service network that is accessible only from inside the cluster. Most applications should use this service type, for which Kubernetes automates the management.

Load balancer

This resource instructs RHOCP to activate a load balancer in a cloud environment. A load balancer instructs Kubernetes to interact with the cloud provider that the cluster is running in, to provision a load balancer. The load balancer then provides an externally accessible IP address to the application.

Take all necessary precautions before deploying this service type. Load balancers are typically too expensive to assign one for each application in a cluster. Furthermore, applications that use this service type become accessible from networks outside the cluster. Additional security configuration is required to prevent unintended access.

NodePort

With this method, Kubernetes exposes a service on a port on the node IP address. The port is exposed on all cluster nodes, and each node redirects traffic to the endpoints (pods) of the service.

A NodePort service requires allowing direct network connections to a cluster node, which is a security risk.

ExternalName

This service tells Kubernetes that the DNS name in the `externalName` field is the location of the resource that backs the service. When a DNS request is made against the Kubernetes DNS server, it returns the `externalName` in a _Canonical Name (CNAME)_ record, and directs the client to look up the returned name to get the IP address.

### Using Routes for External Connectivity

RHOCP provides resources to expose your applications to external networks outside the cluster. You can expose HTTP and HTTPS traffic, TCP applications, and also non-TCP traffic. However, you should expose only HTTP and TLS-based applications to external access. Applications that use other protocols, such as databases, are usually not exposed to external access (from outside a cluster). Routes and ingress are the main resources for handling ingress traffic.

RHOCP provides the `route` resource to expose your applications to external networks. With routes, you can access your application with a unique hostname that is publicly accessible. Routes rely on a Kubernetes ingress controller to redirect the traffic from the public IP address to pods. By default, Kubernetes provides an ingress controller, starting from the 1.24 release. For RHOCP clusters, the OpenShift ingress operator provides the ingress controller. RHOCP clusters can also use various third-party ingress controllers that can be deployed in parallel with the OpenShift ingress controller.

Routes provide ingress traffic to services in the cluster. Routes were created before Kubernetes ingress objects, and provide more features. Routes provide advanced features that Kubernetes ingress controllers might not support through a standard interface, such as TLS re-encryption, TLS passthrough, and split traffic for blue-green deployments.

To create a route with the `oc` CLI, use the ``oc expose service _`service-name`_`` command. Include the `--hostname` option to provide a custom hostname for the route.

```sh
[user@host ~]$ oc expose service api-frontend \ --hostname api.apps.acme.com
```

If you omit the hostname, then RHOCP automatically generates a hostname with the following structure: `<route-name>-<project-name>.<default-domain>`. For example, if you create a `frontend` route in an `api` project, in a cluster that uses `apps.example.com` as the wildcard domain, then the route hostname is as follows:

```
frontend-api.apps.example.com
```

### Important

The DNS server that hosts the wildcard domain is unaware of any route hostnames; it resolves any name only to the configured IPs. Only the RHOCP router knows about route hostnames, and treats each one as an HTTP virtual host.

Invalid wildcard domain hostnames, or hostnames that do not correspond to any route, are blocked by the RHOCP router and result in an HTTP 503 error.

Consider the following settings when creating a route:

- The name of a service. The route uses the service to determine the pods to direct the traffic to.
- A hostname for the route. A route is always a subdomain of your cluster wildcard domain. For example, if you are using a wildcard domain of `apps.dev-cluster.acme.com`, and need to expose a `frontend` service through a route, then the route name is as follows:

```
frontend.apps.dev-cluster.acme.com.
```

RHOCP can also automatically generate a hostname for the route.
- An optional path, for path-based routes.
- A target port that the application listens to. The target port corresponds to the port that you define in the `targetPort` key of the service.
- An encryption strategy, depending on whether you need a secure or an insecure route.

The following listing shows a minimal definition for a route:

```yaml
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: a-simple-route # 1
  labels: # 2
    app: API
    name: api-frontend
spec:
  host: api.apps.acme.com # 3
  to:
    kind: Service
    name: api-frontend # 4
  port: 8080 # 5
    targetPort: 8443
```

|     |                                                                                                                                                                       |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | The name of the route. This name must be unique.                                                                                                                      |
| 2   | A set of labels that you can use as selectors.                                                                                                                        |
| 3   | The hostname of the route. This hostname must be a subdomain of your wildcard domain, because RHOCP routes the wildcard domain to the routers.                        |
| 4   | The service to redirect the traffic to. Although you use a service name, the route uses this information only to determine the list of pods that receive the traffic. |
| 5   | Port mapping from a router to an endpoint in the service endpoints. The target port on pods that are selected by the service that this route points to.               |

**Note

Some ecosystem components have an integration with ingress resources, but not with route resources. For this case, RHOCP automatically creates managed route objects when an ingress object is created. These route objects are deleted when the corresponding ingress objects are deleted.

You can delete a route by using the ``oc delete route _`route-name`_`` command.

```sh
[user@host ~]$ oc delete route myapp-route
```

You can also expose a service from the web console by clicking the Networking → Routes menu. Click Create Route and customize the name, the hostname, the path, and the service to route to by using the form view or the YAML manifest.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/deploy/routes/assets/routeform.png)|

### Using Ingress Objects for External Connectivity

An ingress is a Kubernetes resource that provides some of the same features as routes (which are an RHOCP resource). Ingress objects accept external requests and transfer the requests based on the route. You can enable only certain types of traffic: HTTP, HTTPS and server name identification (SNI), and TLS with SNI. Standard Kubernetes ingress resources are typically minimal. Many common features that applications rely on, such as TLS termination, path redirecting, and sticky sessions, depend on the ingress controller. Kubernetes does not define the configuration syntax. In RHOCP, routes are generated to meet the conditions that the ingress object specifies.

**Note

The ingress resource is commonly used for Kubernetes. However, the route resource is the preferred method for external connectivity in RHOCP.

To create an ingress object, use the `oc create ingress ingress-name --rule=URL_route=service-name:port-number` command. Use the `--rule` option to provide a custom rule in the `host/path=service:port[,tls=secretname]` format. If the TLS option is omitted, then an insecure route is created.

```sh
[user@host ~]$ oc create ingress ingr-sakila \ --rule="ingr-sakila.apps.ocp4.example.com/*=sakila-service:8080"
```

The following listing shows a minimal definition for an ingress object:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend # 1
spec:
  rules: # 2
  - host: "www.example.com" # 3
    http:
      paths:
      - backend: # 4
          service:
            name: frontend
            port:
              number: 80
        pathType: Prefix
        path: /
  tls: # 5
  - hosts:
    - www.example.com
    secretName: example-com-tls-certificate
```

|     |                                                                                                                                                                                                           |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | The name of the ingress object. This name must be unique.                                                                                                                                                 |
| 2   | The HTTP or HTTPS rule for the ingress object.                                                                                                                                                            |
| 3   | The host for the ingress object. Applies the HTTP rule to the inbound HTTP traffic of the specified host.                                                                                                 |
| 4   | The backend to redirect traffic to. Defines the service name, port number, and port names for the ingress object. To connect to the back end, incoming requests must match the host and path of the rule. |
| 5   | The configuration of TLS for the ingress object; it is required for secured paths. The host in the TLS object must match the host in the rules object.                                                    |

You can delete an ingress object by using the ``oc delete ingress _`ingress-name`_`` command.

```sh
[user@host ~]$ oc delete ingress example-ingress
```

### Sticky Sessions

Sticky sessions enable stateful application traffic by ensuring that all requests reach the same endpoint. RHOCP uses cookies to configure session persistence for ingress and route resources. The ingress controller selects an endpoint to handle any user requests, and creates a cookie for the session. The cookie is passed back in response to the request, and the user sends back the cookie with the next request in the session. The cookie tells the ingress controller which endpoint is handling the session, to ensure that client requests use the cookie so that they are routed to the same pod.

RHOCP auto-generates the cookie name for ingress and route resources. You can overwrite the default cookie name by using the `annotate` command with either the `kubectl` or the `oc` commands. With this annotation, the application that receives route traffic knows the cookie name.

The following example configures a cookie for an ingress object:

```sh
[user@host ~]$ oc annotate ingress ingr-example \ ingress.kubernetes.io/affinity=cookie
```

The following example configures a cookie named `myapp` for a route object:

```sh
[user@host ~]$ oc annotate route route-example \ router.openshift.io/cookie_name=myapp
```

After you annotate the route, capture the route hostname in a variable:

```sh
[user@host ~]$ ROUTE_NAME=$(oc get route <route_name> \ -o jsonpath='{.spec.host}')
```

Then, use the `curl` command to save the cookie and access the route:

```sh
[user@host ~]$ curl $ROUTE_NAME -k -c /tmp/cookie_jar
```

The cookie is passed back in response to the request, and is saved to the `/tmp/cookie_jar` directory. Use the `curl` command and the cookie that was saved from the previous command to connect to the route:

```sh
[user@host ~]$ curl $ROUTE_NAME -k -b /tmp/cookie_jar
```

By using the saved cookie, the request is sent to the same pod as the previous request.

### Load Balance and Scale Applications

Developers and administrators can choose to manually scale the number of replica pods in a deployment. More pods might be needed for an anticipated surge in traffic, or the pod count might be reduced to reclaim resources that the cluster can use elsewhere.

You can change the number of replicas in a deployment resource manually by using the `oc scale` command.

```sh
[user@host ~]$ oc scale --replicas 5 deployment/scale
```

The deployment resource propagates the change to the replica set. The replica set reacts to the change by creating pods (replicas) or by deleting existing ones, depending on whether the new intended replica count is less than or greater than the existing count.

Although you can manipulate a replica set resource directly, the recommended practice is to manipulate the deployment resource instead. A new deployment creates either a replica set or a replication controller, and direct changes to a previous replica set or replication controller are ignored.

#### Load Balance Pods

A Kubernetes service serves as an internal load balancer. Standard services act as a load balancer or a proxy, and give access to the workload object by using the service name. A service identifies a set of replicated pods to transfer the connections that it receives.

A router uses the service selector to find the service and the endpoints, or pods, that back the service. When both a router and a service provide load balancing, RHOCP uses the router to load-balance traffic to pods. A router detects relevant changes in the IP addresses of its services, and adapts its configuration accordingly. Custom routers can thereby communicate modifications of API objects to an external routing solution.

RHOCP routers map external hostnames, and load-balance service endpoints over protocols that pass distinguishing information directly to the router. The hostname must exist in the protocol for the router to determine where to send it.

## Summary

- Many resources in Kubernetes and RHOCP create or affect pods.
- Resources are created imperatively or declaratively. The imperative strategy instructs the cluster what to do. The declarative strategy defines the state that the cluster matches.
- The `oc new-app` command creates resources that are determined via heuristics.
- The main way to deploy an application is by creating a deployment.
- The workload API includes several resources to create pods. The choice between resources depends on for how long and how often the pod needs to run.
- A _job_ resource executes a one-time task on the cluster via a pod. The cluster retries the job until it succeeds, or it retries a specified number of attempts.
- Resources are organized into projects and are selected via labels.
- A _route_ connects a public-facing IP address and a DNS hostname to an internal-facing _service_ IP address. 
- Services provide network access between pods, whereas routes provide network access to pods from users and applications outside the RHOCP cluster.

# Chapter 5: Manage Storage for Application Configuration and Data

## Externalize the Configuration of Applications

### Objectives

- Configure applications by using Kubernetes secrets and configuration maps to initialize environment variables and to provide text and binary configuration files.


### Configuring Kubernetes Applications

When an application is run in Kubernetes with a pre-existing image, the application uses the default configuration. This action is valid for testing purposes. However, for production environments, you might need to customize your applications before deploying them.

With Kubernetes, you can use manifests in `JSON` and `YAML` formats to specify the intended configuration for each application. You can define the name of the application, labels, the image source, storage, environment variables, and more.

The following snippet shows an example of a `YAML` manifest file of a deployment:

```yaml
apiVersion: apps/v1 #1
kind: Deployment #2
metadata: #3
  name: hello-deployment
spec: #4
  replicas: 1
  selector:
    matchLabels:
      app: hello-deployment
  template:
    metadata:
      labels:
        app: hello-deployment
    spec: #5
      containers:
      - env: #6
        - name: ENV_VARIABLE_1
          valueFrom:
            secretKeyRef:
              key: hello
              name: world
        image: quay.io/hello-image:latest

```

|     |                                                                                                                                                                               |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | API version of the resource.                                                                                                                                                  |
| 2   | `Deployment` resource type.                                                                                                                                                   |
| 3   | In this section, you specify the metadata of your application, such as the name.                                                                                              |
| 4   | You can define the general configuration of the resource that is applied to the deployment, such as the number of replicas (pods), the selector label, and the template data. |
| 5   | In this section, you specify the configuration for your application, such as the image name, the container name, ports, environment variables, and more.                      |
| 6   | You can define the environment variables to configure your application needs.                                                                                                 |

Sometimes your application requires configuring a combination of files. For example, at the time of creation, a database deployment must have preloaded databases and data. You most commonly configure applications by using environment variables, external files, or command-line arguments. This process of configuration externalization ensures that the application is portable across environments when the container image, external files, and environment variables are available in the environment where the application runs.

Kubernetes provides a mechanism to externalize the configuration of your applications by using configuration maps and secrets.

You can use configuration maps to inject containers with configuration data. The _ConfigMap_ (configuration map) namespaced objects provide ways to inject configuration data into containers, which helps to maintain platform independence of the containers. These objects can store fine-grained information, such as individual properties, or coarse-grained information, such as entire configuration files or JSON blobs (JSON sections). The information in configuration maps does not require protection.

The following listing shows an example of a configuration map:

```yaml
apiVersion: v1
kind: ConfigMap #1
metadata:
  name: example-configmap
  namespace: my-app
data: #2
  example.property.1: hello
  example.property.2: world
  example.property.file: |-
    property.1=value-1
    property.2=value-2
    property.3=value-3
binaryData: #3
  bar: L3Jvb3QvMTAw
```

|     |                                                                                                                                                       |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `ConfigMap` resource type.                                                                                                                            |
| 2   | Contains the configuration data.                                                                                                                      |
| 3   | Points to an encoded file in base64 that contains non-UTF-8 data, for example, a binary Java keystore file. Place a key followed by the encoded file. |

Applications often require access to sensitive information. For example, a back-end web application requires access to database credentials to query a database. Kubernetes and OpenShift use secrets to hold sensitive information. For example, you can use secrets to store the following types of sensitive information:

- Passwords
- Sensitive configuration files
- Credentials to an external resource, such as an SSH key or OAuth token

The following listing shows an example of a secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-secret
  namespace: my-app
type: Opaque #1
data: #2
  username: bXl1c2VyCg==
  password: bXlQQDU1Cg==
stringData: #3
  hostname: myapp.mydomain.com
  secret.properties: |
    property1=valueA
    property2=valueB
```

|     |                                        |
| --- | -------------------------------------- |
| 1   | Specifies the type of secret.          |
| 2   | Specifies the encoded string and data. |
| 3   | Specifies the decoded string and data. |

A secret is a namespaced object and it can store any type of data. Data in a secret is Base64-encoded, and is not stored in plain text. Secret data is not encrypted; you can decode the secret from Base64 format to access the original data. The following example shows the decoded values for the `username` and `password` objects from the `example-secret` secret:

```sh
[user@host] echo bXl1c2VyCg== | base64 --decode
myuser
[user@host] echo bXlQQDU1Cg== | base64 --decode
myP@55
```

Kubernetes and OpenShift support the following types of secrets:

- Opaque secrets: An opaque secret store key and value pairs that contain arbitrary values, and are not validated to conform to any convention for key names or values.
- Service account tokens: Store a token credential for applications that authenticate to the Kubernetes API.
- Basic authentication secrets: Store the needed credentials for basic authentication. The data parameter of the secret object must contain the user and the password keys that are encoded in the Base64 format.
- SSH keys: Store data that is used for SSH authentication.
- TLS certificates: Store a certificate and a key that are used for TLS.
- Docker configuration secrets: Store the credentials for accessing a container image registry.

When you store information in a specific secret resource type, Kubernetes validates that the data conforms to the type of secret.

>Note
By default, configuration maps and secrets are not encrypted. To encrypt your secret data at rest, you must encrypt the Etcd database. When enabled, Etcd encrypts the following resources: secrets, configuration maps, routes, OAuth access tokens, and OAuth authorization tokens. Encrypting the Etcd database is outside the scope of the course.

For more information, refer to the _Encrypting Etcd Data_ chapter in the Red Hat OpenShift Container Platform 4.14 _Security and Compliance_ documentation at [https://docs.redhat.com/en/documentation/openshift_container_platform/4.14/html-single/security_and_compliance/index#about-etcd_encrypting-etcd](https://docs.redhat.com/en/documentation/openshift_container_platform/4.14/html-single/security_and_compliance/index#about-etcd_encrypting-etcd)

### Creating Secrets

If a pod requires access to sensitive information, then create a secret for the information before you deploy the pod. Both the `oc` and `kubectl` command-line tools provide the `create secret` command. Use one of the following commands to create a secret:

- Create a generic secret that contains key-value pairs from literal values that are typed on the command line:

```sh
 [user@host ~]$ oc create secret generic secret_name \ --from-literal key1=secret1 \ --from-literal key2=secret2
```
   
- Create a generic secret by using key names that are specified on the command line and values from files:

```sh
 [user@host ~]$ kubectl create secret generic ssh-keys \ --from-file id_rsa=/path-to/id_rsa \ --from-file id_rsa.pub=/path-to/id_rsa.pub
```
  
- Create a TLS secret that specifies a certificate and the associated key:

```sh
 [user@host ~]$ oc create secret tls secret-tls \ --cert /path-to-certificate --key /path-to-key
```


To create an opaque secret from the web console, click the Workloads → Secrets menu. Click Create and select Key/value secret. Complete the form with the key name, and specify the value by writing it in the following section, or by extracting it from a file.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/storage/configs/assets/kvscrt.png)|

To create a secret from the web console that stores the credentials for accessing a container image registry, click the Workloads → Secrets menu. Click Create and select Image pull secret. Complete the form or upload a configuration file with the secret name, select the authentication type, and add the registry server address, the username, password, and email credentials.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/storage/configs/assets/impllscrt.png)|

### Creating Configuration Maps

The syntax for creating a configuration map and for creating a secret closely match. You can enter key-value pairs on the command line, or use the content of a file as the value of a specified key. You can use either the `oc` or `kubectl` command-line tools to create a configuration map. The following command shows how to create a configuration map:

```sh
[user@host ~]$ kubectl create configmap my-config \ --from-literal key1=config1 --from-literal key2=config2
```

You can also use the `cm` shortname to create a configuration map.

```sh
[user@host ~]$ oc create cm my-config \ --from-literal key1=config1 --from-literal key2=config2
```

To create a configuration map from the web console, click the Workloads → ConfigMaps menu. Click Create ConfigMap and complete the configuration map by using the form view or the YAML view.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/storage/configs/assets/configmap-create.png)|

You can use files on each key that you add by clicking Browse beside the Value field. The Key field must be the name of the added file in the Value field.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/storage/configs/assets/configmap-file.png)|

>Note
Use a binary data key instead of a data key if the file uses the binary format, such as a PNG file.

### Using Configuration Maps and Secrets to Initialize Environment Variables

You can use configuration maps to populate individual environment variables that configure your application. Unlike secrets, the information in configuration maps does not require protection. The following listing shows an initialization example of environment variables:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-map-example
  namespace: example-app #1
data:
  database.name: sakila #2
  database.user: redhat #3
```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO21-1)|The project where the configuration map resides. `ConfigMap` objects can be referenced only by pods in the same project.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO21-2)|Initializes the `database.name` variable to the `sakila` value.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO21-3)|Initializes the `database.user` variable to the `redhat` value.|

You can then use the configuration map to populate environment variables for your application. The following example shows a pod resource that populates specific environment variables by using a configuration map.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-map-example-pod
  namespace: example-app
spec:
  containers:
    - name: example-container
      image: registry.example.com/mysql-80:1-237
      command: [ "/bin/sh", "-c", "env" ]
      env: #1
        - name: MYSQL_DATABASE #2
          valueFrom:
            configMapKeyRef:
              name: config-map-example #3
              key: database.name #4
        - name: MYSQL_USER
          valueFrom:
            configMapKeyRef:
              name: config-map-example #5
              key: database.user #6
              optional: true #7
```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO22-1)|The attribute to specify environment variables for the pod.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO22-2)|The name of a pod environment variable where you are populating a key's value.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO22-3) [![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO22-5)|Name of the `ConfigMap` object to pull the environment variables from.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO22-4) [![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO22-6)|The environment variable to pull from the `ConfigMap` object.|
|[![7](https://rol.redhat.com/rol/static/roc/Common_Content/images/7.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO22-7)|Sets the environment variable as optional. The pod is started even if the specified `ConfigMap` object and keys do not exist.|

The following example shows a pod resource that injects all environment variables from a configuration map:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-map-example-pod2
  namespace: example-app
spec:
  containers:
    - name: example-container
      image: registry.example.com/mysql-80:1-237
      command: [ "/bin/sh", "-c", "env" ]
  envFrom: #1
    - configMapRef:
        name: config-map-example #2
  restartPolicy: Never

```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO23-1)|The attribute to pull all environment variables from a `ConfigMap` object.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO23-2)|The name of the `ConfigMap` object to pull environment variables from.|

You can use secrets with other Kubernetes resources such as pods, deployments, builds, and more. You can specify secret keys or volumes with a mount path to store your secrets. The following snippet shows an example of a pod that populates environment variables with data from the `test-secret` Kubernetes secret:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-example-pod
spec:
  containers:
    - name: secret-test-container
      image: busybox
      command: [ "/bin/sh", "-c", "export" ]
      env: #1
        - name: TEST_SECRET_USERNAME_ENV_VAR
          valueFrom: #2
            secretKeyRef: #3
              name: test-secret #4
              key: username #5

```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO24-1)|Specifies the environment variables for the pod.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO24-2)|Indicates the source of the environment variables.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO24-3)|The `secretKeyRef` source object of the environment variables.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO24-4)|Name of the secret, which must exist.|
|[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_using_configuration_maps_and_secrets_to_initialize_environment_variables-CO24-5)|The key that is extracted from the secret is the username for authentication.|

In contrast with configuration maps, the values in secrets are always encoded (not encrypted), and their access is restricted to fewer authorized users.

### Using Secrets and Configuration Maps as Volumes

To expose a secret to a pod, you must first create the secret in the same namespace, or project, as the pod. In the secret, assign each piece of sensitive data to a key. After creation, the secret contains key-value pairs.

The following command creates a generic secret that contains key-value pairs from literal values that are typed on the command line: `user` with the `demo-user` value, and `root_password` with the `zT1kTgk` value.

[user@host ~]$ **`oc create secret generic demo-secret \ --from-literal user=demo-user \ --from-literal root_password=zT1KTgk`**

You can also create a generic secret by specifying key names on the command line and values from files:

[user@host ~]$ **`oc create secret generic demo-secret \ --from-file user=/tmp/demo/user \ --from-file root_password=/tmp/demo/root_password`**

You can mount a secret to a directory within a pod. Kubernetes creates a file for each key in the secret that uses the name of the key. The content of each file is the decoded value of the secret. The following command shows how to mount secrets in a pod:

[user@host ~]$ **`oc set volume deployment/demo \ ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg) --add --type secret \ ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg) --secret-name demo-secret \ ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg) --mount-path /app-secrets`** ![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_using_secrets_and_configuration_maps_as_volumes-CO25-1)|Modify the volume configuration in the `demo` deployment.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_using_secrets_and_configuration_maps_as_volumes-CO25-2)|Add a new volume from a secret.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_using_secrets_and_configuration_maps_as_volumes-CO25-3)|Use the `demo-secret` secret.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_using_secrets_and_configuration_maps_as_volumes-CO25-4)|Make the secret data available in the `/app-secrets` directory in the pod. The content of the `/app-secrets/user` file is `demo-user`. The content of the `/app-secrets/root_password` file is `zT1KTgk`.|

To assign a secret as a volume to a deployment from the web console, list the available secrets from the Workloads → Secrets menu.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/storage/configs/assets/secrets-list.png)|

Select a secret and click Add Secret to workload.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/storage/configs/assets/secret-slct.png)|

Select the workload, choose the Volume option, and define the mount path for the secret.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/storage/configs/assets/secret-volume.png)|

Similar to secrets, you must first create a configuration map before a pod can consume it. The configuration map must exist in the same namespace, or project, as the pod. The following command shows how to create a configuration map from an external configuration file:

[user@host ~]$ **`oc create configmap demo-map \ --from-file=config-files/httpd.conf`**

You can similarly add a configuration map as a volume by using the following command:

[user@host ~]$ **`oc set volume deployment/demo \ --add --type configmap \ --configmap-name demo-map \ --mount-path /app-secrets`**

To confirm that the volume is attached to the deployment, use the following command:

[user@host ~]$ **`oc set volume deployment/demo`**
 demo
  configMap/demo-map as volume-du9in
    mounted at /app-secrets

You can also use the `oc set env` command to set application environment variables from either secrets or configuration maps. In some cases, you can modify the names of the keys to match the names of environment variables by using the `--prefix` option. In the following example, the `user` key from the `demo-secret` secret sets the `MYSQL_USER` environment variable, and the `root_password` key from the `demo-secret` secret sets the `MYSQL_ROOT_PASSWORD` environment variable. If the key name from the secret is lowercase, then the corresponding environment variable is converted to uppercase to match the pattern that the `--prefix` option defines.

[user@host ~]$ **`oc set env deployment/demo \ --from secret/demo-secret --prefix MYSQL_`**

>Note
You cannot assign configuration maps by using the web console.

### Updating Secrets and Configuration Maps

Secrets and configuration maps occasionally require updates. OpenShift provides the `oc extract` command to ensure that you have the latest data. You can save the data to a specific directory by using the `--to` option. Each key in the secret or configuration map creates a file with the same name as the key. The content of each file is the value of the associated key. If you run the `oc extract` command more than one time, then you must use the `--confirm` option to overwrite the existing files. You can also use the `--confirm` option to create the target directory for the extracted content.

```sh
[user@host ~]$ oc extract secret/demo-secrets -n demo \ --to /tmp/demo --confirm
[user@host ~]$ ls /tmp/demo/
user  root_password
[user@host ~]$ cat /tmp/demo/root_password
zT1KTgk
[user@host ~]$ echo k8qhcw3m0 > /tmp/demo/root_password
```
After updating the locally saved files, use the `oc set data` command to update the secret or configuration map. For each key that requires an update, specify the name of a key and the associated value. If a file contains the value, then use the `--from-file` option.

```sh
[user@host ~]$ oc set data secret/demo-secrets -n demo \ --from-file /tmp/demo/root_password
```

You must restart pods that use environment variables for the pods to read the updated secret or configuration map. Pods that use a volume mount to reference secrets or configuration maps receive the updates without a restart by using an eventually consistent approach. By default, the `kubelet` agent watches for changes to the keys and values that are used in volumes for pods on the node. The `kubelet` agent detects changes and propagates the changes to the pods to keep volume data consistent. Despite the automatic updates that Kubernetes provides, a restart of the pod is still required if the software reads configuration data only at startup time.

### Deleting Secrets and Configuration Maps

Similar to other Kubernetes resources, you can use the `delete` command to delete secrets and configuration maps that are no longer needed or in use.

```sh
[user@host ~]$ kubectl delete secret/demo-secrets -n demo

[user@host ~]$ oc delete configmap/demo-map -n demo
```

## Provision Persistent Data Volumes

### Objectives

- Provide applications with persistent storage volumes for block and file-based data.

### Kubernetes Persistent Storage

Containers have ephemeral storage by default. The lifetime of this ephemeral storage does not extend beyond the life of the individual pod, and this ephemeral storage cannot be shared across pods. When a container is deleted, all the files and data inside it are also deleted. To preserve the files, containers use persistent storage volumes.

Because OpenShift Container Platform uses the Kubernetes persistent volume (PV) framework, cluster administrators can provision persistent storage for a cluster. Developers can use persistent volume claims (PVCs) to request PV resources without specific knowledge of the underlying storage infrastructure.

Two ways exist to provision storage for the cluster: static and dynamic. Static provisioning requires the cluster administrator to create persistent volumes manually. Dynamic provisioning uses storage classes to create the persistent volumes on demand.

Administrators can use storage classes to provide persistent storage. Storage classes describe types of storage for the cluster. Cluster administrators create storage classes to manage storage services or storage tiers of a service. Rather than specifying provisioned storage, PVCs instead refer to a storage class.

Developers use PVCs to add persistent volumes to their applications. Developers need not know details of the storage infrastructure. With static provisioning, developers use previously created PVs, or ask a cluster administrator to manually create persistent volumes for their applications. With dynamic provisioning, developers declare the storage requirements of the application, and the cluster creates a PV to fill the request.

### Persistent Volumes

Not all storage is equal. Storage types vary in cost, performance, and reliability. Multiple storage types are usually available for each Kubernetes cluster.

The following list of commonly used storage volume types and their use cases is not exhaustive.

**configMap

The `configMap` volume externalizes the application configuration data. This use of the `configMap` resource ensures that the application configuration is portable across environments and can be version-controlled.

**emptyDir

An `emptyDir` volume provides a per-pod directory for scratch data. The directory is usually empty after provisioning. `emptyDir` volumes are often required for ephemeral storage.

**hostPath

A `hostPath` volume mounts a file or directory from the host node into your pod. To use a `hostPath` volume, the cluster administrator must configure pods to run as privileged. This configuration grants access to other pods in the same node.

Red Hat does not recommend the use of `hostPath` volumes in production. Instead, Red Hat supports `hostPath` mounting for development and testing on a single-node cluster. Although most pods do not need a `hostPath` volume, it does offer a quick option for testing if an application requires it.

**iSCSI

Internet Small Computer System Interface (iSCSI) is an IP-based standard that provides block-level access to storage devices. With iSCSI volumes, Kubernetes workloads can consume persistent storage from iSCSI targets.

**local

You can use `Local` persistent volumes to access local storage devices, such as a disk or partition, by using the standard PVC interface. `Local` volumes are subject to the availability of the underlying node, and are not suitable for all applications.

**NFS

An NFS (Network File System) volume can be accessed from multiple pods at the same time, and thus provides shared data between pods. The `NFS` volume type is commonly used because of its ability to share data safely. Red Hat recommends to use NFS only for non-production systems.

#### Volume Access Mode

Persistent volume providers vary in capabilities. A volume uses access modes to specify the modes that it supports. For example, NFS can support multiple read/write clients, but a specific NFS PV might be exported on the server as read-only. OpenShift defines the following access modes, as summarized in the following table:

**Table 5.1. Volume Access Modes**

|Access mode|Abbreviation|Description|
|:--|:--|:--|
|`ReadWriteOnce`|RWO|A single node mounts the volume as read/write.|
|`ReadOnlyMany`|ROX|Many nodes mount the volume as read-only.|
|`ReadWriteMany`|RWX|Many nodes mount the volume as read/write.|

  

Developers must select a volume type that supports the required access level by the application. The following table shows some example supported access modes:

**Table 5.2. Access Mode Support**

|Volume type|RWO|ROX|RWX|
|:--|:--|:--|:--|
|configMap|Yes|No|No|
|emptyDir|Yes|No|No|
|hostPath|Yes|No|No|
|iSCSI|Yes|Yes|No|
|local|Yes|No|No|
|NFS|Yes|Yes|Yes|

  

#### Volume Modes

Kubernetes supports two volume modes for persistent volumes: `Filesystem` and `Block`. If the volume mode is not defined for a volume, then Kubernetes assigns the default volume mode, `Filesystem`, to the volume.

OpenShift Container Platform can provision raw block volumes. These volumes do not have a file system, and can provide performance benefits for applications that either write to the disk directly or that implement their own storage service. Raw block volumes are provisioned by specifying `volumeMode: Block` in the PV and PVC specification.

The following table provides examples of storage options with block volume support:

**Table 5.3. Block Volume Support**

|Volume plug-in|Manually provisioned|Dynamically provisioned|
|:--|:--|:--|
|AWS EBS|Yes|Yes|
|Azure disk|Yes|Yes|
|Cinder|Yes|Yes|
|Fibre channel|Yes|No|
|GCP|Yes|Yes|
|iSCSI|Yes|No|
|local|Yes|No|
|Red Hat OpenShift Data Foundation|Yes|Yes|
|VMware vSphere|Yes|Yes|

  

#### Manually Creating a PV

Use a `PersistentVolume` manifest file to manually create a persistent volume. The following example creates a persistent volume from a fiber channel storage device that uses block mode.

```yaml
apiVersion: v1
kind: PersistentVolume #1
metadata:
  name: block-pv #2
spec:
  capacity:
    storage: 10Gi #3
  accessModes:
    - ReadWriteOnce #4
  volumeMode: Block #5
  persistentVolumeReclaimPolicy: Retain #6
  fc: #7
    targetWWNs: ["50060e801049cfd1"]
    lun: 0
    readOnly: false

```

|     |                                                                                                                                                    |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `PersistentVolume` is the resource type for PVs.                                                                                                   |
| 2   | Provide a name for the PV, which subsequent claims use to access the PV.                                                                           |
| 3   | Specify the amount of storage that is allocated to this volume.                                                                                    |
| 4   | The storage device must support the access mode that the PV specifies.                                                                             |
| 5   | The `volumeMode` attribute is optional for `Filesystem` volumes, but is required for `Block` volumes.                                              |
| 6   | The `persistentVolumeReclaimPolicy` determines how the cluster handles the PV when the PVC is deleted. Valid options are `Retain` or `Delete`.     |
| 7   | The remaining attributes are specific to the storage type. In this example, the `fc` object specifies the `Fiber Channel` storage type attributes. |

If the previous manifest is in a file named `my-fc-volume.yaml`, then the following command can create the PV resource on RHOCP:

```sh
[user@host]$ oc create -f my-fc-volume.yaml
```

To create a persistent volume from the web console, click the Storage → PersistentVolumes menu.

### Persistent Volume Claims

A persistent volume claim (PVC) resource represents a request from an application for storage. A PVC specifies the minimal storage characteristics, such as capacity and access mode. A PVC does not specify a storage technology, such as iSCSI or NFS.

The lifecycle of a PVC is not tied to a pod, but to a namespace. Multiple pods from the same namespace but with potentially different workload controllers can connect to the same PVC. You can also sequentially connect storage to and detach storage from different application pods, to initialize, convert, migrate, or back up data.

Kubernetes matches each PVC to a persistent volume (PV) resource that can satisfy the requirements of the claim. It is not an exact match. A PVC might be bound to a PV with a larger disk size than is requested. A PVC that specifies single access might be bound to a PV that is shareable for multiple concurrent accesses. Rather than enforcing policy, PVCs declare what an application needs, which Kubernetes provides on a best-effort basis.

#### Creating a PVC

A PVC belongs to a specific project. To create a PVC, you must specify the access mode and size, among other options. A PVC cannot be shared between projects. Developers use a PVC to access a persistent volume (PV). Persistent volumes are not exclusive to projects, and are accessible across the entire OpenShift cluster. When a PV binds to a PVC, the PV cannot be bound to another PVC.

To add a volume to an application deployment, create a `PersistentVolumeClaim` resource, and add it to the application as a volume. Create the PVC by using either a Kubernetes manifest or the `oc set volumes` command. In addition to either creating a PVC or using an existing PVC, the `oc set volumes` command can modify a deployment to mount the PVC as a volume within the pod.

To add a volume to an application deployment, use the `oc set volumes` command:
```sh
[user@host ~]$ oc set volumes deployment/example-application \ #1
--add \ #2
--name example-pv-storage \ #3
--type persistentVolumeClaim \ #4
--claim-mode rwo \ #5
--claim-size 15Gi \ #6
--mount-path /var/lib/example-app \ #7
--claim-name example-pv-claim #8
```

|     |                                                                                                                                                                  |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Specify the name of the deployment that requires the PVC resource.                                                                                               |
| 2   | Setting the `add` option to `true` adds volumes and volume mounts for containers.                                                                                |
| 3   | The `name` option specifies a volume name. If not specified, a name is autogenerated.                                                                            |
| 4   | The supported types, for the `add` operation, include `emptyDir`, `hostPath`, `secret`, `configMap`, and `persistentVolumeClaim`.                                |
| 5   | The `claim-mode` option defaults to `ReadWriteOnce`. The valid values are `ReadWriteOnce (RWO)`, `ReadWriteMany (RWX)`, and `ReadOnlyMany (ROX)`.                |
| 6   | Create a claim with the given size in bytes, if specified along with the persistent volume type. The size must use SI notation, for example, 15, 15 G, or 15 Gi. |
| 7   | The `mount-path` option specifies the mount path inside the container.                                                                                           |
| 8   | The `claim-name` option provides the name for the PVC, and is required for the `persistentVolumeClaim` type.                                                     |

The command creates a PVC resource and adds it to the application as a volume within the pod.

The command updates the deployment for the application with `volumeMounts` and `volumes` specifications.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  _...output omitted..._
  namespace: storage-volumes #1
  _...output omitted..._
spec:
  _...output omitted..._
  template:
  _...output omitted..._
    spec:
  _...output omitted..._
        **`volumeMounts:`**
        - mountPath: /var/lib/example-app #2
          name: example-pv-storage #3
  _...output omitted..._
      **`volumes:`**
      - name: example-pv-storage #4
        persistentVolumeClaim:
          claimName: example-pv-claim #5
  _...output omitted..._

```

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_creating_a_pvc-CO28-1)|The deployment, which must be in the same namespace as the PVC.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_creating_a_pvc-CO28-2)|The mount path in the container.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_creating_a_pvc-CO28-3) [![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_creating_a_pvc-CO28-4)|The volume name, which is used to specify the volume that is associated with the mount.|
|[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_creating_a_pvc-CO28-5)|The claim name that is bound to the volume.|

The following example specifies a PVC by using a YAML manifest to create a `PersistentVolumeClaim` API object:

---
```yaml
apiVersion: v1
kind: PersistentVolumeClaim #1
metadata:
  name: example-pv-claim #2
  labels:
    app: example-application
spec:
  accessModes:
    - ReadWriteOnce #3
  resources:
    requests:
      storage: 15Gi #4
```

|     |                                                                                                                                                                                                                              |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `PersistentVolumeClaim` is the resource type for a PVC.                                                                                                                                                                      |
| 2   | Use this name in the `claimName` field of the `persistentVolumeClaim` element in the `volumes` section of a deployment manifest.                                                                                             |
| 3   | Specify the access mode that this PVC requests. The storage class provisioner must provide this access mode. If persistent volumes are created statically, then an eligible persistent volume must provide this access mode. |
| 4   | The storage class creates a persistent volume that matches this size request. If persistent volumes are created statically, then an eligible persistent volume must be at least the requested size.                          |

Use the `oc create` command to create the PVC from the manifest file.

```sh
[user@host ~]$ oc create -f pvc_file_name.yaml
```

Use `oc get pvc` to view the available PVCs in the current namespace.

```bash
[user@host ~]$ oc get pvc
NAME         STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   ...
db-pod-pvc   Bound    pvc-13...ca45   1Gi        RWO            nfs-storage    ...
```
To create a persistent volume claim from the web console, click the Storage → PersistentVolumesClaims menu.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/storage/volumes/assets/storage-pvc.png)|

Click Create PersistentVolumeClaim and complete the form by adding the name, the storage class, the size, the access mode, and the volume mode.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/storage/volumes/assets/pvc-form.png)|

#### Kubernetes Dynamic Provisioning

PVs are defined by a `PersistentVolume` API object, which is from existing storage in the cluster. The cluster administrator must statically provision some storage types. Alternatively, the Kubernetes persistent volume framework can use a `StorageClass` object to dynamically provision PVs.

When you create a PVC, you specify a storage amount, the required access mode, and a storage class to describe and classify the storage. The control loop in the RHOCP control node watches for new PVCs, and binds the new PVC to an appropriate PV. If an appropriate PV does not exist, then a provisioner for the storage class creates one.

Claims remain unbound indefinitely if a matching volume does not exist or if a volume cannot be created with any available provisioner that services a storage class. Claims are bound when matching volumes become available. For example, a cluster with many manually provisioned 50 Gi volumes would not match a PVC that requests 100 Gi. The PVC can be bound when a 100 Gi PV is added to the cluster.

Use `oc get storageclass` to view the storage classes that the cluster provides.

```bash
[user@host ~]$ oc get storageclass
NAME                    PROVISIONER                                  ...
nfs-storage (default)   k8s-sigs.io/nfs-subdir-external-provisioner  ...
lvms-vg1             topolvm.io                           ...
```
In the example, the `nfs-storage` storage class is marked as the default storage class. When a default storage class is configured, the PVC must explicitly name any other storage class to use, or can set the `storageClassName` annotation to "", to be bound to a PV without a storage class.

The following `oc set volume` command example uses the `claim-class` option to specify a dynamically provisioned PV.

```bash
[user@host ~]$ oc set volumes deployment/example-application \ --add --name example-pv-storage --type pvc --claim-class nfs-storage \ --claim-mode rwo --claim-size 15Gi --mount-path /var/lib/example-app \ --claim-name example-pv-claim
```
**Note

Because a cluster administrator can change the default storage class, Red Hat recommends that you always specify the storage class when you create a PVC.

#### PV and PVC Lifecycles

When you create a PVC, you request a specific amount of storage, access mode, and storage class. Kubernetes binds the PVC to an appropriate PV. If an appropriate PV does not exist, then a provisioner for the storage class creates one.

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/storage/volumes/assets/pv-lifecycle.svg)

Figure 5.17: PV lifecycle

PVs follow a lifecycle based on their relationship to the PVC.

**Available

After a PV is created, it becomes _available_ for any PVC to use in the cluster in any namespace.

**Bound

A PV that is _bound_ to a PVC is also bound to the same namespace as the PVC, and no other PVC can use it.

**In Use

You can delete a PVC if no pods actively use it. The _Storage Object in Use Protection_ feature prevents the removal of bound PVs and PVCs that pods are actively using. Such removal can result in data loss. Storage Object in Use Protection is enabled by default.

If a user deletes a PVC that a pod actively uses, then the PVC is not removed immediately. PVC removal is postponed until no pods actively use the PVC. Also, if a cluster administrator deletes a PV that is bound to a PVC, then the PV is not removed immediately. PV removal is postponed until the PV is no longer bound to a PVC.

**Released

After the developer deletes the PVC that is bound to a PV, the PV is _released_, and the storage that the PV used can be reclaimed.

**Reclaimed

The reclaim policy of a persistent volume tells the cluster what to do with the volume after it is released. A volume's reclaim policy can be Retain or Delete.

**Table 5.4. Volume Reclaim Policy**

|Policy|Description|
|:--|:--|
|Retain|Enables manual reclamation of the resource for those volume plug-ins that support it.|
|Delete|Deletes both the `PersistentVolume` object from OpenShift Container Platform and the associated storage asset in external infrastructure.|

  

#### Deleting a Persistent Volume Claim

To delete a volume, use the `oc delete` command to delete the PVC. The storage class reclaims the volume after removing the PVC.

```bash
[user@host ~]$ oc delete pvc/example-pvc-storage
```

## Select a Storage Class for an Application

### Objectives

- Match applications with storage classes that provide storage services to satisfy application requirements.
    

### Storage Class Selection

Storage classes are a way to describe types of storage for the cluster and to provision dynamic storage on demand. The cluster administrator determines the meaning of a storage class, which is also called a profile in other storage systems. For example, an administrator can create one storage class for development use, and another for production use.

Kubernetes supports multiple storage back ends. The storage options differ in cost, performance, reliability, and function. An administrator can create different storage classes for these options. As a result, developers can select the storage solution that fits the needs of the application. Developers do not need to know the storage infrastructure details.

Recall that an administrator selects the default storage class for dynamic provisioning. A default storage class enables Kubernetes to automatically provision a persistent volume claim (PVC) that does not specify a storage class. Because an administrator can change the default storage class, a developer should explicitly set the storage class for an application.

#### Reclaim Policy

Outside the application function, the developer must also consider the impact of the reclaim policy on storage requirements. A reclaim policy determines what happens to the data on a PVC after the PVC is deleted. When you are finished with a volume, you can delete the PVC object from the API, which enables reclamation of the resource. Kubernetes releases the volume when the PVC is deleted, but the volume is not yet available for another claim. The previous claimant's data remains on the volume and must be handled according to the policy. To keep your data, choose a storage class with a `retain` reclaim policy.

By using the `retain` reclaim policy, when you delete a PVC, only the PVC object is deleted from the cluster. The Persistent Volume (PV) that backed the PVC, the physical storage device that the PV used, and your data still exist. To reclaim the storage and use it in your cluster again, the cluster administrator must take manual steps.

To manually reclaim a PV as a cluster administrator, follow these steps:

1. Delete the PV.
    
```
[user@host ~]$ oc delete pv <pv-name>
```

   The associated asset in the external storage infrastructure, such as an AWS EBS, GCE PD, Azure Disk, or Cinder volume, still exists after the PV is deleted.
    
2. At this point, the cluster administrator can create another PV by using the same storage and data from the previous PV. A developer could then mount the new PV and access the data from the previous PV.
    
3. Alternatively, the cluster administrator can remove the data on the storage asset, and then delete the storage asset.
    

To automatically delete the PV, the data, and the physical storage for a deleted PVC, you must choose a storage class that uses the `delete` reclaim policy. This reclaim policy automatically reclaims your storage volume when the PVC is deleted. The `delete` reclaim policy is the default setting for all storage provisioners that adhere to the Kubernetes Container Storage Interface (CSI) standards. If you use a storage class that does not specify a reclaim policy, then the `delete` reclaim policy is used.

For more information about the Kubernetes Container Storage Interface standards, refer to the _Kubernetes CSI Developer Documentation_ website at [https://kubernetes-csi.github.io/docs/](https://kubernetes-csi.github.io/docs/).

#### Kubernetes and Application Responsibilities

Kubernetes does not change how an application relates to storage. The application is responsible for working with its storage devices and for ensuring data integrity and consistency. Kubernetes storage does not prevent an application from doing dangerous things, such as sharing a data volume between two databases that require exclusive access to data.

Because a PVC is a storage device that your Linux host mounts, an improperly configured application could behave unexpectedly. For example, you could have an iSCSI LUN, which is expressed as an RWO PVC that is not supposed to be shared, and then mount that same PVC on two pods of the same host. Whether this situation is problematic depends on the applications.

Usually, it is fine for two processes on the same host to share a disk. After all, many applications on your personal machine share a local disk. However, nothing prevents one text editor from overwriting and losing all edits from another text editor. The use of Kubernetes storage must come with the same caution.

Single-node access (RWO) and shared access (RWX) do not ensure that files can be shared safely and reliably. RWO means that only one cluster node can read and write to the PVC. Alternatively, with RWX, Kubernetes provides a storage volume that any pod can access for reading or writing.

### Use Cases for Storage Classes

The administrator creates storage classes that serve the needs of the developers. A `storageClass` object defines each storage class, and the object contains information about the storage provider and the capabilities of the storage medium. The provider creates PVs to match the specifications of the storage class. Administrators can create storage classes with various functional levels, based on many factors.

**Storage volume modes

A storage class with `block` volume mode support can increase performance for applications that can use raw block devices. Consider using a storage class with `Filesystem` volume mode support for applications that share files or that provide file access.

**Quality of Service (QoS) levels

A Solid State Drive (SSD) provides excellent speed and support for frequently accessed files. Use a lower cost and a slower hard drive (HDD) for files that are accessed less often.

**Administrative support tier

A production-tier storage class can include volumes that are backed up often. In contrast, a development-tier storage class might include volumes that are not configured with a backup schedule.

Storage classes can use a combination of these factors and others to best fit the needs of the developers.

Kubernetes matches PVCs with the best available PV that is not bound to another PVC. The PV must provide the access mode that is specified in the PVC, and the volume must be at least as large as the requested size in the PVC. The supported access modes depend on the capabilities of the storage provider. A PVC can specify additional criteria, such as the name of a storage class. If a PVC cannot find a PV that matches all criteria, then the PVC enters a `pending` state and waits until an appropriate PV becomes available.

PVCs can request a specific storage class by specifying the `storageClassName` attribute. This method of selecting a specific storage class ensures that the storage medium is a good fit for the application requirements. Only PVs of the requested storage class can be bound to the PVC. The cluster administrator can configure dynamic provisioners to service storage classes. The cluster administrator can also create a PV on demand that matches the specifications in the PVC.

### Create a Storage Class

The following YAML excerpt describes the basic definition for a `StorageClass` object. A cluster administrator or a `storage-admin` user creates globally scoped `StorageClass` objects. The following resource shows the parameters for configuring a storage class. This example uses the AWS ElasticBlockStore (EBS) object definition.

```yaml
apiVersion: storage.k8s.io/v1 #1
kind: StorageClass #2
metadata:
  name: io1-gold-storage #3
  annotations: #4
    storageclass.kubernetes.io/is-default-class: 'false'
    description:'Provides RWO and RWOP Filesystem & Block volumes'
    ...
parameters: #5
  type: io1
  iopsPerGB: "10"
    ...
provisioner: kubernetes.io/aws-ebs #6
reclaimPolicy: Delete #7
volumeBindingMode: Immediate #8
allowVolumeExpansion: true #9

```

|     |                                                                                                                             |
| --- | --------------------------------------------------------------------------------------------------------------------------- |
| 1   | A required item that specifies the current API version.                                                                     |
| 2   | A required item that specifies the API object type.                                                                         |
| 3   | A required item that specifies the name of the storage class.                                                               |
| 4   | An optional item that specifies annotations for the storage class.                                                          |
| 5   | An optional item that specifies the required parameters for the specific provisioner; this object differs between plug-ins. |
| 6   | A required item that specifies the type of provisioner that is associated with this storage class.                          |
| 7   | An optional item that specifies the selected reclaim policy for the storage class.                                          |
| 8   | An optional item that specifies the selected volume binding mode for the storage class.                                     |
| 9   | An optional item that specifies the volume expansion setting.                                                               |

Several attributes, such as the API version, API object type, and annotations, are common for Kubernetes objects, whereas other attributes are specific to storage class objects.

**Parameters

Parameters can configure file types, change storage types, enable encryption, enable replication, and so on. Each provisioner has different parameter options. Accepted parameters depend on the storage provisioner. For example, the `io1` value for the `type` parameter, and the `iopsPerGB` parameter, are specific to EBS. When a parameter is omitted, the storage provisioner uses the default value.

**Provisioners

The `provisioner` attribute identifies the source of the storage medium plug-in. Provisioners with names that begin with a `kubernetes.io` value are available by default in a Kubernetes cluster.

**ReclaimPolicy

The default reclaim policy, `Delete`, automatically reclaims the storage volume when the PVC is deleted. Reclaiming storage in this way can reduce the storage costs. The `Retain` reclaim policy does not delete the storage volume, so that data is not lost if the wrong PVC is deleted. This reclaim policy can result in higher storage costs if space is not manually reclaimed.

**VolumeBindingMode

The `volumeBindingMode` attribute determines how volume attachments are handled for a requesting PVC. Using the default `Immediate` volume binding mode creates a PV to match the PVC when the PVC is created. This setting does not wait for the pod to use the PVC, and thus can be inefficient. The `Immediate` binding mode can also cause problems for storage back ends that are topology-constrained or are not globally accessible from all nodes in the cluster. PVs are also bound without the knowledge of a pod's scheduling requirements, which might result in unschedulable pods.

By using the `WaitForFirstConsumer` mode, the volume is created after the pod that uses the PVC is in use. With this mode, Kubernetes creates PVs that conform to the pod's scheduling constraints, such as resource requirements and selectors.

**AllowVolumeExpansion

When set to a `true` value, the storage class specifies that the underlying storage volume can be expanded if more storage is required. Users can resize the volume by editing the corresponding PVC object. This feature can be used only to grow a volume, not to shrink it.

The cluster administrator can use the `create` command to create a storage class from a YAML manifest file. The resulting storage class is non-namespaced, and thus is available to all projects in the cluster.
```bash
[user@host ~]$ oc create -f <storage-class-filename.yaml>
```

To create a storage class from the web console, click the Storage → StorageClasses menu. Click Create StorageClass and complete the form or the YAML manifest.

#### Cluster Storage Classes

Use the `oc get storageclass` command to view the storage class options that are available in a cluster.

```bash
[user@host ~]$ oc get storageclass
```

A regular cluster user can view the attributes of a storage class by using the `describe` command. The following example queries the attributes of the storage class with the name `lvms-vg1`.
```bash
[user@host ~]$ oc describe storageclass lvms-vg1
IsDefaultClass:        No
Annotations:           description=Provides RWO and RWOP Filesystem & Block volumes
Provisioner:           topolvm.io
Parameters:            csi.storage.k8s.io/fstype=xfs,topolvm.io/device-class=vg1
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
```

The `describe` command can help a developer to decide whether the storage class is a good fit for an application. If none of the storage classes in the cluster are appropriate for the application, then the developer can request the cluster administrator to create a PV with the required features.

### Storage Class Usage

Recall that the `oc set volume` command can add a PVC and an associated PV to a deployment. A YAML manifest file can declare the parameters of a PVC independently from the deployment. This method is the preferred option to support repeatability, configuration management, and version control. Use the `storageClassName` attribute to specify the storage class for the PVC.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-block-pvc
spec:
  accessModes:
  - RWO
  volumeMode: Block
  storageClassName: <storage-class-name>
  resources:
    requests:
      storage: 10Gi
```

Use the `create` command to create the resource from the YAML manifest file.

```bash
[user@host ~]$ oc create -f <my-pvc-filename.yaml>
```

Use the `--claim-name` option with the `set volume` command to add the pre-existing PVC to a deployment.

```bash
[user@host ~]$ oc set volume deployment/<deployment-name> \
--add --name <my-volume-name> \
--claim-name my-block-pvc \
--mount-path /var/tmp
```

## Manage Non-shared Storage with Stateful Sets

### Objectives

- Deploy applications that scale without sharing storage.
    

### Application Clustering

Clustering applications, such as MySQL and Cassandra, typically require persistent storage to maintain the integrity of the data and files that the application uses. When many applications require persistent storage at the same time, multi-disk provisioning might not be possible due to the limited amount of available resources.

Shared storage solves this problem by allocating the same resources from a single device to multiple services.

### Storage Services

File storage solutions provide the directory structure that is found in many environments. Using file storage is ideal when applications generate or consume reasonable volumes of organized data. Applications that use file-based implementations are prevalent, easy to manage, and provide an affordable storage solution.

File-based solutions are a good fit for data backup and archiving, due to their reliability, as are also file sharing and collaboration services. Most data centers provide file storage solutions, such as a network-attached storage (NAS) cluster, for these scenarios.

_Network-attached storage (NAS)_ is a file-based storage architecture that makes stored data accessible to networked devices. NAS gives networks a single access point for storage with built-in security, management, and fault-tolerant capabilities. Out of the multiple data transfer protocols that networks can run, two protocols are fundamental to most networks: _internet protocol (IP)_ and _transmission control protocol (TCP)_.

The files that are transferred across these protocols can be formatted as one of the following protocols:

- _Network File Systems (NFS)_: This protocol enables remote hosts to mount file systems over a network and to interact with those file systems as though they are mounted locally.
    
- _Server Message Blocks (SMB)_: This protocol implements an application-layer network protocol that is used to access resources on a server, such as file shares and shared printers.
    

NAS solutions can provide file-based storage to applications within the same data center. This approach is common to many application architectures, including the following architectures:

- Web server content
    
- File share services
    
- FTP storage
    
- Backup archives
    

These applications take advantage of data reliability and the ease of file sharing that is available by using file storage. Additionally, for file storage data, the OS and file system handle the locking and caching of the files.

Although familiar and prevalent, file storage solutions are not ideal for all application scenarios. One particular pitfall of file storage is poor handling of large data sets or unstructured data.

Block storage solutions, such as Storage Area Network (SAN) and iSCSI technologies, provide access to raw block devices for application storage. These block devices function as independent storage volumes, such as the physical drives in servers, and typically require formatting and mounting for application access.

Using block storage is ideal when applications require faster access for optimizing computationally heavy data workloads. Applications that use block-level storage implementations gain efficiencies by communicating at the raw device level, instead of relying on operating system layer access.

Block-level approaches enable data distribution on blocks across the storage volume. Blocks also use basic metadata, including a unique identification number for each block of data, for quick retrieval and reassembly of blocks for reading.

SAN and iSCSI technologies provide applications with block-level volumes from network-based storage pools. Using block-level access to storage volumes is common for application architectures, including the following architectures:

- SQL Databases (single node access).
    
- Virtual Machines (multinode access).
    
- High-performance data access.
    
- Server-side processing applications.
    
- Multiple block device RAID configurations.
    

Application storage that uses several block devices in a RAID configuration benefits from the data integrity and performance that the various arrays provide.

With Red Hat OpenShift Container Platform (RHOCP), you can create customized storage classes for your applications. With the NAS and the SAN storage technologies, RHOCP applications can use either the NFS protocol for file-based storage, or the block-level protocol for block storage.

### Introduction to Stateful Sets

A stateful application is characterized by acting according to past states or transactions, which affect the current state and future ones of the application. Using a stateful application simplifies recovery from failures by starting from a certain point in time.

A stateful set is the representation of a set of pods with consistent identities. These identities are defined as a network with a single stable DNS, hostname, and storage from as many volume claims as the stateful set specifies. A stateful set guarantees that a given network identity maps to the same storage identity.

Deployments represent a set of containers within a pod. Each deployment can have many active replicas, depending on the user specification. These replicas can be scaled up or down, as needed. A replica set is a native Kubernetes API object that ensures that the specified number of pod replicas are running. Deployments are used for stateless applications by default, and they can be used for stateful application by attaching a persistent volume. All pods in a deployment share a volume and PVC.

In contrast with deployments, stateful set pods do not share a persistent volume. Instead, stateful set pods each have their own unique persistent volumes. Pods are created without a replica set, and each replica records its own transactions. Each replica has its own identifier, which is maintained in any rescheduling. You must configure application-level clustering so that stateful set pods have the same data.

Stateful sets are the best option for applications, such as databases, that require consistent identities and non-shared persistent storage.

### Working with Stateful Sets

With Kubernetes, you can use manifest files to specify the intended configuration of a stateful set. You can define the name of the application, labels, the image source, storage, environment variables, and more.

The following snippet shows an example of a `YAML` manifest file for a stateful set:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: dbserver #1
spec:
  selector:
    matchLabels:
      app: database #2
  replicas: 3 #3
  template:
    metadata:
      labels:
        app: database #4
    spec:
      containers:
      - env: #5
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              key: user
              name: sakila-cred
        image: registry.ocp4.example.com:8443/redhattraining/mysql-app:v1 #6
        name: database #7
        ports: #8
        - containerPort: 3306
          name: database
        volumeMounts: #9
        - mountPath: /var/lib/mysql
          name: data
      terminationGracePeriodSeconds: 10
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ] #10
      storageClassName: "lvms-vg1" #11
      resources:
        requests:
          storage: 1Gi ![12](https://rol.redhat.com/rol/static/roc/Common_Content/images/12.svg)
```

|      |                                                                                                                                    |
| ---- | ---------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Name of the stateful set.                                                                                                          |
| 2, 4 | Application labels.                                                                                                                |
| 3    | Number of replicas.                                                                                                                |
| 5    | Environment variables, which can be explicitly defined, or by using a secret object.                                               |
| 6    | Image source.                                                                                                                      |
| 7    | Container name.                                                                                                                    |
| 8    | Container ports.                                                                                                                   |
| 9    | Mount path information for the persistent volumes for each replica. Each persistent volume has the same configuration.             |
| 10   | The access mode of the persistent volume. You can choose between the `ReadWriteOnce`, `ReadWriteMany`, and `ReadOnlyMany` options. |
| 11   | The storage class that the persistent volume uses.                                                                                 |
| 12   | Size of the persistent volume.                                                                                                     |

---
**Note

Stateful sets can be created only by using manifest files. The `oc` and `kubectl` CLI do not have commands to create stateful sets imperatively.

---

Create the stateful set by using the `create` command:

```sh
[user@host ~]$ oc create -f statefulset-dbserver.yml
```

Verify the creation of the stateful set named `dbserver`:

```sh
[user@host ~]$ kubectl get statefulset
NAME       READY   AGE
dbserver   3/3     6s
```

Verify the status of the instances:

```sh
[user@host ~]$ oc get pods
NAME         READY   STATUS    RESTARTS   AGE
dbserver-0   1/1     Running   0          85s
dbserver-1   1/1     Running   0          82s
dbserver-2   1/1     Running   0          79s
```

Verify the status of the persistent volumes:

```sh
[user@host ~]$ kubectl get pvc
NAME              STATUS   VOLUME             CAPACITY   ACCESS MODES   STORAGECLASS ...
data-dbserver-0   Bound    pvc-c28f61ee-...   1Gi        RWO            nfs-storage  ...
data-dbserver-1   Bound    pvc-ddbe6af1-...   1Gi        RWO            nfs-storage  ...
data-dbserver-2   Bound    pvc-8302924a-...   1Gi        RWO            nfs-storage  ...
```
Notice that three PVCs were created. Confirm that persistent volumes are attached to each instance:

```sh
[user@host ~]$ oc describe pod dbserver-0
_...output omitted..._
Volumes:
  data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  data-dbserver-0
_...output omitted..._

[user@host ~]$ oc describe pod dbserver-1
_...output omitted..._
Volumes:
  data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  data-dbserver-1
_...output omitted..._

[user@host ~]$ oc describe pod dbserver-2
_...output omitted..._
Volumes:
  data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  data-dbserver-2
_...output omitted..._
```


>Note
You must configure application-level clustering for stateful set pods to have the same data.


You can update the number of replicas of the stateful set by using the `scale` command:

```sh
[user@host ~]$ oc scale statefulset/dbserver --replicas 1
NAME         READY   STATUS    RESTARTS  ...
dbserver-0   1/1     Running   0         ...
```

To delete the stateful set, use the `delete statefulset` command:

```sh
[user@host ~]$ kubectl delete statefulset dbserver
statefulset.apps "dbserver" deleted
```
Notice that the PVCs are not deleted after the execution of the `oc delete statefulset` command:

```
[user@host ~]$ oc get pvc
NAME              STATUS   VOLUME             CAPACITY   ACCESS MODES   STORAGECLASS ...
data-dbserver-0   Bound    pvc-c28f61ee-...   1Gi        RWO            nfs-storage  ...
data-dbserver-1   Bound    pvc-ddbe6af1-...   1Gi        RWO            nfs-storage  ...
data-dbserver-2   Bound    pvc-8302924a-...   1Gi        RWO            nfs-storage  ...
```
You can create a stateful set from the web console by clicking the Workloads → StatefulSets menu. Click Create StatefulSet and customize the YAML manifest.


## Summary

- Configuration maps are objects that provide mechanisms to inject configuration data into containers.
    
- The values in secrets are always encoded (not encrypted).
    
- A persistent volume claim (PVC) resource represents a request from an application for storage, and specifies the minimal storage characteristics, such as the capacity and access mode.
    
- Kubernetes supports two volume modes for persistent volumes: `Filesystem` and `Block`.
    
- Storage classes are a way to describe types of storage for the cluster and to provision dynamic storage on demand.
    
- A reclaim policy determines what happens to the data on a PVC after the PVC is deleted.
    
- A storage class with block volume mode support can improve performance for applications that can use raw block devices.
    
- A stateful set is the representation of a set of pods with consistent identities.
    
- Stateful set pods are assigned individual persistent volumes.


# Chapter 6: Configure Applications for Reliability

## Application High Availability with Kubernetes

### Objectives

- Describe how Kubernetes tries to keep applications running after failures.
    

### Concepts of Deploying Highly Available Applications

_High availability_ (HA) is a goal of making applications more robust and resistant to runtime failures. Implementing HA techniques decreases the likelihood that an application is completely unavailable to users.

In general, HA can protect an application from failures in the following contexts:

- From itself in the form of application bugs
    
- From its environment, such as networking issues
    
- From other applications that exhaust cluster resources
    

Additionally, HA practices can protect the cluster from applications, such as one with a memory leak.

### Writing Reliable Applications

At its core, cluster-level HA tooling mitigates worst-case scenarios. HA is not a substitute for fixing application-level issues, but augments developer mitigations. Although required for reliability, application security is a separate concern.

Applications must work with the cluster so that Kubernetes can best handle failure scenarios. Kubernetes expects the following behaviors from applications:

- Tolerates restarts
    
- Responds to health probes, such as the startup, readiness, and liveness probes
    
- Supports multiple simultaneous instances
    
- Has well-defined and well-behaved resource usage
    
- Operates with restricted privileges
    

Although the cluster can run applications that lack the preceding behaviors, applications with these behaviors better use the reliability and HA features that Kubernetes provides.

Most HTTP-based applications provide an endpoint to verify application health. The cluster can be configured to observe this endpoint and mitigate potential issues for the application.

The application is responsible for providing such an endpoint. Developers must decide how the application determines its state.

For example, if an application depends on a database connection, then the application might respond with a healthy status only when the database is reachable. However, not all applications that make database connections need such a check. This decision is at the discretion of the developers.

### Kubernetes Application Reliability

If an application pod crashes, then it cannot respond to requests. Depending on the configuration, the cluster can automatically restart the pod. If the application fails without crashing the pod, then the pod does not receive requests. However, the cluster can do so only with the appropriate health probes.

Kubernetes uses the following HA techniques to improve application reliability:

- Restarting pods: By configuring a restart policy on a pod, the cluster restarts misbehaving instances of an application.
    
- Probing: By using health probes, the cluster knows when applications cannot respond to requests, and can automatically act to mitigate the issue.
    
- Horizontal scaling: When the application load changes, the cluster can scale the number of replicas to match the load.
    

These techniques are explored throughout this chapter.


## Application Health Probes

### Objectives

- Describe how Kubernetes uses health probes during deployment, scaling, and failover of applications.
    

### Kubernetes Probes

Health probes are an important part of maintaining a robust cluster. _Probes_ enable the cluster to determine the status of an application by repeatedly probing it for a response.

A set of health probes affect a cluster's ability to do the following tasks:

- Crash mitigation by automatically attempting to restart failing pods
    
- Failover and load balancing by sending requests only to healthy pods
    
- Monitoring by determining whether and when pods are failing
    
- Scaling by determining when a new replica is ready to receive requests
    

### Authoring Probe Endpoints

Application developers are expected to code health probe endpoints during application development. These endpoints determine the health and status of the application. For example, a data-driven application might report a successful health probe only if it can connect to the database.

Because the cluster calls them often, health probe endpoints should be quick to perform. Endpoints should not perform complicated database queries or many network calls.

### Probe Types

Kubernetes provides the following types of probes: startup, readiness, and liveness. Depending on the application, you might configure one or more of these types.

#### Readiness Probes

A _readiness probe_ determines whether the application is ready to serve requests. If the readiness probe fails, then Kubernetes prevents client traffic from reaching the application by removing the pod's IP address from the service resource.

Readiness probes help to detect temporary issues that might affect your applications. For example, the application might be temporarily unavailable when it starts, because it must establish initial network connections, load files in a cache, or perform initial tasks that take time to complete. The application might occasionally need to run long batch jobs, which make it temporarily unavailable to clients.

Kubernetes continues to run the probe even after the application fails. If the probe succeeds again, then Kubernetes adds back the pod's IP address to the service resource, and requests are sent to the pod again.

In such cases, the readiness probe addresses a temporary issue and improves application availability.

#### Liveness Probes

Like a readiness probe, a _liveness probe_ is called throughout the lifetime of the application. Liveness probes determine whether the application container is in a healthy state. If an application fails its liveness probe enough times, then the cluster restarts the pod according to its restart policy.

Unlike a startup probe, liveness probes are called after the application's initial start process. Usually, this mitigation is by restarting or re-creating the pod.

#### Startup Probes

A _startup probe_ determines when an application's startup is completed. Unlike a liveness probe, a startup probe is not called after the probe succeeds. If the startup probe does not succeed after a configurable timeout, then the pod is restarted based on its `restartPolicy` value.

Consider adding a startup probe to applications with a long start time. By using a startup probe, the liveness probe can remain short and responsive.

### Types of Tests

When defining a probe, you must specify one of the following types of test to perform:

HTTP GET

Each time that the probe runs, the cluster sends a request to the specified HTTP endpoint. The test is considered a success if the request responds with an HTTP response code between `200` and `399`. Other responses cause the test to fail.

Container command

Each time that the probe runs, the cluster runs the specified command in the container. If the command exits with a status code of `0`, then the test succeeds. Other status codes cause the test to fail.

TCP socket

Each time that the probe runs, the cluster attempts to open a socket to the container. The test succeeds only if the connection is established.

### Timings and Thresholds

All the types of probes include timing variables. The _period seconds_ variable defines how often the probe runs. The _failure threshold_ defines how many failed attempts are required before the probe itself fails.

For example, a probe with a failure threshold of `3` and period seconds of `5` can fail up to three times before the overall probe fails. Using this probe configuration means that the issue can exist for 10 seconds before it is mitigated. However, running probes too often can waste resources. Consider these values when setting probes.

### Adding Probes via YAML

Because probes are defined on a pod template, probes can be added to workload resources such as deployments. To add a probe to an existing deployment, update and apply the YAML file or use the `oc edit` command. For example, the following YAML excerpt defines a deployment pod template with a probe:

apiVersion: apps/v1
kind: Deployment
_...output omitted..._
spec:
_...output omitted..._
  template:
    spec:
      containers:
      - name: web-server
        _...output omitted..._
        livenessProbe: ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
          failureThreshold: 6 ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
          periodSeconds: 10 ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)
          httpGet: ![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)
            path: /health ![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)
            port: 3000 ![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_adding_probes_via_yaml-CO32-1)|Defines a liveness probe.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_adding_probes_via_yaml-CO32-2)|Specifies how many times the probe must fail before mitigating.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_adding_probes_via_yaml-CO32-3)|Defines how often the probe runs.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_adding_probes_via_yaml-CO32-4)|Sets the probe as an HTTP request and defines the request port and path.|
|[![5](https://rol.redhat.com/rol/static/roc/Common_Content/images/5.svg)](https://rol.redhat.com/rol/app/#_adding_probes_via_yaml-CO32-5)|Specifies the HTTP path to send the request to.|
|[![6](https://rol.redhat.com/rol/static/roc/Common_Content/images/6.svg)](https://rol.redhat.com/rol/app/#_adding_probes_via_yaml-CO32-6)|Specifies the port to send the HTTP request over.|

### Adding Probes via the CLI

The `oc set probe` command adds or modifies a probe on a deployment. For example, the following command adds a readiness probe to a deployment called `front-end`:

[user@host ~]$ **`oc set probe deployment/front-end \ --readiness \ ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg) --failure-threshold 6 \ ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg) --period-seconds 10 \ ![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg) --get-url http://:8080/healthz`** ![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)

|   |   |
|---|---|
|[![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)](https://rol.redhat.com/rol/app/#_adding_probes_via_the_cli-CO33-1)|Defines a readiness probe.|
|[![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)](https://rol.redhat.com/rol/app/#_adding_probes_via_the_cli-CO33-2)|Sets how many times the probe must fail before mitigating.|
|[![3](https://rol.redhat.com/rol/static/roc/Common_Content/images/3.svg)](https://rol.redhat.com/rol/app/#_adding_probes_via_the_cli-CO33-3)|Sets how often the probe runs.|
|[![4](https://rol.redhat.com/rol/static/roc/Common_Content/images/4.svg)](https://rol.redhat.com/rol/app/#_adding_probes_via_the_cli-CO33-4)|Sets the probe as an HTTP request, and defines the request port and path.|

### Adding Probes via the Web Console

To add or modify a probe on a deployment from the web console, navigate to the Workloads → Deployments menu and select a deployment.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/reliability/probes/assets/deployments.png)|

Click Actions and then click Add Health Checks.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/reliability/probes/assets/addprobes.png)|

Click Edit probe to specify the readiness type, the HTTP headers, the path, the port, and more.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/reliability/probes/assets/editprobes.png)|

---
**Note

The `set probe` command is exclusive to RHOCP and `oc`.

---


## Reserve Compute Capacity for Applications

### Objectives

- Configure an application with resource requests so Kubernetes can make scheduling decisions.
    

### Kubernetes Pod Scheduling

The Red Hat OpenShift Container Platform (RHOCP) pod scheduler determines the placement of new pods onto nodes in the cluster. The pod scheduler algorithm follows a three-step process:

Filtering nodes

A pod can define a node selector that matches the labels in the cluster nodes. Only labels that match are eligible.

Additionally, the scheduler filters the list of running nodes by evaluating each node against a set of predicates. A pod can define _resource requests_ for compute resources such as CPU, memory, and storage. Only nodes with enough available computer resources are eligible.

The filtering step reduces the list of eligible nodes. In some cases, the pod could run on any of the nodes. In other cases, all the nodes are filtered out, so the pod cannot be scheduled until a node with the appropriate prerequisites becomes available.

If all nodes are filtered out, then a `FailedScheduling` event is generated for the pod.

Prioritizing the filtered list of nodes

By using multiple priority criteria, the scheduler determines a weighted score for each node. Nodes with higher scores are better candidates to run the pod.

Selecting the best fit node

The candidate list is sorted according to these scores, and the node with the highest score is selected to host the pod. If multiple nodes have the same high score, then one node is selected in a round-robin fashion. After a host is selected, then a `Scheduled` event is generated for the pod.

The scheduler is flexible and can be customized for advanced scheduling situations. Customizing the scheduler is outside the scope of this course.

### Compute Resource Requests

For such applications that require a specific amount of compute resources, you can define a resource request in the pod definition of your application deployment. The resource requests assign hardware resources for the application deployment.

Resource requests specify the minimum required compute resources necessary to schedule a pod. The scheduler tries to find a node with enough compute resources to satisfy the pod requests.

In Kubernetes, memory resources are measured in bytes, and CPU resources are measured in CPU units. CPU units are allocated by using millicore units. A millicore is a CPU core, either virtual or physical, that is split into 1000 units. A request value of `"1000 m"` allocates an entire CPU core to a pod. You can also use fractional values to allocate CPU resources. For example, you can set the CPU resource request to a `0.1` value, which represents 100 millicores (`100 m`). Likewise, a CPU resource request with a `1.0` value represents an entire CPU or 1000 millicores (`1000 m`).

You can define resource requests for each container in either a deployment or a deployment configuration resource. If resources are not defined, then the container specification shows a `resources: {}` line.

In your deployment, modify the `resources: {}` line to specify the chosen requests. The following example defines a resource request of 100 millicores (`100 m`) of CPU and one gibibyte (`1 Gi`) of memory.

```yaml
_...output omitted..._
    spec:
      containers:
      - image: quay.io/redhattraining/hello-world-nginx:v1.0
        name: hello-world-nginx
        resources:
          requests:
            cpu: "100m"
            memory: "1Gi"
```

If you use the `edit` command to modify a deployment or a deployment configuration, then ensure that you use the correct indentation. Indentation mistakes can result in the editor refusing to save changes. Alternatively, use the `set resources` command that the `kubectl` and `oc` commands provide, to specify resource requests. The following command sets the same requests as the preceding example:

```sh
[user@host ~]$ oc set resources deployment hello-world-nginx \ --requests cpu=10m,memory=1gi
```

The `set resource` command works with any resource that includes a pod template, such as the deployments and job resources.

### Inspecting Cluster Compute Resources

Cluster administrators can view the available and used compute resources of a node. For example, as a cluster administrator, you can use the `describe node` command to determine the compute resource capacity of a node. The command shows the capacity of the node and how much of that capacity is allocatable. It also displays the amount of allocated and requested resources on the node.
```sh
[user@host ~]$ oc describe node master01
Name:               master01
Roles:              control-plane,master,worker
_...output omitted..._
Capacity:
  cpu:                  8
  ephemeral-storage:    125293548Ki
  hugepages-1Gi:        0
  hugepages-2Mi:        0
  memory:               20531668Ki
  pods:                 250
Allocatable:
  cpu:                  7500m
  ephemeral-storage:    114396791822
  hugepages-1Gi:        0
  hugepages-2Mi:        0
  memory:               19389692Ki
  pods:                 250
_...output omitted..._
Non-terminated Pods:                                (88 in total)
  ... Name           CPU Requests  CPU Limits  Memory Requests  Memory Limits ...
  ... ----           ------------  ----------  ---------------  -------------
  ... controller-... 10m (0%)      0 (0%)      20Mi (0%)        0 (0%)        ...
  ... metallb-...    50m (0%)      0 (0%)      20Mi (0%)        0 (0%)        ...
  ... metallb-...    0 (0%)        0 (0%)      0 (0%)           0 (0%)        ...
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           `Requests`       Limits
  --------           `--------`       ------
  cpu                `3183m (42%)`    1202m (16%)
  memory             `12717Mi (67%)`  1350Mi (7%)
...output omitted...
```

RHOCP cluster administrators can also use the `oc adm top pods` command. This command shows the compute resource usage for each pod in a project. You must include the `--namespace` or `-n` options to specify a project. Otherwise, the command returns the resource usage for pods in the currently selected project.

The following command displays the resource usage for pods in the `openshift-dns` project:

```sh
[user@host ~]$ oc adm top pods -n openshift-dns
NAME                  CPU(cores)   MEMORY(bytes)
dns-default-5kpn5     1m           33Mi
node-resolver-6kdxp   0m           2Mi
```

Additionally, cluster administrators can use the `oc adm top node` command to view the resource usage of a cluster node. Include the node name to view the resource usage of a particular node.
```sh
[user@host ~]$ **`oc adm top node master01`**
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
master01   1250m        16%    10268Mi         54%
```

## Limit Compute Capacity for Applications

### Objectives

- Configure an application with resource limits so Kubernetes can protect other applications from it.

Memory and CPU requests that you define for containers help Red Hat OpenShift Container Platform (RHOCP) to select a compute node to run your pods. However, these resource requests do not restrict the memory and CPU that the containers can use. For example, setting a memory request at 1 GiB does not prevent the container from consuming more memory.

Red Hat recommends that you set the memory and CPU requests to the peak usage of your application. In contrast, by setting lower values, you overcommit the node resources. If all the applications that are running on the node start to use resources above the values that they request, then the compute nodes might run out of memory and CPU.

In addition to requests, you can set memory and CPU _limits_ to prevent your applications from consuming too many resources.

### Setting Memory Limits

A memory limit specifies the amount of memory that a container can use across all its processes.

As soon as the container reaches the limit, the compute node selects and then kills a process in the container. When that event occurs, RHOCP detects that the application is not working any more, because the main container process is missing, or because the health probes report an error. RHOCP then restarts the container according to the pod `restartPolicy` attribute, which defaults to `Always`.

RHOCP relies on Linux kernel features to implement resource limits, and to kill processes in containers that reach their memory limits:

Control groups (cgroups)

RHOCP uses control groups to implement resource limits. Control groups are a Linux kernel mechanism for controlling and monitoring system resources, such as CPU and memory.

Out-of-Memory killer (OOM killer)

When a container reaches its memory limit, the Linux kernel triggers the OOM killer subsystem to select and then kill a process.

You must set a memory limit when the application has a memory usage pattern that you must mitigate, such as when the application has a memory leak. A memory leak is a bug in the application, which occurs when the application uses some memory but does not free it after use. If the leak appears in an infinite service loop, then the application uses more and more memory over time, and can end up consuming all the available memory on the system. For these applications, setting a memory limit prevents them from consuming all the node's memory. The memory limit also enables OpenShift to regularly restart applications to free up their memory when they reach the limit.

To set a memory limit for the container in a pod, use the `oc set resources` command:

[user@host ~]$ **`oc set resources deployment/hello --limits memory=1Gi`**

In addition to the `oc set resources` command, you can define resource limits from a file in the YAML format:

```yaml
apiVersion: apps/v1
kind: Deployment
_...output omitted..._
  spec:
    containers:
    - image: registry.access.redhat.com/ubi9/nginx-120:1-86
      name: hello
      resources:
        requests:
          cpu: 100m
          memory: 500Mi
        `limits:`
          cpu: 200m
          `memory: 1Gi`
```

When RHOCP restarts a pod because of an OOM event, it updates the pod's `lastState` attribute, and sets the reason to `OOMKilled`:

```
[user@host ~]$ oc get pod hello-67645f4865-vvr42 -o yaml
_...output omitted..._
status:
_...output omitted..._
  containerStatuses:
  - containerID: cri-o://806b...9fe7
    image: registry.access.redhat.com/ubi9/nginx-120:1-86
    imageID: registry.access.redhat.com/ubi9/nginx-120:1-86@sha256:1403...fd34
    `lastState:`
      terminated:
        containerID: cri-o://bbc4...9eb2
        exitCode: 137
        finishedAt: "2023-03-08T07:56:06Z"
        `reason: OOMKilled`
        startedAt: "2023-03-08T07:51:43Z"
    name: hello
    ready: true
    restartCount: 1
_...output omitted..._
```

To set a memory limit for the container in a pod from the web console, select a deployment, and click Actions → Edit resource limits.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/reliability/limits/assets/selectresource.png)|

Set memory limits by increasing or decreasing the memory on the Limit section.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/reliability/limits/assets/memory-limits.png)|

### Setting CPU Limits

CPU limits work differently from memory limits. When the container reaches the CPU limit, RHOCP inhibits the container's processes, even if the node has available CPU cycles. The application continues to work, but at a slower pace.

In contrast, if you do not set a CPU limit, then the container can consume as much CPU as is available on the node. If the node's CPU is under pressure, for example because several containers are running CPU-intensive tasks, then the Linux kernel shares the CPU resource between all these containers, according to the CPU requests value for the containers.

You must set a CPU limit when you require a consistent application behavior across clusters and nodes. For example, if the application runs on a node where the CPU is available, then the application can execute at full speed. On the other hand, if the application runs on a node with CPU pressure, then the application executes at a slower pace.

The same behavior can occur between your development and production clusters. Because the two environments might have different node configurations, the application might run differently when you move it from development to production.

### Note

Clusters can have differences in hardware configuration beyond what limits observe. For example, two clusters' nodes might have CPUs with equal core count and unequal clock speeds.

Requests and limits do not account for these hardware differences. If your clusters differ in such a way, take care that requests and limits are appropriate for both configurations.

By setting a CPU limit, you mitigate the differences between the configuration of the nodes, and you experience a more consistent behavior.

To set a CPU limit for the container in a pod, use the `oc set resources` command:

```sh
[user@host ~]$ oc set resources deployment/hello --limits cpu=200m
```

You can also define CPU limits from a file in the YAML format:

```yaml
apiVersion: apps/v1
kind: Deployment
_...output omitted..._
  spec:
    containers:
    - image: registry.access.redhat.com/ubi9/nginx-120:1-86
      name: hello
      resources:
        requests:
          cpu: 100m
          memory: 500Mi
        `limits:`
          `cpu: 200m`
          memory: 1Gi
```

To set a CPU limit for the container in a pod from the web console, select a deployment, and click Actions → Edit resource limits.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/reliability/limits/assets/selectresource.png)|

Set CPU limits by increasing or decreasing the CPU on the Limit section.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/reliability/limits/assets/cpu-limits.png)|

### Viewing Requests, Limits, and Actual Usage

By using the RHOCP command-line interface, cluster administrators can view compute usage information on individual nodes. The `oc describe node` command displays detailed information about a node, including information about the pods that are running on the node. For each pod, it shows CPU requests and limits, as well as memory requests and limits. If you do not specify a request or limit, then the pod shows a zero for that column. The command also displays a summary of all the resource requests and limits.

```sh
[user@host ~]$ oc describe node master01
Name:               master01
Roles:              control-plane,master,worker
_...output omitted..._
Non-terminated Pods:                                (88 in total)
  ... Name           CPU Requests  CPU Limits  Memory Requests  Memory Limits ...
  ... ----           ------------  ----------  ---------------  -------------
  ... controller-... 10m (0%)      0 (0%)      20Mi (0%)        0 (0%)        ...
  ... metallb-...    50m (0%)      0 (0%)      20Mi (0%)        0 (0%)        ...
  ... metallb-...    0 (0%)        0 (0%)      0 (0%)           0 (0%)        ...
_...output omitted..._
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests       `Limits`
  --------           --------       `------`
  cpu                3183m (42%)    `1202m (16%)`
  memory             12717Mi (67%)  `1350Mi (7%)`
_...output omitted..._
```

The `oc describe node` command displays requests and limits. The `oc adm top` command shows resource usage. The `oc adm top nodes` command shows the resource usage for nodes in the cluster. You must run this command as the cluster administrator.

The `oc adm top pods` command shows the resource usage for each pod in a project.

The following command displays the resource usage for the pods in the current project:

```sh
[user@host ~]$ oc adm top pods -n openshift-console
NAME                         CPU(cores)   MEMORY(bytes)
hello-67645f4865-vvr42       121m         309Mi
intradb-6f8d7cfffb-fz55b     0m           440Mi
```

To visualize the consumption of resources from the web console, select a deployment, and click the Metrics tab. From this tab, you can view the usage for memory, CPU, the file system, and incoming and outgoing traffic.

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/reliability/limits/assets/metrics-usage.png)|


## Application Autoscaling

### Objectives

- Configure a horizontal pod autoscaler for an application.

Kubernetes can autoscale a deployment based on current load on the application pods, by means of a `HorizontalPodAutoscaler` (HPA) resource type.

A horizontal pod autoscaler resource uses performance metrics that the OpenShift Metrics subsystem collects. The Metrics subsystem comes preinstalled in OpenShift. To autoscale a deployment, you must specify resource requests for pods so that the horizontal pod autoscaler can calculate the percentage of usage.

The autoscaler works in a loop. Every 15 seconds by default, it performs the following steps:

- The autoscaler retrieves the details of the metric for scaling from the HPA resource.
- For each pod that the HPA resource targets, the autoscaler collects the metric from the metric subsystem.
- For each targeted pod, the autoscaler computes the usage percentage, from the collected metric and from the pod resource requests.
- The autoscaler computes the average usage and the average resource requests across all the targeted pods. It establishes a usage ratio from these values, and then uses the ratio for its scaling decision.
    

The simplest way to create a horizontal pod autoscaler resource is by using the `oc autoscale` command, for example:

```sh
[user@host ~]$ oc autoscale deployment/hello --min 1 --max 10 --cpu-percent 80
```

The previous command creates a horizontal pod autoscaler resource that changes the number of replicas on the `hello` deployment to keep its pods under 80% of their total requested CPU usage.

The `oc autoscale` command creates a horizontal pod autoscaler resource by using the name of the deployment as an argument (`hello` in the previous example).

The maximum and minimum values for the horizontal pod autoscaler resource accommodate bursts of load and avoid overloading the OpenShift cluster. If the load on the application changes too quickly, then it might help to keep several spare pods to cope with sudden bursts of user requests. Conversely, too many pods can use up all cluster capacity and impact other applications that use the same OpenShift cluster.

To get information about horizontal pod autoscaler resources in the current project, use the `oc get` command. For example:

```sh
[user@host ~]$ oc get hpa
NAME   REFERENCE               TARGETS        MINPODS  MAXPODS  REPLICAS  ...
hello  Deployment/hello  <unknown>/80%        1        10       1         ...
scale  Deployment/scale        60%/80%        2        10       2         ...
```

---
**Important**

The horizontal pod autoscaler initially has a value of `<unknown>` in the `TARGETS` column. It might take up to five minutes before `<unknown>` changes to display a percentage for current usage.

A persistent value of `<unknown>` in the `TARGETS` column might indicate that the deployment does not define resource requests for the metric. The horizontal pod autoscaler does not scale these pods.

Pods that are created by using the `oc create deployment` command do not define resource requests. Using the OpenShift autoscaler might therefore require editing the deployment resources, creating custom YAML or JSON resource files for your application, or adding limit range resources to your project that define default resource requests.

---

In addition to the `oc autoscale` command, you can create a horizontal pod autoscaler resource from a file in the YAML format.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hello
spec:
  minReplicas: 1   # 1
  maxReplicas: 10  # 2
  metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 80  # 3
        type: Utilization
    type: Resource
  scaleTargetRef:  # 4
    apiVersion: apps/v1
    kind: Deployment
    name: hello
```

|     |                                                                                                                                                                                                                                                      |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Minimum number of pods.                                                                                                                                                                                                                              |
| 2   | Maximum number of pods.                                                                                                                                                                                                                              |
| 3   | Ideal average CPU usage for each pod. If the global average CPU usage is above that value, then the horizontal pod autoscaler starts new pods. If the global average CPU usage is below that value, then the horizontal pod autoscaler deletes pods. |
| 4   | Reference to the name of the deployment resource.                                                                                                                                                                                                    |

Use the ``oc apply -f _`hello-hpa`_.yaml`` command to create the resource from the file.

The preceding example creates a horizontal pod autoscaler resource that scales based on CPU usage. Alternatively, it can scale based on memory usage by setting the resource name to `memory`, as in the following example:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hello
spec:
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - resource:
      name: `memory`
      target:
        averageUtilization: 80
_...output omitted..._
```

To create a horizontal pod autoscaler resource from the web console, click the Workloads → HorizontalPodAutoscalers menu. Click Create HorizontalPodAutoscaler and customize the YAML manifest.

>Note
If an application uses more overall memory as the number of replicas increases, then it cannot be used with memory-based autoscaling.

# Chapter 7: Manage Application Updates

## Container Image Identity and Tags

### Objectives

- Relate container image tags to their identifier hashes, and identify container images from pods and containers on Kubernetes nodes.

### Kubernetes Image Tags

The full name of a container image is composed of several parts. For example, you can decompose the `registry.access.redhat.com/ubi9/nginx-120:1-86` image name into the following elements:

- The registry server is `registry.access.redhat.com`.
- The namespace is `ubi9`.
- The name is `nginx-120`. In this example, the name of the image includes the version of the software, Nginx version 1.20.
- The tag, which points to a specific version of the image, is `1-86`. If you omit the tag, then most container tools use the `latest` tag by default.

Multiple tags can refer to the same image version. The following screen capture of the Red Hat Ecosystem Catalog at https://catalog.redhat.com/software/containers/explore lists the tags for the `ubi9/nginx-120` image:

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/updates/ids/assets/nginx-120-tags.png)

In this case, the `1.86`, `latest`, and `1` tags point to the same image version. You can use any of these tags to refer to that version.

The `latest` and `1` tags are _floating tags_, because they can point to different image versions over time. For example, when developers publish a new version of the image, they change the `latest` tag to point to that new version. They also update the `1` tag to point to the latest release of that version, such as `1-87` or `1-88`.

As a user of the image, by specifying a floating tag, you ensure that you always consume the up-to-date image version that corresponds to the tag.

#### Floating Tag Issues

Vendors, organizations, and developers who publish images manage their tags and establish their own lifecycle for floating tags. They can reassign a floating tag to a new image version without notice.

As a user of the image, you might not notice that the tag that you were using now points to a different image version.

Suppose that you deploy an application on OpenShift and use the `latest` tag for the image. The following series of events might occur:

1. When OpenShift deploys the container, it pulls the image with the `latest` tag from the container registry.
2. Later, the image developer pushes a new version of the image, and reassigns the `latest` tag to that new version.
3. OpenShift relocates the pod to a different cluster node, for example because the original node fails.
4. On that new node, OpenShift pulls the image with the `latest` tag, and thereby retrieves the new image version.
5. Now the OpenShift deployment runs with a new version of the application, without your awareness of that version update.

A similar issue is that when you scale up your deployment, OpenShift starts new pods. On the nodes, OpenShift pulls the `latest` image version for these new pods. As a result, if a new version is available, then your deployment runs with containers that use different versions of the image. Application inconsistencies and unexpected behavior might occur.

To prevent these issues, select an image that is guaranteed not to change over time. You thus gain control over the lifecycle of your application: you can choose when and how OpenShift deploys a new image version.

You can select a static image version in several ways:

- Use a tag that does not change, instead of relying on floating tags.
- Use OpenShift image streams for tight control over the image versions. Another section in this course discusses image streams further.
- Use the _SHA (Secure Hash Algorithm)_ image ID instead of a tag when referencing an image version.

The distinction between a floating and non-floating tag is not a technical one, but a convention. Although it is discouraged, there is no mechanism to prevent a developer from pushing a different image to an existing tag. Thus, you must specify the SHA image ID to guarantee that the referenced container image does not change.

#### Using SHA Image ID

Developers assign tags to images. In contrast, an SHA image ID, or _digest_, is a unique identifier that the container registry computes and assigns to images. The SHA ID is an immutable string that refers to a specific image version. Using the SHA ID for identifying an image is the most secure approach.

To refer to an image by its SHA ID, replace ``name:tag`` with ``name@`SHA-ID` `` in the image name. The following example uses the SHA image ID instead of a tag.

```
registry.access.redhat.com/ubi9/nginx-120@`sha256:1be2006abd21735e7684eb4cc6eb62...`
```

To retrieve the SHA image ID from the tag, use the `oc image info` command.

---
**Note**

A multi-architecture image references images for several CPU architectures. Multi-architecture images include an index that points to the images for different platforms and CPU architectures.

---

For these images, the `oc image info` command requires you to select an architecture by using the `--filter-by-os` option:

```sh
[user@host ~]$ oc image info registry.access.redhat.com/ubi9/nginx-120:1-86
error: the image is a manifest list and contains multiple images - use --filter-by-os to select from:

  OS            DIGEST
  linux/amd64   sha256:1be2006abd21735e7684eb4cc6eb6295346a89411a187e37cd4...
  linux/arm64   sha256:d765193e823bb89b878d2d2cb8be0e0073839a6c19073a21485...
  linux/ppc64le sha256:0dd0036620f525b3ba9a46f9f1c52ac70414f939446b2ba3a07...
  linux/s390x   sha256:d8d95cc17764b82b19977bc7ef2f60ff56a3944b3c7c14071dd...
```

The following example displays the SHA ID for the image that the `1-86` tag currently points to.

```sh
[user@host ~]$ oc image info --filter-by-os linux/amd64 \ registry.access.redhat.com/ubi9/nginx-120:1-86
Name:      registry.access.redhat.com/ubi9/nginx-120:1-86
`Digest:    sha256:1be2006abd21735e7684eb4cc6eb​6295346a89411a187e37cd4a3aa2f1bd13a5`
Manifest List: sha256:5bc635dc946fedb4ba391470e8f84f9860e06a1709e30206a95ed9955...
Media Type:    application/vnd.docker.distribution.manifest.v2+json
_...output omitted..._
```

You can also use the `skopeo inspect` command. The output format differs from the `oc image info` command, although both commands report similar data.

If you use the ``oc debug node/node-name`` command to connect to a compute node, then you can list the locally available images by running the `crictl images --digests --no-trunc` command. The `--digests` option instructs the command to display the SHA image IDs, and the `--no-trunc` option instructs the command to display the full SHA string; otherwise, the command displays only the first characters.

```sh
[user@host ~]$ oc debug node/node-name
Temporary namespace openshift-debug-csn2p is created for debugging node...
Starting pod/node-name-debug ...
To use host binaries, run chroot /host
Pod IP: 192.168.50.10
If you dont see a command prompt, try pressing enter.
sh-4.4# chroot /host
sh-4.4# crictl images --digests --no-trunc \ registry.access.redhat.com/ubi9/nginx-120:1-86
IMAGE                                     TAG  DIGEST              IMAGE ID    ...
registry.access.redhat.com/ubi9/nginx-120 1-86 `sha256:1be2...13a5`  2e68...949e ...
```

The `IMAGE ID` column displays the local image identifier that the container engine assigns to the image. This identifier is not related to the SHA ID.

The container image format relies on SHA-256 hashes to identify several image components, such as the image layers or the image metadata. Because some commands also report these SHA-256 strings, ensure that you use the SHA-256 hash that corresponds to the SHA image ID. Commands often refer to the SHA image ID as the image digest.

### Selecting a Pull Policy

When you deploy an application, OpenShift selects a compute node to run the pod. On that node, OpenShift pulls the image and then starts the container.

By setting the `imagePullPolicy` attribute in the deployment resource, you can control how OpenShift pulls the image.

The following example shows the `myapp` deployment resource. The pull policy is set to `IfNotPresent`.

```sh
[user@host ~]$ oc get deployment myapp -o yaml
apiVersion: apps/v1
kind: Deployment
_...output omitted..._
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: myapp
    spec:
      containers:
      - image: registry.access.redhat.com/ubi9/nginx-120:1-86
        imagePullPolicy: IfNotPresent
        name: nginx-120
_...output omitted..._
```

The `imagePullPolicy` attribute can take the following values:

`IfNotPresent`

If the image is already on the compute node, because another container is using it or because OpenShift pulled the image during a preceding pod run, then OpenShift uses that local image. Otherwise, OpenShift pulls the image from the container registry.

If you use a floating tag in your deployment, and the image with that tag is already on the node, then OpenShift does not pull the image again, even if the floating tag might point to a newer image in the source container registry.

OpenShift sets the `imagePullPolicy` attribute to `IfNotPresent` by default when you use a tag or the SHA ID to identify the image.

`Always`

OpenShift always verifies whether an updated version of the image is available on the source container registry. To do so, OpenShift retrieves the SHA ID of the image from the registry. If a local image with that same SHA ID is already on the compute node, then OpenShift uses that image. Otherwise, OpenShift pulls the image.

If you use a floating tag in your deployment, and an image with that tag is already on the node, then OpenShift queries the registry anyway to ensure that the tag still points to the same image version. However, if the developer pushed a new version of the image and updated the floating tag, then OpenShift retrieves that new image version.

OpenShift sets the `imagePullPolicy` attribute to `Always` by default when you use the `latest` tag, or when you do not specify a tag.

`Never`

OpenShift does not pull the image, and expects the image to be already available on the node. Otherwise, the deployment fails.

To use this option, you must prepopulate your compute nodes with the images that you plan to use. You use this mechanism to improve speed or to avoid relying on a container registry for these images.

### Pruning Images from Cluster Nodes

When OpenShift deletes a pod from a compute node, it does not remove the associated image. OpenShift can reuse the images without having to pull them again from the remote registry.

Because the images consume disk space on the compute nodes, OpenShift needs to remove, or _prune_, the unused images when disk space becomes sparse. The `kubelet` process, which runs on the compute nodes, includes a garbage collector that runs every five minutes. If the usage of the file system that stores the images is above 85%, then the garbage collector removes the oldest unused images. Garbage collection stops when the file system usage drops below 80%.

The reference documentation at the end of this lecture includes instructions to adjust these default thresholds.

From a compute node, you can run the `crictl imagefsinfo` command to retrieve the name of the file system that stores the images:

```sh
[user@host ~]$ oc debug node/node-name
Temporary namespace openshift-debug-csn2p is created for debugging node...
Starting pod/_`node-name`_-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.50.10
If you don't see a command prompt, try pressing enter.
sh-4.4# **`chroot /host`**
sh-4.4# **`crictl imagefsinfo`**
{
  "status": {
    "timestamp": "1674465624446958511",
    "fsId": {
      `"mountpoint": "/var/lib/containers/storage/overlay-images"`
    },
    `"usedBytes"`: {
      "value": "`` `1318560` ``"
    },
    "inodesUsed": {
      "value": "446"
    }
  }
}
```

From the preceding command output, the file system that stores the images is `/var/lib/containers/storage/overlay-images`. The images consume 1318560 bytes of disk space.

From the compute node, you can use the `crictl rmi` to remove an unused image. However, pruning objects by using the `crictl` command might interfere with the garbage collector and the `kubelet` process.

It is recommended that you rely on the garbage collector to prune unused objects, images, and containers from the compute nodes. The garbage collector is configurable to better fulfill custom needs that you might have.

## Update Application Image and Settings

### Objectives

- Update applications with minimal downtime by using deployment strategies.

### Application Code, Configuration, and Data

Modern applications loosely couple code, configuration, and data. Configuration files and data are not hard-coded as part of the software. Instead, the software loads the configuration and data from an external source. This externalization enables deploying an application to different environments without requiring a change to the application source code.

OpenShift provides configuration map, secret, and volume resources to store the application configuration and data. The application code is available through container images.

Because OpenShift deploys applications from container images, developers must build a new version of the image when they update the code of their application. Organizations usually use a continuous integration and continuous delivery (CI/CD) pipeline to automatically build the image from the application source code, and then to push the resulting image to a container registry.

You use OpenShift resources, such as configuration maps and secrets, to update the configuration of the application. To control the deployment process of a new image version, you use a `Deployment` object.

### Deployment Strategies

Deploying functional application changes or new versions to users is a significant phase of the CI/CD pipelines, where you add value to the development process.

Introducing application changes carries risks, such as downtime during the deployment, bugs, or reduced application performance. You can reduce or mitigate some risks with testing and validation stages in your pipelines.

Application or service downtime can result in lost business, disruption to other services that depend on yours, and violations of service level agreements, among others. To reduce downtime and minimize risks in deployments, use a _deployment strategy_. A deployment strategy changes or upgrades an application in a way that minimizes the impact of those changes.

In OpenShift, you use `Deployment` objects to define deployments and deployment strategies. The `RollingUpdate` and the `Recreate` strategies are the main OpenShift deployment strategies.

To select the `RollingUpdate` or `Recreate` strategies, you set the `.spec.strategy.type` property of the `Deployment` object. The following snippet shows a `Deployment` object that uses the `Recreate` strategy:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
_...output omitted..._
spec:
  progressDeadlineSeconds: 600
  replicas: 10
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: myapp2
  `strategy:     type: Recreate`
  template:
_...output omitted..._
```
#### Rolling Update Strategy

The `RollingUpdate` strategy consists of updating a version of an application in stages. It replaces one instance after another until all instances are replaced.

In this strategy, both versions of the application run simultaneously, and it scales down instances of the previous version only when the new version is ready. The main drawback is that this strategy requires compatibility between the versions in the deployment.

The following graphic shows the deployment of a new version of an application by using the `RollingUpdate` strategy:

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/updates/rollout/assets/rolling-strategy.svg)

1. Some application instances run a code version that needs updating (v1). OpenShift scales up a new instance with the updated application version (v2). Because the new instance with version v2 is not ready, only the version v1 instances fulfill customer requests.
2. The instance with v2 is ready and accepts customer requests. OpenShift scales down an instance with version v1, and scales up a new instance with version v2. Both versions of the application fulfill customer requests.
3. The new instance with v2 is ready and accepts customer requests. OpenShift scales down the remaining instance with version v1.
4. No instances remain to replace. The application update was successful, and without downtime.

The `RollingUpdate` strategy supports continuous deployment, and eliminates application downtime during deployments. You can use this strategy if the different versions of your application can run at the same time.

---
**Note**

The `RollingUpdate` strategy is the default strategy if you do not specify a strategy on the `Deployment` objects.

---

The following snippet shows a `Deployment` object that uses the `RollingUpdate` strategy:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
_...output omitted..._
spec:
  progressDeadlineSeconds: 600
  replicas: 10
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: myapp2
  strategy:
    rollingUpdate:
       maxSurge: 25%
       maxUnavailable: 50%
    type: RollingUpdate
  template:
_...output omitted..._
```

Out of many parameters to configure the `RollingUpdate` strategy, the preceding snippet shows the `maxSurge` and `maxUnavailable` parameters.

During a rolling update, the number of pods for the application varies, because OpenShift starts new pods for the new revision, and removes pods from the previous revision. The `maxSurge` parameter indicates how many pods OpenShift can create above the normal number of replicas. The `maxUnavailable` parameter indicates how many pods OpenShift can remove below the normal number of replicas. You can express these parameters as percentages or as a number of pods.

If you do not configure a readiness probe for your deployment, then during a rolling update, OpenShift starts sending client traffic to new pods as soon as they are running. However, the application inside a container might not be immediately ready to accept client requests. The application might have to load files to cache, establish a network connection to a database, or perform initial tasks that might take time to complete. Consequently, OpenShift redirects client requests to a container that is not yet ready, and these requests fail.

Adding a readiness probe to your deployment prevents OpenShift from sending traffic to new pods that are not ready.

#### Recreate Strategy

In this strategy, all the instances of an application are killed first, and are then replaced with new ones. The major drawback of this strategy is that it causes a downtime in your services. For a period, no application instances are available to fulfill requests.

The following graphic shows the deployment of a new version of an application that uses the `Recreate` strategy:

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/updates/rollout/assets/recreate-strategy.svg)

1. The application has some instances that run a code version to update (v1).
2. OpenShift scales down the running instances to zero. This action causes application downtime, because no instances are available to fulfill requests.
3. OpenShift scales up new instances with a new version of the application (v2). When the new instances are booting, the downtime continues.
4. The new instances finished booting, and are ready to fulfill requests. This step is the last step of the `Recreate` strategy, and it resolves the application outage.

You can use this strategy when your application cannot have different simultaneously running code versions. You might also use it to execute data migrations or data transformations before the new code starts. This strategy is not recommended for applications that need high availability, for example, medical systems.

### Rolling out Applications

When you update a `Deployment` object, OpenShift automatically rolls out the application. If you apply several modifications in a row, such as modifying the image version, updating environment variables, and configuring the readiness probe, then OpenShift rolls out the application for each modification.

To prevent these multiple deployments, pause the rollout, apply all your modifications to the `Deployment` object, and then resume the rollout. OpenShift then performs a single rollout to apply all your modifications:

- Use the `oc rollout pause` command to pause the rollout of the `myapp` deployment:

```sh
[user@host ~]$ oc rollout pause deployment/myapp
```

- Apply all your modifications to the `Deployment` object. The following example modifies the image, an environment variable, and the readiness probe.
```sh
[user@host ~]$ oc set image deployment/myapp \ nginx-120=registry.access.redhat.com/ubi9/nginx-120:1-86
[user@host ~]$ oc set env deployment/myapp NGINX_LOG_TO_VOLUME=1
[user@host ~]$ oc set probe deployment/myapp --readiness --get-url http://:8080
```

- Resume the rollout:
```sh
[user@host ~]$ oc rollout resume deployment/myapp
```

OpenShift rolls out the application to apply all your modifications to the `Deployment` object.


You can follow a similar process when you create and configure a new deployment:

- Create the deployment, and set the number of replicas to zero. This way, OpenShift does not roll out your application, and no pods are running.
```sh
[user@host ~]$ oc create deployment myapp2 \ --image registry.access.redhat.com/ubi9/nginx-120:1-86 --replicas 0
[user@host ~]$ oc get deployment/myapp2
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
myapp2   `0/0     0            0`           9s
```

- Apply the configuration to the `Deployment` object. The following example adds a readiness probe.
```sh
[user@host ~]$ oc set probe deployment/myapp2 --readiness --get-url http://:8080
```

- Scale up the deployment. OpenShift rolls out the application.
```sh
[user@host ~]$ oc scale deployment/myapp2 --replicas 10
[user@host ~]$ oc get deployment/myapp2
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
myapp2   `10/10   10           10`          18s
```

#### Monitoring Replica Sets

Whenever OpenShift rolls out an application from a `Deployment` object, it creates a `ReplicaSet` object. Replica sets are responsible for creating and monitoring the pods. If a pod fails, then the `ReplicaSet` object deploys a new one.

To deploy pods, replica sets use the pod template definition from the `Deployment` object. OpenShift copies the template definition from the `Deployment` object when it creates the `ReplicaSet` object.

When you update the `Deployment` object, OpenShift does not update the existing `ReplicaSet` object. Instead, it creates another `ReplicaSet` object with the new pod template definition. Then, OpenShift rolls out the application according to the update strategy.

Thus, several `ReplicaSet` objects for a deployment can exist at the same time on your system. During a rolling update, the old and the new `ReplicaSet` objects coexist and coordinate the rollout of the new application version. After the rollout completes, OpenShift keeps the old `ReplicaSet` object so that you can roll back if the new application version does not operate correctly.

The following graphic shows a `Deployment` object and two `ReplicaSet` objects. The old `ReplicaSet` object for version 1 of the application does not run any pods. The current `ReplicaSet` object for version 2 of the application manages three replicated pods.

![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/updates/rollout/assets/replicaset.svg)

Do not directly change or delete `ReplicaSet` objects, because OpenShift manages them through the associated `Deployment` objects. The `.spec.revisionHistoryLimit` attribute in `Deployment` objects specifies how many `ReplicaSet` objects OpenShift keeps. OpenShift automatically deletes the extra `ReplicaSet` objects. Also, when you delete a `Deployment` object, OpenShift deletes all the associated `ReplicaSet` objects.

Run the `oc get replicaset` command to list the `ReplicaSet` objects. OpenShift uses the `Deployment` object name as a prefix for the `ReplicaSet` objects.

```sh
[user@host ~]$ oc get replicaset
NAME                DESIRED   CURRENT   READY   AGE
myapp2-574968dd59   0         0         0       3m27s
myapp2-76679885b9   10        10        10      22s
myapp2-786cbf9bc8   0         0         0       114s
```

The preceding output shows three `ReplicaSet` objects for the `myapp2` deployment. Whenever you modified the `myapp2` deployment, OpenShift created a `ReplicaSet` object. The second object in the list is active and monitors 10 pods. The other `ReplicaSet` objects do not manage any pods. They represent the previous versions of the `Deployment` object.

During a rolling update, two `ReplicaSet` objects are active. The old `ReplicaSet` object is scaling down, and at the same time the new object is scaling up:

```sh
[user@host ~]$ oc get replicaset
NAME                DESIRED   CURRENT   READY   AGE
myapp2-574968dd59   0         0         0       13m
myapp2-5fb5766df5   4         4         2       21s  # 1
myapp2-76679885b9   8         8         8       10m  # 2
myapp2-786cbf9bc8   0         0         0       11m
```

|     |                                                                                                                                                                                                       |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | The new `ReplicaSet` object already started four pods, but the `READY` column shows that the readiness probe succeeded for only two pods so far. These two pods are likely to receive client traffic. |
| 2   | The `ReplicaSet` object already scaled down from 10 to 8 pods.                                                                                                                                        |

#### Managing Rollout

Because OpenShift preserves `ReplicaSet` objects from earlier deployment versions, you can roll back if you notice that the new version of the application does not work.

Use the `oc rollout undo` command to roll back to the preceding deployment version. The command uses the existing `ReplicaSet` object for that version to roll back the pods. The command also reverts the `Deployment` object to the preceding version.

```sh
[user@host ~]$ oc rollout undo deployment/myapp2
```
Use the `oc rollout status` command to control the rollout process:

```sh
[user@host ~]$ oc rollout status deployment/myapp2
deployment "myapp2" successfully rolled out
```

If the rollout operation fails, because you specify a wrong container image name or the readiness probe fails, then OpenShift does not automatically roll back your deployment. In this case, run the `oc rollout undo` command to revert to the preceding working configuration.

By default, the `oc rollout undo` command rolls back to the preceding deployment version. If you need to roll back to an earlier revision, then list the available revisions and add the ``--to-revision _`rev`_`` option to the `oc rollout undo` command.

- Use the `oc rollout history` command to list the available revisions:
```sh
[user@host ~]$ oc rollout history deployment/myapp2
deployment.apps/myapp2
REVISION  CHANGE-CAUSE
1         <none>
3         <none>
4         <none>
5         <none>
```

---
**Note**

The `CHANGE-CAUSE` column provides a user-defined message that describes the revision. You can store the message in the `kubernetes.io/change-cause` deployment annotation after every rollout:

```sh
[user@host ~]$ oc annotate deployment/myapp2 \
kubernetes.io/change-cause="Image updated to 1-86"
deployment.apps/myapp2 annotated
[user@host ~]$ oc rollout history deployment/myapp2
deployment.apps/myapp2
REVISION  CHANGE-CAUSE
1         <none>
3         <none>
4         <none>
5         Image updated to 1-86
```

---

- Add the `--revision` option to the `oc rollout history` command for more details about a specific revision:
```sh
[user@host ~]$ oc rollout history deployment/myapp2 --revision 1
deployment.apps/myapp2 with revision #1
Pod Template:
  Labels: app=myapp2
    pod-template-hash=574968dd59
  Containers:
    nginx-120:
      Image:       registry.access.redhat.com/ubi9/nginx-120:1-86
      Port:        <none>
      Host Port:   <none>
      Environment: <none>
      Mounts:      <none>
   Volumes: <none>
```

The `pod-template-hash` attribute is the suffix of the associated `ReplicaSet` object. You can inspect that `ReplicaSet` object for more details by using the `oc describe replicaset myapp2-574968dd59` command, for example.


- Roll back to a specific revision by adding the `--to-revision` option to the `oc rollout undo` command:
```sh
[user@host ~]$ oc rollout undo deployment/myapp2 --to-revision 1
```

If you use floating tags to refer to container image versions in deployments, then the resulting image when you roll back a deployment might have changed in the container registry. Thus, the image that you run after the rollback might not be the original one that you used.

To prevent this issue, use OpenShift image streams for referencing images instead of floating tags. Another section in this course discusses image streams further.

## Reproducible Deployments with OpenShift Image Streams

### Objectives

- Ensure reproducibility of application deployments by using image streams and short image names.

### Image Streams

The distinction between a floating and a non-floating tag is a convention rather than technical. Although it is discouraged, no mechanism exists to prevent a developer from pushing a different image to an existing tag. Therefore, a pod that references an image tag is not guaranteed to retrieve the same image when the pod is restarted. Image streams resolve this problem and also provide essential rollback capabilities.

Image streams are one of the main differentiators between OpenShift and upstream Kubernetes. Whereas Kubernetes resources reference container images directly, OpenShift resources, such as build configurations, reference image streams. OpenShift also extends Kubernetes resources, such as Kubernetes Deployments, with annotations that make them work with OpenShift image streams.

With image streams, OpenShift can ensure reproducible, stable deployments of containerized applications and also rollbacks of deployments to their latest known-good state.

Image streams provide a stable, short name to reference a container image that is independent of any registry server and container runtime configuration.

#### Image Stream Tags

An _image stream_ represents one or more sets of container images. Each set, or stream, is identified by an _image stream tag_. Unlike container images in a registry server, which have multiple tags from the same image repository (or user or organization), an image stream can have multiple image stream tags that reference container images from different registry servers and from different image repositories.

An image stream provides default configurations for a set of image stream tags. Each image stream tag references one stream of container images, and can override most configurations from its associated image stream.

In the following illustration, a deployment that uses the `Image-4` image can be rolled back to using the `Image-3` image, because the `ISTAG-B` image stream tag keeps a history of used images:

|   |
|---|
|![](https://static.ole.redhat.com/rhls/courses/do180-4.14/images/updates/imagestreams/images/imagestreams/openshift-image-streams.svg)|

An image stream tag stores a copy of the metadata about its current container image, including the SHA image ID. The SHA image ID is a unique identifier that the container registry computes and assigns to images. Storing metadata supports faster search and inspection of container images, because you do not need to reach its source registry server.

You can also configure an image stream tag to store the source image layers in the OpenShift internal container registry, which acts as a local image cache. Storing image layers locally avoids the need to fetch these layers from their source registry server. Consumers of the cached image, such as pods and deployments, just reference the internal registry as the source registry of the image.

To better visualize the relationship between image streams and image stream tags, you can explore the `openshift` project that is pre-created in all OpenShift clusters. You can see many image streams in that project, including the `php` image stream:

```bash
[user@host ~]$ oc get is -n openshift -o name
...output omitted...
imagestream.image.openshift.io/nodejs
imagestream.image.openshift.io/perl
`imagestream.image.openshift.io/php`
imagestream.image.openshift.io/postgresql
imagestream.image.openshift.io/python
...output omitted...
```

Several tags exist for the `php` image stream, and an image stream tag resource exists for each tag:

```bash
[user@host ~]$ oc get istag -n openshift | grep php
8.0-ubi9      image-registry ...    6 days ago
8.0-ubi8      image-registry ...    6 days ago
7.4-ubi8      image-registry ...    6 days ago
7.3-ubi7      image-registry ...    6 days ago
```

The `oc describe` command on an image stream shows information from both the image stream and its image stream tags:

```sh
[user@host ~]$ oc describe is php -n openshift
Name:                   php
Namespace:              openshift
...output omitted...
Tags:                   5

`8.0-ubi9`
  tagged from registry.access.redhat.com/ubi9/php-80:latest
...output omitted...

`8.0-ubi8` (latest)
  tagged from registry.access.redhat.com/ubi8/php-80:latest
...output omitted...

`7.4-ubi8`
  tagged from registry.access.redhat.com/ubi8/php-74:latest
..output omitted...

`7.3-ubi7`
  tagged from registry.access.redhat.com/ubi7/php-73:latest
...output omitted...
```
In the previous example, each of the `php` image stream tags refers to a different image name.

#### Image Names, Tags, and IDs

The textual name of a container image is a string. This name is sometimes interpreted as being made of multiple components, such as `registry-host-name/repository-or-organization-or-user-name/image-name:tag-name`, but splitting the image name into its components is a matter of convention, not of structure.

A SHA image ID is a SHA-256 hash that uniquely identifies an immutable container image. You cannot modify a container image. Instead, you create a container image that has a new ID. When you push a new container image to a registry server, the server associates the existing textual name with the new image ID.

When you start a container from an image name, you download the image that is currently associated with that image name. The image ID behind that name might change at any moment, and the next container that you start might have a different image ID. If the image that is associated with an image name has any issues, and you know only the image name, then you cannot roll back to an earlier image.

OpenShift image stream tags keep a history of the latest image IDs that they fetched from a registry server. The history of image IDs is the stream of images from an image stream tag. You can use the history inside an image stream tag to roll back to a previous image, if for example a new container image causes a deployment error.

Updating a container image in an external registry does not automatically update an image stream tag. The image stream tag keeps the reference to the last image ID that it fetched. This behavior is crucial to scaling applications, because it isolates OpenShift from changes that happen at a registry server.

Suppose that you deploy an application from an external registry, and after a few days of testing with a few users, you decide to scale its deployment to enable a larger user population. In the meantime, your vendor updates the container image on the external registry. If OpenShift had no image stream tags, then the new pods would get the new container image, which is different from the image on the original pod. Depending on the changes, this new image could cause your application to fail. Because OpenShift stores the image ID of the original image in an image stream tag, it can create new pods by using the same image ID and avoid any incompatibility between the original and updated image.

OpenShift keeps the image ID of the first pod, and ensures that new pods use the same image ID. OpenShift ensures that all pods use the same image.

To better visualize the relationship between an image stream, an image stream tag, an image name, and an image ID, refer to the following `oc describe is` command, which shows the source image and current image ID for each image stream tag:

```sh
[user@host ~]$ oc describe is php -n openshift
Name:                   php
Namespace:              openshift
_...output omitted..._

`8.0-ubi9`
  tagged from registry.access.redhat.com/ubi9/php-80:latest
_...output omitted..._
  * registry.access.redhat.com/ubi9/php-80@sha256:`2b82...f544`
      2 days ago

`8.0-ubi8` (latest)
  tagged from registry.access.redhat.com/ubi8/php-80:latest
  * registry.access.redhat.com/ubi8/php-80@sha256:`2c74...5ef4`
      2 days ago
_...output omitted..._
```
If your OpenShift cluster administrator already updated the `php:8.0-ubi9` image stream tag, the `oc describe is` command shows multiple image IDs for that tag:

```sh
[user@host ~]$ oc describe is php -n openshift
Name:                   php
Namespace:              openshift
_...output omitted..._

`8.0-ubi9`
  tagged from registry.access.redhat.com/ubi9/php-80:latest
_...output omitted..._
  * registry.access.redhat.com/ubi9/php-80@sha256:`2b82...f544`
      2 days ago
    registry.access.redhat.com/ubi9/php-80@sha256:`8840...94f0`
      5 days ago
    registry.access.redhat.com/ubi9/php-80@sha256:`506c...5d90`
      9 days ago
```

In the previous example, the asterisk (*) shows which image ID is the current one for each image stream tag. It is usually the latest one to be imported, and the first one that is listed.

When an OpenShift image stream tag references a container image from an external registry, you must explicitly update the image stream tag to get new image IDs from the external registry. By default, OpenShift does not monitor external registries for changes to the image ID that is associated with an image name.

You can configure an image stream tag to check the external registry for updates on a defined schedule. By default, new image stream tags do not check for updated images.

#### Image Stream Use Cases

Image streams provide an evolution approach for the container workflows of an organization, and can improve deployment reliability and resilience.

As an example, an organization could start by downloading container images directly from the Red Hat public registry and later set up an enterprise registry as a mirror of those images to save bandwidth. OpenShift users would not notice any change, because they still refer to these images by using the same image stream name. Users of the RHEL container tools would notice the change, because those users would need either to change the registry names in their commands, or to change their container engine configurations to search for the local mirror first.

In other scenarios, the indirection that an image stream provides can be helpful. Suppose that you start with a database container image that has security issues, and the vendor takes too long to update the image with fixes. Later, you find another vendor who provides an alternative container image for the same database, where those security issues are already fixed, and even better, with a track record of providing timely updates to them. If those container images have a compatible configuration of environment variables and volumes, you could change your image stream to point to the image from the alternative vendor.

Red Hat provides hardened, supported container images, such as the MariaDB database, that work mostly as drop-in replacements of container images from popular open source projects. Replacing unreliable image sources with supported Red Hat alternatives, when available, is a preferred use of image streams.

### Creating Image Streams and Tags

In addition to the image streams in the `openshift` project, you can create image streams in your project so that the resources in that project, such as `Deployment` objects, can use them.

Use the `oc create is` command to create an image stream in the current project. The following example creates an image stream named `keycloak`:

```sh
[user@host ~]$ oc create is keycloak
```

After you create the image stream, use the `oc create istag` command to add image stream tags. The following example adds the `20.0` tag to the `keycloak` image stream. In this example, the image stream tag refers to the `quay.io/keycloak/keycloak:20.0.2` image from the Quay.io public repository.

```sh
[user@host ~]$ oc create istag keycloak:20.0 \
  --from-image quay.io/keycloak/keycloak:20.0.2
```

Repeat the preceding command if you need more image stream tags:

```sh
[user@host ~]$ oc create istag keycloak:19.0 \
  *--from-image quay.io/keycloak/keycloak:19.0
```

Use the ``oc tag _`SOURCE-IMAGE`_ _`IMAGE-STREAM-TAG`_`` command to update an image stream tag with a new source image reference. The following example changes the `keycloak:20.0` image stream tag to point to the `quay.io/keycloak/keycloak:20.0.3` image:

```sh
[user@host ~]$ oc tag quay.io/keycloak/keycloak:20.0.3 keycloak:20.0
```

Use the `oc describe is` command to verify that the image stream tag points to the SHA ID of the source image:

```sh
[user@host ~]$ oc describe is keycloak
Name:             keycloak
Namespace:        myproject
Created:          5 minutes ago
Labels:           <none>
Annotations:      openshift.io/image.dockerRepositoryCheck=2023-01-31T11:12:44Z
Image Repository: image-registry.openshift-image-registry.svc:5000/.../keycloak
Image Lookup:     local=false
Unique Images:    3
Tags:             2

20.0
  tagged from quay.io/keycloak/keycloak:20.0.3

  * quay.io/keycloak/keycloak@sha256:`c167...62e9`
      47 seconds ago
    quay.io/keycloak/keycloak@sha256:`5569...b311`
      5 minutes ago

19.0
  tagged from quay.io/keycloak/keycloak:19.0

  * quay.io/keycloak/keycloak@sha256:`40cc...ffde`
      5 minutes ago
```

#### Importing Image Stream Tags Periodically

When you create an image stream tag, OpenShift configures it with the SHA ID of the source image that you specify. After that initial creation, the image stream tag does not change, even if the developer pushes a new version of the source image.

By using image stream tags, you are in control of the images that your applications are using. If you want to use a new image version, then you manually need to update the image stream tag to point to that new version.

However, for some container registries that you trust, or for some specific images, you might prefer the image stream tags to automatically refresh.

For example, Red Hat regularly updates the images from the Red Hat Ecosystem Catalog with bug and security fixes. To benefit from these updates as soon as Red Hat releases them, you can configure your image stream tags to regularly refresh.

OpenShift can periodically verify whether a new image version is available. When it detects a new version, it automatically updates the image stream tag. To activate that periodic refresh, add the `--scheduled` option to the `oc tag` command.

```sh
[user@host ~]$ oc tag quay.io/keycloak/keycloak:20.0.3 keycloak:20.0 --scheduled
```

By default, OpenShift verifies the image every 15 minutes. This period is a setting that your cluster administrators can adapt.

#### Configuring Image Pull-through

When OpenShift starts a pod that uses an image stream tag, it pulls the corresponding image from the source container registry.

When the image comes from a registry on the internet, pulling the image can take time, or even fail in case of a network outage. Some public registries have bandwidth throttling rules that can slow down your downloads further.

To mitigate these issues, you can configure your image stream tags to cache the images in the OpenShift internal container registry. The first time that OpenShift pulls the image, it downloads the image from the source repository and then stores the image in its internal registry. After that initial pull, OpenShift retrieves the image from the internal registry.

To activate image pull-through, add the `--reference-policy local` option to the `oc tag` command.

```sh
[user@host ~]$ oc tag quay.io/keycloak/keycloak:20.0.3 keycloak:20.0 \
  --reference-policy local
```

### Using Image Streams in Deployments

When you create a `Deployment` object, you can specify an image stream instead of a container image from a registry. Using an image stream in Kubernetes workload resources, such as deployments, requires preparation:

- Create the image stream object in the same project as the `Deployment` object.
    
- Enable the local lookup policy in the image stream object.
    
- In the `Deployment` object, reference the image stream tag by its name, such as `keycloak:20.0`, and not by the full image name from the source registry.
    

#### Enabling the Local Lookup Policy

When you use an image stream in a `Deployment` object, OpenShift looks for that image stream in the current project. However, OpenShift searches only the image streams that you enabled the local lookup policy for.

Use the `oc set image-lookup` command to enable the local lookup policy for an image stream:

```sh
[user@host ~]$ oc set image-lookup keycloak
```

Use the `oc describe is` command to verify that the policy is active:

```sh
[user@host ~]$ oc describe is keycloak
Name:             keycloak
Namespace:        myproject
Created:          3 hours ago
Labels:           <none>
Annotations:      openshift.io/image.dockerRepositoryCheck=2023-01-31T11:12:44Z
Image Repository: image-registry.openshift-image-registry.svc:5000/.../keycloak
`Image Lookup:     local=true`
Unique Images:    3
Tags:             2
_...output omitted..._
```

You can also retrieve the local lookup policy status for all the image streams in the current project by running the `oc set image-lookup` command without parameters:

```sh
[user@host ~]$ oc set image-lookup
NAME          LOCAL
keycloak      true
zabbix-agent  false
nagios        false
```

To disable the local lookup policy, add the `--enabled=false` option to the `oc set image-lookup` command:

```sh
[user@host ~]$ oc set image-lookup keycloak --enabled=false
```

#### Configuring Image Streams in Deployments

When you create a `Deployment` object by using the `oc create deployment` command, use the `--image` option to specify the image stream tag:

```sh
[user@host ~]$ oc create deployment mykeycloak --image keycloak:20.0
```

When you use a short name, OpenShift looks for a matching image stream in the current project. OpenShift considers only the image streams that you enabled the local lookup policy for. If it does not find an image stream, then OpenShift looks for a regular container image in the allowed container registries. The reference documentation at the end of this lecture describes how to configure these allowed registries.

You can also use image streams with other Kubernetes workload resources:

- `Job` objects that you can create by using the following command:
    
    [user@host ~]$ **``oc create job _`NAME`_ --image _`IMAGE-STREAM-TAG`_ -- _`COMMAND`_``**
    
- `CronJob` objects that you can create by using the following command:
    
    [user@host ~]$ **``oc create cronjob _`NAME`_ --image _`IMAGE-STREAM-TAG`_ \``**
      **``--schedule _`CRON-SYNTAX`_ -- _`COMMAND`_``**
    
- `Pod` objects that you can create by using the following command:
    
    [user@host ~]$ **``oc run _`NAME`_ --image _`IMAGE-STREAM-TAG`_``**
    

Another section in this course discusses how changing an image stream tag can automatically roll out the associated deployments.


## Automatic Image Updates with OpenShift Image Change Triggers

### Objectives

- Ensure automatic update of application pods by using image streams with Kubernetes workload resources.

### Using Triggers to Manage Images

Image stream tags record the SHA ID of the source container image. Thus, an image stream tag always points to an immutable image.

If a new version of the source image becomes available, then you can change the image stream tag to point to that new image. However, a `Deployment` object that uses the image stream tag does not roll out automatically. For an automatic rollout, you must configure the `Deployment` object with an image trigger.

If you update an image stream tag to point to a new image version, and you notice that this version does not work as expected, then you can revert the image stream tag. `Deployment` objects for which you configured a trigger automatically roll back to that previous image.

Other Kubernetes workloads also support image triggers, such as `Pod`, `CronJob`, and `Job` objects.

### Configuring Image Trigger for Deployments

Before you can configure image triggers for a `Deployment` object, ensure that the `Deployment` object is using image stream tags for its containers:

- Create the image stream object in the same project as the `Deployment` object.
- Enable the local lookup policy in the image stream object by using the `oc set image-lookup` command.
- In the `Deployment` object, reference the image stream tags by their names, such as `keycloak:20`, and not by the full image names from the source registry.

Image triggers apply at the container level. If your `Deployment` object includes several containers, then you can specify a trigger for each one. Before you can set triggers, retrieve the container names:

```sh
[user@host ~]$ oc get deployment mykeycloak -o wide
NAME         READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS ...
mykeycloak   0/1     1            0           6s    `keycloak`   ...
```

Use the `oc set triggers` command to configure an image trigger for the container inside the `Deployment` object. Use the `--from-image` option to specify the image stream tag to watch.

```sh
[user@host ~]$ oc set triggers deployment/mykeycloak --from-image keycloak:20 \
  --containers keycloak
```

To provide automatic image rollout for `Deployment` objects, OpenShift adds the `image.openshift.io/triggers` annotation to store the configuration in JSON format.

The following example retrieves the content of the `image.openshift.io/triggers` annotation, and then uses the `jq` command to display the configuration in a more readable format:

```sh
[user@host ~]$ oc get deployment mykeycloak \
  -o jsonpath='{.metadata.annotations.image\.openshift\.io/triggers}' | jq .
[
  {
    "from": {
      "kind": "ImageStreamTag",
      "name": "keycloak:20"
    },
    "fieldPath": "spec.template.spec.containers[?(@.name==\"keycloak\")].image"
  }
]
```

The `fieldPath` attribute is a JSONPath expression that OpenShift uses to locate the attribute that stores the container image name. OpenShift updates that attribute with the new image name and SHA ID whenever the image stream tag changes.

For a more concise view, use the `oc set triggers` command with the name of the `Deployment` object as an argument:

```sh
[user@host ~]$ oc set triggers deployment/mykeycloak
NAME                    TYPE    VALUE                   AUTO
deployments/mykeycloak  config                                  # [1]
deployments/mykeycloak  image   keycloak:20 (keycloak)  true    # [2]
```

|     |                                                                                                                                                                                         |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | OpenShift uses the configuration trigger to roll out the deployment whenever you change its configuration, such as to update environment variables or to configure the readiness probe. |
| 2   | OpenShift watches the `keycloak:20` image stream tag that the `keycloak` container uses.                                                                                                |

The `true` value under the `AUTO` column indicates that the trigger is enabled.

You can disable the configuration trigger by using the `oc rollout pause` command, and you can re-enable it by using the `oc rollout resume` command.

You can disable the image trigger by adding the `--manual` option to the `oc set triggers` command:

```sh
[user@host ~]$ oc set triggers deployment/mykeycloak --manual \
  --from-image keycloak:20 --containers keycloak
```

You re-enable the trigger by using the `--auto` option:

```sh
[user@host ~]$ oc set triggers deployment/mykeycloak --auto \
  --from-image keycloak:20 --containers keycloak
```

You can remove the triggers from all the containers in the `Deployment` object by adding the `--remove-all` option to the command:

```sh
[user@host ~]$ oc set triggers deployment/mykeycloak --remove-all
```

#### Rolling out Deployments

A `Deployment` object with an image trigger automatically rolls out when the image stream tag changes.

The image stream tag might change because you manually update it to point to a new version of the source image. The image stream tag might also change automatically if you configure it for periodic refresh, by adding the `--scheduled` option to the `oc tag` command. When the image stream tag automatically changes, all the `Deployment` objects with a trigger that refers to that image stream tag also roll out.

#### Rolling Back Deployments

For Kubernetes `Deployment` objects, to roll back the deployment, you revert the image stream tag. By reverting the image stream tag, OpenShift rolls out the `Deployment` object to use the previous image that the image stream tag is pointing to again.

### Managing Image Stream Tags

You can create image streams and image stream tags in several ways. The following commands perform the same operation. They all create the `keycloak` image stream if it does not exist, and then create the `keycloak:20.0.2` image stream tag:

```sh
[user@host ~]$ oc create istag keycloak:20.0.2  --from-image quay.io/keycloak/keycloak:20.0.2

[user@host ~]$ oc import-image keycloak:20.0.2  --from quay.io/keycloak/keycloak:20.0.2 --confirm

[user@host ~]$ oc tag quay.io/keycloak/keycloak:20.0.2 keycloak:20.0.2
```

You can rerun the `oc import-image` and `oc tag` commands to update the image stream tag from the source image. If the source image changes, then the commands update the image stream tag to point to that new version. However, you can use the `oc create istag` command only to initially create the image stream tag. You cannot update tags by using that command.

Use the `--help` option for more details about the commands.

You can create several image stream tags that point to the same image. The following command creates the `keycloak:20` image stream tag, which points to the same image as the `keycloak:20.0.2` image stream tag. In other words, the `keycloak:20` image stream tag is an alias for the `keycloak:20.0.2` image stream tag.

```sh
[user@host ~]$ oc tag --alias keycloak:20.0.2 keycloak:20
```

The `oc describe is` command reports that both tags point to the same image:

```sh
[user@host ~]$ oc describe is keycloak
Name:       keycloak
Namespace:  myproject
...output omitted...

`20.0.2 (20)`
  tagged from quay.io/keycloak/keycloak:20.0.2

  * quay.io/keycloak/keycloak@sha256:5569...b311
      3 minutes ago
```

Using aliases is a similar concept to floating tags for container images. Suppose that a new image version is available in the Quay.io repository. You could create an image stream tag for that new image:

```sh
[user@host ~]$ oc create istag keycloak:20.0.3 --from-image quay.io/keycloak/keycloak:20.0.3
imagestreamtag.image.openshift.io/keycloak:20.0.3 created
[user@host ~]$ oc describe is keycloak
Name:       keycloak
Namespace:  myproject
_...output omitted..._

`20.0.3`
  tagged from quay.io/keycloak/keycloak:20.0.3

  * quay.io/keycloak/keycloak@sha256:c167...62e9
      36 seconds ago

20.0.2 `(20)`
  tagged from quay.io/keycloak/keycloak:20.0.2

  * quay.io/keycloak/keycloak@sha256:5569...b311
      About an hour ago
```

The `keycloak:20` image stream tag does not change. Therefore, the `Deployment` objects that use that tag do not roll out.

After testing the new image, you can move the `keycloak:20` tag to point to the new image stream tag:

```sh
[user@host ~]$ oc tag --alias keycloak:20.0.3 keycloak:20
Tag keycloak:20 set up to track keycloak:20.0.3.
[user@host ~]$ oc describe is keycloak
Name:       keycloak
Namespace:  myproject
_...output omitted..._

`20.0.3 (20)`
  tagged from quay.io/keycloak/keycloak:20.0.3

  * quay.io/keycloak/keycloak@sha256:c167...62e9
      10 minutes ago

20.0.2
  tagged from quay.io/keycloak/keycloak:20.0.2

  * quay.io/keycloak/keycloak@sha256:5569...b311
      About an hour ago
```

Because the `keycloak:20` image stream tag points to a new image, OpenShift rolls out all the `Deployment` objects that use that tag.

If the new application does not work as expected, then you can roll back the deployments by resetting the `keycloak:20` tag to the previous image stream tag:

```sh
[user@host ~]$ oc tag --alias keycloak:20.0.2 keycloak:20
```

By providing a level of indirection, image streams give you control over managing the container images that you use in your OpenShift cluster.

