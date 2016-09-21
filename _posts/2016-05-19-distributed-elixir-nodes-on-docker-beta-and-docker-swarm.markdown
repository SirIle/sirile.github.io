---
layout: "post"
title: "Distributed Elixir nodes on Docker Beta and Docker Swarm"
date: "2016-05-19 01:12"
commentIssueId: 15
---

{::options parse_block_html="true" /}
<div class="toc">
Contents

<!--lint disable -->
* toc
{:toc}
</div>

## General

In this post I'll take a look into running distributed Elixir nodes both on
Docker Beta and Docker Swarm. When I started trying out Elixir I quickly ran
into [this article by Jon
Harrington](http://blog.jonharrington.org/elixir-and-docker/) about running
Elixir on Docker and also found out that Marlus Saraiva had done an excellent
job of [minimizing the Elixir
containers](https://github.com/msaraiva/alpine-erlang) so that they run on
Alpine.

The examples in the first article were a bit dated and more cumbersome than they
would need to be with the modern Docker networking and also instructed about
creating Elixir images which were based on Ubuntu. This post combines the
examples of Jon's article with the networking redone and with the examples
running on the Marlus' minimized images.

## Running Elixir nodes on Docker Beta

If you are part of the Docker Beta program on either Mac or Windows, you can run
Elixir nodes easily on the embedded Docker engine. For connectivity it's easiest
to create a network and join the Elixir nodes to that network so that they can
directly call each other.

~~~bash
docker network create elixir
~~~

Then start an Elixir node called _socrates_. For networking to work it's easiest
to use fully qualified domain names, in this case _socrates.local_

~~~bash
docker run -it --name socrates.local --net elixir --rm msaraiva/elixir iex --name socrates@socrates.local --cookie monster
~~~

And in another terminal start a second node called _plato_

~~~bash
docker run -it --name plato.local --net elixir --rm msaraiva/elixir iex --name plato@plato.local --cookie monster
~~~

As there are no environment variables set, the docker client points to the local
beta-instance. Now the nodes should be able to ping each other. In the terminal
window for _socrates_ you can run

~~~bash
iex(socrates@socrates.local)1> Node.ping(:"plato@plato.local")
:pong
~~~

and of course from _plato_ you can ping _socrates_

~~~bash
iex(plato@plato.local)1> Node.ping(:"socrates@socrates.local")
:pong
~~~

If you're using boot2docker, the commands are the same, just point the docker
client to the boot2docker instance.

## Setting up the Swarm with overlay networking

In these examples I'll use a locally created Swarm. Changing the commands so
that the Swarm runs in for example AWS instead is easy and can be done by
following the instructions in [this earlier blog]({% post_url
2015-11-25-getting-overlay-networking-to-work-in-aws-with-docker-19 %}).

### Create Infra node with Consul

~~~bash
docker-machine create -d virtualbox infra
docker $(docker-machine config infra) run -d -p "8500:8500" -h consul progrium/consul -server -bootstrap
~~~

### Create swarm-0 (Swarm master) and the overlay network

~~~bash
docker-machine create -d virtualbox --swarm --swarm-master --swarm-discovery="consul://$(docker-machine ip infra):8500" --engine-opt="cluster-store=consul://$(docker-machine ip infra):8500" --engine-opt="cluster-advertise=eth1:2376" swarm-0
docker $(docker-machine config swarm-0) network create --driver overlay elixir
~~~

### (Optionally create more Swarm nodes)

~~~bash
docker-machine create -d virtualbox --swarm --swarm-discovery="consul://$(docker-machine ip infra):8500" --engine-opt="cluster-store=consul://$(docker-machine ip infra):8500" --engine-opt="cluster-advertise=eth1:2376" swarm-1
~~~

If you're creating more Swarm nodes, just change the node name to be unique in
the Swarm.

### Target the Docker client to the Swarm master

~~~bash
eval $(docker-machine env --swarm swarm-0)
~~~

## Two Elixir nodes pinging each other over the overlay network

Open up two terminal windows. In the first window start the first Elixir node
called _socrates_. The commands are exactly the same as when running on the
local Docker Beta. Remember to target the client to the Swarm master in all the
terminals or point the client directly to Swarm master with the config
parameter.

~~~bash
docker run -it --name socrates.local --rm --net elixir msaraiva/elixir iex --name socrates@socrates.local --cookie monster
~~~

In the second window start the node called _plato_

~~~bash
docker $(docker-machine config --swarm swarm-0) run -it --name plato.local --rm --net elixir msaraiva/elixir iex --name plato@plato.local --cookie monster
~~~

Now from _socrates_ you can ping _plato_

~~~bash
iex(socrates@socrates.local)1> Node.ping(:"plato@plato.local")
:pong
~~~

and of course again from _plato_ you can ping _socrates_

~~~bash
iex(plato@plato.local)1> Node.ping(:"socrates@socrates.local")
:pong
~~~

and to show that they are indeed on different Docker nodes you can run

~~~bash
$ docker $(docker-machine config --swarm swarm-0) ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS               NAMES
73ff85734c36        msaraiva/elixir     "iex --name plato@pla"   43 seconds ago       Up 42 seconds                           swarm-0/plato.local
e6213947adca        msaraiva/elixir     "iex --name socrates@"   About a minute ago   Up About a minute                       swarm-1/socrates.local
~~~

in this case plato is on swarm-0 and socrates is on swarm-1.

## Creating PG2 group and joining the distributes nodes

In [Jon's article](http://blog.jonharrington.org/elixir-and-docker/) he gives an
example about joining three Elixir nodes to a process group (PG2). The example
uses nodes calles _plato_, _socrates_ and _aristotle_. If all the nodes share
the same group and have been introduced to each other, then they are a part of
the same process group. In three separate terminals start the three nodes

~~~bash
# Terminal 1
docker $(docker-machine config --swarm swarm-0) run -it --name socrates.local --rm --net elixir msaraiva/elixir iex --name socrates@socrates.local --cookie monster
# Terminal 2
docker $(docker-machine config --swarm swarm-0) run -it --name plato.local --rm --net elixir msaraiva/elixir iex --name plato@plato.local --cookie monster
# Terminal 3
docker $(docker-machine config --swarm swarm-0) run -it --name aristotle.local --rm --net elixir msaraiva/elixir iex --name aristotle@aristotle.local --cookie monster
~~~

Then in _socrates_ create the process group

~~~bash
iex(socrates@socrates.local)1> :pg2.start()
{:ok, #PID<0.65.0>}
iex(socrates@socrates.local)2> :pg2.create(:group)
:ok
iex(socrates@socrates.local)3> :pg2.join(:group, self)
:ok
iex(socrates@socrates.local)4> :pg2.get_members(:group)
[#PID<0.63.0>]
~~~

Now we have created the process group and added ourself to it. Now in _plato_

~~~bash
iex(plato@plato.local)1> :pg2.start()
{:ok, #PID<0.65.0>}
iex(plato@plato.local)2> :pg2.create(:group)
:ok
iex(plato@plato.local)3> :pg2.join(:group, self)
:ok
iex(plato@plato.local)4> :pg2.get_members(:group)
[#PID<0.63.0>]
~~~

We created the same group, but now the nodes need to be introduced to each
other. Let's ping from _socrates_

~~~bash
iex(socrates@socrates.local)5> Node.ping(:"plato@plato.local")
:pong
iex(socrates@socrates.local)6> :pg2.get_members(:group)
[#PID<9068.63.0>, #PID<0.63.0>]
~~~

Both nodes have joined. Finally, in _aristotle_ type

~~~bash
iex(aristotle@aristotle.local)1> :pg2.start()
{:ok, #PID<0.65.0>}
iex(aristotle@aristotle.local)2> :pg2.create(:group)
:ok
iex(aristotle@aristotle.local)3> :pg2.join(:group, self)
:ok
iex(aristotle@aristotle.local)4> Node.ping(:"plato@plato.local")
:pong
iex(aristotle@aristotle.local)5> :pg2.get_members(:group)
[#PID<0.63.0>, #PID<9065.63.0>, #PID<9118.63.0>]
~~~

You can see all the nodes have joined the same process group. For more examples
refer to [Jon's article](http://blog.jonharrington.org/elixir-and-docker/).

## Conclusion

In this article I demonstrated how to easily run Elixir nodes on Docker Beta and
Docker Swarm. When running on Swarm the Elixir nodes can be distributed over
multiple machines and easily ran on different clouds achieving practically
limitless scalability. Compared to the earlier articles there is now no need to
figure out the IP addresses of the nodes as Docker networking supports DNS based
node discovery and using the minimized images makes testing and development
quick. Have fun experimenting and implementing with Elixir!
