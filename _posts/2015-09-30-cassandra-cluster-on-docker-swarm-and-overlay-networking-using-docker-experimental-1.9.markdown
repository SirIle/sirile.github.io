---
layout: post
title: Cassandra cluster on Docker Swarm and Overlay Networking using Docker Experimental 1.9
date: '2015-09-30 02:26'
commentIssueId: 11
---

<!--lint disable -->
{::options parse_block_html="true" /}
<div class="toc">
Contents

<!--lint disable -->
* toc
{:toc}
</div>

## General

In an [earlier blog]({% post_url
2015-07-15-docker-swarm-and-experimental-multihost-networking-with-docker-machine-and-boot2docker %})
I demonstrated how a local boot2docker based Docker experimental using overlay
networking on Docker Swarm can be set up. In this article I show how a Cassandra
cluster can be set-up on top of the overlay network.

A boot2docker image containing the experimental version is used. Information
about building it can be found [here]({% post_url
2015-08-27-using-docker-19-experimental-with-docker-machine-and-virtualbox-driver-boot2docker %}).

## Overview of the Architecture

The set-up consists of an infra-node which contains the Consul server used as
the back-end for overlay networking. As the overlay networking is still in an
experimental stage it has some issues. It for example breaks the outside
connectivity of the containers. This prevents building of images. For this
reason infra-node is used for building.

To have a shorter development feedback cycle, a private registry is set up on
the infra node so that the swarm members can get their images from there instead
of having to use Docker Hub for building and distributing the images.

In addition to the infra-node a swarm master and another swarm-node are created.
Adding more swarm nodes is trivial.

### The Cassandra image

