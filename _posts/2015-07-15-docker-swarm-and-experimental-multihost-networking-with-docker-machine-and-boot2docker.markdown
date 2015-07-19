---
layout: post
title: "Docker Swarm and experimental multihost networking with docker-machine and boot2docker"
date: "2015-07-15 01:11"
commentIssueId: 7
---

{::options parse_block_html="true" /}
<div class="toc">
Contents

* toc
{:toc}
</div>

## General

After getting the [overlay networking to work with Docker experimental and docker-machine locally using boot2docker]({% post_url 2015-07-14-docker-multihost-networking-using-overlay-driver-with-docker-machine-and-boot2docker %}), I wanted to get Docker Swarm to work with overlay network as described in [Docker's blog](https://github.com/docker/docker/blob/master/experimental/compose_swarm_networking.md).

There were some differences in setting the system up when compared to running it on Digital Ocean like in their example, but with some tweaking the example works.

The idea is that first you create a node for running Consul and then a swarm master and one or multiple swarm nodes that run the containers. As all the nodes are configured to use overlay networking by default, the created containers can directly talk to each other without any extra tweaking.

In this article I follow the steps that are described in Docker's blog and note the differences when running the example locally. I won't be detailing the use of Docker Compose here as it is exactly the same as in the original article.

**Note!** At the time of writing this (15th July 2015) VirtualBox 5.0 doesn't play along with the current docker-machine version (0.3.0). If you have upgraded to VirtualBox 5.0, you need to downgrade to 4.3.28, which (on Mac with brew using the [caskroom/versions tap](https://github.com/caskroom/homebrew-cask/blob/master/USAGE.md#additional-taps-optional)) can be done with

{% highlight bash %}
brew tap caskroom/versions
brew cask uninstall virtualbox
brew cask install virtualbox4328100309
{% endhighlight %}

## Set up the swarm with multi-host networking

### Create the Consul node and start Consul

{% highlight bash %}
docker-machine create -d virtualbox --virtualbox-boot2docker-url=http://sirile.github.io/files/boot2docker-1.8.iso consul
docker $(docker-machine config consul) run -d -p 8500:8500 progrium/consul -server -bootstrap-expect 1
{% endhighlight %}

The boot2docker.iso image that [supports Docker experimental]({% post_url 2015-07-02-using-docker-18-experimental-with-docker-machine-and-virtualbox-driver-boot2docker %}) is downloaded as needed and Consul server is started.

### Create a swarm token

{% highlight bash %}
export SWARM_TOKEN=$(docker $(docker-machine config consul) run swarm create)
{% endhighlight %}

### Create swarm master

{% highlight bash %}
docker-machine create -d virtualbox --virtualbox-boot2docker-url=http://sirile.github.io/files/boot2docker-1.8.iso --engine-opt="default-network=overlay:multihost" --engine-opt="kv-store=consul:$(docker-machine ip consul):8500" --engine-label="com.docker.network.driver.overlay.bind_interface=eth1" swarm-0
{% endhighlight %}

The difference here (apart from running on boot2docker instead of Digital Ocean) is that the bind interface is eth1 instead of eth0. Reason for this is explained [here]({% post_url 2015-07-14-docker-multihost-networking-using-overlay-driver-with-docker-machine-and-boot2docker %}).

### Start swarm manually

As Docker Machine doesn't yet support multi-host networks, Swarm needs to be started manually

{% highlight bash %}
docker $(docker-machine config swarm-0) run -d --restart="always" --net="bridge" swarm:latest join --addr "$(docker-machine ip swarm-0):2376" "token://$SWARM_TOKEN"
docker $(docker-machine config swarm-0) run -d --restart="always" --net="bridge" -p "3376:3376" -v "$HOME/.docker/machine/machines/swarm-0:/etc/docker" swarm:latest manage --tlsverify --tlscacert="/etc/docker/ca.pem" --tlscert="/etc/docker/server.pem" --tlskey="/etc/docker/server-key.pem" -H "tcp://0.0.0.0:3376" --strategy spread "token://$SWARM_TOKEN"
{% endhighlight %}

Another difference is that the certificates are exposed directly from the host. If the location for the certificates is different in the set-up the volume share needs to be changed appropriately.

### Create a Swarm node

{% highlight bash %}
docker-machine create -d virtualbox --virtualbox-boot2docker-url=http://sirile.github.io/files/boot2docker-1.8.iso --engine-opt="default-network=overlay:multihost" --engine-opt="kv-store=consul:$(docker-machine ip consul):8500" --engine-label="com.docker.network.driver.overlay.bind_interface=eth1" --engine-label="com.docker.network.driver.overlay.neighbor_ip=$(docker-machine ip swarm-0)" swarm-1
docker $(docker-machine config swarm-1) run -d --restart="always" --net="bridge" swarm:latest join --addr "$(docker-machine ip swarm-1):2376" "token://$SWARM_TOKEN"
{% endhighlight %}

Here the difference is again the eth1 binding for the overlay network interface. Multiple Swarm nodes can be created the same way, just change the name for the instance.

### Point Docker at the Swarm

{% highlight bash %}
export DOCKER_HOST=tcp://"$(docker-machine ip swarm-0):3376"
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH="$HOME/.docker/machine/machines/swarm-0"
{% endhighlight %}

## Testing connectivity

Now you can proceed with the steps in [the original blog](https://github.com/docker/docker/blob/master/experimental/compose_swarm_networking.md#run-containers-and-get-them-communicating) or if you just want to quickly test that communication between swarm members really works, start a few containers and because of the spread strategy they should be located to different nodes. Run for example

{% highlight bash %}
docker run -it --rm --name test busybox
{% endhighlight %}

and in a second terminal run (remember to set the environment variables)

{% highlight bash %}
docker run -it --rm --name second busybox
{% endhighlight %}

now you should with the command `docker ps -a` (after setting the environment variables correctly) in a third console see something like

{% highlight bash %}
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                                     NAMES
53f322453a6a        busybox             "/bin/sh"              8 seconds ago       Up 8 seconds                                                  swarm-0/second
94c6b5a8b6d2        busybox             "/bin/sh"              33 seconds ago      Up 33 seconds                                                 swarm-1/test
f0a6bbea946b        swarm:latest        "/swarm join --addr    6 minutes ago       Up 6 minutes        2375/tcp                                  swarm-1/grave_hopper
c9fa1d1a6b59        swarm:latest        "/swarm manage --tls   7 minutes ago       Up 7 minutes        2375/tcp, 192.168.99.101:3376->3376/tcp   swarm-0/condescending_payne
43648614825d        swarm:latest        "/swarm join --addr    8 minutes ago       Up 7 minutes        2375/tcp                                  swarm-0/sharp_bhaskara
{% endhighlight %}

as you can see the 'test' and 'second' containers are on different hosts (swarm-1 and swarm-0 respectively).

Now the different containers should be able to ping each other. In 'test' container:

{% highlight bash %}
/ # ping second
PING second (172.21.0.2): 56 data bytes
64 bytes from 172.21.0.2: seq=0 ttl=64 time=0.336 ms
64 bytes from 172.21.0.2: seq=1 ttl=64 time=0.684 ms
{% endhighlight %}

and in 'second' container

{% highlight bash %}
/ # ping test
PING test (172.21.0.1): 56 data bytes
64 bytes from 172.21.0.1: seq=0 ttl=64 time=0.592 ms
64 bytes from 172.21.0.1: seq=1 ttl=64 time=0.761 ms
{% endhighlight %}

## Next steps

Now building and testing a multi-node set-up is possible also locally without needing to use a cloud provider. It might be a good idea to wait for the tooling to mature a bit so that docker-machine can catch up with the changes to the networking model and support building a swarm also with overlay network. Also as Docker 1.8 is released quite a lot of the tweaks should become unnecessary.

Next I'll probably see if I can get Cassandra to run on multiple nodes without the actual instances being aware of the infrastructure set-up.
