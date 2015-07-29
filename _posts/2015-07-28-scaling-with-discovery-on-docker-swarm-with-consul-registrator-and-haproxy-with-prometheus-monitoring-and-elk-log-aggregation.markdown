---
layout: post
title: 'Scaling with discovery on Docker Swarm with Consul, Registrator and HAProxy with Prometheus monitoring and ELK log aggregation'
date: '2015-07-28 13:39'
commentIssueId: 9
---

{::options parse_block_html="true" /}
<div class="toc">
Contents

* toc
{:toc}
</div>

## General

After playing around quite a lot with automatic scaling using [Registrator triggered HAProxy configuration generation]({% post_url 2015-05-18-using-haproxy-and-consul-for-dynamic-service-discovery-on-docker %}) I wanted to combine it with Docker Swarm. At the same time I wanted to see if [log aggregation using ELK-stack]({% post_url 2015-06-26-elasticsearch-logstash-kibana-and-logspout-on-docker %}) (ElasticSearch, LogStash and Kibana) could be made pluggable and add Prometheus based metrics collection as an option.

There are a few frameworks that achieve pretty much the same result, but I wanted to build from scratch to get to know how the components work. The end result is a handful of shell scripts that can be used to set up the Docker Swarm and add or remove logging and monitoring to all the nodes. If new nodes are added, just by running the script again they are joined to the monitoring and log aggregation system.

A three node swarm set-up with logging and monitoring running five instances of a test service can be done with:

{% highlight bash %}
git clone https://github.com/SirIle/docker-multihost.git && cd docker-multihost/swarm
./createInfraNode.sh
for i in {0..2}; do ./createSwarmNode.sh $i; done
./addLogging.sh
./addMonitoring.sh
for i in {1..5}; do ./startService.sh hello/v1 sirile/node-test; done
{% endhighlight %}

This automatically starts a HAProxy which acts as the rest endpoint and directs traffic with the url pattern of /hello/v1 to the application containers. HAProxy itself could be started on any of the swarm nodes. If wanted this can be controlled with labels. The scripts display the public IP address of the endpoint. As Consul is used, joining the local Consul server to the one controlling the swarm is also an option and then finding the application becomes just http://rest.service.consul/hello/v1.

![Overview_of_Swarm](/images/Overview_of_Swarm.png)

### Target architecture

As Docker overlay networking matures I'll move towards target architecture where I have front-end nodes that expose ports outside and application nodes that are only reachable through the overlay networking. At the moment this set-up doesn't yet work and registrator doesn't understand overlay networking services at all so Consul based service discovery can't be used with it. Docker Swarm labels can be used to distinguish different roles and so direct the applications to specific swarm members, but that isn't used in this example as all the nodes can work in all the roles.

![Swarm_target_architecture](/images/Swarm_target_architecture.png)

#### Cloud service providers

Everything should work also when running against a cloud based service providers, only change should be in the _createInfraNode.sh_ and _createSwarmNode.sh_ where instead of using the VirtualBox driver another driver (with the required extra configuration like AWS token) should be used.

### Prerequisites

The following tools need to be installed for the scripts to work. Everything has been tested on Mac, but should work also on Linux. For Windows tweaking would be necessary, but all the tools have been ported, so just tweaking the scripts should be enough.

- VirtualBox (5.0.0)
- Docker Machine (0.3.1)
- Docker (1.7.1)

#### Docker Compose

I tried using Docker Compose to define the nodes, but as it doesn't yet [support](https://github.com/docker/compose/issues/1377) [variable](https://github.com/docker/compose/issues/495) [expansion](https://github.com/docker/compose/issues/64) passing the required information for the services isn't at the moment possible. I may explore it again later as there have been a lot of requests for that functionality.

## Components

The basic architecture consists of an infra node which runs the Consul master. In addition it can run the Prometheus server doing the performance metrics aggregation and the ELK stack for log aggregation and browsing.

