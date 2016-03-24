---
layout: post
title: Trying out Kubernetes 1.2
date: '2016-03-24 12:22'
commentIssueId: 14
---

{::options parse_block_html="true" /}
<div class="toc">
Contents

* toc
{:toc}
</div>

## General

In this post I'll go over setting up a simple application on the newly released Kubernetes 1.2 version. I am not that familiar with earlier Kubernetes versions as I haven't been a fan of all the necessary configuration to get simple things running, but with version 1.2 a lot of progress has been made to make k8s easier and simpler to use.

## Installation

Installing Kubernetes on OS X is straightforward. You just run the two lines

{% highlight bash %}
export KUBERNETES_PROVIDER=vagrant
curl -sS https://get.k8s.io | bash
{% endhighlight %}

and Kubernetes is downloaded. The package is large (over 425MB) and will take some time to download.

The script immediately starts up the Vagrant based local environment with two nodes, master and node-1. While starting up the virtual machines are automatically provisioned and updated which takes several minutes. The setup should be completed with a success message.

## Controlling

The downloaded package is automatically extracted to a subdirectory called kubernetes. The scripts used for controlling kubernetes are located in kubernetes/cluster. This directory can either be added to the path or most of the work can be done in that directory as then also Vagrant knows about the status of the VirtualBox environments without having to use global-status and machine ids.

You can access the Kubernetes dashboard and Cockpit through the URLs that are listed after the setup has completed. I tried installing a simple service through Cockpit and the end result is different than what you get by installing using the new `kubectl.sh run` command, so I decided to stick with the commandline.

### Suspending and resuming

The status of the Vagrant controlled VMs can be checked either by running `vagrant global-status` or if you're in the kubernetes/cluster directory just by issuing `vagrant status`. With `vagrant suspend` the VMs are saved and they can later on be resumed with `vagrant up`. If you are not in the kubernetes/cluster folder, you need to provide the ids for the commands to work.

### Accessing the nodes

Nodes can be accessed with `vagrant ssh nodename` if you are in the kubernetes/cluster directory or `vagrant ssh id` if you use the global id.

### Changing the number of nodes

The number of installed nodes can be altered with an environment variable. It can be set after the initial set-up has finished and then the new node can be added by running the `kube-up.sh` script, but at least for me the validation script got stuck and I started over with `vagrant destroy` and then `./kube-up.sh`.

{% highlight bash %}
export NUM_NODES=2
./kube-up.sh
{% endhighlight %}

## Starting a simple service

With version 1.2 Kubernetes has simplified the application definition so that the yaml-based configuration isn't  needed. With the run command a service can be started and the configuration can also be later edited so that the yaml-based configuration file is provided by Kubernetes for editing.

### Running a simple image

Creating and starting a service with the new run-command looks like

{% highlight bash %}
./kubectl.sh run iletest --image=sirile/go-image-test --replicas=2 --port=80 --expose
{% endhighlight %}

This starts the defined image on two pods and automatically creates the service which exposes the pods. It creates a *deployment* and corresponding *ReplicaSets* instead of *ReplicationControllers*. As most of the examples still use *ReplicationControllers* I was quite confused when I tried to get the service exposed.

### Exposing the image via LoadBalancer

By default the started Deployment is of the type *ClusterIP* which means that it is visible inside the cluster, but not to outside world. For external visibility the service type needs to be changed into *LoadBalancer* or *NodePort*. As *NodePort* would mean that the service is only visible on that exact node it is running on, *LoadBalancer* makes more sense.

#### Editing a service

Editing a service can be done with

{% highlight bash %}
./kubectl.sh edit service/iletest
{% endhighlight %}

As I normally use Atom as the default editor I had to change the editor to wait mode which can be done with

{% highlight bash %}
export EDITOR="atom --wait"
{% endhighlight %}

After using that for a while I found out that as Kubernetes checks the file for validity when it's saved, a better feedback loop was achieved when using joe as the editor, so I decided to go with

{% highlight bash %}
export EDITOR="joe"
{% endhighlight %}

When you save the file (in this case by pressing ctrl+k+x) if the syntax is incorrect, the header of the file tells you what are allowed values, where the error was and the file is only really saved and taken into use when everything is okay. That's excellent usability and should be the new norm in everything that needs configuring.

#### Changing the service type

{% highlight yaml %}
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: 2016-03-24T09:53:58Z
  name: iletest
  namespace: default
  resourceVersion: "232"
  selfLink: /api/v1/namespaces/default/services/iletest
  uid: 54404fdb-f1a6-11e5-942b-080027242396
