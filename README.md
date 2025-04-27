# explore-kubernetes

## Introduction
- Kubernetes orchestrates and manages collections of containers (often using container runtimes like containerd).
- It takes care of scaling, distribution, and connectivity among these containers. Think of it as a _system to manage many containers and the infrastructure they run on_.

---

## Pods
- Pod is *smallest and simplest* unit in Kubernetes object model. It represents one (sometimes more) running containers in a cluster.
- Pods are just wrappers around containers.
	- You can think of it as a Docker container with a little extra Kubernetes magic.
	- The container is the actual application, and the Pod is the Kubernetes abstraction that manages the container and the resources it needs to run.
- Pods are *ephemeral*, its designed to be spun up, torn down, and restarted at a moment's notice.
	- If a Pod encounters a problem, it can be easily terminated and replaced with a new, healthy instance.
	- It also promotes immutability. Instead of manually patching or updating _existing_ environments, you spin up new versions of the entire environment.
	- As a developer, you'll need to understand it's a bad idea to store persistent data on Pod.
- Every Pod in has a _unique internal-to-k8s IP address_.
	- By giving each Pod a unique IP, Kubernetes simplifies communication and service discovery within the cluster.
	- Pods within the same Node or across different Nodes can easily communicate.
	- All the resources inside a k8s cluster are _virtualized_. So, the IP address of a Pod is not the same as the IP address of the Node it's running on. It's a virtual IP address that is only accessible from within the cluster.

### Trashing Pods
- Pods that keep crashing and restarting. Usually caused by:
	- There's a bug introduced in the latest image
	- Application is misconfigured and can't start properly
	- Dependency of the application is misconfigured
	- Application is trying to use too much memory and is being killed by Kubernetes
- **CrashLoopBackoff**
	- This is a type of pod status, this means the container is crashing (exiting with error code 1)
	- Because Kubernetes will automatically restart the container, but each time it tries to restart, it crashes again, it will wait longer and longer in between restart (that's why it's called backoff)

---

## Deployments
- A _[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)_ provides declarative updates for Pods and ReplicaSets.
	- You describe your desired state in Deployment, the Deployment Controller will make the current state match the desired state.
- There are 5 main top level fields
	- `apiVersion`
	- `kind: Deployment`: specifies the type of object you're configuring
	- `metadata`: metadata about the deployment, when it was created, name and its ID
	- `spec`: most important, specifies the desired state of the deployment, like num of replicas
	- `status`: current state of the deployment, won't edit this directly, kubernetes inserts this and let you see the current state.

Sample deployment file
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synergychat-api
  labels:
    app: synergychat-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: synergychat-api
  template:
    metadata:
      labels:
        app: synergychat-api
    spec:
      containers:
        - name: synergychat-api
          image: bootdotdev/synergychat-api:latest
          env:
            - name: API_PORT
              valueFrom:
                configMapKeyRef:
                  name: synergychat-api-configmap
                  key: API_PORT
```

---

## Replica Sets
- A [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) maintains a stable set of replica Pods running at any given time. It's the thing that makes sure that the number of Pods you want running is the same as the number of Pods that are actually running.
- Deployment is a higher-level abstraction that manages ReplicaSets for you.
- You will probably **never use ReplicaSets directly**.

---

## ConfigMaps
- ConfigMaps are a great way to manage innocent environment variables in Kubernetes. Things like:
	- Ports
	- URLs of other services
	- Feature flags
	- Settings that can change between envs, like `DEBUG` mode
- However, they are **not cryptographically secure**. i.e. they are not encrypted and can be accessed by anyone with access to the cluster
- For sensitive information, use [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) or a third-party solution.

Sample config file map
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: synergychat-crawler-configmap
data:
  CRAWLER_PORT: "8080"
  CRAWLER_KEYWORDS: love,hate,joy,sadness,anger,disgust,fear,surprise
```

There are two ways to reference the values from a configmap
```yaml
# first method: envFrom
spec:
  ...
  template:
    ...
    spec:
      containers:
        - name: synergychat-crawler
          image: bootdotdev/synergychat-crawler:latest
          envFrom: # use envFrom
            - configMapRef:
                name: synergychat-crawler-configmap

# second method: configMapKeyRef (individual config)
spec:
  ...
  template:
    ...
    spec:
      containers:
        - name: synergychat-api
          image: bootdotdev/synergychat-api:latest
          env:
            - name: API_PORT
              valueFrom:
                configMapKeyRef:
                  name: synergychat-api-configmap
                  key: API_PORT
```

---

## Services
- Services provide a stable endpoints for pods.
- They are an _abstraction used to provide a stable endpoint_ (fixed address) and load balance traffic across group of Pods.
- **Service Types**
	- `ClusterIP`
		- This is the _default_ service type.
		- It is the IP address that the service is bound to on the internal Kubernetes network
	- `NodePort`
		- Expose the service on each node's ip at a static port
	- `LoadBalancer`
		- Creates an external load balancer in the current cloud environment if supported
		- Assigns a fixed external IP to the service
	- `ExternalName`
		- Maps the service to the contents of externalName field (usually hostname)
		- The mapping configures the cluster's DNS server to return a CNAME record with that external hostname value. No proxy is setup.

Sample service file
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: synergychat-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

