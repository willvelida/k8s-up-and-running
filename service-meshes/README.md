# Service Meshes

There are three general capabilities provided by most service meshes:

1. Network Encryption and Authorization.
2. Traffic Shaping.
3. Observability.

## Encryption and Authentication with Mutual TLS

- Encryption of network traffic between Pods is a key component to security in microservices.
- Encryption provided by Mutual TLS is a popular use case for a Service Mesh.
- Leaving the implementation of encryption to individual teams leads to problems, which can include a negative impact to reliability, or provide no real security.
- Installing a service mesh on your Kubernetes cluster automatically provides encryption to network traffic between every Pod in the cluster.
- The service mesh adds a sidecar container to every Pod, which transparently intercepts all network communication.
- In addition to securing the communication, mTLS adds identity to the encryption using client certificates so your app securely knows the identity of every network client.

## Traffic Shaping

- In the real world, there are multiple instances of microservices running within the application.
- When rolling out updates, you may have different versions of your service running simultaneously.
- "Dog fooding" (testing your own software) - enable us to do **traffic shaping**, or routing of requests to different implementations based on the characteristics of the request.
- In this case, you would create an experiment where all the traffic from your company's internal users went to service Y, while all traffic from the rest of the world still goes to service X.
- Traffic Shaping opens up other deployment scenarios like A/B Testing, Blue/Green Deployments, etc.
- Service Meshes change this by building experimentations into the mesh itself. We define the experiment, and the mesh implements it for us.

## Introspection

- Finding errors in our code is a large part of how most developers spend their days.
- Debugging is even more difficult when apps are spread across multiple microservices.
- It's hard to stitch together a single request when it spans multiple Pods.
- The information needed for debugging must be stitched back together from multiple sources, assuming that the relevant information was collected in the first place.
- Service Meshes provide introspection, because it knows where requests were routed, and it can keep track of the information required to put a complete request trace back together.
- Furthermore, the service mesh is implemented once for an entire cluster, meaning that the same request tracing works no matter which team developed the service.
- The monitoring data is entirely consistent across all of the different services joined together by a cluster-wide service mesh.

## Do you really need a service mesh?

- Service Meshes can add complexity to your design, due to it being deeply integrated into the core communication of your microservices.
- When a service mesh fails, your entire app stops working.
- Before you adopt one, you need to be sure that you can fix problems in the Service Mesh when they occur.
- You must also be ready to monitor the software releases for the service mesh to make sure you pick up the latest software and bug fixes (including the overhead of rolling the new version up).
- Cloud providers that provide managed service meshes make this easier, however this can come with extra complexity and cost.
- If you do decide to use a Service Mesh, its best to apply it at the cluster level, and all microservices adopt it at the same time.

## Observability for Service Mesh Implementations

- There's lots of different implementations fo service meshes in the CNCF ecosystem.
- Because the service mesh is transparently intercepting network traffic from your application Pod, a part of the service mesh needs to be present within every one of your Pods.
- Forcing developers to add something to each container image would introduce significant friction as well as make it more difficult to manage the service mesh version.
- As a result, most service meshes add a sidecar container to every Pod in the mesh, because the sidecar sits in the same network stack as the application Pod, it can use tools like `iptables` or `eBPF` to introspect and intercept network traffic coming from your app container and process it into the service mesh.
- Most service mesh implementation depend on a mutating admission controller to automatically add the service mes sidecar to all Pods that are created in a particular cluster.
- The service mesh admission controller modifies the Pod definition by adding the sidecar.
- Because this admission controller is installed by the cluster administrator, it implements a service mesh for the entire cluster.
- Service mesh implementations take advantages of `custom resource definitions (CRD)` to add specialized resources to your cluster that are not part of the default installation.