# Table of Contents

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