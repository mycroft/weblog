---
title: "100 Days of K8S with Typescript"
date: 2022-09-25T20:54:51+02:00
draft: true
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