# Common kubectl commands

- [Namespaces](#namespaces)
- [Contexts](#contexts)
- [Viewing Kubernetes API Objects](#viewing-kubernetes-api-objects)
- [Creating, updating, and destroying Kubernetes objects](#creating-updating-and-destroying-kubernetes-objects)
- [Labeling and Annotating Objects](#labeling-and-annotating-objects)
- [Debugging Commands](#debugging-commands)
- [Cluster Management](#cluster-management)
- [Built-in Help](#built-in-help)
- [Useful Links](#useful-links)

## Namespaces

- Kubernetes uses [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) to organize objects in cluster.
- `kubectl` CLI interacts with the `default` namespace by....default.
- If you want to use a different namespace, pass the `--namespace` flag.

For example:

```bash
kubectl --namespace=mynamespace
```

- You can also use the shorthand `-n`.
- If you want to interact with all namespaces, use the `--all-namespaces` flag.

## Contexts

- If you want to change the namespace you're working in more permanently, use a *context*.
- This gets recorded in the `$HOME/.kube/config` config file, which also stores how you find and authenticate to your cluster.

For example, to create a context with a different default namespace, use:

```bash
kubectl config set-context my-context --namespace=mynamespace
```

Then to use the context, you have to run the following:

```bash
kubectl config use-context my-context
```

- You can use contexts to manage different clusters or different users for authentication using the `--users` or `--clusters` flags.

## Viewing Kubernetes API Objects

- Every object in Kubernetes is represented by a RESTful resource.
- The `kubectl` command makes HTTP requests to these URLs to access Kubernetes objects that reside at these paths.
- The most basic command for viewing K8s objects is `kubectl get`.
- If you run `kubectl get <resource>`, you'll see all resources of that type in your current namespace.
- For a specific object name, run `kubectl get <resource> <obj-name>`.
- To get more information about an object, we can pass in the `-o wide` flag. We can also get a JSON output of our object using `-o json`, or YAML with `-o yaml`.

You can also extract specific fields from the object using JSONPath query language to select a specific field. To do this, we can run the following:

```bash
kubectl get pods my-pod -o jsonpath --template={.status.podIP}
```

We can also view multiple objects of different types using comma separated list of types, which will display all of those object types within a given namespace:

```bash
kubectl gets pods,services
```

If you are interested in more details for a particular object, we can use the `describe` command:

```bash
kubectl describe <resource-name> <obj-name>
```

If you want to see a list of supported fields for each supported type of Kubernetes object, you can use the `explain` command:

```bash
kubectl explain <type>
```

If you want to observe the state for a particular K8s object to see changes to it as it occurs, you can add the `--watch` flag to your command.

## Creating, updating, and destroying Kubernetes objects.

- Objects in the Kubernetes API are represented as JSON or YAML files
- These files are returned by the server in response to a query or POST to the server as part of the request.
- You can use YAML or JSON files to create, update, or delete objects in Kubernetes.

For example, if we have a YAML file containing a K8s objects, we can run the following:

```bash
kubectl apply -f obj.yaml
```

You don't need to specify what type of Kubernetes object is in the file either, and we can use the sample `apply` command to update changes to our object.

If you want to see what changes `apply` will make **without actually making those changes**, you can use the `--dry-run` flag to print the objects to the terminal without actually sending it to the server.

If you want to make interactive changes to your Kubernetes objects instead of doing in the file, you can use the `edit` command:

```bash
kubectl edit <resource-name> <obj-name>
```

This will download the latest object state, and launch an editor containing the definition (**VIM FTW!!!**)

The `apply` command also records the history of previous configs in an annotation within the object. We can manipulate these record using the `edit-last-applied, set-last-applied` and `view-last-applied` commands.

If you want to delete a Kubernetes resource, we can do so by running:

```bash
kubectl delete -f obj.yaml
```

or

```bash
kubectl delete <resource-name> <obj-name>
```

## Labeling and Annotating Objects

- Labels and Annotations are tags for your objects.
- You can update the labels and annotations on any Kubernetes object using the `label` and `annotate` commands.

For example, you can add a label to a Kubernetes object like so (also the same syntax for annotations):

```bash
kubectl label <object-type> <resource-name> <label-key>=<label-value>
```

By default, you can't overwrite existing labels. To do that, you need to add the `--overwrite` flag.

If you want to remove a label, you can do this by passing in the label name like so:

```bash
kubectl label <object-type> <resource-name> <label-key>-
```

## Debugging Commands

To see logs for a running container:

```bash
kubectl logs <pod-name>
```

If you have multiple containers within a Pod, you can choose the container to view by using the `-c` flag.

By default, `kubectl logs` lists the current logs and exits. If you want ot stream the logs back to the terminal without exiting, you can add the `-f` (follow) CLI flag.

You can also execute a command in a running container by running the following:

```bash
kubectl exec -it <pod-name> -- bash
```

If you don't have bash or some other terminal within your container, you can always attach it to the running process:

```bash
kubectl attach -it <pod-name>
```

You can also copy files to and from a container using the following:

```bash
kubectl cp <pod-name>:<file-path> <local-file-path>
```

The above is an example of copying a file from Kubernetes to your local, but the reverse is also possible!

- You can also access your Pod via the network using `port-forward`.
- This forwards network traffic from the local machine to your Pod, enabling you to securely tunnel network traffic through to containers that might not be exposed publicly.

For example, to open up a connection that forwards traffic from the local machine on port 8080 to the remote container on port 80:

```bash
kubectl port-forward <pod-name> 8080:80
```

If you want to view Kubernetes events, we can run the following to see a list of the latest 10 events on all objects within a given namespace:

```bash
kubectl get events
```

You can also stream events as they happen by appending `--watch`, or view events in all namespaces using the `-A` flag.

If you're interested in how your cluster is using resources, you can use the `top` command to see the list of resources in use either by Pods or nodes. For example:

```bash
kubectl top nodes
```

> [!NOTE]
> These `top` commands only work if a metrics server is running in your cluster. Metric servers are present in nearly every managed K8s environment, but if `top` fails for you cluster, you need to install a metrics server!

## Cluster Management

- You can also use `kubectl` to manage the cluster instead.
- Most commonly used to cordon and drain a particular node.
- When we cordon a node, you prevent future Pods from being scheduled onto that machine.
- When we drain a node, you remove any Pods that are running on that machine.
- Once a machine is repaired, you can use `kubectl uncordon` to re-enable Pods being able to be scheduled on that machine.
- There is no `undrain` command!.

## Built-in help!

Finally, `kubectl` comes with built-in `help` commands, which you can use like so:

```bash
kubectl help
```

or 

```bash
kubectl help <command-name>
```

## Useful links

- [Kubernetes Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Configure Access to Multiple Clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
- [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)