# Service Discovery

- [The Service Object](#the-service-object)
- [Service DNS](#service-dns)
- [Readiness Checks](#readiness-checks)
- [Looking beyond the cluster](#looking-beyond-the-cluster)
- [Load Balancer Integration](#load-balancer-integration)
- [Endpoints](#endpoints)
- [Manual Service Discovery](#manual-service-discovery)
- [kube-proxy and Cluster IPs](#kube-proxy-and-cluster-ips)

## The Service Object

- Real Service Discovery in Kubernetes starts with a Service Object.
- Service objects are ways to create a named label selector.

We can use `kubectl expose` to create a service like so:

```bash
$ kubectl create deployment alpaca-prod --image=willvelida/kuard --port=8080
$ kubectl scale deployment alpaca-prod --replicas=3
$ kubectl expose deployment alpaca-prod
$ kubectl create deployment bandicoot-prod --image=willvelida/kuard --port=8080
$ kubectl scale deployment bandicoot-prod --replicas=2
$ kubectl expose deployment bandicoot-prod
$ kubectl get services -o wide
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE   SELECTOR
alpaca-prod      ClusterIP   10.0.217.1     <none>        8080/TCP   76s   app=alpaca-prod
bandicoot-prod   ClusterIP   10.0.195.103   <none>        8080/TCP   9s    app=bandicoot-prod
kubernetes       ClusterIP   10.0.0.1       <none>        443/TCP    5d    <none>
```

After running these commands, we have 3 services. Taking a look at the `SELECTOR` column, we see that the `alpaca-prod` service simply gives a name to the selector and specifies which ports to talk to for that service.

The `kubectl expose` command will pull both the label selector and the relevant ports from the deployment definition.

The service is assigned a new type of virtual IP called a `cluster IP`. This is a special IP address the system will load balance across all of the Pods that are identified by the selector.

## Service DNS

- Because the cluster IP is virtual, it's stable and appropriate to give it a DNS address.
- Within a namespace, it's easy as just using the service name to connect to one of the Pods identified by a service.
- Kubernetes provides a DNS service exposed to Pods running in the cluster. This was installed as a system component when the cluster was first created.
- The DNS service is managed by Kubernetes, and it provides DNS names for cluster IPs.

## Readiness Checks

- Service Objects track which of your Pods is ready via a readiness check.

We can modify our deployment to add the readiness check like so:

```bash
$ kubectl edit deployment/alpaca-prod
```

This will fetch the current version of `alpaca-prod` deployment and bring up an editor. Once we save and quit the editor, it'll write the object to Kubernetes. Add the following:

```yaml
      containers:
      - image: willvelida/kuard
	readinessProbe:
	  httpGet:
	    path: /ready
	    port: 8080
	periodSeconds: 2
	initialDelaySeconds: 0
	failureThreshold: 3
	successThreshold: 1
```

This sets up the Pods in this deployment so that they will be checked for readiness via an HTTP GET to `/ready` on port 8080. This is done every 2 seconds starting as soon as the Pod comes up. If the check fails 3 times, then the Pods are considered not ready.

Only ready Pods are sent traffic.

## Looking Beyond the Cluster

- At some point, we need to let traffic into the cluster.
- The most portable way to do this is to use **NodePorts**.
- In addition to a cluster IP, the system picks a port (or one we specify), and every node in the cluster then forwards traffic to that port to the service.
- With this, if you can reach any node in the cluster, you can contact a service.
- You can use NodePort without knowing where any of the Pods for that service are running.
- You can integrate this with hardware or software Load Balancers to expose the service further.

We can do this by like so:

```
$ kubectl edit service alpaca-prod
```

We then change the `spec.type` field to `NodePort`, and the system will assign it a new NodePort:

```bash
$ kubectl describe service alpaca-prod
Name:                     alpaca-prod
Namespace:                default
Labels:                   app=alpaca-prod
Annotations:              <none>
Selector:                 app=alpaca-prod
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.0.217.1
IPs:                      10.0.217.1
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  31139/TCP
Endpoints:                10.244.2.33:8080,10.244.3.25:8080,10.244.3.26:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Now we can hit any of our cluster nodes on that port to access the service. If you're on the same network, you can access it directly.

## Load Balancer Integration

- If you have a cluster that's configured to integrate with external load balancers, you can use the `LoadBalancer` type.
- This builds on `NodePort` by additionally configuring the cloud to create a new load balancer and direct it at nodes in your cluster.

We can do this by changing our service `spec.type` to `LoadBalancer`. When we run `kubectl get services`, we'll see that we now have an external IP for our service:

```bash
$ kubectl describe service alpaca-prod
Name:                     alpaca-prod
Namespace:                default
Labels:                   app=alpaca-prod
Annotations:              <none>
Selector:                 app=alpaca-prod
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.0.217.1
IPs:                      10.0.217.1
LoadBalancer Ingress:     20.227.72.70
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  31139/TCP
Endpoints:                10.244.2.33:8080,10.244.3.25:8080,10.244.3.26:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason                Age   From                Message
  ----    ------                ----  ----                -------
  Normal  Type                  71s   service-controller  NodePort -> LoadBalancer
  Normal  EnsuringLoadBalancer  71s   service-controller  Ensuring load balancer
  Normal  EnsuredLoadBalancer   60s   service-controller  Ensured load balancer
```

- You'll often want to expose your application only within your private network.
- To achieve this, use an `internal` load balancer.

## Endpoints

- Some applications want to be able to use services without using a cluster IP.
- This is done through the `Endpoints` object.
- For every `Service` object, Kubernetes creates a buddy `Endpoints` object that contains the IP addresses for that Service:

```bash
$ kubectl describe endpoints alpaca-prod
Name:         alpaca-prod
Namespace:    default
Labels:       app=alpaca-prod
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2025-01-14T23:20:19Z
Subsets:
  Addresses:          10.244.2.33,10.244.3.25,10.244.3.26
  NotReadyAddresses:  <none>
  Ports:
    Name     Port  Protocol
    ----     ----  --------
    <unset>  8080  TCP

Events:  <none>
```

- To use a service, an advanced application can talk to the Kubernetes API directly to look up endpoints and call them.
- The Kubernetes API even has the capability to "watch" objects and be notified as soon as they change.
- Clients can react immediately as soon as the IPs associated with a service change.

## Manual Service Discovery

- Kubernetes Services are built on top of label selectors over Pods.
- That means that you can use the Kubernetes API to do service discovery without using Service Objects at all.

With `kubectl` (and via the API) we can easily see what IPs are assigned to each Pod in our example deployments:

```bash
$ kubectl get pods -o wide --show-labels
NAME                              READY   STATUS    RESTARTS   AGE    IP            NODE                              NOMINATED NODE   READINESS GATES   LABELS
alpaca-prod-7b794cc88c-b5xkz      1/1     Running   0          109m   10.244.2.33   aks-default-78443803-vmss000001   <none>           <none>            app=alpaca-prod,pod-template-hash=7b794cc88c     
alpaca-prod-7b794cc88c-cpkjk      1/1     Running   0          110m   10.244.3.25   aks-default-78443803-vmss000002   <none>           <none>            app=alpaca-prod,pod-template-hash=7b794cc88c     
alpaca-prod-7b794cc88c-cqhfd      1/1     Running   0          109m   10.244.3.26   aks-default-78443803-vmss000002   <none>           <none>            app=alpaca-prod,pod-template-hash=7b794cc88c     
bandicoot-prod-75b768ff74-p26gh   1/1     Running   0          108m   10.244.2.34   aks-default-78443803-vmss000001   <none>           <none>            app=bandicoot-prod,pod-template-hash=75b768ff74  
bandicoot-prod-75b768ff74-w46xm   1/1     Running   0          109m   10.244.3.27   aks-default-78443803-vmss000002   <none>           <none>            app=bandicoot-prod,pod-template-hash=75b768ff74
```

This is great, but what if we have a lot of Pods?

We'd want to filter this based on the labels applied as part of the deployment, like so:

```bash
$ kubectl get pods -o wide --selector=app=alpaca-prod
NAME                           READY   STATUS    RESTARTS   AGE    IP            NODE                              NOMINATED NODE   READINESS GATES
alpaca-prod-7b794cc88c-b5xkz   1/1     Running   0          111m   10.244.2.33   aks-default-78443803-vmss000001   <none>           <none>
alpaca-prod-7b794cc88c-cpkjk   1/1     Running   0          111m   10.244.3.25   aks-default-78443803-vmss000002   <none>           <none>
alpaca-prod-7b794cc88c-cqhfd   1/1     Running   0          111m   10.244.3.26   aks-default-78443803-vmss000002   <none>           <none>
```

At this point, we have some basic service discovery.

You can always use labels to identify the set of Pods you are interested in, get all of the Pods for those labels, and dig out the IP address. **However**, keeping the correct set of labels to use in sync can be tricky. This is why the Service object was created.

## kube-proxy and Cluster IPs

- Cluster IPs are stable virtual UPs that load balance traffic across all of the endpoints in a service.
- This is performed by a component running on every node in the cluster called the `kube-proxy`.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nlq09sl37yjmokpkhgxp.png)

- The `kube-proxy` watches for new services in the cluster via the API server.
- It then programs a set of `iptables` rules in the kernel of that host to rewrite destinations of packets so they are directed at one of the endpoints for that service.
- If the set of endpoints change, the set of `iptables` rules is rewritten.
- The cluster IP itself is usually assigned by the API server as the service is created.
- However, when creating the service, the user can specify a specific cluster IP.
- Once set, the cluster IP cannot be modified without deleting and recreating the Service object.