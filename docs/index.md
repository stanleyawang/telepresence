Have you ever wanted the quick development cycle of local code while still having your code run within a remote Kubernetes cluster?
Telepresence allows you to run your code locally while still:

1. Giving your code access to Services in a remote Kubernetes cluster.
2. Giving your code access to cloud resources like AWS RDS or Google PubSub.
3. Allowing Kubernetes to access your code as if it were in a normal pod within the cluster.

**IMPORTANT:** Telepresence is currently in the prototyping stage, and we expect it to change rapidly based on user feedback.
Please [file bugs and feature requests](https://github.com/datawire/telepresence/issues)!

## Theory of operation

Let's assume you have a web service which listens on port 8080, and has a Dockerfile which gets built to an image called `examplecom/servicename`.
Your service depends on other Kubernetes `Service` instances (`thing1` and `thing2`), and on a cloud database.

The Kubernetes production environment looks like this:

<div class="mermaid">
graph LR
  subgraph Kubernetes in Cloud
    code["k8s.Pod: servicename"]
    s1["k8s.Service: servicename"]-->code
    code-->s2["k8s.Service: thing1"]
    code-->s3["k8s.Service: thing2"]
    code-->c1>"Cloud Database (AWS RDS)"]
  end
</div>

Currently Telepresence works by running your code locally in a Docker container, and forwarding requests to/from the remote Kubernetes cluster.

<div class="mermaid">
graph LR
  subgraph Laptop
    code["servicename, in container"]---client[Telepresence client]
  end
  subgraph Kubernetes in Cloud
    client-.-proxy["k8s.Pod: Telepresence proxy"]
    s1["k8s.Service: servicename"]-->proxy
    proxy-->s2["k8s.Service: thing1"]
    proxy-->s3["k8s.Service: thing2"]
    proxy-->c1>"Cloud Database (AWS RDS)"]
  end
</div>

