# Deployments

- [About Deployments](#about-deployments)
- [Your first Deployment](#your-first-deployment)
- [Creating Deployments](#creating-deployments)
- [Managing Deployments](#managing-deployments)
- [Updating Deployments](#updating-deployments)
    - [Scaling a Deployment](#scaling-a-deployment)
    - [Updating a container image](#updating-a-container-image)
    - [Rollout History](#rollout-history)
- [Deployment Strategies](#deployment-strategies)
    - [Recreate Strategy](#recreate-strategy)
    - [RollingUpdate Strategy](#rollingupdate-strategy)
    - [Slowing Rollouts to Ensure Service Health](#slowing-rollouts-to-ensure-service-health)
- [Deleting a Deployment](#deleting-a-deployment)

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

Kubernetes Deployments maintain a history of rollouts, which can be useful both for understanding the previous state of the Deployment and for rolling back to a specific version.

We can see the Deployment history by running:

```bash
$ kubectl rollout history deployment kuard

REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         Update to hello web app
```

The revision history is given in oldest to newest older. A revision number is incremented for each rollout.

If we're interested in more details about a particular revision, we can add the `--revision` flag to view details about that specific revision:

```bash
$ kubectl rollout history deployment kuard --revision=3
deployment.apps/kuard with revision #3
Pod Template:
  Labels:       pod-template-hash=6bddd7847b
        run=kuard
  Annotations:  kubernetes.io/change-cause: Update to hello web app
  Containers:
   kuard:
    Image:      willvelida/hello-web-app
    Port:       <none>
    Host Port:  <none>
    Limits:
      cpu:      500m
      memory:   128Mi
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
  Node-Selectors:       <none>
  Tolerations:  <none>
```

Let's say there's an issue with a rollout, we can simply undo the rollout by running:

```bash
$ kubectl rollout undo deployments kuard
```

The undo command works regardless of the stage of the rollout (whether partially completed or fully completed).

**As always, we should look to apply rollouts declaratively over imperative commands. It's better to revert the YAML file for our deployment, and apply that instead**.

Additionally, we can rollout undo to a specific version like so:

```bash
$ kubectl rollout undo deployments kuard --to-revision=3
```

By default, the last 10 revisions of a Deployment are kept attached to the Deployment object itself.

If we need to keep Deployment histories around for a long time, we can set a maximum history size for the Deployment revision history. We can so this using the `revisionHistoryLimit` property in our Deployment specification:

```yaml
spec:
    revisionHistoryLimit: 14
```

## Deployment Strategies

Kubernetes deployments support two different rollout strategies, `Recreate` and `RollingUpdate`:

### Recreate Strategy

- The `Recreate` strategy is the simpler of the two.
- It simply updates the ReplicaSet it manages to use the new image and terminates all of the Pods associated with the Deployment.
- The ReplicaSet notices that it no longer has any replicas and re-creates all Pods using the new image.
- Once the Pods are re-created, they are running the new version.

While this strategy is fast and simple, it will result in downtime.

This strategy should only be used for test Deployments where a Service downtime is acceptable.

### RollingUpdate Strategy

- The `RollingUpdate` strategy is the generally preferable strategy for any user-facing service.
- While it is slower than `Recreate`, it is more sophisticated and robust.
- Using `RollingUpdate`, you can roll out a new version of your service while it is still receiving user traffic, without any downtime.


`RollingUpdate` strategy works by updating a few Pods at a time, moving incrementally until all of the Pods are running the new version of your software.

### Managing multiple versions of your service

- This means that for a while, both the new and old services will receive requests and send traffic.
- It's critical that each version of your software, and each of its clients, is capable of talking interchangeably with both a slightly older and newer version of your software.

### Configuring a rolling update

`RollingUpdate` is a fairly generic strategy

- It can be used to update a variety of applications in a variety of settings.
- You can tune `RollingUpdate` to suit our needs.
- There are two parameters that we can use to tune the rolling update behavior: `maxUnavailable` and `maxSurge`

`maxUnavailable`

- This sets the maximum number of Pods that can be unavailable during a rolling update.
- We can set it to an absolute number, or to a percentage.
- Using a percentage is a good approach for most services, since the value is correctly applied regardless of the desired number of replicas in the Deployment.

`maxSurge`

- This controls how many extra resources can be created to achieve a rollout.
- Imagine we have a service of 10 replicas, and we set the `maxUnavailable` to 0 and `maxSurge` to 20%.
- The first thing the rollout will do is scale the new ReplicaSet up by 2 replicas, for a total of 12 (120%) in the service.
- It will then scale the old ReplicaSet down to 8 replicas (for a total of 10) - so 8 old and 2 new in the service.
- This process proceeds until the rollout is complete.
- At any time, the capacity of the service is guaranteed to be at least 100% and the maximum extra resources used for the rollout are limited to an additional 20% of all resources.

## Slowing Rollouts to Ensure Service Health

Staged rollouts are meant to ensure that the rollout results in a healthy, stable service running the new software version.

To do this, the Deployment controller always waits until a Pod reports that it is ready before moving on to update the next Pod.

> [!NOTE]
> The Deployment controller examines the Pod's status as determined by its readiness checks. Readiness checks are part of the Pod's health checks. If you want to use Deployments to reliably roll out your software, you *have* to specify readiness health checks for the containers in your Pod. Without these checks, the Deployment controller is running without knowing the Pod's status.

In most real-world scenarios, we want to wait for a period of time to have confidence that the new version is operating correctly before you move on to updating the next Pod.

For Deployments, this is defined by the `minReadySeconds` parameter:

```yaml
spec:
    minReadySeconds: 60
```

This means that the Deployment must wait 60 seconds after seeing a Pod become healthy before moving on to updating the next Pod.

We may also want to set a timeout that limits how long the system will wait.

To do this, we can use the Deployment parameter `progressDeadlineSeconds`:

```yaml
spec:
    progressDeadlineSeconds: 600
```

This sets the progress deadline to 10 minutes. If any particular stage in the rollout fails to progress in 10 minutes, then the Deployment is marked as failed, and all attempts to move the Deployment forward are halted.

**It's important to note that this timeout is given in terms of Deployment progress, not the overall length of a Deployment**. In this context, progress is defined as any time the Deployment creates or deleted a Pod. When that happens, the timeout clock resets to zero:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4zf1tf0bl1hoq6m7l9go.png)

## Deleting a Deployment

If we want to delete a Deployment, we can do it imperatively:

```bash
$ kubectl delete deployments kuard
```

You can also do it using the declarative YAML file for your Deployment:

```bash
$ kubectl delete -f kuard-deployment.yaml
```

Deleting the Deployment means deleting the entire service. It also deletes the ReplicaSets it manages, and any Pods that the ReplicaSet manages.

If we just want to delete the Deployment, and not everything else, we can do so by adding the `--cascade=false` flag to delete only the Deployment object.