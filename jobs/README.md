# Jobs

- A Job creates Pods that run until successful termination.
- In contrast, a regular Pod will continually restart regardless of its exit code.
- Jobs are useful for things you only want to do once, such as database migrations or batch jobs.
- If run as a regular Pod, your database migration tasks would run in a loop, continually repopulating the database after every exit.

Chapters

- [The Job Object](#the-job-object)
- [Job Patterns](#job-patterns)
    - [One Shot](#one-shot-pattern)
    - [Parallelism](#parallelism)
    - [Work Queues](#work-queues)
- [CronJobs](#cronjobs)

## The Job Object

- The Job object is responsible for creating and managing Pods defined in a template in the job specification
- These Pods generically run until successful completion.
- The job object coordinates running a number of Pods in parallel.
- If the Pod fails before a successful termination, the job controller will create a new Pod based on the Pod template in the job specification.
- Given that Pods have to be scheduled, there is a chance that your job will not execute if the scheduler does not find the required resources.
- Also, due to the nature of distributed systems, there is a small chance that duplicate Pods will be created for a specific task during certain failure scenarios.

## Job Patterns

- Jobs are designed to manage batch-like workloads where items are processed by one or more Pods.
- By default, each job runs a single Pod once until termination.
- This job pattern is defined by 2 primary attributes of the job:
    - The number of job completions.
    - The number of Pods to run in Parallel.
- In the case of the "run once until completion" pattern, the `completions` and `parallelism` parameters are set to 1.

### One Shot pattern

- One-shot jobs provide a way to run a single Pod once until successful termination.
- First, the Pod must be created and submitted to the Kubernetes API.
- This is done using a Pod template defined in the job configuration.
- Once a job is up and running, the Pod backing the job must be monitored for successful termination.
- If a job fails for any reason, the job controller is responsible for re-creating the Pod until a successful termination occurs.

There are multiple ways to create a one-shot job in Kubernetes. The easiest way to do this is to use the `kubectl` command-line tool:

```bash
$ kubectl run -i oneshot --image=willvelida/kuard --restart=OnFailure --command /kuard -- --keygen-enable --keygen-exit-on-complete --keygen-num-to-gen 10
If you don't see a command prompt, try pressing enter.
2025/01/24 00:16:44 (ID 0 1/10) Item done: SHA256:aC5CT5xlXkAVQzBgGGHBRUM0TYH7CcwJDGaz2O6807E
2025/01/24 00:16:48 (ID 0 2/10) Item done: SHA256:OOCDA+DfhO++gmMzE6kbOGxdo8II+enehahaVdNCOVk
2025/01/24 00:16:51 (ID 0 3/10) Item done: SHA256:7h63OV5P7cdzB9AdwZlXd2+EVBiheg6DZ2XOICqiVuo
2025/01/24 00:16:57 (ID 0 4/10) Item done: SHA256:0yPnam4CFxbW3QuyCW/aaxTyhFdl4QV8eupZZ3sfcNc
2025/01/24 00:17:01 (ID 0 5/10) Item done: SHA256:1MRTT6Rng/lhIYr8ceGYqltcDjcuaeSPnlDV6cEZyKo
2025/01/24 00:17:02 (ID 0 6/10) Item done: SHA256:kQMtgwKOl/RcgoWj1EtTQoVdaTZasnuhcpWyRtao/G0
2025/01/24 00:17:04 (ID 0 7/10) Item done: SHA256:cKloaUH4fUbOPjCzSNqGwcMrhKTjaKWdgLnUD9NiLcg
2025/01/24 00:17:05 (ID 0 8/10) Item done: SHA256:oQb5lka7wQShYYGx+4798w9E1+6VxNBnjSyQuWQDTh8
2025/01/24 00:17:06 (ID 0 9/10) Item done: SHA256:RNmA4lKdU+cuLq8IBufCojofUklz31yG/HCPJwEEip8
2025/01/24 00:17:09 (ID 0 10/10) Item done: SHA256:RWTmr2+/61wHBMPImdfVAgB+wjvwaOMfdeaSLX6UEq0
```

There are some thing to note here:

- The `-i` option to `kubectl` indicates that this is an interactive command. `kubectl` will wait until the job is running and then show the log output from the first Pod in the job.
- `--restart=OnFailure` is the option that tells `kubectl` to create a Job object.
-  All the options after the `--` are command-line arguments to the container image.

After the job is completed, the Job object and related Pod are retained so that we can inspect the log output.

This job we just ran won't show up in `kubectl get jobs` unless we pass the `-a` flag. Without the flag, `kubectl` hides completed jobs.

To delete the job, we can run this command:

```bash
$ kubectl delete pods oneshot
```

We can also create one-shot jobs using YAML:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: oneshot
spec:
  ttlSecondsAfterFinished: 100
  template:
    spec:
      containers:
      - name: kuard
        image: willvelida/kuard
        imagePullPolicy: Always
        command: 
          - "/kuard"
        args:
          - "--keygen-enbale"
          - "--keygen-exit-on-complete"
          - "--keygen-num-to-gen=10"
      restartPolicy: OnFailure
```

We can then create the job by applying it:

```bash
$ kubectl delete pods oneshot
pod "oneshot" deleted
```

Then `describe` te onshot job:

```bash
$ kubectl describe jobs oneshot
Name:                        oneshot
Namespace:                   default
Selector:                    batch.kubernetes.io/controller-uid=8855bed1-e4df-40ca-a3cf-5b58aa0f65c4
Labels:                      batch.kubernetes.io/controller-uid=8855bed1-e4df-40ca-a3cf-5b58aa0f65c4
                             batch.kubernetes.io/job-name=oneshot
                             controller-uid=8855bed1-e4df-40ca-a3cf-5b58aa0f65c4
                             job-name=oneshot
Annotations:                 <none>
Parallelism:                 1
Completions:                 1
Completion Mode:             NonIndexed
Suspend:                     false
Backoff Limit:               6
TTL Seconds After Finished:  100
Start Time:                  Fri, 24 Jan 2025 11:43:31 +1100
Pods Statuses:               1 Active (0 Ready) / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  batch.kubernetes.io/controller-uid=8855bed1-e4df-40ca-a3cf-5b58aa0f65c4
           batch.kubernetes.io/job-name=oneshot
           controller-uid=8855bed1-e4df-40ca-a3cf-5b58aa0f65c4
           job-name=oneshot
  Containers:
   kuard:
    Image:      willvelida/kuard
    Port:       <none>
    Host Port:  <none>
    Command:
      /kuard
    Args:
      --keygen-enbale
      --keygen-exit-on-complete
      --keygen-num-to-gen=10
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  98s   job-controller  Created pod: oneshot-sgd77
```

We can then view the results of the job by looking at the logs of the Pod that was created:

```bash
$ kubectl logs oneshot-9qdhp
2025/01/24 00:47:12 Starting kuard version: test
2025/01/24 00:47:12 **********************************************************************
2025/01/24 00:47:12 * WARNING: This server may expose sensitive
2025/01/24 00:47:12 * and secret information. Be careful.
2025/01/24 00:47:12 **********************************************************************
2025/01/24 00:47:12 Config:
{
  "address": ":8080",
  "debug": false,
  "debug-sitedata-dir": "./sitedata",
  "keygen": {
    "enable": true,
    "exit-code": 0,
    "exit-on-complete": true,
    "memq-queue": "",
    "memq-server": "",
    "num-to-gen": 10,
    "time-to-run": 0
  },
  "liveness": {
    "fail-next": 0
  },
  "readiness": {
    "fail-next": 0
  },
  "tls-address": ":8443",
  "tls-dir": "/tls"
}
2025/01/24 00:47:12 Could not find certificates to serve TLS
2025/01/24 00:47:12 Serving on HTTP on :8080
2025/01/24 00:47:12 (ID 0) Workload starting
2025/01/24 00:47:14 (ID 0 1/10) Item done: SHA256:vpFtbmJSZp82q5QpxwwM7AKRo9SMRbVww/o35FBUCTg
2025/01/24 00:47:20 (ID 0 2/10) Item done: SHA256:dUkTstc4L5rWfMKJtWhKhnZ84DQ+Uu/R/3h31YliKfk
2025/01/24 00:47:22 (ID 0 3/10) Item done: SHA256:H8Va9XxS7akEZ3cX9tspzIhzy1PSLUYns3gf2M/M/kQ
2025/01/24 00:47:37 (ID 0 4/10) Item done: SHA256:oqHGEYCLMpR+8nueMVLf0wQH5PC7AKL+Zv4MUnkmkhc
2025/01/24 00:47:43 (ID 0 5/10) Item done: SHA256:+uLDwAWcRYQvzyoPlsRKR6jGrObJDSjX6j2HKUOnTHI
2025/01/24 00:47:44 (ID 0 6/10) Item done: SHA256:8t304/TBHVZcFA09QX4bS6y9Zb7k8ntj6MgJF66y4TI
2025/01/24 00:47:50 (ID 0 7/10) Item done: SHA256:81kVHYP/FxuJ9XhkP6h449DliScMaUUbgnLMJ+WghbU
2025/01/24 00:47:53 (ID 0 8/10) Item done: SHA256:OMCbhmo5KwWg8iByWyRXSWpc5PQj6Dm39QUAKLYLV+o
2025/01/24 00:47:56 (ID 0 9/10) Item done: SHA256:Dcv0gGQ0T2GgXUZ5fa4PING3YYaM6KPQYodV/6ZxH7Q
2025/01/24 00:47:58 (ID 0 10/10) Item done: SHA256:MBvmFTY7kvMOrPJAODjWH191kWMtez01a9RGDSiLBVc
2025/01/24 00:47:58 (ID 0) Workload exiting
```

This job ran successfully, but what happens if something fails? Let's modify the job to cause a failure:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: oneshot
spec:
  ttlSecondsAfterFinished: 100
  template:
    spec:
      containers:
      - name: kuard
        image: willvelida/kuard
        imagePullPolicy: Always
        command: 
          - "/kuard"
        args:
          - "--keygen-enable"
          - "--keygen-exit-on-complete"
          - "--keygen-exit-code=1"
          - "--keygen-num-to-gen=3"
      restartPolicy: OnFailure
```

Let's look at the Pod status:

```bash
kubectl get pod -l job-name=oneshot
NAME            READY   STATUS             RESTARTS      AGE
oneshot-2ctjv   0/1     CrashLoopBackOff   1 (15s ago)   35s
```

It's not uncommon to have a bug someplace that causes the program to crash as soon as it starts.

In this case, Kubernetes will wait a bit before restarting the Pod to avoid a crash loop that would eat resources on the node.

This is all handled locally to the node by the `kubelet` without the job being involved at all.

Let's try something else:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: oneshot
spec:
  ttlSecondsAfterFinished: 100
  template:
    spec:
      containers:
      - name: kuard
        image: willvelida/kuard
        imagePullPolicy: Always
        command: 
          - "/kuard"
        args:
          - "--keygen-enable"
          - "--keygen-exit-on-complete"
          - "--keygen-num-to-gen=10"
      restartPolicy: Never
```

Here, we changed the `restartPolicy` from `OnFailure` to `Never`. This tells the `kubenet` not to restart the Pod on failure, but just declare the Pod as failed.

The Job object then notices and creates a replacement Pod. If we're not careful, this will create a lot of "junk" in our cluster.

Always use a `restartPolicy: OnFailure` so failed Pods are rerun in place.

### Parallelism

- Now for our job, generating keys can be slow.
- What we can do is start a bunch of workers together to make this key generation faster.
- We can do this by using a combination of the `completions` and `parallelism` parameters.

Let's do this in a new YAML manifest:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel
  labels:
    chapter: jobs
spec:
  parallelism: 5
  completions: 10
  template:
    metadata:
      labels:
        chapter: jobs
    spec:
      containers:
      - name: kuard
        image: willvelida/kuard
        imagePullPolicy: Always
        command: 
          - "/kuard"
        args:
          - "--keygen-enable"
          - "--keygen-exit-on-complete"
          - "--keygen-num-to-gen=10"
      restartPolicy: OnFailure
```

Start it up:

```bash
$ kubectl apply -f .\job-parallel.yaml        
job.batch/parallel created
```

Now watch as the pods come up, do their thing, and exit:

```bash
$ kubectl get pods -w
NAME                       READY   STATUS    RESTARTS   AGE
kuard-6b86bffcf9-8kdmc     1/1     Running   0          2d20h
kuard-6b86bffcf9-cx6bh     1/1     Running   0          2d20h
kuard-6b86bffcf9-jsjr2     1/1     Running   0          2d20h
kuard-6bddd7847b-6gp8g     0/1     Pending   0          2d15h
nginx-fast-storage-lcktn   1/1     Running   0          24h
parallel-8czv8             1/1     Running   0          8s
parallel-jqsfp             1/1     Running   0          8s
parallel-t5q5s             1/1     Running   0          8s
parallel-txv6g             1/1     Running   0          8s
parallel-wnnrr             1/1     Running   0          8s
parallel-wnnrr             0/1     Completed   0          47s
parallel-wnnrr             0/1     Completed   0          49s
parallel-227sw             0/1     Pending     0          0s
parallel-227sw             0/1     Pending     0          0s
parallel-227sw             0/1     ContainerCreating   0          0s
parallel-wnnrr             0/1     Completed           0          49s
parallel-227sw             1/1     Running             0          3s
parallel-t5q5s             0/1     Completed           0          57s
parallel-t5q5s             0/1     Completed           0          58s
```

### Work Queues

A common use case for jobs is to process work from a work queue.

We can create a simple ReplicaSet to manage a singleton work queue daemon. We use this ReplicaSet to ensure that a new Pod will get created in the face of machine failure:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: queue
  labels:
    app: work-queue
    component: queue
    chapter: jobs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: work-queue
      component: queue
      chapter: jobs
  template:
    metadata:
      labels:
        app: work-queue
        component: queue
        chapter: jobs
    spec:
      containers:
        - name: queue
          image: willvelida/kuard
          imagePullPolicy: Always
```

Let's run the work queue with the following command:

```bash
$ kubectl apply -f .\rs-queue.yaml
replicaset.apps/queue created
```

At this point, the work queue daemon should be up and running, let's port forward to connect to it:

```bash
$ kubectl port-forward rs/queue 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

Now we can expose it using a service. This will make it easy for producers and consumers to locate the work queue via DNS:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: queue
  labels:
    app: work-queue
    component: queue
    chapter: jobs
spec:
  selector:
    app: work-queue
    component: queue
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
```

Create the queue service with `kubectl`:

```bash
$ kubectl apply -f .\service-queue.yaml
service/queue created
```

We're now ready to put a bunch of work items in the queue. Let's use curl to communicate to the work queue through the `kubectl port-forward` we set up earlier:

```bash
# Create a work queue called 'keygen'
curl -X PUT localhost:8080/memq/server/queues/keygen

# Create 100 work items and load up the queue
for i in work-item-{0..99}; do
    curl -X POST localhost:8080/memq/server/queues/keygen/enqueue -d "$i"
done
```

Run these commands, and you should see some messages being sent to the queue

```bash
{"kind":"message","id":"faa94e9189c2faafb0c9b340e18a9177","body":"work-item-90","creationTimestamp":"2025-01-24T02:41:00.531335781Z"}
{"kind":"message","id":"a7e82bdc52cf52f05e823498ef3312ad","body":"work-item-91","creationTimestamp":"2025-01-24T02:41:00.7194962Z"}
{"kind":"message","id":"a0baca8bd0ee628af1ec47b00c7c75c0","body":"work-item-92","creationTimestamp":"2025-01-24T02:41:01.066213246Z"}
{"kind":"message","id":"148a88e0eaecc8c16bf505fa22198054","body":"work-item-93","creationTimestamp":"2025-01-24T02:41:01.422181353Z"}
{"kind":"message","id":"086e8a4fc70f8072081059252013ede0","body":"work-item-94","creationTimestamp":"2025-01-24T02:41:01.731689358Z"}
{"kind":"message","id":"4c4d2553fae93f876175aaab5dc58d38","body":"work-item-95","creationTimestamp":"2025-01-24T02:41:02.045731593Z"}
{"kind":"message","id":"29529f5107c9cdae8ae040de82929c00","body":"work-item-96","creationTimestamp":"2025-01-24T02:41:02.368264183Z"}
{"kind":"message","id":"24168b533e1b94ea564696f00907b333","body":"work-item-97","creationTimestamp":"2025-01-24T02:41:02.560345928Z"}
{"kind":"message","id":"704777d533d1f8f7bae96ae679dd55e4","body":"work-item-98","creationTimestamp":"2025-01-24T02:41:03.050379503Z"}
{"kind":"message","id":"db0ec675348ed84243dd4e72294604d2","body":"work-item-99","creationTimestamp":"2025-01-24T02:41:03.452387007Z"}
```

Now we can kick off a job to consume the work queue until it's empty.

We can create a Job to draw work items from the queue, create a key, and then exit once the queue is empty:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: consumers
  labels:
    app: work-queue
    component: queue
    chapter: jobs
spec:
  parallelism: 5
  template:
    metadata:
      labels:
        app: work-queue
        component: queue
        chapter: jobs
    spec:
      containers:
      - name: worker
        image: willvelida/kuard
        imagePullPolicy: Always
        command:
        - "/kuard"
        args:
        - "--keygen-enable"
        - "--keygen-exit-on-complete"
        - "--keygen-memq-server=http://queue:8080/memq/server"
        - "--keygen-memq-queue=keygen"
      restartPolicy: OnFailure
```

Here we're telling the job to start 5 Pods in parallel. As the `completions` parameter is unset, we put the job into worker-pool mode.

Once the first Pod exists with a zero exit code, the job will start winding down and will not start any new Pods.

This means that none of the workers should exit until the work is done and they are all in the process of finishing up.

We can create the consumers job like so:

```bash
$ kubectl apply -f job-consumers.yaml 
job.batch/consumers created
```

We can now view the Pods backing the job:

```bash
$ kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
consumers-b5sbd            1/1     Running   0          14s
consumers-jrc4g            1/1     Running   0          14s
consumers-k2n8h            1/1     Running   0          14s
consumers-msr4w            1/1     Running   0          14s
consumers-z98v5            1/1     Running   0          14s
```

We have 5 Pods running in parallel, which will continue to run until the work queue is empty.

We can then clean up all of the stuff we've created by running:

```bash
$ kubectl delete rs,svc,job -l chapter=jobs
replicaset.apps "queue" deleted
service "queue" deleted
job.batch "consumers" deleted
```

## CronJobs

Sometimes we want to schedule a job to be run on a certain interval. To achieve this, we can use **CronJobs**.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: example-cron
spec:
  # Run every fifth hour
  schedule: "0 */5 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.28
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

The `spec.schedule` field contains the interval for the CronJob in standard `cron` format.