(Future versions may allow you to run your code locally directly, without a local container.
[Let us know](https://github.com/datawire/telepresence/issues/1) if this a feature you want.)

## Installing

You will need the following available on your machine:

* Docker.
* Python (2 or 3). This should be available on any Linux or OS X machine.
* Access to your Kubernetes cluster, with local credentials on your machine.
  You can do test this by running `kubectl get pod` - if this works you're all set.

In order to install, run the following command:

```
curl -L https://github.com/datawire/telepresence/raw/{{ site.data.version.version }}/cli/telepresence -o telepresence
chmod +x telepresence
```

Then move telepresence to somewhere in your `$PATH`, e.g.:

```
mv telepresence /usr/local/bin
```

## Quickstart

Let's try Telepresence out quickly.
First, we'll connect to the Kubernetes cluster:

```console
host$ telepresence start --new-deployment --expose 8080
Generated new deployment 'deployment-12343'.
```

Given that deployment name we can run Docker commands locally that are exposed to the remote cluster.
We'll do it with a shell, so we can try different things.
`telepresence run-local` is a thin wrapper around `docker run`, so we can use standard `docker run` arguments.

In a different terminal we'll start a container using the `alpine` image.
As you can see below, environment variables are set similar to those set in Kubernetes pods to point at a `Service`.
We can send a request to that service and it will get proxied, and we can also use hostnames:

```console
host$ telepresence run-local --deployment deployment-12343 \
      -i -t alpine /bin/sh  # <-- arguments passed to `docker run`
localcontainer$ env
KUBERNETES_SERVICE_HOST=127.0.0.1
KUBERNETES_SERVICE_PORT=60001
localcontainer$ apk add --no-cache curl  # install curl
localcontainer$ curl -k -v "https://127.0.0.1:60001/"
> GET / HTTP/1.1
> User-Agent: curl/7.38.0
> Host: 10.0.0.1
> Accept: */*
> 
< HTTP/1.1 401 Unauthorized
< Content-Type: text/plain; charset=utf-8
< X-Content-Type-Options: nosniff
< Date: Mon, 06 Mar 2017 19:19:44 GMT
< Content-Length: 13
Unauthorized
localcontainer$ curl -k "https://kubernetes.default.svc.cluster.local/"
Unauthorized
localcontainer$ exit
host$
```

We've sent a request to the Kubernetes API service, and we could similarly talk to any `Service` in the remote Kubernetes cluster, even though the container is running locally.

Finally, since we exposed port 8080 on the remote cluster, we can run a local server (within the container) that listens on port 8080 and it will be exposed via port 8080 inside the Kubernetes pods we've created.
Let's say we want to serve some static files from your local machine:
We can mount the current directory as a Docker volume, and run a webserver on port 8080:

```console
host$ echo "hello!" > file.txt
host$ telepresence run-local --deployment deployment-12343 \
      -v $PWD:/files -i -t python:3-slim /bin/sh   # <-- passed to `docker run`
localcontainer$ cd /files
localcontainer$ ls
file.txt
localcontainer$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 ...
```

At this point the code will be accessible from inside the Kubernetes cluster:

<div class="mermaid">
graph TD
  subgraph Laptop
    code["myserver.py on port 8080, in container"]---client[Telepresence client]
  end
  subgraph Kubernetes in Cloud
    client-.-proxy["k8s.Pod: Telepresence proxy, listening on port 8080"]
  end
</div>

Let's send a request to the remote pod.
In a different terminal we figure out the name of the pod running our Telepresence proxy:

```console
$ kubectl get pod
NAME                                 READY     STATUS         RESTARTS   AGE
deployment-12343-3572484030-7k4bk    1/1       Running        0          1m
```

Now we port forward local port 1234 to port 8080 on that pod:

```console
$ kubectl port-forward deployment-12343-3572484030-7k4bk 1234:8080 &
```

And now we can send requests, via the remote Kubernetes cluster, back to our local code:

```console
$ curl http://localhost:1234/file.txt
hello!
```

More usefully, if you have a `Service` configured to point at the pod running the proxy you will be able to access your local code from other places in the Kubernetes cluster.

## In-depth usage

Let's look in a bit more detail at using Telepresence.

Your Kubernetes configuration will typically have a `Service`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: servicename-service
spec:
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    name: servicename
```

You will also have a `Deployment` that actually runs your code, with labels that match the `Service` `selector`:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: servicename-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        name: servicename
    spec:
      containers:
      - name: servicename
        image: examplecom/servicename:1.0.2
        ports:
        - containerPort: 8080
      - env:
        - name: YOUR_DATABASE_HOST
          value: somewhere.someplace.cloud.example.com
```

In order to run Telepresence you will need to do three things:

1. Replace your production `Deployment` with a custom `Deployment` that runs the Telepresence proxy.
2. Run the Telepresence client locally in Docker.
3. Run your own code in its own Docker container, hooked up to the Telepresence client.

Let's go through these steps one by one.

### 1. Run the Telepresence proxy in Kubernetes

Instead of running the production `Deployment` above, you will need to run a different one that runs the Telepresence proxy instead.
It should only have 1 replica, and it will use a different image, but it should have the same environment variables since you want those available to your local code.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: servicename-deployment
spec:
  replicas: 1  # <-- only one replica
  template:
    metadata:
      labels:
        name: servicename
    spec:
      containers:
      - name: servicename
        image: datawire/telepresence-k8s:{{ site.data.version.version }}  # <-- new image
        ports:
        - containerPort: 8080
      - env:
        - name: YOUR_DATABASE_HOST
          value: somewhere.someplace.cloud.example.com
```

You should apply this file to your cluster:

```console
$ kubectl apply -f telepresence-deployment.yaml
```

### 2. Run the local Telepresence client on your machine

You want to do the following:

1. Expose port 8080 in your code to Kubernetes.
2. Proxy `somewhere.someplace.cloud.example.com` port 5432 via Kubernetes, since it's probably not accessible outside of your cluster.
3. Connect specifically to the `servicename-deployment` pod you created above, in case there are multiple Telepresence users in the cluster.

Services `thing1` and `thing2` will be available to your code automatically so no special parameters are needed for them.
You can do so with the following command line:

```console
$ telepresence start --deployment servicename-deployment \
               --proxy somewhere.someplace.cloud.example.com:5432 \
               --expose 8080
A new environment file named `servicename-deployment.env` was generated.
```

### 3. Run your code locally in a container

You can now run your own code locally inside Docker, attaching it to the network stack of the Telepresence client and using the environment variables Telepresence client extracted:

```console
$ docker run --net=container:servicename-deployment \ 
             --env-file=servicename-deployment.env \
             examplecom/servicename:latest
```

Or, you can use the simpler Telepresence wrapper for `docker run`:

```console
$ telepresence run-local --deployment servicename-deployment \
               # `docker run` arguments: \
               examplecom/servicename:latest
```

Your code is now connected to the remote Kubernetes cluster.

### 4. (Optional) Better local development with Docker

To make Telepresence even more useful, you might want to use a custom Dockerfile setup that allows for code changes to be reflected immediately upon editing.

For interpreted languages the typical way to do this is to mount your source code as a Docker volume, and use your web framework's ability to reload code for each request.
Here are some tutorials for various languages and frameworks:

* [Python with Flask](http://matthewminer.com/2015/01/25/docker-dev-environment-for-web-app.html)
* [Node](http://fostertheweb.com/2016/02/nodemon-inside-docker-container/)

## What Telepresence proxies

Telepresence currently proxies the following:

* The [special environment variables](https://kubernetes.io/docs/user-guide/services/#environment-variables) that expose the addresses of `Service` instances.
  E.g. `REDIS_MASTER_SERVICE_HOST`.
  These will be modified with new values based on the proxying logic, but that should be transparent to the application.
* The standard [DNS entries for services](https://kubernetes.io/docs/user-guide/services/#dns).
  E.g. `redis-master` and `redis-master.default.svc.cluster.local` will resolve to a working IP address.
* TCP connections to other `Service` instances that existed when the proxy was supported.
* Any additional environment variables that a normal pod would have, with the exception of a few environment variables that are different in the local environment.
  E.g. UID and HOME.
* TCP connections to specific hostname/port combinations specified on the command line.
  Typically this would be used for cloud resources, e.g. a AWS RDS database.
* TCP connections *from* Kubernetes to your local code, for ports specified on the command line.

Currently unsupported:

* TCP connections, environment variables, DNS records for `Service` instances created *after* Telepresence is started.
* SRV DNS records matching `Services`, e.g. `_http._tcp.redis-master.default`.
* UDP messages in any direction.
* For proxied addresses, only one destination per specific port number is currently supported.
  E.g. you can't proxy `remote1.example.com:5432` and `remote2.example.com:5432` at the same time.
* Access to volumes, including those for `Secret` and `ConfigMap` Kubernetes objects.

## Help us improve Telepresence!

We are considering various improvements to Telepresence, including:

* [Removing need for Kubernetes credentials](https://github.com/datawire/telepresence/issues/2)
* [Allowing running code locally without a container](https://github.com/datawire/telepresence/issues/1)
* Implementing any of the unsupported features mentioned above.

Please add comments to relevant tickets if you are interested in these features, or [file a new issue](https://github.com/datawire/telepresence/issues/new) if there is no existing ticket for a desired feature or bug report.

## Alternatives

Some alternatives to Telepresence:

* Minikube is a tool that lets you run a Kubernetes cluster locally.
  You won't have access to cloud resources, however, and your development cycle won't be as fast since access to local source code is harder.
  Finally, spinning up your full system may not be realistic if it's big enough.
* Docker Compose lets you spin up local containers, but won't match your production Kubernetes cluster.
  It also won't help you access cloud resources, you will need to emulate them.
* Pushing your code to the remote Kubernetes cluster.
  This is a somewhat slow process, and you won't be able to do the quick debug cycle you get from running code locally.