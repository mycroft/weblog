---
title: "Kubernetes Learning Path"
date: 2022-11-19T12:23:30+01:00
draft: true
summary: |
  Kubernetes from 0 to prod: A exercices based learning path.
---
# Abstract

At work, we're using AWS EKS and I was asked to help people learning basic (& more advanced) Kubenertes usage, in workshop style: Demoes then exercices.

# Steps

## Day 0: Boostrap with Kind

Building from scratch a Kubernetes cluster is hard. There is already a project to help people building a cluster from scratch: [Kubernete the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way). There are easier ways to bootstrap Kube clusters for testing: [Kind](https://kind.sigs.k8s.io) or [Minikube](https://minikube.sigs.k8s.io/docs/start/) are great for this.

Exercice: Launch a 3 workers node cluster with Kind. Make sure to have the config file into /tmp/kube.conf. Running `kubectl get nodes` with it should show the 3 nodes up & ready.

The configuration file to use it:

```yaml
```

Creating the cluster is easy as calling `kind`, giving the config file path, configuring the destination path for the kubeconfig file and giving a cool name for the cluster:

```fish
> kind create cluster --config kind-cluster.yaml --kubeconfig /tmp/kube.conf --name testaroo
enabling experimental podman provider
Creating cluster "testaroo" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼 
 ✓ Preparing nodes 📦 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-testaroo"
You can now use your cluster with:

kubectl cluster-info --context kind-testaroo --kubeconfig /tmp/kube.conf
```

Verifying the nodes:

```sh
> kubectl cluster-info --context kind-testaroo --kubeconfig /tmp/kube.conf
Kubernetes control plane is running at https://127.0.0.1:64240
CoreDNS is running at https://127.0.0.1:64240/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

> set -x KUBECONFIG /tmp/kube.conf

> kubectl get nodes
NAME                     STATUS   ROLES           AGE   VERSION
testaroo-control-plane   Ready    control-plane   52s   v1.25.3
testaroo-worker          Ready    <none>          32s   v1.25.3
testaroo-worker2         Ready    <none>          32s   v1.25.3
testaroo-worker3         Ready    <none>          32s   v1.25.3
```

Resources:
- https://kind.sigs.k8s.io

## Day 1: Namespaces

Namespaces are a mechanism to allow some isolation between groups of resources.

Exercice: With `kubectl`, create a namespace:

```sh
> kubectl create namespace testaroo
namespace/testaroo created
```

Still with `kubectl`, edit the created namespace and add some labels (eg: `app.kubernetes.io/managed-by: nothing`). Then, using `kubectl` with the `-o jsonpath=...` argument, print the value of the added label.

```sh
> kubectl edit namespace testaroo
...

> kubectl get namespace testaroo -o jsonpath="{.metadata.labels.app\.kubernetes\.io\/managed-by}"
nothing
```

Resources:
- https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/
- https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/
- https://kubernetes.io/docs/reference/kubectl/jsonpath/


## Day 2: Launching pods & service

Pods are the simplest workload in kubernetes. A pod can contain one or multiple containers. For the sake of simplicity, it is better to keep one container per pod though. Once pods are alive, it is then possible to bind a service to any port of them, identifying them using selectors.

Exercices:
- Using `kubectl run`, create a pod using the `nginx:latest` image in the `default` namespace.
- Then, create a basic `nodeport` service which targets port 80 and binds the 31234 port on all nodes. Check it by going on any node and verify with `curl`.

```sh
> kubectl run nginx --image=nginx:latest
pod/nginx created

> kubectl create service nodeport nginx --tcp=80 --node-port=31234
service/nginx created
```

Problem: By checking the service, there is no endpoint defined! Fix it by making sure the service selector is targeting something that exists (It can be fixed by editing either the pod or the service). The reason is that when using `kubectl run` it will add a `run=nginx` label and the `service` will select pods with `app=nginx`. Adding the label to the pod will fix it.

```sh
> kubectl patch pod nginx --patch '{"metadata":{"labels": {"app": "nginx"}}}'
service/nginx patched

> k exec -ti nginx -- curl -I http://10.89.0.5:31234/
HTTP/1.1 200 OK
Server: nginx/1.23.2
```

Resources:
- https://kubernetes.io/docs/concepts/workloads/pods/
- https://kubernetes.io/docs/concepts/services-networking/service/
