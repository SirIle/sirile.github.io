---
layout: post
title: "Docker multihost networking using overlay driver with docker-machine and boot2docker"
date: "2015-07-14 14:15"
commentIssueId: 6
---

{::options parse_block_html="true" /}
<div class="toc">
Contents

* toc
{:toc}
</div>

## General

After getting [boot2docker to work with the Docker experimental version]({% post_url 2015-07-02-using-docker-18-experimental-with-docker-machine-and-virtualbox-driver-boot2docker %}) I wanted to try overlay networking between different hosts. The [blog posts on Docker repository](https://github.com/docker/docker/blob/master/experimental/compose_swarm_networking.md) are getting better, but still I encountered a few issues before getting things to work locally using boot2docker instead of the Digital Ocean they use as example.

This article expects that the Docker experimental command line version has been installed locally.

**Note!** At the time of writing this (15th July 2015) VirtualBox 5.0 doesn't play along with the current docker-machine version (0.3.0). If you have upgraded to VirtualBox 5.0, you need to downgrade to 4.3.30, which (on Mac with brew using the [caskroom/versions tap](https://github.com/caskroom/homebrew-cask/blob/master/USAGE.md#additional-taps-optional)) can be done with

{% highlight bash %}
brew tap caskroom/versions
brew cask uninstall virtualbox
brew cask install virtualbox4330101610
{% endhighlight %}

## Infrastructure node

As the overlay driver needs an external key/value datastore and I have anyway settled on a pattern with infrastructure containers, setting a standalone node up with Consul is done first.

### Creating the infra-node with docker-machine

Infra node that is compatible with the docker experimental command line utility can be created with

{% highlight bash %}
docker-machine create -d virtualbox --virtualbox-boot2docker-url=http://sirile.github.io/files/boot2docker-1.8.iso infra
{% endhighlight %}

This downloads a version of boot2docker.iso which has the experimental Docker running.

### Starting services

I've been taking a second look at docker-compose. The Consul service for infra node can be started with the following _docker-compose.yml_

{% highlight yaml %}
consul:
  image: progrium/consul
  ports:
    - "8500:8500"
    - "172.17.42.1:53:53"
    - "172.17.42.1:53:53/udp"
  command: --server -bootstrap-expect 1
  dns: 172.17.42.1

{% endhighlight %}

using the commands

{% highlight bash %}
eval "$(docker-machine env infra)"
docker-compose up -d
{% endhighlight %}

another way to start just consul on infra node is to run

{% highlight bash %}
eval "$(docker-machine env infra)"
docker run -d -p 8500:8500 progrium/consul --server -bootstrap-expect 1
{% endhighlight %}

The docker-compose file sets the Consul as the local DNS server for itself, which isn't necessary for the networking to function.

## First application node

After consul is running on the infrastructure node the first application node can be created. The biggest difference between the Docker blog posts describing the creation of the node is the order of the network interfaces between boot2docker and the Digital Ocean instance they use. With boot2docker the eth0 that is used in the example is tied to the Docker internal IP which doesn't work across nodes. Instead the overlay driver should be tied to eth1. This took a bit of figuring out as the only error was that the second node couldn't connect to the serf service provided by the first node as it was only listening to the internal IP address of 10.0.2.15.

### Creating the app0 node with docker-machine

{% highlight bash %}
docker-machine create -d virtualbox --virtualbox-boot2docker-url=http://sirile.github.io/files/boot2docker-1.8.iso --engine-opt="default-network=overlay:multihost" --engine-opt="kv-store=consul:$(docker-machine ip infra):8500" --engine-label="com.docker.network.driver.overlay.bind_interface=eth1" app0
{% endhighlight %}

### Starting a container on first node

{% highlight bash %}
eval "$(docker-machine env app0)"
docker run --rm -it --name test busybox
{% endhighlight %}

## Second application node

The only notable difference on creating the second (and subsequent) node is the parameter _com.docker.network.driver.overlay.neighbor_ip_ which tells the overlay driver to connect to an existing node in the network.

{% highlight bash %}
docker-machine create -d virtualbox --virtualbox-boot2docker-url=http://sirile.github.io/files/boot2docker-1.8.iso --engine-opt="default-network=overlay:multihost" --engine-opt="kv-store=consul:$(docker-machine ip infra):8500" --engine-label="com.docker.network.driver.overlay.bind_interface=eth1" --engine-label="com.docker.network.driver.overlay.neighbor_ip=$(docker-machine ip app0)" app1
{% endhighlight %}

### Starting a container on the second node

After the second node has been brought up, a container can be started there

{% highlight bash %}
eval "$(docker-machine env app1)"
docker run --rm -it --name second busybox
{% endhighlight %}

## Testing the connectivity

Now you should be able to ping the containers across the nodes from each other. On node app0

{% highlight bash %}
$ ping second
PING second (172.21.0.2): 56 data bytes
64 bytes from 172.21.0.2: seq=0 ttl=64 time=0.700 ms
64 bytes from 172.21.0.2: seq=1 ttl=64 time=0.493 ms
{% endhighlight %}

and on node app1

{% highlight bash %}
$ ping test
PING test (172.21.0.1): 56 data bytes
64 bytes from 172.21.0.1: seq=0 ttl=64 time=0.443 ms
64 bytes from 172.21.0.1: seq=1 ttl=64 time=0.896 ms
{% endhighlight %}

## Next steps

Next I'll try to replicate the whole docker swarm example on top of local boot2docker based environment and after that create an agnostic Cassandra cluster across nodes without need for eloquent startup scripts so that it should adapt and scale with no extra work.
