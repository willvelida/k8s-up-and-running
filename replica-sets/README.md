# ReplicaSets

- [About ReplicaSets](#about-replicasets)
- [Reconciliation Loops](#reconciliation-loops)
- [Relating Pods and ReplicaSets](#relating-pods-and-replicasets)
    - [Adopting Existing Containers](#adopting-existing-containers)
    - [Quarantining Container](#quarantining-containers)
- [Designing with ReplicaSets](#designing-with-replicasets)
- [ReplicaSet Spec](#replicaset-spec)
    - [Pod Templates](#pod-templates)
    - [Labels](#labels)
- [Creating a ReplicaSet](#creating-a-replicaset)
- [Inspecting a ReplicaSet](#inspecting-a-replicaset)
    - [Finding a ReplicaSet from a Pod](#finding-a-replicaset-from-a-pod)
    - [Finding a Set of Pods for a ReplicaSet](#finding-a-set-of-pods-for-a-replicaset)
- [Scaling ReplicaSets](#scaling-replicasets)
    - [Imperative Scaling with kubectl scale](#imperative-scaling-with-kubectl-scale)
    - [Declaratively Scaling with kubectl apply](#declaring-scaling-with-kubectl-apply)
    - [Autoscaling a ReplicaSet](#autoscaling-a-replicaset)
- [Deleting ReplicaSets](#deleting-replicasets)
- [More Information](#more-information)

## About ReplicaSets

More often than not, we want multiple replicas of a container running for a variety of reasons:

1. Redundancy
    - Failure toleration by running multiple instances
2. Scale
    - Higher request-processing capacity by running multiple instances
3. Sharding
    - Different replicas can handle different parts of computation in parallel.

A ReplicaSet acts as a cluster-wide Pod manager, ensuring that the right types and numbers of Pods are running at all times.
 
Because ReplicaSets make it easy to create and manage replicated sets of Pods, they are the building blocks for common app deployment patterns and for self-healing apps at the infrastructure level.

Pods managed by ReplicaSets are automatically rescheduled under certain failure conditions, such as node or network failures.

When we define a ReplicaSet, we define a specification for the Pos we want to create, and a desired number of replicas.

We also need to define a way of finding Pods that the ReplicaSet should control. The act of managing the replicated Pods is an example of a **reconciliation loop**.

## Reconciliation Loops

- The central concept behind a reconciliation loop is the notion of *desired* state vs *observed or current* state.
- Desired state is what you want. With a ReplicaSet, it is the desired number of replicas and the definition of the Pod to replicate.
- In contrast, the current state is the currently observed state of the system.
- The reconciliation loop is constantly running, observing the current state of the world and taking action to ensure observed state matches desired state.
- The benefits of this are it's goal-driven, self-healing, and can be expressed with a few lines of code.

## Relating Pods and ReplicaSets

- The relationship between Pods and ReplicaSets are loosely coupled.
- Though ReplicaSets create and manage the Pods, they don't own the Pods they create.
- ReplicaSets use label queries to identify the set of Pods they should be managing.
- They then use the same Pod API to create the Pods that they are managing.
- ReplicaSets that create multiple Pods and the services that load balance to those Pods are also totally separate, decoupled API objects.

### Adopting existing containers

- There are some occasions where we create Pods imperatively (like a single Pod)
- At some point, we'll need to expand that singleton container into a replicated service.
- Because ReplicaSets are decoupled from the Pods they manage, we can create a ReplicaSet that will adopt the existing Pod and scale out additional copies of those containers.
- This allows us to move from a single imperative Pod to a replicated set of Pods managed by a ReplicaSet.

### Quarantining containers

- Misbehaving pods can still be part of the replicated set.
- In these situations, you can modify the set of labels on the sick Pod, and doing so will disassociate it from the ReplicaSet so that you can debug the Pod.
- The ReplicaSet will notice that a Pod is missing and create a new copy, but because the Pod is still running we can interactively debug it.

## Designing with ReplicaSets

- ReplicaSets are designed to represent a single microservice in your architecture.
- Every Pod the ReplicaSet controller creates is entirely homogeneous.
- Typically, these Pods are then fronted by a Kubernetes service load balancer, which spreads traffic across the Pods that make up the service.
- Generally, ReplicaSets are designed for stateless services.
- When a ReplicaSet is scaled down, an arbitrary Pod is selected for deletion.

## ReplicaSet Spec 

- Like all objects in Kubernetes, we define ReplicaSets using a spec.
- All ReplicaSets must have a unique name (defined using the `metadata.name` field), a `spec` section that describes the number of Pods that should be running cluster-wide at any given time, and a Pod template that describes the Pod to be created when the defined number of replicas is not met.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: kuard
  labels:
    app: kuard
    version: "2"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kuard
      version: "2"
  template:
    metadata:
      labels:
        app: kuard
        version: "2"
    spec:
      containers:
        - name: kuard
          image: willvelida/kuard
```

### Pod Templates

- When current state Pods is less than desired state, the ReplicaSet Controller will create new Pods using a template contained in the ReplicaSet specification.
- The Pods are created in exactly the same way as when you created a Pod from a YAML file.
- Instead of a file though, the ReplicaSet controller creates and submits a Pod manifest based on the Pod template directly to the API server.

Here's an example of a Pod template in a ReplicaSet:

```yaml
template:
    metadata:
        labels:
            app: helloworld
            version: v1
    spec:
        containers:
            - name: helloworld
              image: willvelida/kuard
              ports:
                - containerPort: 80
```

### Labels

- ReplicaSets monitor cluster state using a set of Pod labels to filter Pod listings and track Pods running within a cluster.
- When initially created, a ReplicaSet fetches a Pod listing from the Kubernetes API and filters the results by labels.
- Based on the number of Pods the query returns, the ReplicaSet deletes or created Pods to meet the desired number of replicas.
- These filtering labels are defined in the ReplicaSet `spec` section and are the key to understanding how ReplicaSets works.

## Creating a ReplicaSet

RepliaSets are created by submitting a ReplicaSet object to the Kubernetes API.

We can use the `apply` command to submit the ReplicaSet to the Kubernetes API:

```bash
$ kubectl apply -f kuard-rs.yaml
replicaset.apps/kuard created
```

Once the `kuard` ReplicaSet has been accepted, the ReplicaSet controller will detect that there are no kuard Pods running that match the desired state and create new kuard Pod based on the contents of the Pod template:

```bash
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
kuard-rvz7x                       1/1     Running   0          71s
```

## Inspecting a ReplicaSet

As with Pods and other Kubernetes API objects, if you want to see more details about a ReplicaSet, you can use the `describe` command to provide much more information about its state.

For example:

```bash
$ kubectl describe rs kuard
Name:         kuard
Namespace:    default
Selector:     app=kuard,version=2
Labels:       app=kuard
              version=2
Annotations:  <none>
Replicas:     1 current / 1 desired
Pods Status:  1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=kuard
           version=2
  Containers:
   kuard:
    Image:         willvelida/kuard
    Port:          <none>
    Host Port:     <none>
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  2m37s  replicaset-controller  Created pod: kuard-rvz7x
```

You can see the label selector for the ReplicaSet, as well as the state of all the replicas it manages.

### Finding a ReplicaSet from a Pod

Sometimes you may wonder if a Pod is managed by a ReplicaSet, and if it is, which one. To enable this kind of discovery, the ReplicaSet controller adds an `ownerReferences` section to every Pod that it creates.

If you run the following, look for the `ownerReferences` section:

```bash
$ kubectl get pods <pod-name> -o=jsonpath='{.metadata.ownerReferences[0].name}'
```

If applicable, this will list the name of the ReplicaSet that is managing this Pod.

### Finding a Set of Pods for a ReplicaSet

You can also determine the set of Pods managed by a ReplicaSet. First, get the set of labels using the `kubectl describe` command.

For example:

```bash
$ kubectl get pods -l app=kuard,version=2
NAME          READY   STATUS    RESTARTS   AGE
kuard-rvz7x   1/1     Running   0          15m
```

## Scaling ReplicaSets

You can scale ReplicaSets up or down by updating the `spec.replicas` key on the ReplicaSet object stored in Kubernetes. When you scale up a ReplicaSet, it submits new Pods to the Kubernetes API using the Pod template defined on the ReplicaSet.

### Imperative Scaling with kubectl scale

The easiest way to achieve this is using the `scale` command in `kubectl`. For example, to scale up to 4 replicas:

```bash
$ kubectl scale replicasets kuard --replicas=4
```

While imperative commands are helpful, it's important to update the declarative manifests to match the number of replicas you need.

### Declaring Scaling with kubectl apply

In a declarative world, you make changes by editing the config file in version control and then applying those changes to the cluster.

To scale the `kuard` ReplicaSet, edit the manifest to set the `replicas` count to 4:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: kuard
  labels:
    app: kuard
    version: "2"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kuard
      version: "2"
  template:
    metadata:
      labels:
        app: kuard
        version: "2"
    spec:
      containers:
        - name: kuard
          image: willvelida/kuard
```

Now that the updated `kuard` ReplicaSet is in place, the ReplicaSet controller will detect that the number of desired Pods has changed and that it needs to take action to realize that the desired state.

When we run the `kubectl get pods`, we should see the two additional pods that have been created:

```bash
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
kuard-fs8dm                       1/1     Running   0          3m17s
kuard-rvz7x                       1/1     Running   0          32m
kuard-wcql4                       1/1     Running   0          3m17s
```

### Autoscaling a ReplicaSet

While we will want to have explicit control over the number of replicas in a ReplicaSet, often we want just enough replicas.

This definition depends on the needs of the containers in the ReplicaSet (for example, we may want a NGINX web server to scale based on CPU usage).

**Horizontal Pod Autoscaling** involves creating additional replicas of a Pod, and **vertical scaling** involves increasing the resources required for a particular Pod.

Many solutions also enable *cluster* autoscaling, where the number of machines in the cluster is scaled in response to resource needs.

To scale a ReplicaSet, we can run a command like so:

```bash
$ kubectl autoscale rs kuard --min=2 --max=5 --cpu-percent=80
```

This command creates an autoscaler that scales between two and five replicas with a CPU threshold of 80%.

## Deleting ReplicaSets

When a ReplicaSet is no longer required, it can be deleted using the `kubectl delete` command. By default, this also deletes the Pods that are managed by the ReplicaSet:

```bash
$ kubectl delete rs kuard
replicaset.apps "kuard" deleted
```

Running the `kubectl get pods` commands show that all the kuard Pods created by the kuard ReplicaSet have also been deleted:

```bash
$ kubectl get pods
```

If you don't want to delete the Pods that the ReplicaSet is managing, you can set the `--cascade` flag to `false` to ensure only the ReplicaSet object is deleted and not the Pods:

```bash
$ kubectl delete rs kuard --cascade=false
```

## More Information

- [ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)