---

## Ingress
- An [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) resource exposes services to the outside world and is used often in production environments.
- A request goes through a journey as such
```
client -> ingress managed load balancer -> ingress -> service -> pod
```

Sample ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: synchat.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
    - host: synchatapi.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80

```

- Note: `nginx.ingress.kubernetes.io/rewrite-target` is an annotation, which is some extra information that kubernetes does not know and will not handle.
- Annotation can be anything.
- Because the ingress is not a core kubernetes resources (think of it as an extension), the configuration of the ingress is stored in annotations.

---

## Storage

### Ephemeral Volumes
- Ephemeral volumes are usually used to share data between containers in a pod
- Sample ephemeral volume config
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synergychat-crawler
  labels:
    app: synergychat-crawler
  namespace: crawler
spec:
	...
    spec:
      volumes:
        - name: cache-volume
          emptyDir: {}
      containers:
        - name: synergychat-crawler-1
          image: bootdotdev/synergychat-crawler:latest
          env:
            ...
          volumeMounts:
            - name: cache-volume
              mountPath: /cache
		- name: synergychat-crawler-2
          image: bootdotdev/synergychat-crawler:latest
          env:
            ...
          volumeMounts:
            - name: cache-volume
              mountPath: /cache
```

### Persistent Volumes (PV)
- Instead of simply adding a volume to a deployment, a persistent volume is a cluster-level resource that is created separately from the pod and then attached to the pod. It's similar to a ConfigMap in that way.
- PVs can be created statically or dynamically.
	- **Static PVs**: created by cluster admin
	- **Dynamic PVs** :created automatically when a pod request a volume that doesn't exist yet.
- **Persistent Volume Claims (PVC)**
	- A [persistent volume claim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) is a _request_ for a persistent volume. When using dynamic provisioning, a PVC will automatically create a PV if one doesn't exist that matches the claim.
- Sample PVC file
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: synergychat-api-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
- To attach the volume for your pods
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synergychat-api
  labels:
    app: synergychat-api
spec:
  ...
  template:
    ...
    spec:
      volumes:
        - name: synergychat-api-volume
          persistentVolumeClaim:
            claimName: synergychat-api-pvc
      containers:
        - name: synergychat-api
          image: bootdotdev/synergychat-api:latest
          volumeMounts:
            - name: synergychat-api-volume
              mountPath: /persist
          ...
```

---

## Namespaces
- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) are a way to isolate cluster resources into groups. They're a bit like directories on your computer, but instead of containing files, they contain Kubernetes objects.
- Every resources has a name, in a namespace, the name has to be unique.
- You can create a namespace by simple doing `kubectl create namespace <nsname>`
- To add a resources to a specific namespace, simply add the namespace field in metadata.
```yaml
metadata:
  name: crawler-service
  namespace: crawler
```
- Note: when add the namespace tag to existing resources (in default ns or other ns), it will re-create the resources, you'll have to delete the resources in the original namespace manually.

### Intra-cluster DNS
- Kubernetes automatically create dns entries for each `service`
```sh
<service-name>.<namespace>.svc.cluster.local

# most of the time you can omit .svc.cluster.local and do
<service-name>.<namespace>
```

---

## Scaling
- CPU is [measured in cores](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-cpu), so we can use the suffix `m` to specify milli-cores. For example, `500m` is 500 milli-cores, or 0.5 cores.
- Memory is measured in bytes, so we can use the suffixes `Ki`, `Mi`, and `Gi` to specify kilobytes, megabytes, and gigabytes, respectively. For example, `512Mi` is 512 mebibytes.

### Resource Limits
- Limit is the max amount of resources of the pod is allowed to use.
- It's actively enforced, if the usage > limit, it can be throttled (CPU), or killed (Out of memory)
- Example of setting resource limits
```yaml
spec:
  containers:
    - name: <container-name>
      image: <image-name>
      resources:
        limits:
          memory: <max-memory>
          cpu: <max-cpu>
```

### Resource Request
- Request is the amount of resources the pod is guaranteed to get.
- The scheduler uses it to decide which node to run the pod
- It's not enforced, just reserved. If you request for a pod and the existing node does not have enough resources, it will be put under _pending_.
- Example of setting resource requests
```yaml
spec:
  containers:
    - name: <container-name>
      image: <image-name>
      resources:
		requests:
		  cpu: <request-cpu>
		  memory: <request-memory>
```

### Rule of thumb for Resource Limits and Requests
- Set memory _requests_ ~10% higher than the average memory usage of your pods
- Set CPU _requests_ to 50% of the average CPU usage of your pods
- Set memory _limits_ ~100% higher than the average memory usage of your pods
- Set CPU _limits_ ~100% higher than the average CPU usage of your pods

### Horizontal Pod Autoscaling (HPA)
- A [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) can automatically scale the number of Pods in a Deployment based on observed CPU utilization or other custom metrics.
- It's very common in a Kubernetes environment to have a low number of pods in a deployment, and then scale up the number of pods automatically as CPU usage increases.
- Sample HPA config
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: synergychat-web
  minReplicas: 1
  maxReplicas: 4
  targetCPUUtilizationPercentage: 50
```
- Note: you'll have to remove the "replicas" field in your deployment.
