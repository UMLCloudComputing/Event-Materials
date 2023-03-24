---
aliases:
- "UML Cloud Computing Club Kubernetes Presentation"
---
2023/03/23
Tags: #UML #Kubernetes 

Andrew Aiken
https://infrasec.sh

# What are Containers?
![Deployment evolution](https://d33wubrfki0l68.cloudfront.net/26a177ede4d7b032362289c6fccd448fc4a91174/eb693/images/docs/container_evolution.svg)

## Containerization Tools
- [Containerd](https://containerd.io/)
- [[Docker]]

# Whats Kubernetes
## High Level What
https://kubernetes.io
https://kubernetes.io/docs/home/

An open-source system for automating deployment, scaling, and management of containerized applications

## Why
Each container that you run is repeatable; the standardization from having dependencies included means that you get the same behavior wherever you run it.

Containers decouple applications from underlying host infrastructure. This makes deployment easier in different cloud or OS environments.

# Local Setup
https://kubernetes.io/docs/tasks/tools/

## Docker Desktop
`Docker Desktop -> Settings -> Kubernetes -> Enable Kubernetes`
![[Pasted image 20230323091120.png]]

## K3d
https://k3d.io/
[[K3D]]

```bash
k3d cluster create C3-demo

k3d node create C3-demo-node-1 --cluster C3-demo

k3d cluster delete C3-demo
```

Exposing ports
https://k3d.io/v5.4.6/usage/exposing_services/

## MiniKube
https://minikube.sigs.k8s.io/


# Kubernetes Basic Resources
## Pods
```bash
kubectl run nginx-pod --image=nginx

kubectl delete pod nginx-pod
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

Deploying manifest files
```bash
kubectl apply -f file.yaml

kubectl delete -f file.yaml
```


## Services
Used for communication between pods & externally.

Types of services:
- ClusterIP
- NodePort
- LoadBalancer

#### ClusterIP
Internal networking
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - protocol: TCP
      port: 9898
      targetPort: 9898
```

#### NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 9898
      targetPort: 9898
      nodePort: 30000
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web
spec:
  containers:
  - name: web
    image: stefanprodan/podinfo
    ports:
    - containerPort: 9898
```


#### LoadBalancer
Exposes the Service externally using a cloud provider's load balancer. NodePort and ClusterIP Services, to which the external load balancer routes, are automatically created.
[Documentation](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)


## Deployments
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: stefanprodan/podinfo
        ports:
        - containerPort: 9898
        env:
        - name: FOO
          value: BAR
```

```yaml
kubectl create deployment nginx --replicas=3 --image nginx:latest
```

```bash
for _ in {1..10}; do; curl -s http://localhost:30000/env | jq -r '.[1]'; done
```

#### Editing
```bash
kubectl edit deployment <DEPLOYMENT_NAME>

kubectl rollout status deployment <DEPLOYMENT_NAME>

kubectl scale deployment <DEPLOYMENT_NAME> --replicas=4
```


## Notes
[Lens](https://github.com/lensapp/lens)
[OpenLens](https://github.com/MuhammedKalkan/OpenLens)
[Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
