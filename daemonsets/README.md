# DaemonSets

- Deployments and ReplicaSets are generally about creating a service with multiple replicas for redundancy.
- Generally, the motivation for replicating a Pod to every node is to land some sort of agent or daemon on each node, and the Kubernetes Object for achieving this is the DaemonSet.
- DaemonSets ensure that a copy of a Pod is running across a set of nodes in a Kubernetes cluster.
- They are used to deploy system daemons such as log collectors and monitoring agents, which typically must run on every node.
- DaemonSets share similar functionality with ReplicaSets; both create Pods that are expected to be long-running services and ensure that the desired state and the observed state match.

*ReplicaSets* should be used when your application is decoupled from the node and you can run multiple copies on a given node without special consideration.

*DaemonSets* should be used when a single copy of your application must run on all or a subset of the nodes in the cluster.

You should generally not use scheduling restrictions or other parameters to ensure that Pods do not colocate on the same node.

- You can use labels to run DaemonSet Pods on specific nodes (for example, you may want to run specialized intrusion-detection software on nodes that are exposed to the edge network)
- You can also use DaemonSets to install software on nodes in a cloud-based cluster. To install specific software on each node, a DaemonSet is the right approach.

## DaemonSet Scheduler

- By default, a DaemonSet will create a copy of a Pod on every node unless a node selector is used, which will limit eligible nodes to those with a matching set of labels.
- DaemonSets determine which node a Pod will run on at Pod creation time by specifying the `nodeName` field in the Pod `spec`.
- As a result, Pods created by DaemonSets are ignored by the Kubernetes scheduler.
- DaemonSets are managed by a reconciliation control loop that measures the desired state with the observed state. With this information, the DaemonSet controller creates a Pod on each node that doesn't currently have a matching Pod.
- If a new node is added to the cluster, then the DaemonSet controller notices that it is missing a Pod, and adds the Pod to the new node.

## Creating DaemonSets

DaemonSets are created by submitting a DaemonSet configuration to the Kubernetes API server.

This DaemonSet will create a `fluentd` logging agent on every node in the target cluster:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
        - name: fluentd
          image: fluent/fluentd:v0.14.10
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 200Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

- DaemonSets require a unique name across all DaemonSets in a given Kubernetes namespace.
- Each DaemonSet must include a Pod template spec, which will be used to create Pods as needed
- Unlike ReplicaSets, DaemonSets will create pods on every node in the cluster by default unless a node selector is used.

Once we have a valid DaemonSet configuration in place, we can use `kubectl apply`  to submit the DaemonSet to the API.

```bash
$ kubectl apply -f fluentd.yaml
```

Once it has been successfully applied, we can query its current state using `kubectl describe`

```bash
$ kubectl describe daemonset fluentd
kubectl describe daemonset fluentd
Name:           fluentd
Selector:       app=fluentd
Node-Selector:  <none>
Labels:         app=fluentd
Annotations:    deprecated.daemonset.template.generation: 1
Desired Number of Nodes Scheduled: 3
Current Number of Nodes Scheduled: 3
Number of Nodes Scheduled with Up-to-date Pods: 3
Number of Nodes Scheduled with Available Pods: 3
Number of Nodes Misscheduled: 0
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=fluentd
  Containers:
   fluentd:
    Image:      fluent/fluentd:v0.14.10
    Port:       <none>
    Host Port:  <none>
    Limits:
      memory:  200Mi
    Requests:
      cpu:        100m
      memory:     200Mi
    Environment:  <none>
    Mounts:
      /var/lib/docker/containers from varlibdockercontainers (ro)
      /var/log from varlog (rw)
  Volumes:
   varlog:
    Type:          HostPath (bare host directory volume)
    Path:          /var/log
    HostPathType:
   varlibdockercontainers:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/docker/containers
    HostPathType:
  Node-Selectors:  <none>
  Tolerations:     <none>
Events:
  Type    Reason            Age    From                  Message
  ----    ------            ----   ----                  -------
  Normal  SuccessfulCreate  3m10s  daemonset-controller  Created pod: fluentd-2vr6p
  Normal  SuccessfulCreate  3m10s  daemonset-controller  Created pod: fluentd-72vws
  Normal  SuccessfulCreate  3m10s  daemonset-controller  Created pod: fluentd-htrnq
```

This output indicates a `fluentd` Pod was successfully deployed to all three nodes in our cluster. We can verify this using the `kubectl get pods` command with the `-o` flag to print the nodes where each `fluent` Pod was assigned:

```bash
$ kubectl get pods -l app=fluentd -o wide
NAME            READY   STATUS    RESTARTS   AGE     IP            NODE                              NOMINATED NODE   READINESS GATES
fluentd-2vr6p   1/1     Running   0          4m46s   10.244.2.43   aks-default-78443803-vmss000001   <none>           <none>
fluentd-72vws   1/1     Running   0          4m46s   10.244.0.11   aks-default-78443803-vmss000004   <none>           <none>
fluentd-htrnq   1/1     Running   0          4m46s   10.244.3.46   aks-default-78443803-vmss000002   <none>           <none>
```

With the `fluentd` DaemonSet in place, adding a new node to the cluster will result in a `fluentd` Pod being deployed to that node automatically.

This is what we want when managing daemons and other cluster-wide services.

## Limiting DaemonSets to Specific Nodes

- The most common use case for DaemonSets is to run a Pod across every node in a Kubernetes cluster.
- However, there are some cases where you want to deploy a Pod to only a subset of nodes.
- For example, you may want a workload to have access to GPU that's only available to subset of nodes in your cluster.

### Adding labels to nodes

The first step in limiting DaemonSets to specific nodes is to add the desired set of labels to a subset of nodes. This can be achieved using the `kubectl label` command.

The following command adds the `ssd=true` label to a single node:

```bash
$ kubectl label nodes aks-default-78443803-vmss000001 ssd=true
node/aks-default-78443803-vmss000001 labeled
```

Using a label selector, we can filter nodes based on labels. To list only the nodes that have the `ssd` label set to true, we can use the `--selector` flag:

```bash
$ kubectl get nodes --selector ssd=true
NAME                              STATUS   ROLES    AGE   VERSION
aks-default-78443803-vmss000001   Ready    <none>   9d    v1.30.6
```

### Node Selectors

- Node Selectors can be used to limit what nodes a Pod can run on a given Kubernetes cluster
- Node selectors are defined as part of the Pod spec when creating a DaemonSet.

Here's an example of a DaemonSet configuration that limits NGINX to running only on nodes with the `ssd=true` label set:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: nginx
    ssd: "true"
  name: nginx-fast-storage
spec:
  selector:
    matchLabels:
      app: nginx
      ssd: "true"
  template:
    metadata:
      labels:
        app: nginx
        ssd: "true"
    spec:
      nodeSelector:
        ssd: "true"
      containers:
        - name: nginx
          image: nginx:1.10.0
```

Let's apply it and see what happens:

```bash
$ kubectl apply -f .\nginx-fast-storage.yaml
daemonset.apps/nginx-fast-storage created
```

Since there's only one node with the `ssd=true` label, the `nginx-fast-storage` Pod will only run on that node:

```bash
$ kubectl get pods -l app=nginx -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE                              NOMINATED NODE   READINESS GATES
nginx-fast-storage-lcktn   1/1     Running   0          59s   10.244.2.44   aks-default-78443803-vmss000001   <none>           <none>
```

Adding the `ssd=true` label to additional nodes will cause the `nginx-fast-storage` Pod to be deployed on those Nodes. The inverse is also true: if a required label is removed from a node, the Pod will be removed by the DaemonSet controller.

## Updating a DaemonSet

- DaemonSets can be rolled out using the same `RollingUpdate` strategy that Deployments use.
- You can configure the update strategy using the `spec.updateStrategy.type` field, which should have the value `RollingUpdate`.
- When a DaemonSet has an update strategy of `RollingUpdate`, any change to the `spec.template` field in the DaemonSet will initiate a rolling update.

There are two parameters that control the rolling update of a DaemonSet:

`spec.minReadySeconds` - Determines how long a Pod must be ready before the rolling update proceeds to upgrade subsequent Pods.

`spec.updateStrategy.rollingUpdate.maxUnavailable` - Indicates how many Pods may be simultaneously updated by the rolling update.

- You will likely want to set `spec.minReadySeconds` to a reasonably long value to ensure that your Pod is truly healthy before the rollout proceeds.
- The setting for `spec.updateStrategy.rollingUpdate.maxUnavailable` is more likely to be application specific.
- Increasing the maximum unavailability will make your rollout move faster, but increases the blast radius of a failed rollout.
- The characteristics of your application and cluster environment dictate the relative values of speed vs safety.
- A good approach might be set `maxUnavailable` to 1, and only increase it if users or admins complain about DaemonSet rollout speed.

Once a rolling update has started, we can use `kubectl rollout` commands to see the current status of a DaemonSet rollout.

## Deleting a DaemonSet

To delete a DaemonSet, we can run the following:

```bash
$ kubectl delete -f fluentd.yaml
```

> [!NOTE]
> Deleting a DaemonSet will also delete all the Pods being managed by that DaemonSet. Set the `--cascade` flag to `false` to ensure only the DaemonSet is deleted and not the Pods.