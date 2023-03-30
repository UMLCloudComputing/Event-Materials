---
aliases:
- "UML Cloud Computing Club Kubernetes Presentation 2"
date: 2023/03/30
tags:
- uml
- c3
- kubernetes
---

Andrew Aiken
https://infrasec.sh

## Namespaces
```bash
# In a namespace
kubectl api-resources --namespaced=true

# Not in a namespace
kubectl api-resources --namespaced=false
```

### Set default namespace
```bash
kubectl config set-context --current --namespace=<NAME>
```

### Create
```bash
kubectl create namespace <NAME>
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <NAME>
```

## Configuration

### Configmaps

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-cm
data:
  KEY: VALUE
```

```bash
kubectl create configmap foobar --from-literal=foo=bar

# When specifying a file values should be stored in `KEY: VALUE` format
kubectl create configmap <map-name> --from-file=config_data.txt
```

### Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: generic
data:
  extra: YmFy # Base64
```

```bash
kubectl create secret generic db-user-pass --from-literal=username=devuser
```

### Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: nginx
      envFrom:
        - configMapRef:
            name: my-cm
```

### Mounting

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo-pod
spec:
  containers:
    - name: demo
      image: nginx
      volumeMounts:
        - name: foo
          mountPath: "/etc/foo"
          readOnly: true
  volumes:
    - name: foo
      secret:
        secretName: my-secret
```

## Ingress
OSI Layer 7 load balance built into k8s, still needs NodePort or LoadBalancer service
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingr ess
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
```

#### Docs
[Kubectl Reference Docs](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-ingress-em-)
[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

## Storage
### Persistent Volume
Unit of claimable storage.
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  capacity:
    storage: 1Gi
  hostPath:
    path: /Users/aaiken/temp
```

#### Docs
[Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

### PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  volumeName: my-pv
```

Allocate a block of storage.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: foo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### Mounting
Attach the PVC defined earlier onto a pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: web
      image: nginx
      volumeMounts:
      - mountPath: "/volume"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: my-pvc
```

Mount a volume without defining a claim or volume.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-recycler
spec:
  volumes:
  - name: vol
    hostPath:
      path: /Users/aaiken/temp
  - name: empty-vol
    emptyDir: {}
  containers:
  - name: pv-recycler
    image: nginx
    volumeMounts:
    - name: vol
      mountPath: /volume
    - name: empty-vol
      mountPath: /empty-vol
```

## Security Contexts
Grant other capabilities to pods and/or containers.
When added under `spec.securityContext` applies to the pod, when `spec.containers[].securityContext` applies the the container.

#### Usage
Specify what `user / groups` to run the **pod** security-context-demo-4 as
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-4
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1003
  containers:
  - name: sec-ctx-4
    image: gcr.io/google-samples/node-hello:1.0
```

Grant `NET_ADMIN` & `SYS_TIME` to the **container** sec-ctx-4
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-4
spec:
  containers:
  - name: sec-ctx-4
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
```

#### Docs
[Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

## Service Accounts
Default service account for each namespace, & then mounted to the pod as a volume.
```yaml
kubectl create serviceaccount NAME
```

Token is stored in a k8s secret

Change the service account for the pod `Pod.spec.serviceAccountName`
Configure not to mount `Pod.spec.automountServiceAccountToken: false`
    This can also me set on the service account ([doc](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#opt-out-of-api-credential-automounting))

## Roles
[Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-binding-examples)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "create","delete"]
```

```yaml
kubectl create role developer --namespace=default --verb=list,create,delete --resource=pods
```

### Role Binding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

```yaml
kubectl create rolebinding dev-user-binding --namespace=default --role=developer --user=dev-user
```

### Cluster Roles
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: node-admin
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "watch", "list", "create", "delete"]
```

#### Cluster Role Binding
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin-binding
subjects:
- kind: User
  name: aaiken
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-admin
  apiGroup: rbac.authorization.k8s.io
```

## Network Policies
Restrict access between resources

### Usage
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

