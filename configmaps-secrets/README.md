# Table of Contents

- [ConfigMaps](#configmaps)
    - [Creating ConfigMaps](#creating-configmaps)
    - [Using a ConfigMap](#using-a-configmap)
- [Secrets](#secrets)
    - [Creating Secrets](#creating-secrets)
    - [Consuming Secrets](#consuming-secrets)
    - [Private Container Registries](#private-container-registries)
- [Naming Constraints](#naming-conventions)
- [Managing ConfigMaps and Secrets](#managing-configmaps-and-secrets)
    - [Listing](#listing)
    - [Creating](#creating)
    - [Updating](#updating)

# ConfigMaps

- ConfigMaps are used to provide configuration information for workloads.
- They can be fine-grained information like a string or a composite value in the form of a file.
- One way to think of a ConfigMap is as a Kubernetes object that defines a small filesystem
- They can also be a set of variables that can be used when defining the environment or command line for your containers.
- ConfigMaps are combined with the Pod right before it's run.
- This means that the container image and the Pod definition can be reused by many workloads just by changing the ConfigMap that's used.

## Creating ConfigMaps

Here's how we can create a ConfigMap imperatively:

```bash
$ kubectl create configmap my-config --from-file=my-config.txt --from-literal=extra-parameter=extra=value --from-literal=another-parameter=another-value
configmap/my-config created
```

We can see the YAML equivalent of what we've just created:

```bash
$ kubectl get configmaps my-config -o yaml
apiVersion: v1
data:
  another-parameter: another-value
  extra-parameter: extra=value
  my-config.txt: "parameter1=value1\r\nparameter2=value2"
kind: ConfigMap
metadata:
  creationTimestamp: "2025-01-24T10:13:47Z"
  name: my-config
  namespace: default
  resourceVersion: "6344320"
  uid: bf33b260-1227-415a-acd4-7932d014d590
```

## Using a ConfigMap

There are three main ways to use a ConfigMap:

1. **Filesystem** - You can mount a ConfigMap into a Pod. A file is created for each entry based on the key name. The contents of that file are set to the value
2. **Environment variable** - A ConfigMap can be used to dynamically set the value of an environment variable.
3. **Command-line argument** - Kubernetes supports dynamically creating the command line for a container based on ConfigMap values.

Here's a YAML manifest that brings all of these together:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard-config
  labels:
    name: kuard-config
spec:
  volumes:
    - name: config-volume
      configMap:
        name: my-config
  restartPolicy: Never
  containers:
  - name: kuard-config
    image: willvelida/kuard
    imagePullPolicy: Always
    command:
    - "/kuard"
    - "${EXTRA_PARAM}"
    env:
      # An example of an environment variable used inside the container
      - name: ANOTHER_PARAM
        valueFrom:
          configMapKeyRef:
            name: my-config
            key: another-parameter
      # An example of an environment variable passed to the command to start
      - name: EXTRA_PARAM
        valueFrom:
          configMapKeyRef:
            name: my-config
            key: extra-parameter
    volumeMounts:
      # Mounting the ConfigMap as a set of files
      - name: config-volume
        mountPath: /config
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 8080
```

- For the filesystem method, we create a new volume inside the Pod and give it the name `config-volume`.
- We then define this volume to be a ConfigMap volume and point at the ConfigMap to mount.
- We have to specify where this gets mounted into the `kuard` container with a `volumeMount`. In this case, we are mounting it at `/config`
- Environment variables are specified with a special `valueFrom` member.
- This references the ConfigMap and data key to use within that ConfigMap.
- Command line arguments build on environment variables.

Let's run this Pod, and port-forward to examine how the app sees the world.

```bash
$ kubectl apply -f kuard-config.yaml 
pod/kuard-config created

$ kubectl port-forward kuard-config 8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

In our browser, we should see our environment variables!

# Secrets

- Kubernetes has native support for storing and handling private data with care.
- Secrets enable container images to be created without bundling sensitive data.
- This allows containers to remain portable across environments.
- Secrets are exposed to Pods via explicit declaration in Pod manifests and the Kubernetes API.
- This way, the Kubernetes Secrets API provides an application-centric mechanism for exposing sensitive configuration information to apps that's easy to audit and leverages native OS isolation primitives.

> [!WARNING]
> Kubernetes Secrets are stored in plain text in the `etcd` storage for the cluster. Depending on your requirements, this may not be sufficient. Anyone with admin rights on the cluster will be able to read the Secrets. In recent editions of Kubernetes, support has been added for encrypting the Secrets with a user-supplied key, generally integrated with a cloud key store. Most cloud key stores have integration with Kubernetes Secret Store CSI Driver volumes, enabling us to skip Kubernetes Secrets entirely and rely on the cloud provider's key store.

## Creating Secrets

We can create Secrets using the Kubernetes API or `kubectl`. Secrets hold one or more data elements as a collection of key/value pairs.

We can create a Secret called `kuard-tls` using the `create secret` command:

```bash
$ kubectl create secret generic kuard-tls --from-file=kuard.crt --from-file=kuard.key
secret/kuard-tls created
```

The `kuard-tls` secret has been created with two data elements. To retrieve them, we can run the following:

```bash
$ kubectl describe secrets kuard-tls
Name:         kuard-tls
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
kuard.crt:  133 bytes
kuard.key:  1679 bytes
```

With this Secret in place, we can consume it from a Pod using a Secrets volume

## Consuming Secrets

Secrets can be consumed using the Kubernetes REST API by apps that can call the API directly, but our goal is to keep apps portable.

Instead of accessing Secrets through the API server, we can use a **Secrets volume**. Secret data can be exposed to Pods using the Secrets volume type. Secrets volumes are managed by the `kubelet` and are created at Pod creation time. Secrets are stored on `tmpfs` volumes (aka RAM risks), and are not written to disk on nodes.

Each data element of a Secret is stored in a separate file under the target mount point specified in the volume mount. The `kuard-tls` Secret contains two data elements:

- `kuard.crt`
- `kuard.key`

Mounting the `kuard-tls` Secrets volume to `/tls` results in the following fields: `/tls/kuard.crt` and `/tls/kuard.key`.

Here's an example of how to declare a Secrets volume, which exposes the `kuard-tls` Secret to the `kuard` container under `/tls`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard-tls
  labels:
    name: kuard-tls
spec:
  volumes:
  - name: tls-certs
    secret:
      secretName: kuard-tls
  containers:
  - name: kuard-tls
    image: willvelida/kuard
    imagePullPolicy: Always
    volumeMounts:
    - name: tls-certs
      mountPath: "/tls"
      readOnly: true
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 8443
```

## Private Container Registries

- A special use case for Secrets is to store access credentials for private container registries.
- K8s supports the use of private images, but access to those require credentials.
- Private images can be stored across one or more private registries.
- This presents a challenge for managing credentials for each private registry on every possible node in the cluster.
- **Image Pull Secrets** leverage the Secrets API to automate the distribution of private registry credentials.

Let's take a lok at the following:

```bash
$ kubectl create secret docker-registry my-image-pull-secret \
    --docker-username=<username> \
    --docker-password=<password> \
    --docker-email=<email>
```

We can then enable access to the private registry by referencing the Image pull secret in the Pod manifest file:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard-tls
  labels:
    name: kuard-tls
spec:
  imagePullSecrets:
  - name: my-image-pull-secret
  volumes:
    - name: tls-certs
      secret:
        secretName: kuard-tls
  containers:
  - name: kuard-tls
    image: willvelida/kuard
    imagePullPolicy: Always
    volumeMounts:
    - name: tls-certs
      mountPath: '/tls'
      readOnly: true
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 8443
```

If you are repeatedly pulling from the same registry, you can add the Secrets to the default service account associated with each Pod to avoid having to specify the Secrets in every Pod you create.

## Naming Conventions

- The key names for data items inside a Secret or ConfigMap are defined to a valid environment variable names.
- They may begin with a dot, then are followed by a letter or number, followed by characters including dots, dashes, or underscores.
- Dots cannot be repeated, and dots and underscores or dashes cannot be adjacent to each other.

Here are some examples of valid and invalid names for ConfigMaps and Secrets:

| Valid Key Name | Invalid Key Name |
| -------------- | ---------------- |
| .auth_token | Token..properties |
| Key.pem | auth file_json |
| config_file | _password.txt |

## Managing ConfigMaps and Secrets

ConfigMaps and Secrets are managed through the Kubernetes API.

### Listing

You can use the `kubectl get secrets` command to list all Secrets in the current namespace:

```bash
$ kubectl get secrets
NAME        TYPE     DATA   AGE
kuard-tls   Opaque   2      22h
```

We can also do this for ConfigMaps:

```bash
$ kubectl get configmaps
NAME               DATA   AGE
kube-root-ca.crt   1      17d
my-config          3      2d16h
```

We can use `kubectl describe` to get more details on a single object:

```bash
$ kubectl describe configmaps my-config
Name:         my-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
another-parameter:
----
another-value
extra-parameter:
----
extra=value
my-config.txt:
----
parameter1=value1\r
parameter2=value2

BinaryData
====

Events:  <none>
```

Finally, we can see the raw data (including values in Secrets!) by using a command similar to the following `kubectl get configmap my-config -o yaml` or `kubectl get secret kuard-tls -o yaml`.

### Creating

The easiest way to create a Secret or ConfigMap is via `kubectl create secret generic` or `kubectl create configmap`. There are a variety of ways to specify the data items that go into the Secret or ConfigMap. These can be combined in a single command:

- ```--from-file=<filename>``` = Load from the file with the Secret data key that's the same as the filename.
- ```--from-file=<key>=<filename>``` = Load from the file with the Secret data key explicitly specified.
- ```--from-file=<directory>``` = Load all the files in the specified directory where the filename is an acceptable key name.
- ```--from-literal=<key>=<value>``` = Use the specified key/value pair directly.

### Updating

You can update a ConfigMap or Secret and have it reflected in running applications. There's no need to restart if the app is configured to reread configuration values.

**Update from file**

- If you have a manifest for your ConfigMap or Secret, you can just edit it directly and replace it with a new version using `kubectl replace -f <filename>`. You can also use `kubectl apply -f <filename>` if you created the resource with `kubectl apply`.
- Updating the file can be cumbersome. There is no `kubectl` command that supports loading data from an external file. The data must be stored directly in the YAML manifest.
- The most common use case is when the ConfigMap is defined as part of a directory or list of resources and everything is created and updated together. Often these manifests will be checked into source control.

**Re-create and update**

- If you store the inputs into your ConfigMaps or Secrets as separate files on disk, you can use `kubectl` to recreate the manifest and then use it to update the object, which will look something like this:

```bash
$ kubectl create secrets generic kuard-tls --from-file=kuard.crt --from-file=kuard.key --dry-run -o yaml | kubectl replace -f -
```

- This command line first creates a new Secret with the same name as our existing Secret. If we just stopped there, the Kubernetes API server would return an error complaining that we arey trying to create a Secret that already exists.
- Instead, we tell `kubectl` not to actually send the data to the server but instead to dump the YAML that it *would have sent* to the API server with `stdout`.
- We then pipe that to the `kubectl replace` and use `-f -` to tell it read from `stdin`.
- In this way, we can update a Secret from files on disk without having to manually base64-encode data.

**Edit current version**

- The final way to update a ConfigMap is to use `kubectl edit` to bring up a version of the ConfigMap in your editor so you can tweak it:

```bash
$ kubectl edit configmap my-config
```

You should see the ConfigMap definition in your editor. Make the desired changes and then save and close the editor. The new version of the object will be pushed to the Kubernetes API server.

**Live updates**

- Once a ConfigMap or Secret is updated using the API, it'll be automatically pushed to all volumes that use that ConfigMap or Secret.
- It may take a few seconds, but the file listing and contents of the file, as seen by kuard, will be updated with these new values.
- Using this live update feature, you can update the configuration of apps without restarting them.
