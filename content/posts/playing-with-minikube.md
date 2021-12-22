---
title: "Playing With Minikube"
date: 2021-12-01T20:38:20+01:00
draft: true
summary: "How I spent an evening doing basic kubernetes things for fun and really no profit"
---

Hello World. Today I'll play with minikube. I never done that before, as I'm not a daily kubernetes user at all & needed a refresh.

Goal of this mini-tutorial/notes is to:

- Install & run minikube;
- Starts a few deployment;
- Expose the service in & outside the cluster;
- Add some configuration;
- Add persistent volumes;
- Play a bit with deployments.


# Installation

Check [how to install minikube](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/). Download the `minikube` binary, the `kubectl` one, check you can use kvm2 as a driver (you don't want to start kubernetes in a docker container, right? Right?). And that's it:

```sh
❯ minikube status
🤷  Profile "minikube" not found. Run "minikube profile list" to view all profiles.
👉  To start a cluster, run: "minikube start"
```

# Launch the cluster!

... and don't forget to specify the driver you want to use:

```sh
❯ minikube start --driver kvm2
😄  minikube v1.24.0 on Fedora 35
✨  Using the kvm2 driver based on user configuration
👍  Starting control plane node minikube in cluster minikube
🔥  Creating kvm2 VM (CPUs=2, Memory=6000MB, Disk=20000MB) ...
🐳  Preparing Kubernetes v1.22.3 on Docker 20.10.8 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

~ took 40s 

❯ minikube ssh uptime
 19:48:30 up 2 min,  0 users,  load average: 1.32, 0.48, 0.18
```

... and as minikube is great, it also configured `kubectl` so the "cluster" can be used out of the box:

```sh
❯ kubectl get pods -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS        AGE
kube-system   coredns-78fcd69978-6q8c9           1/1     Running   0               3m14s
kube-system   etcd-minikube                      1/1     Running   0               3m27s
kube-system   kube-apiserver-minikube            1/1     Running   0               3m29s
kube-system   kube-controller-manager-minikube   1/1     Running   0               3m27s
kube-system   kube-proxy-lmcp8                   1/1     Running   0               3m15s
kube-system   kube-scheduler-minikube            1/1     Running   0               3m27s
kube-system   storage-provisioner                1/1     Running   1 (3m10s ago)   3m26s
```

# First pod

Well. The cluster is now up. `kubectl` is correctly configured and can join the cluster. Now, lets create our first app.

The pod deployment file looks like

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: noop
spec:
  containers:
  - name: alpine
    image: alpine:3.15
    command: ["sh", "-c", "while true; do sleep 3600; done"]
```

Deploying the pod is as simple as deploying the `yaml` file:

```sh
❯ kubectl apply -f configs/000-noop.yaml
pod/noop created
```

... and this is it, the app is just running:

```sh
❯ kubectl get pods
NAME     READY   STATUS    RESTARTS   AGE
noop   1/1     Running   0          10m

❯ kubectl get pods -o wide
NAME     READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
noop   1/1     Running   0          10m   172.17.0.3   minikube   <none>           <none>

```

You can then check what's running in the container:

```sh
❯ kubectl exec -ti noop -- ps auxw
PID   USER     TIME  COMMAND
    1 root      0:00 sh -c while true; do sleep 3600; done
    7 root      0:00 sleep 3600
   31 root      0:00 ps auxw

```

What is now I just wanted to start `nginx` but don't have the time to write the `yaml`? Well, lets create a deployment. And dump the deployment as a configuration file:

```sh
❯ kubectl create deployment nginx --image=nginx
deployment.apps/nginx created

❯ kubectl get deploy nginx -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2021-12-01T20:31:09Z"
  generation: 1
  labels:
    app: nginx
  name: nginx
  namespace: default
  resourceVersion: "2389"
  uid: e3775bea-68e8-47a2-a386-8bb44185bd95
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2021-12-01T20:31:19Z"
    lastUpdateTime: "2021-12-01T20:31:19Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2021-12-01T20:31:09Z"
    lastUpdateTime: "2021-12-01T20:31:19Z"
    message: ReplicaSet "nginx-6799fc88d8" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1
```

Well, that's quite a big output. But we found again our pod description.

# Accessing the pod from the cluster using ClusterIP service.

To allow accessing container ports, it is required to use services. There is multiple kind of services, and the first one I used was `ClusterIP`, which allows binding a port for the service in the cluster. What does that mean? It means the following deployment will bind the port 8123 to `nginx` (that's the service name, not the container's!) in the cluster:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
  - port: 8123 # Port exposed in cluster
    targetPort: 80 # Port inside container
    protocol: TCP
  selector:
    app: nginx
```

Note the `selector` which allows selecting the target of our service (by its label). We then are able to query the service:

```sh
❯ kubectl exec -ti noop -- wget http://nginx:8123/ -O - -q
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

After use, we can delete the service by using `kubectl delete svc nginx`:

```sh
❯ kubectl delete svc nginx
service "nginx" deleted
```

# Accessing the pod from outside the cluster using NodePort service.

After `ClusterIP`, next thing to try is `NodePort`. Consider the new service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
  - port: 8123 # Port exposed in cluster
    targetPort: 80 # Port inside container
    protocol: TCP
    nodePort: 31234
  selector:
    app: nginx
  type: NodePort
```

Warning: As it uses the same name that our older service, it will replace it.

This configuration will bind the node port 31234. You'll be then be able to reach the `nginx` server just by going to minikube VM's port directly:

```sh
❯ minikube ssh curl http://localhost:31234/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

You can also take a look at `LoadBalancer` if you want to continue service tests. In the `minikube` context, it doesn't make much sense as we only have one working node.


# Viewing the kubernetes state with dashboard

Well, command line is OK, but more viz can be better. `minikube` provides the nice `dashboard` add-ons. Just invoke it & it will pop up in your browser:

```sh
❯ minikube dashboard
```

# Using a basic configuration

It is possible to append some configuration bits using [configMaps](https://kubernetes.io/docs/concepts/configuration/configmap/), using environment variables or configuration files.

First, create a `configmap`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: testaroo-config
data:
  testaroo: "1"
  testaroo.properties: |
    this is a text file with properties
    in multiple lines in it. Thats cool
```

Then, use it the way you need (either by just adding environment variables or mounting the configuration in the container):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    env:
      - name: TESTAROO
        valueFrom:
          configMapKeyRef:
            name: testaroo-config
            key: testaroo
    volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    - name: config
      configMap:
        name: testaroo-config
        items:
          - key: "testaroo.properties"
            path: "testaroo.properties"
```

Redeploy, then check:

```sh
❯ kubectl exec -ti nginx -- env | grep -i testaroo
TESTAROO=1

❯ kubectl exec -ti nginx -- cat /config/testaroo.properties
this is a text file with properties
in multiple lines in it. Thats cool
```


# Creating some persistent volumes, and using it in containers

One another thing that can be done is to create persistent volume that will be re-used accross pods restarts.

You'll need first to create a [PersistentVolume]():

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0001
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 5Gi
  hostPath:
    path: /data/pv0001
---
```

As we're building a Persistent volume using a new directory inside the `minikube` VM, don't forget to actually create the directory:

```sh
❯ minikube ssh sudo 'mkdir /data/pv0001'
```

... and then use it in the pod, by mounting it. I'll be using another `nginx` pod for this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pv
  labels:
    app: nginx-pv
spec:
  volumes:
    - name: storage-pv
      hostPath:
        path: /data/pv0001
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: storage-pv
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-pv
spec:
  ports:
  - port: 8111 # Port exposed in cluster
    targetPort: 80 # Port inside container
    protocol: TCP
    nodePort: 31111
  selector:
    app: nginx-pv
  type: NodePort
---
```

I've also set up another `NodePort` service to be able to reach the pod.

Well, the only remaining thing to do is to deploy this new configuration:

```sh
❯ kubectl apply -f configs/004-volumes.yaml 
persistentvolume/pv0001 created
pod/nginx-pv created
service/nginx-pv created
```

After a few seconds, lets check the `pv` status:

```sh
❯ kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv0001   5Gi        RWO            Retain           Available                                   35s

❯ kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
nginx-pv                 1/1     Running   0          39s
```

Everything seems fine. Our `pv` is tied to host's `/data/pv0001`, which means if we add data in that, it should be appearing in our pod. Let's check that:

```sh
❯ kubectl exec -ti nginx-pv -- ls -ltra /usr/share/nginx/html
total 8
drwxr-xr-x 3 root root 4096 Dec 21 03:00 ..
drwxr-xr-x 2 root root 4096 Dec 21 16:53 .
```

Everything is empty so far. Let's create some file in it:

```sh
❯ kubectl exec -ti nginx-pv -- sh
# date > /usr/share/nginx/html/date.txt
# cat /usr/share/nginx/html/date.txt
Tue Dec 21 16:54:45 UTC 2021
```

As the NodePort service is here, we can check the file by just doing a simple `curl` query:

```sh
❯ minikube ssh curl http://localhost:31111/date.txt
Tue Dec 21 16:54:45 UTC 2021
```

Now, remove the `nginx-pv` pod & recreate it (& I should eventually use deployments to do this). And the end, verify if the file still here or not:

```sh
❯ kubectl delete pod nginx-pv --force
pod "nginx-pv" force deleted

❯ kubectl apply -f configs/004-volumes.yaml
persistentvolume/pv0001 unchanged
pod/nginx-pv created
service/nginx-pv unchanged

❯ minikube ssh curl http://localhost:31111/date.txt
Tue Dec 21 16:54:45 UTC 2021
```

# Deployments

Last important thing but actually not the least, are the deployments. They can be used to scale, rollout updates, and so.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

This deployment contains 3 replicas of our `nginx` container. Like any other configuration files we wrote until now, it can be deployed with `kubectl apply`:


```sh
❯ kubectl apply -f configs/005-deploy.yaml 

❯ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           55s

❯ kubectl get pod 
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-585449566-sm2l9   1/1     Running   0          63s
nginx-deployment-585449566-wkzgn   1/1     Running   0          63s
nginx-deployment-585449566-xptl9   1/1     Running   0          63s
noop                               1/1     Running   0          28h
```

## Rescale deployment

We're going from 3 to 2 replicas:

```sh
❯ kubectl scale deploy nginx-deployment --replicas=2
deployment.apps/nginx-deployment scaled

❯ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           2m15s
```


## Config rollout

What if we want to change our container configuration ? Well, edit the deployment & apply!

```yaml
...
      containers:
      - name: nginx
        image: nginx:1.20.2
...
```

Then go & deploy:

```sh
❯ kubectl apply -f configs/005-deploy.yaml 
deployment.apps/nginx-deployment configured

❯ kubectl get pod 
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-585449566-sm2l9    1/1     Running             0          4m28s
nginx-deployment-585449566-wkzgn    1/1     Terminating         0          4m28s
nginx-deployment-6b7f76486d-blxhf   1/1     Running             0          4s
nginx-deployment-6b7f76486d-bx5fp   0/1     ContainerCreating   0          1s
nginx-deployment-6b7f76486d-dwxjf   1/1     Running             0          11s
noop                                1/1     Running             0          28h

❯ kubectl describe pod nginx-deployment-6b7f76486d-bx5fp|grep Image
    Image:          nginx:1.20.2
    Image ID:       docker-pullable://nginx@sha256:03f3cb0afb7bd5c76e01bfec0ce08803c495348dccce37bcb82c347b4853c00b
```

During the rollout, the newer pods will be created one by one, while the older ones will be terminated once newer pods are ready.


If you're too lazy, you can also change directly the image on CLI:

```sh
❯ kubectl set image deployment nginx-deployment nginx=nginx:latest
deployment.apps/nginx-deployment image updated

❯ kubectl get pods
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-585449566-7dljw    1/1     Running             0          7s
nginx-deployment-585449566-k47tb    1/1     Running             0          4s
nginx-deployment-585449566-srqj5    0/1     ContainerCreating   0          1s
nginx-deployment-6b7f76486d-bx5fp   1/1     Running             0          3m12s
nginx-deployment-6b7f76486d-dwxjf   1/1     Terminating         0          3m22s
noop                                1/1     Running             0          28h
```

What if something goes wrong? You can revert the rollout to a former version!

```sh
❯ kubectl rollout history deploy nginx-deployment
deployment.apps/nginx-deployment 
REVISION  CHANGE-CAUSE
2         <none>
3         <none>

❯ kubectl rollout undo deploy nginx-deployment --to-revision=2
deployment.apps/nginx-deployment rolled back

❯ kubectl describe deploy nginx-deployment|grep Image
    Image:        nginx:1.20.2
```

We went from 1.20.2 to latest into revision 3. Going back to revision 2 will revert the change and now we're back to former nginx version!



Well, I'm done for today. In this new tutorial, I've rediscover `kubernetes` using `minikube`, and how to perform basic deployment & configuration tasks. There is still a lot to cover, but my goal with this was to start to learn again this orchestration tool.

All configuration files are [here on github](https://github.com/mycroft/k8s-configs).

# Resources

* [Learn Kubernetes](https://kubernetes.io/docs/tutorials/)
* [Minikube docs](https://minikube.sigs.k8s.io/docs/)
* [Kubernetes cheatsheet](https://kubernetes.io/fr/docs/reference/kubectl/cheatsheet/)
* [Kubernetes examples](https://k8s-examples.container-solutions.com)