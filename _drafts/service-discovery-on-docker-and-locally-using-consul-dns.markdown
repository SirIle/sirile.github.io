---
layout: post
title: "Service discovery on Docker and locally using Consul DNS"
date: "2015-04-13 00:35"
---


{::options parse_block_html="true" /}
<div class="toc">
Contents

* toc
{:toc}
</div>

## General

Using DNS for service discovery is extremely simple. It is well suited to finding ..

This article describes how to set up an environment where the local development environment can seamlessly connect to services running in containers which can be located either locally or in the cloud.

_Insert picture here_

With containers on Docker providing the DNS parameter is trivial and Consul can be used as the DNS server. On the development machine things are a bit trickier, but at least OS X supports the resolv mechanism which started me thinking about using either a Consul instance running on the Docker node or a local Consul instance as the DNS server to provide service discovery. A quick comparison of the two possibilities was as follows:

<table>
  <tr><th>Consul on local machine</th><th>Consul on Docker node</th></tr>
  <tr>
    <td><b>Pros</b>
      <ul>
        <li>Easy to set up</li>
        <li>Only privileged action is creating resolv configuration</li>
        <li>No need to run Consul in privileged mode as port can be anything</li>
        <li>No need to change configuration if other nodes change IPs</li>
        <li>By joining local Consul to other Consul networks dev, test etc can be supported</li>
      </ul>
    </td><td><b>Pros</b>
      <ul>
        <li>No need to set up local Consul</li>
      </ul>
    </td>
  </tr><tr>
    <td><b>Cons</b>
      <ul>
        <li>A local Consul needs to be run</li>
        <li>By default docker-consul by Progrium doesn't expose DNS outside of the node</li>
      </ul>
    </td><td><b>Cons</b>
      <ul>
        <li>Configuration needs to be changed if IP of the node changes</li>
      </ul>
    </td>
  </tr>
</table><p/>

This leads me to favor option 1 and setting up Consul locally. I'm using Mac with Yosemite and [brew](http://brew.sh/) as the package manager, but everything should be doable also on Windows. Resolv configuration may need extra steps and maybe the installation of a local DNS service to do the routing of the DNS queries.

## Setting up the Docker node

### Prerequisites

Following tools should be installed to make development easier. The versions I used are in the brackets.

- Docker Machine (0.2.0)
- Docker client (1.6)
- VirtualBox (4.3.26)

On OS X side I used brew and brew cask to install the packages, in Windows I just downloaded the installation packets and installed manually.

### Starting the node

- Create the Docker Machine node

{% highlight bash %}
docker-machine create -d virtualbox dev
{% endhighlight %}

- Start Consul

{% highlight bash %}
docker run --name consul -d -h dev -p `docker-machine ip`:8300:8300 -p `docker-machine ip`:8301:8301 -p `docker-machine ip`:8301:8301/udp -p `docker-machine ip`:8302:8302 -p `docker-machine ip`:8302:8302/udp -p `docker-machine ip`:8400:8400 -p `docker-machine ip`:8500:8500 -p 172.17.42.1:53:53 -p 172.17.42.1:53:53/udp progrium/consul -server -advertise `docker-machine ip` -bootstrap-expect 1
{% endhighlight %}

- Start Registrator

{% highlight bash %}
docker run -d -v /var/run/docker.sock:/tmp/docker.sock -h dev gliderlabs/registrator consul://`docker-machine ip`:8500
{% endhighlight %}

- Start the WS HAProxy

{% highlight bash %}
docker run -d -e SERVICE_NAME=rest --dns 172.17.42.1 -p 80:80 sirile/haproxy
{% endhighlight %}

consul-template -config=/tmp/haproxy.json -dry -once
haproxy -f /etc/haproxy/haproxy.cfg

- Start the service(s)

{% highlight bash %}
docker run -d -e SERVICE_NAME=hello/v1 -e SERVICE_TAGS=rest -h hello1 --name hello1 -p :80 sirile/scala-boot-test
{% endhighlight %}

## Setting up the local side

### Prerequisites

- Consul

### Setting up resolver configuration

Following the [blog](http://passingcuriosity.com/2013/dnsmasq-dev-osx/).

Create directory /etc/resolver

{% highlight bash %}
sudo mkdir -p /etc/resolver
{% endhighlight %}

Create a file called `/etc/resolver/consul` with the contents:

{% highlight bash %}
nameserver 127.0.0.1
port 8600
{% endhighlight %}

### Starting local Consul and joining it to the Docker node

Start local Consul and join it to the one running on Docker Machine

{% highlight bash %}
consul agent -server -join `docker-machine ip` -data-dir /tmp/consul
{% endhighlight %}

## Profit

- Check that ping and browser work as expected

ping rest.service.consul
curl rest.service.consul/hello/v1