Docker Swarm nodes are used to host the applications and also the HAProxy which does load-balancing between different application containers. The first created node (or node id 0) is called swarm-master and the Swarm commands are executed through it. It is the only required server, but more instances can be created and they're automatically joined into the swarm.

There are also a few helpful functions which make finding and controlling containers in the swarm easier.

### Log aggregation

Log aggregation is done by running [ELK-stack](https://www.elastic.co/webinars/introduction-elk-stack) consisting of ElasticSearch, LogStash and Kibana on the infra node and [logspout](https://github.com/gliderlabs/logspout) on the swarm nodes.

### Monitoring

I used the [article at CenturyLink labs](https://labs.ctl.io/monitoring-docker-services-with-prometheus/) as the starting point in adding [Prometheus](http://prometheus.io) as the runtime monitoring information collector. All the swarm nodes have [cAdvisor](https://github.com/google/cadvisor) running which can automatically export the runtime information for Prometheus. Prometheus configuration is re-written if the script is run again and Prometheus is signaled so that it will reload the configuration on the fly.

## Getting the code

The scripts can be checked out with

{% highlight bash %}
git clone https://github.com/SirIle/docker-multihost.git && cd docker-multihost/swarm
{% endhighlight %}

## Creating the Swarm

### Create infra node

Infra node is created with the script

{% highlight bash %}
./createInfraNode.sh
{% endhighlight %}

It first checks if infra node hasn't been already created quitting if it has and then starts the Consul server.

### Create swarm master

Swarm master is the first node and can be created by giving the command a 0 as an argument.

{% highlight bash %}
./createSwarmNode.sh 0
{% endhighlight %}

The command line for starting the Swarm master has the corresponding flag set, but otherwise the node itself is almost the same as for normal nodes. The Consul running on infra is used as the discovery back-end and so the infra node needs to be running.

### Optional: create zero or many swarm members

More swarm members can be created at any time by running the same script with a different id

{% highlight bash %}
./createSwarmNode.sh 1
./createSwarmNode.sh 2
{% endhighlight %}

The started nodes automatically start Registrator and Consul and join the Consul running on infra node.

### Start service(s)

I created two simple containers that can be used to demonstrate scaling called [sirile/node-test](https://github.com/SirIle/node-test) and [sirile/scala-boot-test](https://github.com/SirIle/scala-boot-test). They can both be found from the Docker Hub, so there is no need to build them locally. Node-test is 23.31 MB and scala-boot-test 191 MB (as the JRE is rather large) so downloading them shouldn't take too long. They both output the hostname, current timestamp and the implementation language, Javascript and Node respectively.

Any service can be used as long as it exposes the service through port 80 so that registrator picks it up and HAProxy configuration is written correctly. The real port that is visible outside is controlled by Docker and HAProxy can then direct traffic there based on the given url identifier.

When a service is started using the script it checks that at least one HAProxy container is running and if not, starts it and shows the address of the HAProxy through which the service is available.

{% highlight bash %}
$ ./startService.sh hello/v1 sirile/node-test
** Starting image sirile/node-test with the name hello/v1 **
a1aaa0691a548aa9cc6db024537f83dc0c2f08d7344f1e1e41a69dc28b91db5f
** Service available at http://192.168.99.102/hello/v1 **
{% endhighlight %}

#### Using Consul for service discovery in the containers

Consul has been specified as the default DNS server for the containers. Registered services can be looked up normally, so after starting a container you can do this (based on the previous example):

{% highlight bash %}
$ docker exec -it 16f25b75b4fdee72be6d992c4c7f39f01604c2b4cd5c2a062b0779d031f403f0 ping consul.service.consul
PING consul.service.consul (192.168.99.100): 56 data bytes
64 bytes from 192.168.99.100: seq=0 ttl=63 time=0.208 ms
64 bytes from 192.168.99.100: seq=1 ttl=63 time=0.434 ms
{% endhighlight %}


### Add log aggregation

ELK based log aggregation system can be added with the script

{% highlight bash %}
$ ./addLogging.sh
** Starting LogBox and Kibana on infra **
decc99d53c5033f1758b4b63973daff3b63e1a3923454647ba662fd86ea37d08
c5e9f612143a84550921fafbf6b7a122451f3463ee852c66268eed7881bfeb2f
** Servers in the swarm: swarm-app-1 swarm-master **
** Starting logspout on swarm-app-1
2ca445fd69c85383ea2485004e3ad0584e94910281981dcbfc8f7b7788e48fc0
** Starting logspout on swarm-master
ad5adf8a329eb4243f87652df8da04f722bfec9fa2ee96062ba6f2a773339b2e
** Logging system started, Kibana is available at http://192.168.99.100:5601 **
{% endhighlight %}

This starts the LogBox container on infra node which consists of LogStash and ElasticSearch. It also starts Kibana that acts as the UI and tells the address where it can be found, for example http://192.168.99.100:5601.

LogSpout is started on all swarm nodes and the log traffic is directed to LogStash running on infra node. If the script is run again, it checks if the required containers are running on all servers and starts them as needed, for example after a new swarm node has been added.

#### Stopping the log aggregation

LogBox and all the LogSpout instances can be removed with the script

{% highlight bash %}
./rmLogging.sh
{% endhighlight %}

### Add monitoring

Prometheus based monitoring and the corresponding cAdvisor containers can be started with

{% highlight bash %}
$ ./addMonitoring.sh
** Servers in the swarm: swarm-app-1 swarm-master **
** Starting cAdvisor on swarm-app-1
3f6a32670668d0a54d91df02d6107fc4a8225c3fb2f4637045622130a3b1ed83
** Starting cAdvisor on swarm-master
8a379f5385b9cc9b99d8eadf473bc647db5c313f1df17e1536199b193f0a668d
** Starting Prometheus on infra **
aff2bb500671d7b81d91331845abdb61466f4638888cc552c299869a728a3010
Prometheus ui can be found at http://192.168.99.100:9090
{% endhighlight %}

Then the cAdvisor instances can be directly accessed from port 8080, for example http://192.168.99.101:8080 and the Prometheus UI can be accessed from the infra server at port 9090, for example http://192.168.99.100:9090.

## Using Consul for local service discovery

One of the functions in _docker-functions.sh_ can also return an external IP for a container running a given image. Remember to point the docker client to swarm master.

{% highlight bash %}
$ source docker-functions.sh
$ dock --swarm swarm-master
$ dockip rest
192.168.99.102
{% endhighlight %}

Things can be made even more simple by joining a local Consul to the Consul that is running on infra node and using it as the local DNS server.

### Setting up resolver configuration

OS X can use resolver to enable Consul DNS based service discovery locally. To set it up (following this [blog](http://passingcuriosity.com/2013/dnsmasq-dev-osx/)) the following steps are needed:

Create directory /etc/resolver

{% highlight bash %}
sudo mkdir -p /etc/resolver
{% endhighlight %}

Create a file called `/etc/resolver/consul` with the contents:

{% highlight bash %}
nameserver 127.0.0.1
port 8600
{% endhighlight %}

Now the domain .consul is resolved with the DNS server running locally on port 8600 which is the default port for Consul DNS.

### Starting a local Consul and joining it to infra Consul

First you need to install Consul locally, which can be done through [brew](http://brew.sh) with `brew install consul`. Then you can start a local Consul instance that joins to the Consul server running on infra node with

{% highlight bash %}
consul agent -data-dir=/tmp/consul -join $(docker-machine ip infra)
{% endhighlight %}

### Testing the discovery

Now the HAProxy based endpoint should we reachable at http://rest.service.consul and the example service should reply

{% highlight bash %}
$ curl rest.service.consul/hello/v1
{"hostname":"e736179c1932","time":"2015-07-28T10:07:05.433Z","language":"javascript"}
{% endhighlight %}

## Scripts and files

In this section I'll explain the different scripts quickly. The directory looks like

{% highlight bash %}
$ ls -lF
total 72
-rwxr--r--  1 ilkka.anttonen  562225435  1449 Jul 27 14:56 addLogging.sh*
-rwxr--r--  1 ilkka.anttonen  562225435  2220 Jul 28 10:23 addMonitoring.sh*
-rwxr--r--  1 ilkka.anttonen  562225435   727 Jul 28 10:41 createInfraNode.sh*
-rwxr--r--  1 ilkka.anttonen  562225435  1841 Jul 27 13:55 createSwarmNode.sh*
-rwxr--r--  1 ilkka.anttonen  562225435  1228 Jul 27 14:54 docker-functions.sh*
-rw-r--r--  1 ilkka.anttonen  562225435   984 Jul 28 10:23 prometheus.yml
-rwxr--r--  1 ilkka.anttonen  562225435   286 Jul 27 14:24 rmLogging.sh*
-rwxr--r--  1 ilkka.anttonen  562225435   286 Jul 27 15:30 rmMonitoring.sh*
-rwxr--r--  1 ilkka.anttonen  562225435   805 Jul 27 14:17 startService.sh*
{% endhighlight %}

### docker-functions.sh

This file contains functions that make life with containers in a swarm easier. The functions are also used in the other scripts to prevent duplication of code. Easiest way is to run `source docker-functions.sh`.

#### dock

This is a shortcut that is used to point the docker command line client to a given docker-machine controlled node. For example to point to infra the command is `dock infra` and to point to swarm-master the command is `dock --swarm swarm-master`.

#### dockpsi

This function returns ps information for all containers running a given image. For example to query all the containers on a swarm that run an image containing the string 'node':

{% highlight bash %}
$ dockpsi node
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                          NAMES
f495c4fbe5b8        sirile/node-test    "node /app.js"      2 hours ago         Up 2 hours          192.168.99.102:32778->80/tcp   swarm-app-1/dreamy_euclid
e8d148835aab        sirile/node-test    "node /app.js"      2 hours ago         Up 2 hours          192.168.99.101:32777->80/tcp   swarm-master/suspicious_yalow
86cb5b75ef42        sirile/node-test    "node /app.js"      2 hours ago         Up 2 hours          192.168.99.102:32777->80/tcp   swarm-app-1/cranky_feynman
50f9e99d8144        sirile/node-test    "node /app.js"      2 hours ago         Up 2 hours          192.168.99.101:32776->80/tcp   swarm-master/insane_lalande
f8942be4f752        sirile/node-test    "node /app.js"      2 hours ago         Up 2 hours          192.168.99.102:32776->80/tcp   swarm-app-1/pensive_ptolemy
{% endhighlight %}

#### dockrm

This function removes all the instances of a given image. For example if the image is sirile/node-test you can run `dockrm node` to remove all running containers that run that image.

### addLogging.sh and addMonitoring.sh

These scripts check that the infra node is running, then start the main containers there. LogBox for logging and Prometheus for monitoring. Then they loop through the swarm nodes (with a bit of awk and xargs trickery) and start the required services there if they are not already running. In the case of _addMonitoring.sh_, it also rewrites the Prometheus configuration file _prometheus.yml_ and sends a SIGHUP to the running Prometheus container if it exists so that the configuration is reloaded.

### createInfraNode.sh and createSwarmNode.sh

These scripts create the infra node and swarm nodes (including the master) respectively. The _createSwarmNode.sh_ checks that infra node is running and if it isn't it errors out. It also checks that swarm-master has been created if application node is being created.

### rmLogging.sh and rmMonitoring.sh

These scripts remove containers that take care of logging or monitoring.

### startService.sh

This script is used to start services. It checks that at least one instance of HAProxy image is running and starts it if needed.
