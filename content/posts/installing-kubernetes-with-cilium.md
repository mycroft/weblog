---
title: "Installing Kubernetes With Cilium"
date: 2023-09-25T11:50:17+02:00
tags:
  - kubernetes
  - cilium
summary: |
  Installing Kubernetes 1.28 with cilium on proxmox hosted ubuntu VMs
---

Following [last week VM installation](posts/packer-and-terraform-with-proxmox) in Proxmox with Terraform, now is time to install kubernetes on the newly created VMs.

The installation is done in 4 steps:

- Install pre requirements on all hosts;
- Install main node;
- Install cilium;
- Install worker nodes.

## Pre-requirements

### Update system and install some missing packages

```sh
kube-vm-1$ apt-get update
kube-vm-1$ apt-get install -y ca-certificates curl gnupg apt-transport-https
```

### Install containerd

```sh
kube-vm-1$ nstall -m 0755 -d /etc/apt/keyrings
kube-vm-1$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
kube-vm-1$ chmod a+r /etc/apt/keyrings/docker.gpg

kube-vm-1$ echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
   tee /etc/apt/sources.list.d/docker.list > /dev/null

kube-vm-1$ apt-get update
kube-vm-1$ apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

### Prepare containerd: Install CNI plugins & patch configuration

Default configuration does not suit `Kubernetes`. So configuration is generated & patched from scratch

```sh
kube-vm-1$ mv /etc/containerd/config.toml /etc/containerd/config.toml.backup
kube-vm-1$ containerd config default > /etc/containerd/config.toml
kube-vm-1$ sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

kube-vm-1$ mkdir -p /opt/cni/bin/
kube-vm-1$ wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
kube-vm-1$ tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.3.0.tgz

kube-vm-1$ systemctl enable containerd
kube-vm-1$ systemctl restart containerd
```


### System: Load kernel modules, patch sysctl and disable swap

```sh
kube-vm-1$ cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

kube-vm-1$ modprobe overlay
kube-vm-1$ modprobe br_netfilter

kube-vm-1$ cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

kube-vm-1$ sysctl --system

kube-vm-1$ sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
kube-vm-1$ swapoff -a
```


### Install Kubernetes

```sh
kube-vm-1$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
kube-vm-1$ echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" >> ~/kubernetes.list
kube-vm-1$ mv ~/kubernetes.list /etc/apt/sources.list.d

kube-vm-1$ apt-get update

kube-vm-1$ VERSION="1.28.2-00"
kube-vm-1$ apt-get install -y kubelet=$VERSION kubeadm=$VERSION kubectl=$VERSION kubernetes-cni
kube-vm-1$ apt-mark hold kubelet kubeadm kubectl
```

### Reboot

To apply kernel updates, modules and so, reboot the system.

```sh
kube-vm-1$ reboot
```

You can verify `containerd` is well alive after the reboot.



## Master node

To run the master Kubernetes node, run the appropriate `kubeadm init` command with our basic network configuration (IP addresses ranges).
On the master node, 

```sh
kube-vm-1$ export MASTER_NODE_IP="10.2.1.21"
kube-vm-1$ export K8S_POD_NETWORK_CIDR="192.168.0.0/17"

kube-vm-1$ systemctl enable kubelet

kube-vm-1$ kubeadm init \
  --apiserver-advertise-address=$MASTER_NODE_IP \
  --pod-network-cidr=$K8S_POD_NETWORK_CIDR \
  --ignore-preflight-errors=NumCPU \
  --skip-phases=addon/kube-proxy \
  --control-plane-endpoint $MASTER_NODE_IP \
  | tee /root/kubeadm.output
```

Note: `--skip-phases=addon/kube-proxy` is removing `kube-proxy`. This component will be replaced by `Cilium`.


## Installing Cilium


```sh
kube-vm-1$ CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
kube-vm-1$ CLI_ARCH=amd64
kube-vm-1$ if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
kube-vm-1$ curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
kube-vm-1$ sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
kube-vm-1$ tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
kube-vm-1$ rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

kube-vm-1$ export KUBECONFIG=/etc/kubernetes/admin.conf
kube-vm-1$ cilium install \
  --version 1.14.2 \
  --set ipam.operator.clusterPoolIPv4PodCIDRList='{192.168.0.0/17}' \
  --set kubeProxyReplacement=true
ℹ️  Using Cilium version 1.14.2
🔮 Auto-detected cluster name: kubernetes
🔮 Auto-detected kube-proxy has not been installed
ℹ️  Cilium will fully replace all functionalities of kube-proxy

kube-vm-1$ cilium status --wait
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (using embedded mode)
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium             Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium             Running: 1
                       cilium-operator    Running: 1