### Docs
[Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

## Certificates

### Generate
```bash
# Create key
openssl genrsa -out user.key 4096

# Create csr
openssl req -new -key user.key -subj "/CN=user" -out user.csr
```

### Certificate Signing Request
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: user
spec:
  request: LS0tLS1CUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZVzVuWld4aE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTByczhJTHRHdTYxakx2dHhWTTJSVlRWMDNHWlJTWWw0dWluVWo4RElaWjBOCnR2MUZtRVFSd3VoaUZsOFEzcWl0Qm0wMUFSMkNJVXBGd2ZzSjZ4MXF3ckJzVkhZbGlBNVhwRVpZM3ExcGswSDQKM3Z3aGJlK1o2MVNrVHF5SVBYUUwrTWM5T1Nsbm0xb0R2N0NtSkZNMUlMRVI3QTVGZnZKOEdFRjJ6dHBoaUlFMwpub1dtdHNZb3JuT2wzc2lHQ2ZGZzR4Zmd4eW8ybmlneFNVekl1bXNnVm9PM2ttT0x1RVF6cXpkakJ3TFJXbWlECklmMXBMWnoyalVnald4UkhCM1gyWnVVV1d1T09PZnpXM01LaE8ybHEvZi9DdS8wYk83c0x0MCt3U2ZMSU91TFcKcW90blZtRmxMMytqTy82WDNDKzBERHk5aUtwbXJjVDBnWGZLemE1dHJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR05WdmVIOGR4ZzNvK21VeVRkbmFjVmQ1N24zSkExdnZEU1JWREkyQTZ1eXN3ZFp1L1BVCkkwZXpZWFV0RVNnSk1IRmQycVVNMjNuNVJsSXJ3R0xuUXFISUh5VStWWHhsdnZsRnpNOVpEWllSTmU3QlJvYXgKQVlEdUI5STZXT3FYbkFvczFqRmxNUG5NbFpqdU5kSGxpT1BjTU1oNndLaTZzZFhpVStHYTJ2RUVLY01jSVUyRgpvU2djUWdMYTk0aEpacGk3ZnNMdm1OQUxoT045UHdNMGM1dVJVejV4T0dGMUtCbWRSeEgvbUNOS2JKYjFRQm1HCkkwYitEUEdaTktXTU0xMzhIQXdoV0tkNjVoVHdYOWl4V3ZHMkh4TG1WQzg0L1BHT0tWQW9FNkpsYWFHdTlQVmkKdjlOSjVaZlZrcXdCd0hKbzZXdk9xVlA3SVFjZmg3d0drWm89Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  groups:
  - system:authenticated
  usages:
  - digital signature
  - key encipherrment
  - server auth
```
`spec.request` is base64 of the **user.csr** 
```bash
cat ./user.csr | base64 | tr -d '\n'
```

### Approving request
```bash
# View csr
kubectl get csr

# Approve the csr
kubectl certificate approve user
```

[Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)

## Components

![image.svg](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)

### Control plane components
#### kube-apiserver
The API server is a component of the Kubernetes control plane that exposes the Kubernetes API. The API server is the front end for the Kubernetes control plane.

#### etcd
Consistent and highly-available key value store used as Kubernetes' backing store for all cluster data.

#### kube-scheduler
Control plane component that watches for newly created Pods with no assigned node, and selects a node for them to run on.

#### kube-controller-manager
Control plane component that runs controller processes.
- Node controller: Responsible for noticing and responding when nodes go down.
- Job controller: Watches for Job objects that represent one-off tasks, then creates Pods to run those tasks to completion.
- Endpoints controller: Populates the Endpoints object (that is, joins Services & Pods).
- Service Account & Token controllers: Create default accounts and API access tokens for new namespaces.

### Node components
#### kubelet
An agent that runs on each node in the cluster. It makes sure that containers are running in a Pod.

#### kube-proxy
kube-proxy is a network proxy that runs on each node in your cluster, implementing part of the Kubernetes Service concept.


## Additional Learning
[Kubernetes Website](https://kubernetes.io/)
[Kubectl cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#interacting-with-running-pods)
[Free Kubernetes Cluster](https://github.com/learnk8s/free-kubernetes)
