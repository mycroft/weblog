---
title:  "Setup & running kubernetes"
date:   "2019-05-15T11:25:35+0200"
categories:
  - kubernetes
summary: "First steps trying to use kubernetes"
---

In this series of small articles, I'll install & configure a Kubernetes cluster with a master and a few nodes, and set up a few applications experiments.

I'll only work with Centos 7 systems.


# Installing Kubernetes

The kubernetes installation is mostly similar for both master & workers nodes, with some light differences at the end of the process.

## On both Master & Worker nodes
The following tasks must be made on all nodes:

  - Upgrade all system packages;
  - Disable swap;
  - Enable bridge firewall rules;
  - Install docker-ce and its requirements;
  - Install kubernetes' repository & kubernetes.

### Upgrade all system packages

As usual, prior to any new installation, make sure your system is updated.

```sh
$ yum clean all && yum upgrade -y
```

### Disable selinux & swap

Swap & selinux must be disabled prior installing kubernetes

```sh
$ setenforce 0
$ sed -i 's/^SELINUX=.*/SELINUX=disabled/' /tmp/config
```

A reboot is likely required to take changes in effect permanently.

```sh
$ sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab 
$ swapoff -a
```

### Enable bridge firewall rules

```sh
$ modprobe br_netfilter
$ echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```

### Install docker-ce & its requirements

It is strongly recommended to not use the docker from the Centos repository as it can be quite old & doesn't handle the required features for the kubernetes. Add its official repository & install it:

```sh
$ yum install -y yum-utils device-mapper-persistent-data lvm2
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$ yum install -y docker-ce
$ systemctl enable docker-ce
$ systemctl start docker-ce
```

### Install kubernetes' repository & kubernetes

Prepare a repo file for Kubernetes repository & install kubernetes to complete kubernetes installation. There is no need to start it, just enable it in systemd for now.

```sh
$ cat > /etc/yum.repo.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
           https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

$ yum install -y kubelet kubeadm kubectl
$ systemctl enable kubelet
```

Once you've installed kubernetes, you're mostly good to go. You now need to do only master-only initialization & slave-only join commands.

## On Master nodes only

On master nodes, you need to initialize the kubernetes cluster & install the network layer

```sh
$ kubeadm init --pod-network-cidr=10.244.0.0/16
```

Note that the **--pod-network-cidr** argument is required because the network' stack we will use, **flannel**, requires it.

Write down the join command, or store the join token. The given token has a 24h TTL, but you are still able to generate new ones using _kubeadm token_ commands:

```sh
$ kubeadm token create
e7wy7h.62br9mqdsztr0t1g
$ kubeadm token list
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION   EXTRA GROUPS
e7wy7h.62br9mqdsztr0t1g   23h       2019-05-16T14:06:24+02:00   authentication,signing   <none>        system:bootstrappers:kubeadm:default-node-token
```

Once initialized, you must prepare your user to be able to connect the cluster

```sh
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

And finally, install the network layer as kubernetes does not have one by default. I'll use coreos's flannel for now:

```sh
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## On Worker nodes only

To join the cluster, reuse the given token on each nodes:

```sh
$ kubectl join --token=<given token> --discovery-token-unsafe-skip-ca-verification --master-ip=<master ip>
```

If joining the cluster is successfull, you'll be able to see its status using the _kubectl get nodes_ command on the master:

```sh
$ kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
kub0   Ready    master   29d   v1.14.1
kub1   Ready    <none>   29d   v1.14.1
kub2   Ready    <none>   29d   v1.14.1
kub3   Ready    <none>   29d   v1.14.1
kub4   Ready    <none>   29d   v1.14.1
kub5   Ready    <none>   29d   v1.14.1
```

You are able to check too if all system pods are up & running:

```sh
$ kubectl get -A pods
NAMESPACE     NAME                           READY   STATUS    RESTARTS   AGE
kube-system   coredns-fb8b8dccf-ck9vb        1/1     Running   9          30d
kube-system   coredns-fb8b8dccf-wf6vn        1/1     Running   10         30d
kube-system   etcd-kub0                      1/1     Running   3          30d
kube-system   kube-apiserver-kub0            1/1     Running   8          30d
kube-system   kube-controller-manager-kub0   1/1     Running   11         30d
kube-system   kube-flannel-ds-amd64-58bzx    1/1     Running   3          30d
kube-system   kube-flannel-ds-amd64-6js9j    1/1     Running   4          30d
kube-system   kube-flannel-ds-amd64-7c2rv    1/1     Running   3          30d
kube-system   kube-flannel-ds-amd64-qgjtn    1/1     Running   3          30d
kube-system   kube-flannel-ds-amd64-wckr2    1/1     Running   4          30d
kube-system   kube-flannel-ds-amd64-z2vc4    1/1     Running   6          30d
kube-system   kube-proxy-7ssr5               1/1     Running   4          30d
kube-system   kube-proxy-8fc2c               1/1     Running   3          30d
kube-system   kube-proxy-9lvvb               1/1     Running   3          30d
kube-system   kube-proxy-cfw7m               1/1     Running   4          30d
kube-system   kube-proxy-jvtrr               1/1     Running   3          30d
kube-system   kube-proxy-pdm6h               1/1     Running   4          30d
kube-system   kube-scheduler-kub0            1/1     Running   12         30d
```

You'll be done if all nodes are in the **Ready** state.