Cluster Pods:          0/2 managed by Cilium
Helm chart version:    1.14.2
Image versions         cilium             quay.io/cilium/cilium:v1.14.2@sha256:6263f3a3d5d63b267b538298dbeb5ae87da3efacf09a2c620446c873ba807d35: 1
                       cilium-operator    quay.io/cilium/operator-generic:v1.14.2@sha256:52f70250dea22e506959439a7c4ea31b10fe8375db62f5c27ab746e3a2af866d: 1
```

The `ipam.operator.clusterPoolIPv4PodCIDRList` is used to not use default IPv4 Pool used (10/8) as it conflicts with my LAN. `kubeProxyReplacement=true` will delegate the kube-proxy features to cilium. It is faster than `kube-proxy`, and mandatory for other Cilium features.

At the end of this step, all the pods should be up & running and the node should be in `Ready` step:

```sh
kube-vm-1$ export KUBECONFIG=/etc/kubernetes/admin.conf
kube-vm-1$ kubectl get nodes,pods -A
NAME             STATUS   ROLES           AGE   VERSION
node/kube-vm-1   Ready    control-plane   45m   v1.28.2

NAMESPACE     NAME                                    READY   STATUS    RESTARTS   AGE
kube-system   pod/cilium-jnbp6                        1/1     Running   0          43m
kube-system   pod/cilium-operator-5566bfffd9-cztw9    1/1     Running   0          43m
kube-system   pod/coredns-5dd5756b68-jp742            1/1     Running   0          45m
kube-system   pod/coredns-5dd5756b68-wwgnj            1/1     Running   0          45m
kube-system   pod/etcd-kube-vm-1                      1/1     Running   0          45m
kube-system   pod/kube-apiserver-kube-vm-1            1/1     Running   0          45m
kube-system   pod/kube-controller-manager-kube-vm-1   1/1     Running   0          45m
kube-system   pod/kube-scheduler-kube-vm-1            1/1     Running   0          45m
```

## Adding worker nodes

The `kubeadm init` output was stored in `/root/kubeadm.output`. Make sure to follow the pre-requirement step to prepare the system then `kubeadm join` command:

```sh
kube-vm-2$ kubeadm join 10.2.1.21:6443 --token 1j70mn.52v29x4fecxf8v5u \
	--discovery-token-ca-cert-hash sha256:7fcd5e319d21d8bc02a32db0639066bcddfb479cfb5b758719000b0534d02c19
```

## Adding other control-plane node

You'll need to either copy certificates from initial master node to new control-plane node, or push the secrets in kubernetes and fetch them when joining the cluster. As I'm lazy, I'll use the 2nd way:

On the main control plane node:

```sh
kube-vm-1$ kubeadm init phase upload-certs --upload-certs
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
26a2902e671a417456861f1e17facb11de6410c7aed8a8355be582beefdea183

```

On the new node:

```sh
kube-vm-3$ kubeadm join 10.2.1.21:6443 --token 8qlo8l.yaqd9mwyj3i5jxoy \
  --discovery-token-ca-cert-hash sha256:7fcd5e319d21d8bc02a32db0639066bcddfb479cfb5b758719000b0534d02c19 \
  --control-plane --certificate-key 26a2902e671a417456861f1e17facb11de6410c7aed8a8355be582beefdea183
[... snap ...]

kube-vm-3$ export KUBECONFIG=/etc/kubernetes/admin.conf
kube-vm-3$ kubectl get nodes
NAME        STATUS   ROLES           AGE    VERSION
kube-vm-1   Ready    control-plane   20m    v1.28.2
kube-vm-2   Ready    <none>          19m    v1.28.2
kube-vm-3   Ready    control-plane   107s   v1.28.2
```

## Validating cilium

Finally, it is possible to check if `cilium` is working correctly. Just run `cilium connectivity test`:

```sh
kube-vm-1$ cilium connectivity test
🛈 Monitor aggregation detected, will skip some flow validation steps
✨ [kubernetes] Creating namespace cilium-test for connectivity check...
✨ [kubernetes] Deploying echo-same-node service...
✨ [kubernetes] Deploying DNS test server configmap...
✨ [kubernetes] Deploying same-node deployment...
✨ [kubernetes] Deploying client deployment...
[... snap ...]

⌛ [kubernetes] Waiting for NodePort 10.2.1.22:32668 (cilium-test/echo-same-node) to become ready...
⌛ [kubernetes] Waiting for NodePort 10.2.1.24:30885 (cilium-test/echo-other-node) to become ready...
⌛ [kubernetes] Waiting for NodePort 10.2.1.24:32668 (cilium-test/echo-same-node) to become ready...
🏃 Running tests...
[=] Test [no-policies]
.......
[... snap ...]

✅ All 42 tests (306 actions) successful, 13 tests skipped, 0 scenarios skipped.
```

When tests are completed, the `cilium-test` namespace can be deleted to remove all testing pods and configurations.
