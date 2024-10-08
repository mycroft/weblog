---
title: "Cilium: Enabling Hubble, Ingress Controller on LAN"
date: 2023-09-26T15:31:14+02:00
summary: |
  Enable some Cilium features: Hubble, Ingress on LAN, etc.
tags:
  - kubernetes
  - cilium
  - hubble
---

## Cilium: Enabling Ingress controller support

Cilium integrates an [ingress controller](https://docs.cilium.io/en/stable/network/servicemesh/ingress/), disabled by default. On my bare-metal/VM setup, it is required to enable it, and also add some bits to handle load balancer configuration.

### Update cilium configuration to enable ingress controllers

```sh
kube-vm-1$ cilium upgrade --reuse-values --version 1.14.2 --set ingressController.enabled=true --set ingressController.loadbalancerMode=dedicated
ℹ️  Using Cilium version 1.14.1
🔮 Auto-detected cluster name: kubernetes
🔮 Auto-detected kube-proxy has not been installed
ℹ️  Cilium will fully replace all functionalities of kube-proxy

kube-vm-1$ kubectl -n kube-system rollout restart deployment/cilium-operator
kube-vm-1$ kubectl -n kube-system rollout restart ds/cilium

kube-vm-1$ cilium config view | grep ingress
enable-ingress-controller                         true
enable-ingress-secrets-sync                       true
enforce-ingress-https                             true
ingress-default-lb-mode                           dedicated
ingress-lb-annotation-prefixes                    service.beta.kubernetes.io service.kubernetes.io cloud.google.com
ingress-secrets-namespace                         cilium-secrets
ingress-shared-lb-service-name                    cilium-ingress
```

See [Kubernetes Ingress Support](https://docs.cilium.io/en/stable/network/servicemesh/ingress/) for more details.


### Test Ingress

Install demo app:

```sh
kube-vm-1$ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.11/samples/bookinfo/platform/kube/bookinfo.yaml
[... snap ...]
```

Then an ingress for the app:

```sh
kube-vm-1$ kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.14.2/examples/kubernetes/servicemesh/basic-ingress.yaml
kube-vm-1$ kubectl get ingress
NAME            CLASS    HOSTS   ADDRESS   PORTS   AGE
basic-ingress   cilium   *                 80      13s

```

It will create an ingress, but no address will be attached to it, as there is no load balancer on my setup:

```sh
kube-vm-1$ kubectl get svc
NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
cilium-ingress-basic-ingress   LoadBalancer   10.106.88.127    <pending>     80:31803/TCP,443:31392/TCP   34s
```

Note: There will be 2 LoadBalancer services: One in `kube-system` namespace, used by shared services, and one dedicated to the `ingress`, which is tied to the service namespace (here: `default`).


To "fix" this and the ingress to be reachable on the LAN, I'll:

* Create a `CiliumLoadBalancerIPPool`;
* Have the pool IPs reachable from outside the cluster


### CiliumLoadBalancerIPPool

```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "basic-pool"
spec:
  cidrs:
  - cidr: "10.42.42.0/24"
```

Note: It is not possible to have a `/32` range here as it will reserve the first and last IP of the range, making it mandatory to use a `/30` or wider range (meaning at least 2 IPs).

```sh
kube-vm-1$ kubectl apply -f basic-pool.yaml
ciliumloadbalancerippool.cilium.io/basic-pool created

kube-vm-1$ kubectl get svc
NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
cilium-ingress-basic-ingress   LoadBalancer   10.106.88.127    10.42.42.54   80:31803/TCP,443:31392/TCP   34s
```

As soon as a LB IP pool is created, an IP will be affected to the service. However, this IP is not routed on the LAN as it is not announced. Cilium 1.14 fixes this and can be enabled by:

```sh
kube-vm-1$ cilium upgrade --reuse-values --version 1.14.2 --set l2announcements.enabled=true --set externalIPs.enabled=true --set devices=eth+
kube-vm-1$ kubectl -n kube-system rollout restart deployment/cilium-operator
kube-vm-1$ kubectl -n kube-system rollout restart ds/cilium
```

Prior to complete the setup, it is required to create a policy to allow announcements for services on given devices. See [L2 Announcements / L2 Aware LB](https://docs.cilium.io/en/latest/network/l2-announcements/).

```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: basic-policy
spec:
  interfaces:
  - eth0
  externalIPs: true
  loadBalancerIPs: true
```

Apply it:

```sh
kube-vm-1$ kubectl apply -f policy.yaml
```

The service should be reachable through the LB reserved IP from the LAN:

```sh
desktop$ curl http://10.42.42.54/
<!DOCTYPE html>
<html>
  <head>
    <title>Simple Bookstore App</title>

```

### Using shared ingress

Now, I've an issue I'm unhappy with. By default, `cilium` will create/use `dedicated` loadbalancer services, and then, reserve a pool IP for each `Ingress`. I want to use a single entry point. And make sure IP is always the same.

I'll patch the `kube-system`'s `svc/cilium-ingress` to reserve a single IP from my pool:

```sh
$ kubectl patch service -n kube-system cilium-ingress -p '{"metadata": {"annotations": {"io.cilium/lb-ipam-ips": "10.42.42.42"}}}' --type merge
service/cilium-ingress patched
$ kubectl get service -n kube-system cilium-ingress
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
cilium-ingress   LoadBalancer   10.107.131.201   10.42.42.42   80:30490/TCP,443:32168/TCP   2d
```

Next, I patch the `Ingress` annotations to use the `shared` LoadBalancer:

```sh
$ kubectl patch ingress basic-ingress -p '{"metadata": {"annotations": {"ingress.cilium.io/loadbalancer-mode": "shared"}}}' --type merge
ingress.networking.k8s.io/basic-ingress patched (no change)
$ kubectl get ingress basic-ingress
NAME            CLASS    HOSTS   ADDRESS       PORTS   AGE
basic-ingress   cilium   *       10.42.42.42   80      2d
```

### Setting a CA & TLS certificates

Setting up default TLS certs:

```sh
$ apt-get install -y minica
...
$ mkdir k8s-wildcard && cd k8s-wildcard
$ minica minica '*.k8s'
$ minica '*.svc.k8s'
Creating a auto cert:

Client Cert: false
Common Name: *.svc.k8s
Org:         Example Organization
Cert Flavor: auto
Output crt:  *.svc.k8s.crt
Output key:  *.svc.k8s.key

$ kubectl create secret tls -n kube-system default-cert --cert=*.svc.k8s.crt --key=*.svc.k8s.key

$ cilium upgrade --version 1.14.2 --reuse-values --set ingressController.defaultSecretNamespace=kube-system --set ingressController.defaultSecretName=default-cert 
```

Then, add the `tls` section in the `Ingress`:

```yaml
spec:
  tls:
  - hosts:
    - chocapic.svc.k8s
```

The service should now be available from `https://chocapic.svc.k8s/`:

```sh
$ curl -k --resolve chocapic.svc.k8s:443:10.42.42.42 https://chocapic.svc.k8s/
<html>
  <head>
    <title>Simple Bookstore App</title>
<meta charset="utf-8">
<meta http-equiv="X-UA-Compatible" content="IE=edge">
<meta name="viewport" content="width=device-width, initial-scale=1">

```

## Enabling Hubble

```sh
kube-vm-1$ cilium hubble enable --ui
kube-vm-1$ cilium status
...
Hubble Relay:       OK
...
```

Then:

```sh
kube-vm-1$ kubectl port-forward -n kube-system svc/hubble-ui --address 10.2.1.21 12000:80
```

And open your browser on http://10.2.1.21:12000/ and the Hubble UI should be shown.