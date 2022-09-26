---
title: "100 Days of K8S with Typescript"
date: 2022-09-25T20:54:51+02:00
summary: Let's write some stuff in typescript
categories:
- kubernetes
- typescript
---

This post will be another attempt to learn Kubernetes, using typescript & cdk8s this time.

# Day 1: Workplace initialization & Hello World.

You'll need `node`, `npm` & `typescript`. I've installed those with `brew` (cuz I'm doing this on Mac OS X):

```sh
> brew install node npm typescript
...
> node --version
v18.9.0
> npm --version
8.19.1
> tsc --version
Version 4.8.3
```

You might also want to install `ts-node` so it will not be needed to convert `typescript` to `javascript` before running it with `node`:

```sh
> npm install -g ts-node
...
> npx ts-node --version
v10.9.1
```

Once this little people is installed, let's write our first typescript program, the `hello world` program:

```ts
let message: string = 'Hello, World!';
console.log(message);
```

Then, convert it with `tsc` & run it with `node`:

```sh
> tsc app.ts
> ls -l
total 16
-rw-r--r--  1 pmarie  staff  53 25 Sep 21:38 app.js
-rw-r--r--  1 pmarie  staff  61 13 Sep 23:15 app.ts
> cat app.js
var message = 'Hello, World!';
console.log(message);

> node app.js
Hello, World!
```

Or do it with only one step using `ts-node`:

```sh
> npx ts-node app.ts
Hello, World!
```


# Day 2: Installing cdk8s & creating our first chart

[cdk8s](https://cdk8s.io) is a development framework dedicated to build kubernetes charts using typescript, python or golang. Installing it is as simple as it can be done with any other tool using `brew`:

```sh
> brew install cdk8s
>
```

## Bootstrapping the cdk8s app

Initializing a new application is a bit long, but it will download all nodes modules and build the initial k8s layer to be able to create k8s charts. It can be done with `cdk8s init`: 

```sh
> cdk8s init typescript-app
...
 Your cdk8s typescript project is ready!

   cat help         Print this message
 
  Compile:
   npm run compile     Compile typescript code to javascript (or "yarn watch")
   npm run watch       Watch for changes and compile typescript in the background
   npm run build       Compile + synth

  Synthesize:
   npm run synth       Synthesize k8s manifests from charts to dist/ (ready for 'kubectl apply -f')

 Deploy:
   kubectl apply -f dist/

 Upgrades:
   npm run import        Import/update k8s apis (you should check-in this directory)
   npm run upgrade       Upgrade cdk8s modules to latest version
   npm run upgrade:next  Upgrade cdk8s modules to latest "@next" version (last commit)
```

Building the code & synthetizing the charts is as simple as running `npm run compile` then `cdk8s synth`:

```sh
> npm run compile && cdk8s synth
...
dist/hello-world-ts.k8s.yaml
```

Per default, the chart will be empty. Thanksfully, the [getting started](https://cdk8s.io/docs/latest/getting-started/#importing-constructs-for-the-kubernetes-api) page gives a basic example with a `service` & a `deployment`:

```ts
    const label = { app: 'hello-k8s' };

    new KubeService(this, 'service', {
      metadata: {
        labels: label,
      },
      spec: {
        type: 'LoadBalancer',
        ports: [ { port: 8001, targetPort: IntOrString.fromNumber(8080) } ],
        selector: label
      }
    });

    new KubeDeployment(this, 'deployment', {
      spec: {
        replicas: 2,
        selector: {
          matchLabels: label
        },
        template: {
          metadata: { labels: label },
          spec: {
            containers: [
              {
                name: 'hello-kubernetes',
                image: 'paulbouwer/hello-kubernetes:1.7',
                ports: [ { containerPort: 8080 } ]
              }
            ]
          }
        }
      }
    });
```

After building & creating the charts, the `hello-world-ts.k8s.yaml` file is no longer empty:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hello-k8s
  name: hello-world-ts-service-c8780799
spec:
  ports:
    - port: 8001
      targetPort: 8080
  selector:
    app: hello-k8s
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-ts-deployment-c881e597
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-k8s
  template:
    metadata:
      labels:
        app: hello-k8s
    spec:
      containers:
        - image: paulbouwer/hello-kubernetes:1.7
          name: hello-kubernetes
          ports:
            - containerPort: 8080
```

Note: At the time of writing, I'm using a [k3s](https://k3s.io) for my tests. The created service (with a loadbalancer) will then spawn a pod on each nodes to bind the port (8001). For each other `LoadBalancer` services, using another port will be required. Note also that other k8s installation might require more setup to work correctly.

## Testing the chart

```sh
> kubectl apply -f dist/hello-world-ts.k8s.yaml
service/hello-world-ts-service-c8780799 created
deployment.apps/hello-world-ts-deployment-c881e597 created
> kubectl get services --show-labels --selector='app=hello-k8s'
NAME                              TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE   LABELS
hello-world-ts-service-c8780799   LoadBalancer   10.43.190.35   10.0.0.4      8001:30106/TCP   50m   app=hello-k8s
> kubectl get pods --show-labels --selector='app=hello-k8s'
NAME                                                 READY   STATUS    RESTARTS   AGE   LABELS
hello-world-ts-deployment-c881e597-5c667d695-shtc4   1/1     Running   0          53m   app=hello-k8s,pod-template-hash=5c667d695
hello-world-ts-deployment-c881e597-5c667d695-zhnrm   1/1     Running   0          53m   app=hello-k8s,pod-template-hash=5c667d695
> curl -s http://shuttle:8001/ | pandoc -f html -t plain
[]

Hello world!

  ------- ----------------------------------------------------
  pod:    hello-world-ts-deployment-c881e597-5c667d695-zhnrm
  node:   Linux (5.19.7-200.fc36.x86_64)
  ------- ----------------------------------------------------
```

Applying the example chart using `kubectl apply` created a deployment (with 2 replicas) & a service binding on my installation port `8001` on each k8s nodes. Reaching one of the node is correctly serving the demo container.
