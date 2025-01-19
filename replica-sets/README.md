# ReplicaSets

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
