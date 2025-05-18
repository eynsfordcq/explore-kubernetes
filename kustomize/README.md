# kustomize

## commonLabels, commonAnnotations, namePrefix and nameSuffix
- By defining commonLabels, commonAnnotations, etc in kustomization.yaml, we can reduce redundancy and have a consistent definition.

Example `kustomization.yaml`
```yaml
... # other stuff

commonLabels:
  app: mywebapp

commonAnnotations:
  app: kustom-annotations

namePrefix:
  kustom-

nameSuffix:
  -v1
```

In our deployment.yaml
```yaml
...
metadata:
  name: mywebapp
  namespace: default
# NO LONGER NEEDED #
#   labels:
#     app: mywebapp
spec:
  replicas: 1
# NO LONGER NEEDED #
#   selector:
#     matchLabels:
#       app: mywebapp
  ...
```

Same goes for our service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mywebapp
  namespace: default
# NO LONGER NEEDED #
#   labels:
#     app: mywebapp
spec:
  ports:
    - port: 80
      protocol: TCP
      name: flask
# NO LONGER NEEDED #
#   selector:
#     app: mywebapp
  type: LoadBalancer
```

Notice that it will be added automatically: do `kubectl kustomize .`
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    app: kustom-annotations # <- commonAnnotations auto added
  labels:
    app: mywebapp # <- commonLabels
  name: kustom-mywebapp-v1 # <- prefix and suffix added
  namespace: default
spec:
  ports:
  - name: flask
    port: 80
    protocol: TCP
  selector:
    app: mywebapp
  type: LoadBalancer
---
# similar stuff for our services
```

## configMapGenerator

Example kustomization.yaml
```yaml
... # other stuff

configMapGenerator:
  - name: kustom-map
    env: config.properties
```

Example config.properties
```properties
# nothing really fancy just key value pairs
BG_COLOR=#12181b
FONT_COLOR=#FFFFFF
CUSTOM_HEADER=My Custom Message
```

Reference it from deployment
```yaml
          envFrom:
            - configMapRef:
                name: kustom-map
```

Do `kubectl kustomize .` and observe that a configMap is generated
```yaml
apiVersion: v1
data:
  BG_COLOR: '#12181b'
  CUSTOM_HEADER: My Custom Message
  FONT_COLOR: '#FFFFFF'
kind: ConfigMap
metadata:
  annotations:
    app: kustom-annotations
  labels:
    app: mywebapp
  name: kustom-kustom-map-v1-56dm69g45b # <- note this, it adds suffix prefix and give it a id
---
...
apiVersion: apps/v1
kind: Deployment
...
spec:
  ...
  template:
    ...
    spec:
      containers:
      - envFrom:
        - configMapRef:
            name: kustom-kustom-map-v1-56dm69g45b # <- automatically reference the full name
        ...
```

Apply the web app
```sh
kubectl apply -k .

# see pods
kubectl get pods

# see service
kubectl get services

# port forward 
kubectl port-forward services/kustom-mywebapp-v1 8080:80

# go to localhost:8080, it should work
```

## Overlays
- Useful when you want to deploy the app as different environments e.g. `Prod, Staging, Dev`

Directory structure
```plaintext
.
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── dev
    │   ├── config.properties
    │   ├── kustomization.yaml
    │   └── replicas.yaml
    └── prod
        ├── config.properties
        ├── kustomization.yaml
        └── replicas.yaml
```

In the individual kustomization.yaml (overlays/dev/kustomization.yaml and overlays/prod/kustomization.yaml)
```yaml
# specify the base
resources:
  - ../../base

# overwrite some attributes
namespace: dev

# apply patches (in here we change the replicas to two)
patches:
  - path: replicas.yaml

# in the base we won't have config.properties anymore, remove it in the base/kustomization.yaml
# add it here
configMapGenerator:
  - name: kustom-map
    env: config.properties
```

Observe generated file do `kubectl kustomize overlays/dev`
```yaml
apiVersion: v1
data:
  BG_COLOR: '#12181b'
  CUSTOM_HEADER: My Dev App # <- from overlays/dev/config.properties
  FONT_COLOR: '#FFFFFF'
kind: ConfigMap
metadata:
  name: kustom-map-b77bbtd9bm
  namespace: dev # <- namespace overriden
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    app: kustom-annotations
  labels:
    app: mywebapp
  name: kustom-mywebapp-v1
  namespace: dev
spec:
  replicas: 2 # <- patch from replicas.yaml
  ...
```