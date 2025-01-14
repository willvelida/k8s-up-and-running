# Labels and Annotations

- Labels and Annotations are fundamental concepts in Kubernetes that let you work in set of things that map to how you think about your application.
- You can organize, mark, and cross-index all of your resources to represent the groups that make the most sense for your application.

**Labels** are key/value pairs that can be attached to Kubernetes objects such as Pods or ReplicaSets. They can be arbitrary and are useful for attaching identifying information to Kubernetes objects. Labels provide the foundation for grouping objects.

**Annotations** provide a storage mechanism that resembles labels. Key/Value pairs designed to hold nonidentifying information that tools and libraries can leverage. Unlike labels, annotations are not meant for querying, filtering, or otherwise differentiating Pods from each other.

## Labels

- Labels provide identifying metadata for objects.
- These are used for grouping, viewing, and operating.
- Labels have a simple key/value pair syntax.
- Labels can be broken down into two parts: an optional prefix and a name, separated by a slash.
- The prefix must be a DNS subdomain with a 253 character limit.
- The key name is required and have a maximum length of 63 characters.
- When domain names are used in labels and annotations, they are expected to be aligned to that particular entity in some way.

## Applying Labels

Let's create some deployments with some labels

```bash
$ kubectl run alpaca-prod --image=willvelida/kuard --labels="ver=1,app=alpaca,env=prod"
$ kubectl run alpaca-test --image=willvelida/kuard --labels="ver=2,app=alpaca,env=test"
$ kubectl run bandicoot-prod --image=willvelida/kuard --labels="ver=2,app=bandicoot,env=prod"
$ kubectl run bandicoot-staging --image=willvelida/kuard --labels="ver=2,app=bandicoot,env=staging"
```

At this point, we should have 4 pods:

```bash
$ kubectl get pods --show-labels
NAME                READY   STATUS    RESTARTS   AGE     LABELS
alpaca-prod         1/1     Running   0          2m30s   app=alpaca,env=prod,ver=1
alpaca-test         1/1     Running   0          97s     app=alpaca,env=test,ver=2
bandicoot-prod      1/1     Running   0          55s     app=bandicoot,env=prod,ver=2
bandicoot-staging   1/1     Running   0          29s     app=bandicoot,env=staging,ver=2
```

## Modifying Labels

You can also apply or update labels on objects after you create them:

```bash
kubectl label deployments alpaca-test "canary=true"
```

> ![NOTE]
> In the above example, the `kubectl label` command will only change the label on the deployment itself, not any of the objects the deployment creates. To change those, you'll need to change the template embedded in the deployment.

You can also use the `-L` option to `kubectl get` to show a label value as a column. You can also remove a label by applying a dash-suffix:

```bash
kubectl label deployments alpaca-test "canary-"
```

## Label Selectors

- Label selectors are used to filter Kubernetes objects based on a set of labels.
- Selectors use a simple syntax for Boolean expressions.
- They are used both by end users (via kubectl) and by different types of objects.
- Each deployment (via a ReplicaSet) creates a set of Pods using the labels specified in the template embedded in the deployment (This is configured by the `kubectl run` command)

For example, if we only wanted to list Pods that have the `ver` label set to `2`, we could use the `--selector` flag:

```bash
$ kubectl get pods --selector="ver=2"
```

## Label Selectors in API Objects

- A Kubernetes object uses a label selector to refer to a set of other Kubernetes objects.
- Instead of a simple string, K8s use a parsed structure.

There are two forms of this. Most objects support a newer version of selector operations like so:

```yaml
selector:
    matchLabels:
        app: alpaca
    matchExpressions:
        - {key: ver, operator: In, values: [1,2]}
```

This uses compact YAML syntax. This is an item in a list that is a map with three entries.

The last entry has a value that is a list with two items. All of the terms are evaluated as a logical AND.

The older form of specifying selectors only supports the `=` operator. This selects target objects where its set of key/value pairs all match the object. The selector `app=alpaca,ver=1` would be represented by this:

```yaml
selector:
    app: alpaca
    ver: 1
```

## Labels in the Kubernetes Architecture

- Labels play a critical role in linking various related Kubernetes objects
- In many cases, objects need to relate to one another, and these relationships are defined by labels and label selectors.
- Labels are a powerful glue that holds a Kubernetes application together.

## Annotations

- Annotations provide a place to store additional metadata for Kubernetes objects where the sole purpose of the metadata is assisting tools and libraries.
- It's a way for other programs driving K8s via an API to store some opaque data with an object.
- Annotations can be used for the tool itself or to pass configuration information between external systems.

While labels are used to identify and group objects, annotations are used to provide extra information about where an object came from, how to use it, or policy around that object.

When in doubt, add information to an object as an annotation and promote it to a label if you find yourself wanting to use it in a selector.

Annotations are used to:

- Keep track of a reason for the latest update to an object.
- Communicate a specialized scheduling policy to a specialized scheduler.
- Extend data about the last tool to update the resource and how it was updated.
- Attach build, release, or image information that isn't appropriate for labels.
- Enable the Deployment object to keep track of ReplicaSets that it is managing for rollouts.
- Provide extra data to enhance the visual quality or usability of a UI.
- Prototype alpha functionality in Kubernetes.

Annotations are used in various places in Kubernetes, primarily used for rolling deployments. During rolling deployments, annotations are used to track rollout status and provide the necessary information required to roll back a deployment to a previous state.

Annotations are defined in the common `metadata` section in every Kubernetes object.

```yaml
metadata:
    annotations:
        example.com/will-velida: "https://www.willvelida.com/"
```