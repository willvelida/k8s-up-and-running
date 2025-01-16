# HTTP Load Balancing with Ingress

- A critical part of any app is getting network traffic to and from that application
- Service Objects operate at Layer 4, meaning that it only forwards TCP and UDP connections and doesn't look inside of those connections.
- Kubernetes calls its HTTP-based load-balancing system **Ingress**.
- The Ingress controller is a software system made up of two parts.
    - The first is the Ingress proxy, which is exposed outside the cluster using a service of type `LoadBalancer`.
    - This proxy sends request to "upstream" servers.
    - The second component is the Ingress reconciler, or operator.
    - This is responsible for reading and monitoring Ingress objects in the Kubernetes API and reconfiguring the Ingress proxy to route traffic as specified in the Ingress resource.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ygbnthsorkkm8jobg0xb.png)

- [Ingress Spec vs Ingress Controllers](#ingress-spec-vs-ingress-controllers)
- [Configuring DNS](#configuring-dns)
- [Using Ingress](#using-ingress)
    - [Simplest ways](#simplest-ways)
    - [Using hostname](#using-hostnames)
    - [Using paths](#using-paths)
- [Advanced Ingress Topics and Gotchas](#advanced-ingress-topics-and-gotchas)
    - [Running multiple ingress controllers](#running-multiple-ingress-controllers)
    - [Multiple Ingress Objects](#multiple-ingress-objects)
    - [Ingress and Namespaces](#ingress-and-namespaces)
    - [Path Rewriting](#path-rewriting)
    - [Serving TLS](#serving-tls)
- [More Information](#more-information)

## Ingress Spec vs Ingress Controllers

- Ingress is very different from other regular objects in Kubernetes.
- It is split into a common resource specification and a controller implementation, meaning that there is no "standard" Ingress controller built into Kubernetes, users must install one.
- We can create and modify Ingress objects just like every other object, but there is no code running to act on those objects by default.
- It's up to us to install and manage an outside controller.

## Configuring DNS

- To make Ingress work, we need to configure DNS entries to the external address of our load balancer.
- You can map multiple hostnames to a single external endpoint and the Ingress Controller will direct incoming request to the appriopriate upstream service.
- The ExternalDNS project is a cluster add-on that you can use to manage DNS records for you.

## Using Ingress

Once you have an Ingress controller configured, you can start to use it in the following ways:

### Simplest ways

The simplest way to use Ingress is to have it blindly pass everything that it sees through an upstream service.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
  labels:
    name: simple-ingress
spec:
  defaultBackend:
    service:
      name: alpaca
      port:
        number: 8080
```

We can create this Ingress with `kubectl`

```bash
$ kubectl apply -f simple-ingress.yaml
```

- This sets things up so that any HTTP request that hits the Ingress Controller is forwarded on to the alpaca service.

### Using Hostnames

- Things start to get interesting when we direct traffic based on properties of the request.
- The most common example of this is to have the Ingress system look at the HTTP host header (which is set to the DNS domain in the original URL) and direct traffic based on that header.

We can add another Ingress object for directing traffic to the alpaca service for any traffic directed to `alpaca.example.com`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
  labels:
    name: host-ingress
spec:
  rules:
  - host: alpaca.example.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: alpaca
            port: 
              number: 8080
```

### Using Paths

- Another thing we can do is look at the path in a HTTP request.
- We can do this easily by specifying a path in the `paths` entry.
- In the below example, we direct everything coming into the http://bandicoot.example.com to the `bandicoot` service, but we also send http://bandicoot.example.com/a to the `alpaca` service.
- This type of scenario can be used to host multiple services on different paths of a single domain.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
  labels:
    name: path-ingress
spec:
  rules:
  - host: bandicoot.example.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: bandicoot
            port: 
              number: 8080
      - pathType: Prefix
        path: "/a/"
        backend:
          service:
            name: alpaca
            port:
              number: 8080
```

- When there are multiple paths on the same host listed in the Ingress system, the longest prefix matches.
- So in this example, traffic starting with `/a/` is forwarded to the `alpaca` service, while all other traffic is directed to `bandicoot`
- As requests get proxied to the upstream service, the path remains unmodified.


## Advanced Ingress Topics and Gotchas

- Ingress supports some fancy features (based on the Ingress Controller implementation).
- Many of these features are exposes via annotations on the Ingress object, which can be hard to validate and are easy to get wrong.
- Many of these annotations might apply to the entire Ingress object and so can be more general than we need.
- To scope these down, you can split a single Ingress object into multiple Ingress objects.
- The Ingress Controller should read them and merge them together.

### Running multiple Ingress Controllers

- There are multiple Ingress controllers, and you may want to run multiple Ingress controllers on a single cluster.
- To solve this, the IngressClass resource exists so that an Ingress resource can request a particular implementation.
- When you create an Ingress resource, you use the `spec.ingressClassName` field to specify the specific Ingress resource.

> [!NOTE]
> Prior to K8s version 1.18, we used the `kubernetes.io/ingress.class` annotation. Move away from this, as it will likely be deprecated.

- If the `spec.ingressClassName` annotation is missing, a default Ingress controller is used.
- It's specified by adding the `ingressclass.kubernetes.io/is-default-class` annotation to the correct IngressClass resource.

### Multiple Ingress Objects

- If we specify multiple Ingress objects, the Ingress controllers should read them all and try to merge them into a coherent configuration.
- However, if you specify duplicate and conflicting configurations, the behavior is undefined.
- It's likely that different Ingress Controllers will behave differently.
- Even a single implementation may do different things depending on non-obvious factors.

### Ingress and Namespaces

- Due to security concerns, an Ingress object can refer to only an upstream service in the same namespace.
- This means that you can't use an Ingress object to point a subpath to a service in another namespace.
- However, multiple Ingress objects in different namespaces can specify subpaths for the same host.
- These Ingress objects are then merged to come up with the final config for the Ingress Controller.
- This cross-namespace behavior means that coordinating Ingress globally across the cluster is necessary.
- If not coordinated correctly, an Ingress object in one namespace could cause problems in another.

### Path Rewriting

- Some Ingress controller implementation support path rewriting.
- This can be used to modify the path in the HTTP request as it gets proxied.
- This is usually specified by an annotation on the Ingress object and applies to all request that are specified by that object.

For example, if we were using the NGINX Ingress controller, we could specify an annotation of `nginx.ingress.kubernetes.io/rewrite-target: /`. This can sometimes make upstream services work on a subpath even if they weren't built to do so.

There are multiple implementations that not only implement path rewriting, but also support regular expressions when specifying the path. How this is done is implementation-specific.

Path rewriting isn't a silver bullet, and can often lead to bugs.

### Serving TLS

- When serving websites, it's necessary to do so securely using TLS and HTTPS. Ingress supports this.

First, we have to specify a Secret with their TLS certificate and keys. We can create a Secret imperatively like so:

```bash
$ kubectl create secret tls <secret-name> --cert <certificate-pem-file> --key <private-key-pem-file>
```

Or via yaml like so:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret-name
type: kubernetes.io/tls
data:
  tls.cert: <base64 encoded certificate>
  tls.key: <base64 encoded private key>
```

- Once you have the certificate uploaded, we can reference it in an Ingress object.
- This specifies a list of certificates along with the hostnames that those certificates should be used for.
- Again, if multiple Ingress objects specify certificates for the same hostname, the behavior is undefined.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  labels:
    name: tls-ingress
spec:
  tls: 
    - hosts:
      - alpaca.example.com
      secretName: tls-secret-name
  rules:
  - host: alpaca.example.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: alpaca
            port: 
              number: 8080
```

## More Information

- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [Gateway](https://kubernetes.io/docs/concepts/services-networking/gateway/)