The Cassandra image used is almost uncustomized and can be found from
[GitHub](https://github.com/SirIle/miniboxes/tree/master/minicassandra). A
startup-script has been added which modifies the cassandra.yaml so that the seed
list is changed to include a value that is given as part of the startup command.
If the value is omitted it defaults to 127.0.0.1. Also the listen address is
cleared so that the hostname is used instead of localhost so that the cluster
can work. RPC connections are allowed from everywhere. By default the Cassandra
image uses quite a lot of memory, so the amount of memory can throttled using
Docker options.

## Setting up the environment

### Downloading the experimental boot2docker-image and setting it as the default

As downloading the image every time takes considerable time, it's easier to
download the image once and set it as the default for docker-machine. (Thanks
JoyceBabu for the tip!).

~~~bash
curl -L http://sirile.github.io/files/boot2docker-1.9.iso > $HOME/.docker/machine/cache/boot2docker-1.9.iso
export VIRTUALBOX_BOOT2DOCKER_URL=file://$HOME/.docker/machine/cache/boot2docker-1.9.iso
~~~

Afterwards unsetting the variable is enough to revert to default image.

### Creating the infra node

Infra node contains the private registry and Consul. The IP is given so that the
locally built images can be pushed to the private registry without setting up
TLS. If the created node gets another IP from the VirtualBox DHCP server it may
be easiest to reset the DHCP server (I disabled and re-enabled it from the
VirtualBox GUI) and restart the machine. Otherwise getting the private registry
to work may require quite a lot of tinkering.

#### Create the node using Docker Machine

~~~bash
docker-machine create --driver virtualbox  --virtualbox-memory 2048  --engine-insecure-registry 192.168.99.100:5000 infra
~~~

#### Start the private registry

~~~bash
docker $(docker-machine config infra) run -d -p 5000:5000 --restart=always --name registry registry:2
~~~

#### Start Consul

~~~bash
docker $(docker-machine config infra) run -d -p 8500:8500 progrium/consul -server -bootstrap-expect 1
~~~

#### Create the Swarm token

~~~bash
export SWARM_TOKEN=$(docker $(docker-machine config infra) run swarm create)
~~~

### Creating the Swarm master (swarm-0)

#### Create the node using Docker Machine

~~~bash
docker-machine create -d virtualbox  --engine-opt="default-network=overlay:multihost" --engine-opt="kv-store=consul:$(docker-machine ip infra):8500" --engine-label="com.docker.network.driver.overlay.bind_interface=eth1" --engine-insecure-registry $(docker-machine ip infra):5000 swarm-0
~~~

#### Start swarm

~~~bash
docker $(docker-machine config swarm-0) run -d --restart="always" --net="bridge" swarm:latest join --addr "$(docker-machine ip swarm-0):2376" "token://$SWARM_TOKEN"
docker $(docker-machine config swarm-0) run -d --restart="always" --net="bridge" -p "3376:3376" -v "$HOME/.docker/machine/machines/swarm-0:/etc/docker" swarm:latest manage --tlsverify --tlscacert="/etc/docker/ca.pem" --tlscert="/etc/docker/server.pem" --tlskey="/etc/docker/server-key.pem" -H "tcp://0.0.0.0:3376" --strategy spread "token://$SWARM_TOKEN"
~~~

### Creating the swarm node (swarm-1)

This step can be repeated and more nodes can be created by just changing the
machine name.

#### Create the node using Docker Machine

~~~bash
docker-machine create -d virtualbox  --engine-opt="default-network=overlay:multihost" --engine-opt="kv-store=consul:$(docker-machine ip infra):8500" --engine-label="com.docker.network.driver.overlay.bind_interface=eth1" --engine-label="com.docker.network.driver.overlay.neighbor_ip=$(docker-machine ip swarm-0)" --engine-insecure-registry $(docker-machine ip infra):5000 swarm-1
~~~

#### Start the swarm agent

~~~bash
docker $(docker-machine config swarm-1) run -d --restart="always" --net="bridge" swarm:latest join --addr "$(docker-machine ip swarm-1):2376" "token://$SWARM_TOKEN"
~~~

### Getting the Cassandra image to local registry

If you want to build the image it can be found from
[here](https://github.com/SirIle/miniboxes/tree/master/minicassandra). Otherwise
you can pull the image from Dockerhub and push it to local registry.

~~~bash
docker $(docker-machine config infra) pull sirile/minicassandra
docker $(docker-machine config infra) tag sirile/minicassandra $(docker-machine ip infra):5000/cass
docker $(docker-machine config infra) push $(docker-machine ip infra):5000/cass
~~~

### Starting the images

#### Point docker client to swarm master

~~~bash
export DOCKER_HOST=tcp://"$(docker-machine ip swarm-0):3376"
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH="$HOME/.docker/machine/machines/swarm-0"
~~~

#### Start a few Cassandra instances

The last parameter is the seed list that is substituted to the _cassandra.yaml_
configuration file. Allow for some time between starting the instances so that
the cluster builds up correctly. The instance logs can be read using the normal
`docker logs` command, for example `docker logs cass1`.

~~~bash
docker run -d --name cass1 192.168.99.100:5000/cass cass1
docker run -d --name cass2 192.168.99.100:5000/cass cass1
docker run -d --name cass3 192.168.99.100:5000/cass cass1
~~~

## Testing the set-up

### Show how the instances are spread between nodes

~~~bash
$ docker ps
CONTAINER ID        IMAGE                      COMMAND             CREATED              STATUS              PORTS               NAMES
eab9a8726ca9        192.168.99.100:5000/cass   "/start.sh cass1"   48 seconds ago       Up 48 seconds                           swarm-1/cass3
84191d24d9a1        192.168.99.100:5000/cass   "/start.sh cass1"   About a minute ago   Up About a minute                       swarm-0/cass2
d800115d147f        192.168.99.100:5000/cass   "/start.sh cass1"   2 minutes ago        Up 2 minutes                            swarm-1/cass1
~~~

### Checking the ring

~~~bash
$ docker exec cass1 /cassandra/bin/nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens  Owns (effective)  Host ID                               Rack
UN  172.21.0.1  51.61 KB   256     66.4%             ad94770f-0119-427b-a603-1fb278fbad46  rack1
UN  172.21.0.3  66.49 KB   256     63.2%             708e5394-4e2f-4c48-9254-cbb8e1d6ed90  rack1
UN  172.21.0.2  66.05 KB   256     70.4%             6eb5464b-771f-4242-913f-d399f05411d2  rack1
~~~

### Starting cqlsh

~~~bash
$ docker exec -it cass1 /cassandra/bin/cqlsh
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 2.1.9 | CQL spec 3.2.0 | Native protocol v3]
Use HELP for help.
cqlsh>
~~~

## Final thoughts

Everything seems to work as expected. The port for connecting from outside of
the swarm can be exposed with the normal Docker procedures of defining -p on one
of the instances.
