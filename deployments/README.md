# Deployments

## About Deployments

- The Deployment object exists to manage the release of new versions of your application.
- Deployments represent deployed applications in a way that transcends any particular version.
- Additionally, Deployments enable you to easily move from one version of your code to the next.
- This *rollout* process is specifiable and careful.
- It waits for a user-configurable amount of time between upgrading individual Pods, and it uses health checks to ensure that the new version of the app is operating correctly and stops the deployment if too many failures occur.
- Using Deployments, we can simply and reliably roll out new software versions without downtime or errors.
- Deployments are controlled by a Deployment controller that runs in the Kubernetes cluster itself.

## Your first Deployment

Like all objects in Kubernetes, a Deployment can be represented as a YAML object that provides details about what we want to run:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuard
  labels:
    run: kuard
spec:
  selector:
    matchLabels:
      run: kuard
  replicas: 1 
  template:
    metadata:
      labels:
        run: kuard    
    spec:
      containers:
      - name: kuard
        image: willvelida/kuard
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
```

We can then create the deployment like so:

```bash
$ kubectl create -f kuard-deployment.yaml
```

Deployments manage ReplicaSets. We can see this relationship defined by labels and a label selector. We can see the label selector by looking at the Deployment object:

```bash
$ kubectl get deployments kuard -o jsonpath --template '{.spec.selector.matchLabels}'
{"run":"kuard"}
```

From this you can see that the Deployment is managing a ReplicaSet with the label `run=kuard`. We can use this in a label selector query across ReplicaSets to find that specific ReplicaSet:

```bash
$ kubectl get replicasets --selector=run=kuard
NAME               DESIRED   CURRENT   READY   AGE
kuard-6b86bffcf9   1         1         1       4m26s
```

Now we can see the relationship between a Deployment and a ReplicaSet in action. We can resize the Deployment using the imperative `scale` command:

```bash
$ kubectl scale deployments kuard --replicas=2
deployment.apps/kuard scaled
```

Now if we list that ReplicaSet again, we should see:

```bash
kubectl get replicasets --selector=run=kuard                      
NAME               DESIRED   CURRENT   READY   AGE
kuard-6b86bffcf9   2         2         2       10m
```

Scaling the Deployment has also scaled the ReplicaSet it controls.

Let's try the opposite, scaling the ReplicaSet:

```bash
$ kubectl scale replicasets kuard-6b86bffcf9 --replicas=1
replicaset.apps/kuard-6b86bffcf9 scaled
```

Now get that ReplicaSet again:

```bash
$ kubectl get replicasets --selector=run=kuard
NAME               DESIRED   CURRENT   READY   AGE
kuard-6b86bffcf9   2         2         2       11m
```

That's odd, it still has two replicas!

- This is because K8s is an online, healing system.
- The top-level Deployment object is managing the ReplicaSet.
- When you adjust the number of Replicas to 1, it no longer matches the desired state of the Deployment, which has set replicas to 2.
- The Deployment controller notices this, and takes action to ensure observed state matches desired state.
- To change this, we'll need to delete the Deployment.

## Creating Deployments

We can download the Deployment into a YAML file like so:

```bash
$ kubectl get deployments kuard -o yaml > .\kuard-deployment.yaml
$ kubectl replace -f .\kuard-deployment.yaml --save-config
deployment.apps/kuard replaced
```

The Deployment spec has a very similar structure to the ReplicaSet spec.
There is a Pod template, which contains a number of containers that are created for each replica managed  by the Deployment.
In addition to the Pod specification, there is also a `strategy` object:

```yaml
strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
```

This `strategy` object dictates the  different ways in which a rollout of new software can proceed.

There are two strategies supported by Deployments:

1. `Recreate`
2. `RollingUpdate`

## Managing Deployments

As with all Kubernetes objects, we can get detailed information about your Deployment via the `kubectl describe` command. This command provides an overview of the Deployment configuration:

```bash
$ kubectl describe deployments kuard
Name:                   kuard
Namespace:              default
CreationTimestamp:      Tue, 21 Jan 2025 15:51:17 +1100
Labels:                 run=kuard
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               run=kuard
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  run=kuard
  Containers:
   kuard:
    Image:      willvelida/kuard
    Port:       <none>
    Host Port:  <none>
    Limits:
      cpu:         500m
      memory:      128Mi
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   kuard-6b86bffcf9 (2/2 replicas created)
Events:
  Type    Reason             Age                From                   Message
  ----    ------             ----               ----                   -------
  Normal  ScalingReplicaSet  33m                deployment-controller  Scaled up replica set kuard-6b86bffcf9 to 1
  Normal  ScalingReplicaSet  22m (x2 over 27m)  deployment-controller  Scaled up replica set kuard-6b86bffcf9 to 2 from 1
```

There's a lot of information here! Two of the most important peices of information in the output are `OldReplicaSets` and `NewReplicaSet`.

These fields point to the ReplicaSet objects this Deployment is currently managing.

There is also a `kubectl rollout` command for Deployments, which we can use to obtain the history of rollouts associated with a particular Deployment `kubectl rollout history`.

## Updating Deployments

Deployments are declarative objects that describe a deployed application. The two most common operations on a Deployment are scaling and application updates.

### Scaling a Deployment

To scale up a Deployment, you would edit your YAML file to increase the number of replicas:

```yaml
spec:
    replicas: 3
```

Once we've committed this change, we can update the Deployment object using the `kubectl apply` command:

```bash
$ kubectl apply -f .\kuard-deployment.yaml
deployment.apps/kuard configured
```

This will increase the size of the ReplicaSet it manages and eventually create a new Pod managed by the Deployment:

```bash
$ kubectl get deployments kuard
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
kuard   3/3     3            3           42m
```

### Updating a Container Image

- Another common use case for updating a Deployment is to roll out a new version of the software running in one or more containers.
- We can also edit the YAML file to do so:

```yaml
containers:
- image: willvelida/kuard-blue
  imagePullPolicy: Always
```

We can then annotate the template for the Deployment to record some information about the update:

```yaml
spec:
  ...
  template:
    metadata:
        annotations:
            kubernetes.io/change-cause: "Update to Blue Kuard"
```

> [!NOTE]
> Make sure you add this annotation to the template and not the deployment itself. Also, do not update the `change-cause` annotation when doing simple scaling operations. This will trigger a new rollout.

Again, we can use `kubectl apply` to update the Deployment:

```bash
$ kubectl apply -f kuard-deployment.yaml
```

After you update the Deployment, it will trigger a rollout, which you can then monitor via the `kubectl rollout` command:

```bash
$ kubectl rollout status deployments kuard
Waiting for deployment "kuard" rollout to finish: 1 out of 3 new replicas have been updated...
```

You can see the old and new ReplicaSets managed by the Deployment along with the images being used. Both the old and new ReplicaSets are kept around in case you want to roll back:

```bash
$ kubectl get replicasets -o wide
NAME               DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES                     SELECTOR
kuard-59787787     1         1         0       3m18s   kuard        willvelida/hello-web-app   pod-template-hash=59787787,run=kuard
kuard-6b86bffcf9   3         3         3       3m43s   kuard        willvelida/kuard           pod-template-hash=6b86bffcf9,run=kuard
```

If you're in the middle of a rollout and want to pause it, run this command:

```bash
$ kubectl rollout pause deployments kuard
```

If you want to resume again, run this:

```bash
$ kubectl rollout resume deployments kuard
```

### Rollout History