spec:
  clusterIP: 10.247.93.166
  ports:
  - nodePort: 31948
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: iletest
  sessionAffinity: None
  type: ClusterIP # <- Change this to LoadBalancer
status:
  loadBalancer: {}
{% endhighlight %}

After saving the file it is immediately taken into use.

## Testing the service

The documentation says that LoadBalancer implementation is provider specific and it looks like the Vagrant version exposes the LoadBalancer on all the nodes to the NodePort. When testing with curl it looks like services do get called from both nodes successfully.

### Finding out the port

The port can be queried with

{% highlight bash %}
$ ./kubectl.sh describe service/iletest
Name:			iletest
Namespace:		default
Labels:			<none>
Selector:		run=iletest
Type:			LoadBalancer
IP:			10.247.93.166
Port:			<unset>	80/TCP
NodePort:		<unset>	31948/TCP
Endpoints:		10.246.62.3:80,10.246.62.4:80,10.246.85.5:80 + 1 more...
Session Affinity:	None
No events.
{% endhighlight %}

### Finding out the node IPs

Vagrant based VMs always have the IPs so that node-1 is 10.245.1.3, node-2 is 10.245.1.4 and so on, but these IPs can also be found out with the query

{% highlight bash %}
$ ./kubectl.sh get -o json nodes/kubernetes-node-1 | grep -A1 LegacyHostIP
                "type": "LegacyHostIP",
                "address": "10.245.1.3"
{% endhighlight %}

### Testing the service using curl or browser

The query with the IP and port looks like

{% highlight bash %}
$ curl 10.245.1.3:31948
<!DOCTYPE html><html><head><title>Node scaling demo</title></head><body><h1>iletest-836207442-vyd50</h1><svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 400"><circle cx="50" cy="50" r="48" fill="#50a8d0" stroke="#000"/><path d="M50,2a48,48 0 1 1 0,96a24 24 0 1 1 0-48a24 24 0 1 0 0-48" fill="#000"/><circle cx="50" cy="26" r="6" fill="#000"/><circle cx="50" cy="74" r="6" fill="#FFF"/></svg></body></html>
$ curl 10.245.1.4:31948
<!DOCTYPE html><html><head><title>Node scaling demo</title></head><body><h1>iletest-836207442-x82ho</h1><svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 400"><circle cx="50" cy="50" r="48" fill="#381808" stroke="#000"/><path d="M50,2a48,48 0 1 1 0,96a24 24 0 1 1 0-48a24 24 0 1 0 0-48" fill="#000"/><circle cx="50" cy="26" r="6" fill="#000"/><circle cx="50" cy="74" r="6" fill="#FFF"/></svg></body></html>
{% endhighlight %}

With browser there seems to be some affinity where refreshing the page gives the same instance, but for example trying with a few different browsers (and incognito mode) shows that the results do come from different instances.

### Listing Kubernetes services

The Kubernetes services can be listed with

{% highlight bash %}
$ ./kubectl.sh cluster-info
Kubernetes master is running at https://10.245.1.2
Heapster is running at https://10.245.1.2/api/v1/proxy/namespaces/kube-system/services/heapster
KubeDNS is running at https://10.245.1.2/api/v1/proxy/namespaces/kube-system/services/kube-dns
kubernetes-dashboard is running at https://10.245.1.2/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard
Grafana is running at https://10.245.1.2/api/v1/proxy/namespaces/kube-system/services/monitoring-grafana
InfluxDB is running at https://10.245.1.2/api/v1/proxy/namespaces/kube-system/services/monitoring-influxdb
{% endhighlight %}

The default username and password for the services are vagrant/vagrant.

## Conclusion

Setting up a local Vagrant based multi-node Kubernetes set-up is now easy and straightforward.

For doing simple development using Kubernetes locally seems like quite a overkill as the installation takes a lot of time and the set-up takes up quite a lot of resources. For managing a production ready environment it looks like a good fit, although then controlling things on a more fine-grained level using yaml-based configuration will make more sense. A lot of simplification has been achieved with version 1.2, although many examples still refer to older terminology and the best practices will surely change.

It will be interesting to see if Kubernetes will take into use the simpler networking model that was introduced with Docker 1.9. The Docker version in Kubernetes 1.2 seems to be 1.9 series, but I'd guess that they will quite quickly move to the 1.10 series.

For some reason the included Innoflux and Grafana don't seem to start if there is only one worker node running, but with two worker nodes they started successfully.
