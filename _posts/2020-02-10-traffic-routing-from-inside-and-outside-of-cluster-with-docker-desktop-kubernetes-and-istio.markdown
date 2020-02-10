---
layout: post
title:  "Traffic routing from inside and outside of cluster with Docker Desktop, Kubernetes and Istio"
commentIssueId: 19
---

{::options parse_block_html="true" /}
<div class="toc">
Contents

* toc
{:toc}
</div>

This post is about routing traffic with Istio to different versions of a given service. It shows how
the routing works from outside of the cluster as well as from the inside and how to visualize the
traffic and debug configurations.

The set-up has been tested on Docker Desktop for Mac version 2.2.0.0 which includes Kubernetes
version 1.15.5.

## Prerequisites

This test expects a clean Kubernetes set-up running on Docker Desktop. Also `istioctl` needs to be
installed, which can be done on Mac with `brew install istioctl`.

**N.B.** Configure enough memory for the Docker Desktop as running out of memory can lead to weird
issues. For this test 4 GB seems to be adequate.

### Installing Istio

Download Istio set-up folder with `curl -L https://istio.io/downloadIstio | sh -`. Then cd into the
folder and install Istio in demo mode with `istioctl manifest apply --set profile=demo`.

### Accessing the Istio-dashboard

Kiali (the dashboard) is enabled with the demo-profile. It uses the default secret for the username
and password (admin/admin). It can be accessed with `istioctl dashboard kiali`.

## Test image

As a test image we use the [http-https-echo](https://github.com/mendhak/docker-http-https-echo) by
mendhak which is a service that echoes all the requests back to the caller with all the information
that the original call contained, like http-headers, which enables us to easily distinguish between
different containers and versions that host the service.

## Creating the components

To be able to test the settings as we go on, the creation is started with the namespace and then
from the "bottom" up, ie. Deployment->Service->VirtualService. This way after creating the
Deployment (which creates the Pods) we can test it using port forward and after creating the
Service, we can test the service from inside the cluster with the Kubernetes dns name.

### Namespace

Namespace for the service is called *echo* and creating it is done with `kubectl create namespace
echo`. After creation the automated sidecar proxy creation is enabled for the namespace with
`kubectl label namespace echo istio-injection=enabled`.

### Deployment

As we'll be testing versioning of the services later, we add a *v1* to the deployment now. As a
sidenote, especially the Deployment object has a lot of repeating information.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-deployment-v1
  namespace: echo
  labels:
    app: echo
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
      version: v1
  template:
    metadata:
      labels:
        app: echo
        version: v1
    spec:
      containers:
        - name: echo
          image: mendhak/http-https-echo
          ports:
            - containerPort: 80
```

After applying the configuration, the running pod information can be found with `kubectl get pods -n
echo`. A port can be forwarded to the running pod with `kubectl port-forward -n echo pod/<pod-name>
8080:80`. Then point a browser to [http://localhost:8080](http://localhost:8080) and you should see
the echo give a reply back.

### Service

Next level is the Service. It is pretty straightforward and can be created with the following
configuration

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-service
  namespace: echo
spec:
  selector:
    app: echo
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
```

One way to check that the service works is to create a container inside the cluster and do a `curl`
from there. It can be achieved with `kubectl -n echo run debug -i -t --rm --image=sirile/netdebug`.
After a shell is given, run either `curl echo-service` or `http echo-service` and you should get a
reply.

**N.B.** This bypasses the Istio traffic management as the traffic doesn't go through a Gateway
that is listened by a VirtualService, rather than going straight to the Pods controlled by the
Service. This will only become visible later, but may lead to a bit of confusion as the original
`curl` will work and not require any url-prefixes that have been configured in the VirtualService.
The system will work as specified later when the mesh is added to the gateway definitions.

### Destination

As we are using a subset for the service selection, we need a Destination to support it.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: echo
  namespace: echo
spec:
  host: echo-service
  subsets:
    - name: v1
      labels:
        version: v1
```

### VirtualService

As istio creates a default ingress gateway with the demo-profile, we just use it.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: echo-route
  namespace: echo
spec:
  hosts:
    - "*" # Wildcard can be used as we don't have mesh in the gateways
  gateways:
    - istio-system/ingressgateway
  http:
    - name: "echo-route"
      match:
        - uri:
            prefix: "/echo"
      route:
        - destination:
            host: echo-service
```

### Testing the routing

For testing some load needs to be generated against the service. For this I use
[loadtest](https://github.com/alexfernandez/loadtest) which is a handy tool to generate constant
traffic. It can be installed with `yarn global add loadtest`. After the previous commands have been
succesfully run, some test load can be generated with `loadtest --rps 5 http://localhost/echo` which
generates a constant load of 5 requests per second.

The generated load and traffic-flow can be examined through the kiali-dashboard.

## Adding versions for services

Now we can create a second version of the echo-deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-deployment-v2
  namespace: echo
  labels:
    app: echo
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
      version: v2
  template:
    metadata:
      labels:
        app: echo
        version: v2
    spec:
      containers:
        - name: echo
          image: mendhak/http-https-echo
          ports:
            - containerPort: 80
```

### Destination v2

Destination is updated to also include v2.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: echo
  namespace: echo
spec:
  host: echo-service
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

### VirtualService v2

VirtualService is updated to distribute load to both v1 and v2. This set-up will also distribute
traffic correctly from inside of the service mesh by including it as a gateway. That means that
wildcard can't be used as the hosts definition any more.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: echo-route
  namespace: echo
spec:
  hosts:
    - "echo-service.echo.svc.cluster.local"
    - "echo.local" # Wildcard can't be used
  gateways:
    - istio-system/ingressgateway
    - mesh # Mesh is specified as a Gateway
  http:
    - name: "echo-route"
      match:
        - uri:
            prefix: "/echo"
      route:
        - destination:
            host: echo-service
            subset: v1
          weight: 75
        - destination:
            host: echo-service
            subset: v2
          weight: 25

```

### Testing the routing from the outside

From the outside the host-header must be passed, so the test looks like `http localhost/echo
host:echo.local` or `curl --header 'host:echo.local' localhost/echo`. To generate constant load, the
command `loadtest --rps 5 -H 'host:echo.local' http://localhost/echo` can be used.

### Testing the routing from the inside of cluster

Start the container inside the cluster with `kubectl run -n echo debug -i -t --rm
--image=sirile/netdebug`. From there the services can be called with `http echo-service/echo`. The
distribution of calls to v1 and v2 should follow the correct specifications.

**N.B.** Now the curl from the inside of the cluster needs to have the proper url-prefix added for
the traffic to flow to the destination specified in the VirtualService.

## Conclusions

Setting up the versioning of the services and having the load distributed correctly from outside and
inside of the cluster isn't exactly trivial and some of the information needs some digging. At first
I didn't understand that the mesh had to be added as a Gateway for Istio to able to control the
traffic that originates from inside the mesh, but after figuring it out things worked as expected.
Visualization of the cluster and the traffic flow inside it with Kiali is impressive.

## Kiali with visualization of traffic

![Dashboard-image](/images/kiali.png "Kiali dashboard")
