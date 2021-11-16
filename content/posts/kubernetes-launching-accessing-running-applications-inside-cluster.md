---
title:  "Kubernetes: Launching & accessing running applications inside the cluster"
date:   "2019-05-22T14:24:52+0200"
categories:
  - kubernetes
summary: "Launching & getting into running apps in kubernetes"
---

There are multiple ways to start an application inside a kubernetes cluster, and I'll describe on this page how to run a simple one and the multiple ways to access it for debugging reasons. Please note that I will stay inside the cluster perimeter, and will not write about ClusterIP, LoadBalancer etc. in this one.

# Starting a pod without a deployment

The simple way to run an app - [a pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) - in to use *kubectl run*:

```sh
$ kubectl get pods
No resources found.

$ kubectl run --generator=run-pod/v1 --image=pmarie/httpd-sample:v2 httpd-sample
pod/httpd-sample created

$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
httpd-sample   1/1     Running   0          3s
```

The application I'm using, [httpd-sample](https://hub.docker.com/r/pmarie/httpd-sample) is a small golang debug httpd.

My pod is called **httpd-sample** and you can get its own description using the *describe* command:

```sh
$ kubectl describe pod httpd-sample
Name:               httpd-sample
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               kub5/10.0.2.32
Start Time:         Thu, 16 May 2019 14:30:46 +0200
Labels:             run=httpd-sample
Annotations:        <none>
Status:             Running
IP:                 10.244.2.8
[...]
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m57s  default-scheduler  Successfully assigned default/httpd-sample to kub5
  Normal  Pulled     3m56s  kubelet, kub5      Container image "pmarie/httpd-sample:v2" already present on machine
  Normal  Created    3m56s  kubelet, kub5      Created container httpd-sample
  Normal  Started    3m56s  kubelet, kub5      Started container httpd-sample
```

You can also get its logs (ie: its output) using *logs*:

```sh
$ kubectl logs httpd-sample
2019/05/16 12:30:46 --- Starting up on 8080
```

# Accessing the app

When an app is running, there are multiple ways to access it: through a proxy using the kubernetes API, using a port-forward or a service...

## Running the Kubernetes kubectl proxy

The [*kubectl proxy*](https://kubernetes.io/docs/concepts/cluster-administration/proxies/) allows to access applications through a [REST API](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#directly-accessing-the-rest-api):

```sh
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
[...]
```

On another terminal:

```sh
$ curl http://127.0.0.1:8001/version
{
  "major": "1",
  "minor": "14",
  "gitVersion": "v1.14.1",
  "gitCommit": "b7394102d6ef778017f2ca4046abbaa23b88c290",
  "gitTreeState": "clean",
  "buildDate": "2019-04-08T17:02:58Z",
  "goVersion": "go1.12.1",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Note that you can query all kubernetes' API endpoints this way. Feel free to query *http://127.0.0.1:8001* directly. Once reachable, you will be able to query any http port of your pods using the pod name and its port number:


```sh
$ export POD_NAME=httpd-sample
$ curl http://127.0.0.1:8001/api/v1/namespaces/default/pods/${POD_NAME}:8080/proxy/
Date: Thu May 16 09:19:23
RemoteAddr: 10.244.0.0:43356
Host: 127.0.0.1:8001
RequestURI: /
User-agent: curl/7.29.0
Random-Number: 8674665223082153551
```

## Accessing the app through port-forward

It is possible to forward any port using [kubectl port-forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/):

```sh
$ kubectl port-forward httpd-sample 60123:8080
Forwarding from 127.0.0.1:60123 -> 8080
Forwarding from [::1]:60123 -> 8080
[...]
```

Your app will be reachable using the newly opened port:

```sh
$ curl http://localhost:60123
Date: Thu May 16 09:22:04
RemoteAddr: 127.0.0.1:55342
Host: localhost:60123
RequestURI: /
User-agent: curl/7.29.0
Random-Number: 4037200794235010051
```

## Running a shell inside the app's container

As any docker container, you'll be able ot run processes within it or even [run a shell](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/):

```sh
$ kubectl exec httpd-sample -ti /bin/sh
/app # ls
httpd-sample
```