# Pods

- [Pods in Kubernetes](#pods-in-kubernetes)
- [The Pod Manifest](#the-pod-manifest)
    - [Creating a Pod](#creating-a-pod)
    - [Creating a Pod Manifest](#creating-a-pod-manifest)
    - [Running Pods](#running-pods)
    - [Listing Pods](#listing-pods)
    - [Pod Details](#pod-details)
    - [Deleting a Pod](#deleting-a-pod)
- [Accessing Your Pod](#accessing-your-pod)
    - [Getting more information with Logs](#getting-more-information-with-logs)
    - [Running commands in your container with exec](#running-commands-in-your-container-with-exec)
    - [Copying files to and from containers](#copying-files-to-and-from-containers)
- [Health Checks](#health-checks)
    - [Liveness Probes](#liveness-probes)
    - [Readiness Probes](#readiness-probe)
    - [Startup Probe](#startup-probes)
    - [Other types of health checks](#other-types-of-health-checks)
- [Resource Management](#resource-management)
    - [Resource Requests](#resource-requests)
    - [Capping Resource Usage with Limits](#capping-resource-usage-with-limits)
- [Persisting Data with Volumes](#persisting-data-with-volumes)
    - [Using Volumes with Pods](#using-volumes-with-pods)
    - [Different ways of using Volumes with Pods](#different-ways-of-using-volumes-with-pods)
- [More Information](#more-information)


## Pods in Kubernetes

- A Pod is a collection of application containers and volumes running in the same execution environment.
- Pods, not containers, are the smallest deployable artifact in a cluster.
- All of the containers in a Pod land on the same machine.
- Each container within a Pod runs in its own cgroup, but they share a number of Linux namespaces.
- Applications running in the same Pod share the same IP address and port space (networking space), have the same hostname, and can communicate using native interprocess communication channels.
- Applications in different Pods are isolated from each other.

## The Pod Manifest

- Pods are described in a manifest file.
- The Kubernetes API accepts and processes Pod manifests before storing them in persistent storage (etcd).
- The scheduler also uses the Kubernetes API to find Pods that haven't been scheduled onto a Node.
- It then places the pods onto nodes depending on resources and constraints expressed in Pod manifests.

### Creating a Pod.

We can create a Pod imperatively by running the following command:

```bash
kubectl run <pod-name> --generator=run-pod/v1 --image=<pod-image>
```

We can see the status of this Pod by running:

```bash
kubectl get pods
```

And we can delete it by running the following:

```bash
kubectl delete pods/<pod-name>
```

### Creating a Pod manifest

- You can write Pod manifests using YAML or JSON. (YAML is preferred)
- Pod manifests should be treated in the same way as source code, with comments to explain things to new team members.
- Pod manifests include a couple of key fields and attributes

We can define a Pod manifest like so:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard
  labels:
    name: kuard
spec:
  containers:
  - name: kuard
    image: willvelida/kuard
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 8080
        name: http
        protocol: TCP
```

### Running Pods

To create the above Pod manifest, we can run the following:

```bash
kubectl apply -f kuard-pod.yaml
```

### Listing Pods

Using `kubectl`, we can list all the pods running in the cluster. So for example:

```bash
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
kuard   1/1     Running   0          5s
```

Here, we can see the name of the Pod, as well as the number of ready containers, and the status of the Pod. We can also see how many times the Pod has restarted, and how old the Pod is.

### Pod Details

Kubernetes maintains numerous events about Pods that are present in the event stream, but are not attached to the Pod object.

We can use the `describe` command to find out more information about our Pod. For example:

```
kubectl describe pods kuard
```

This will output information about the Pod, including the basics, containers, and events related to the Pod. So for example:

```bash
Name:             kuard
Namespace:        default
Priority:         0
Service Account:  default
Node:             aks-default-78443803-vmss000002/10.224.0.5
Start Time:       Mon, 13 Jan 2025 18:30:00 +1100
Labels:           name=kuard
Annotations:      <none>
Status:           Running
IP:               10.244.1.23
IPs:
  IP:  10.244.1.23
Containers:
  kuard:
    Container ID:   containerd://ae1918ba5f422a33e517158f71b4cf7b27727c29698af89fd04268d22d95e43b
    Image:          willvelida/kuard
    Image ID:       docker.io/willvelida/kuard@sha256:e6dc719c81bfd5e0f3789339389910094fd7bad7e52a29ed6325fac299bd51ec
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 13 Jan 2025 18:30:02 +1100
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  128Mi
    Requests:
      cpu:        500m
      memory:     128Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-ljhqh (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
Volumes:
  kube-api-access-ljhqh:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Guaranteed
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/memory-pressure:NoSchedule op=Exists
                             node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m30s  default-scheduler  Successfully assigned default/kuard to aks-default-78443803-vmss000002
  Normal  Pulling    3m29s  kubelet            Pulling image "willvelida/kuard"
  Normal  Pulled     3m28s  kubelet            Successfully pulled image "willvelida/kuard" in 1.567s (1.568s including waiting). Image size: 12272126 bytes.
  Normal  Created    3m28s  kubelet            Created container kuard
  Normal  Started    3m28s  kubelet            Started container kuard
```

### Deleting a Pod

When you want to delete the pod, you can delete it either by name, or through the same file that you used to create it:

```bash
kubectl delete pods/kuard
```

or

```bash
kubectl delete -f kuard-pod.yaml
```

When a Pod is deleted, **it is not immediately killed**. Instead, you'll see it in `Terminating` state. 

All Pods have a termination grace period (default is 30 seconds). When a Pod is transitioned to `Terminating`, it no longer receives new requests. It allows the Pod to finish requests that it may be in the middle of processing before it's terminated.

## Accessing your Pod

- Once your pod is running, you're going to want to access it for a variety of reasons.
- You may want to view its logs to debug a problem that you're seeing, or even execute other commands to help debug it.

### Getting more information with Logs

You can view the current logs from the running instance using `kubectl logs`. You can add the `-f` flag to stream the logs continuously.

```bash
kubectl logs kuard
```

The `kubectl logs` command always tries to get logs from the currently running container. Adding the previous `--previous` flag will get logs from a previous instance of the container.

### Running commands in your container with exec

Sometimes you need to execute commands in the context of the container itself. You can do so like this:

```bash
kubectl exec <pod-name> -- date
```

You can also get an interactive session using the `-it` flag.

### Copying Files to and from Containers

- Generally, copying files into a container is an antipattern.
- You should treat the contents of a container as immutable.
- Occassionally, it's the most immediate way to stop the bleeding and restore your service to health.
- Once the issue is over, build a new image with the file, and roll it out.

## Health Checks

- When you run your app as a Container in Kubernetes, it's kept alive for us using a *process health check*.
- This health check simply ensures that the main process for your app is always running. If not, Kubernetes restarts it.
- However in most cases, this is insufficient. To address this, Kubernetes has a application *liveness check* that run application specific logic to verify that the app is not just running, but is functioning properly.
- Since these liveness checks are application-specific, we have to define them in the Pod manifest.

### Liveness Probes

- Liveness probes are defined per container, which means each container defined inside a Pod must be checked separately.

We can define a health check like so:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard
  labels:
    name: kuard
spec:
  containers:
  - name: kuard
    image: willvelida/kuard
    livenessProbe:
      httpGet:
        path: /healthy
        port: 8080
      initialDelaySeconds: 5
      timeoutSeconds: 1
      periodSeconds: 10
      failureThreshold: 3
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 8080
        name: http
        protocol: TCP
```

- This manifest uses a `httpGet` probe to perform an HTTP GET request against the `/healthy` endpoint of port 8080.
- The probe sets an `initialDelaySeconds` of 5, meaning it won't be called until 5 seconds after all Pods in the container are created.
- The probe must respond within the 1-second timeout, and the HTTP status code must be equal or greater than 200 and less than 400 to be successful.
- Kubernetes will call the probe every 10 seconds, and if it fails 3 times, the container will fail and restart.

### Readiness Probe

- Readiness describes when a container is ready to serve user requests.
- Containers that fail readiness checks are removed from service load balancers.
- Readiness probes are configured similarly to liveness probes.
- Combining Readiness and Liveness probes helps ensure only healthy containers are running within the cluster.

### Startup Probes

- When a Pod is started, the startup probe is run before any other probing of the Pod has started.
- The startup probe proceeds until it either times out (the Pod restarts in this case), or it succeeds, and the liveness probe takes over.
- Startup probes enable you to poll slowly for a slow-starting container, while also enabling a responsive liveness check once the slow-starting container has initialized.

### Other types of health checks

- In addition to HTTP, Kubernetes also supports `tcpSocket` health checks that open a TCP socket. If it succeeds, the probe succeeds.
- This is useful for non-HTTP applications, such as Databases.
- Kubernetes also allows `exec`probes. These execute a script or program in the context of a container.
- These are great for custom application logic that doesn't fit neatly into a HTTP call.

## Resource Management

- Kubernetes also enables users to increase the overall utilization of the compute nodes that make up the cluster.
- We measure this with the utilization metric. **Utilization is defined as the amount of a resource actively being used divided by the amount of resource that has been purchased**
- We have to tell Kubernetes about the resources an app needs so that Kubernetes can find the optimal packing of containers onto machines.
- Kubernetes allows users to specify two different resource metrics:
    - **Resource requests** specify the minimum amount of a resource required to run the application.
    - **Resource limits** specify the maximum amount of a resource that an application can consume.

### Resource Requests

- When a Pod requests the resources required to run its containers, Kubernetes guarantees that these resources are available to the Pod.
- This is most commonly CPU and Memory, but can also include GPU.

Here's an example of how to allocate resources to a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard
  labels:
    name: kuard
spec:
  containers:
  - name: kuard
    image: willvelida/kuard
    livenessProbe:
      httpGet:
        path: /healthy
        port: 8080
      initialDelaySeconds: 5
      timeoutSeconds: 1
      periodSeconds: 10
      failureThreshold: 3
    resources:
      requests:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 8080
        name: http
        protocol: TCP
```

> [!NOTE]
> Resources are requested per container, not Pod. The total resources requested by the Pod is the sum of all resources needed for the Container.

Requests are used when scheduling Pods to nodes. The Kubernetes scheduler will ensure that the sum of all requests of all Pods on a node does not exceed the capacity of the node.

### Capping Resource Usage with Limits

- In addition to setting the resources required by a Pod, you can also set a maximum on its resource usage with *limits*.
- When we set limits on a container, the kernel is configured to ensure that consumption cannot exceed these limits.

So instead of using the *requests* property in our Pod definition, we can define limits as well using the *limits* property, like so:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard
  labels:
    name: kuard
spec:
  containers:
  - name: kuard
    image: willvelida/kuard
    livenessProbe:
      httpGet:
        path: /healthy
        port: 8080
      initialDelaySeconds: 5
      timeoutSeconds: 1
      periodSeconds: 10
      failureThreshold: 3
    resources:
      requests:
        memory: "128Mi"
        cpu: "500m"
      limits:
        memory: "256Mi"
        cpu: "1000m"
    ports:
      - containerPort: 8080
        name: http
        protocol: TCP
```

## Persisting Data with Volumes

- When a Pod is deleted or a container restarts, any and all data in the container's filesystem is also deleted.
- This is often a good thing for stateless applications.
- In other cases, we need access to persistent disk storage.

### Using Volumes with Pods

To add a volume to a Pod manifest, we need to add two things to our Pod configuration:

1. `spec.volumes` - This array defines all of the volumes that may be accessed by containers in the Pod manifest. Not all containers are required to mount all volumes defined by the Pod.
2. `volumeMounts` - This array defines the volumes that are mounted into a particular container and the path where each volume should be mounted. Two different containers in a Pod can mount the same volume at different mount paths.

So with that in mind, we can define our mounts in our Pod like so:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard
  labels:
    name: kuard
spec:
  volumes:
    - name: "kuard-data"
      hostPath:
        path: "/var/lib/kuard"
  containers:
  - name: kuard
    volumeMounts:
      - mountPath: "/data"
        name: "kuard-data"
    image: willvelida/kuard
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 8080
        name: http
        protocol: TCP
```

### Different ways of using Volumes with Pods

There are a variety of ways you can use data in your application. The following are some of these ways and the recommended patterns for Kubernetes.

*Communication/synchronization*

- To achieve synchronization, Pods can use an `emptyDir` volume.
- This volume to scoped to the Pod's lifespan, but can also be shared between Containers in the Pod, allowing them to communicate.

*Cache*

- An app may use a volume that helps performance, but not required for correct operation of it (For example, rendering images)
- You want such a cache to survive a container restart due to a health-check failure, and thus `emptyDir` works well for the cache use case as well.

*Persistent Data*

- Sometimes you need a volume for truly persistent data.
- To achieve this, Kubernetes supports a wide variety of remote network storage volumes, including NFS, Azure File and Disk etc.

*Mounting the host filesystem*

- Other apps don't need persistent volumes, but they do need some access to the underlying host filesystem.
- Kubernetes supports the `hostPath` volume, which can mount arbitrary locations on the worker node into the container.

## More Information

- [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)