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


# Day 3: Testing using jest

Today, I'm testing testing. It seems like it is possible to test javascript or typescript stuff with [jest](https://jestjs.io). When generating first time our cdk8s app, it created already a basic unit test. Let's install `jest` before continuing:

```sh
> npm install -y jest
```

jest is able to record snapshots of the current state, so it can be re-tested later. This can be done thanks to:

```sh
> jest --updateSnapshot
```

I added some basic test to verify the generated service. I'm testing it is a label value, the defined ports and the type . It looks like this:

```ts
  test('HelloWorldTsHasService', () => {
    const app = Testing.app();
    const chart = new HelloWorldChart(app, 'test-chart');
    const results = Testing.synth(chart);

    var service = results.find((obj) => {
      return obj.kind === 'Service';
    });

    expect(service.metadata.labels['app']).toBe('hello-k8s');
    expect(service.spec.ports.length).toBe(1);
    expect(service.spec.ports[0].port).toBe(8001);
    expect(service.spec.ports[0].targetPort).toBe(8080);
    expect(service.spec.type).toBe('LoadBalancer');
  });
```

Running the test looked like this:

```sh
> jest
 PASS  ./main.test.ts
  TestHelloWorldTs
    ✓ BasicHelloWorldSnapshot (5 ms)
    ✓ HelloWorldTsHasService (1 ms)

Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshots:   1 passed, 1 total
Time:        1.192 s
Ran all test suites.
```


# Day 4: Playing with kubernetes/client-node.

Today, I took a look to the javascript kubernetes client, aka [kubernetes/client-node](https://github.com/kubernetes-client/javascript).

First, I created a new project and imported the library:

```sh
> npm install -y @kubernetes/client-node
```

Then, I wrote some code:

```ts
import _, { CoreV1Api, AppsV1Api, KubeConfig } from '@kubernetes/client-node';

const kc = new KubeConfig();
kc.loadFromDefault();

const k8sApi = kc.makeApiClient(CoreV1Api);
const k8sAppsApi = kc.makeApiClient(AppsV1Api);

async function listDeploymentsForNamespace(namespace: string) {
    const res = await k8sAppsApi.listNamespacedDeployment(namespace);
    res.body.items.forEach((o: _.V1Deployment) => {
        console.log("namespace:" + namespace + " " + o.metadata?.name);
    });
}

async function listNamespacesAndDeployments() {
    const res = await k8sApi.listNamespace();
    res.body.items.forEach((o: _.V1Namespace) => {
        if (typeof o.metadata?.name === 'string') {
            listDeploymentsForNamespace(o.metadata?.name);
        }
    });
}

listNamespacesAndDeployments();
```

The snippet is quite small, and it took me some time to write it as I really hate async code. My goal was to write a couple of api calls and do stuff with those, and the only thing I succeeded was to list namespaces & deployments in those. At first, the result was messy (async code!), but eventually I made it work. The result of this:

```sh
> npx ts-node hello-world.ts
namespace:default hello-world-ts-deployment-c881e597
namespace:cert-manager cert-manager
namespace:cert-manager cert-manager-cainjector
namespace:cert-manager cert-manager-webhook
namespace:kube-system local-path-provisioner
namespace:kube-system coredns
namespace:kube-system traefik
namespace:kube-system metrics-server
```
