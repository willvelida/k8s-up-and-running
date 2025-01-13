# Pods

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

## Creating a Pod.

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

## Creating a Pod manifest

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

## Running Pods

To create the above Pod manifest, we can run the following:

```bash
kubectl apply -f kuard-pod.yaml
```

## Listing Pods

Using `kubectl`, we can list all the pods running in the cluster. So for example:

```bash
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
kuard   1/1     Running   0          5s
```

Here, we can see the name of the Pod, as well as the number of ready containers, and the status of the Pod. We can also see how many times the Pod has restarted, and how old the Pod is.

## Pod Details

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

## Deleting a Pod

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

## Getting more information with Logs

You can view the current logs from the running instance using `kubectl logs`. You can add the `-f` flag to stream the logs continuously.

```bash
kubectl logs kuard
```

The `kubectl logs` command always tries to get logs from the currently running container. Adding the previous `--previous` flag will get logs from a previous instance of the container.

## Running commands in your container with exec

Sometimes you need to execute commands in the context of the container itself. You can do so like this:

```bash
kubectl exec <pod-name> -- date
```

You can also get an interactive session using the `-it` flag.

## Copying Files to and from Containers

- Generally, copying files into a container is an antipattern.
- You should treat the contents of a container as immutable.
- Occassionally, it's the most immediate way to stop the bleeding and restore your service to health.
- Once the issue is over, build a new image with the file, and roll it out.

## Health Checks

- When you run your app as a Container in Kubernetes, it's kept alive for us using a *process health check*.
- This health check simply ensures that the main process for your app is always running. If not, Kubernetes restarts it.
- However in most cases, this is insufficient. To address this, Kubernetes has a application *liveness check* that run application specific logic to verify that the app is not just running, but is functioning properly.
- Since these liveness checks are application-specific, we have to define them in the Pod manifest.

## Liveness Probes

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

## Readiness Probe

- Readiness describes when a container is ready to serve user requests.
- Containers that fail readiness checks are removed from service load balancers.
- Readiness probes are configured similarly to liveness probes.
- Combining Readiness and Liveness probes helps ensure only healthy containers are running within the cluster.

## Startup Probes

- When a Pod is started, the startup probe is run before any other probing of the Pod has started.
- The startup probe proceeds until it either times out (the Pod restarts in this case), or it succeeds, and the liveness probe takes over.
- Startup probes enable you to poll slowly for a slow-starting container, while also enabling a responsive liveness check once the slow-starting container has initialized.

## Other types of health checks

- In addition to HTTP, Kubernetes also supports `tcpSocket` health checks that open a TCP socket. If it succeeds, the probe succeeds.
- This is useful for non-HTTP applications, such as Databases.
- Kubernetes also allows `exec`probes. These execute a script or program in the context of a container.
- These are great for custom application logic that doesn't fit neatly into a HTTP call.