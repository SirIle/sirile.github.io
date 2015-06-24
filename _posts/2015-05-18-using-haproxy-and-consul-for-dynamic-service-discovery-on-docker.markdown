---
layout: post
title: Using HAProxy and Consul for dynamic service discovery on Docker
date: '2015-05-18 14:08'
commentIssueId: 1
---

{::options parse_block_html="true" /}
<div class="toc">
Contents

* toc
{:toc}
</div>

## General

As I have been using Docker more I have started going towards simpler, stand alone containers. This aligns well with the current rise of microservices. To effectively use microservices a mechanism for service registration and discovery is needed so that the benefits of easy scaling can be realized.

I try to keep things as portable as possible, so no proprietary tools are used. I leverage heavily the excellent work done by Jeff Lindsay ([progrium](http://progrium.com/blog/)).

This will start a series of posts where I look at creating a Docker based runtime environment with dynamic service registration and discovery with support for automatic vertical and horizontal scaling. As I'm using Docker Machine, the environment can as easily be a local VirtualBox based one or reside on Amazon Web Services, Google Compute Engine, Microsoft Azure or anywhere Docker Machine supports.

### Source code and pre-built images

Code for the HAProxy image can be found [here](https://github.com/SirIle/miniboxes). Code for the scala-boot-test which is used as the example application can be found [here](https://github.com/SirIle/scala-boot-test). More information about the scala-boot-test can be found from this [blog post](http://sirile.github.io/2015/04/08/docker-gradle-scala-boot.html). All Docker images can also be found from Dockerhub, so all the docker commands are runnable and the images are automatically pulled from the central repository.

### Requirements for discovery mechanism

The following requirements were identified when starting

- Flexible URL-based proxying of requests
- Automatic configuration when services are spawned or destroyed
- Logging to stdout so that logs can easily be harvested
- Support for vertical and horizontal scaling
  - Many instances of a service on one or many nodes
  - Multiple nodes and requests proxied across the nodes
- Support for versioned services
- Support for different types of load balanced entities (start with rest/http)

### Choosing HAProxy

There are a few good alternatives to use to proxy requests to services. Two of them stand out: [nginx](http://nginx.org) and [HAProxy](http://www.haproxy.org).

Nginx is geared towards proxying http(s) traffic whereas HAProxy can be used to proxy also other traffic, even low level TCP and UDP. Because of this versatility of HAProxy it was chosen.

### Other Components

[Consul](https://www.consul.io) is used as the service registry and key/value-store.

[Registrator](https://github.com/gliderlabs/registrator) is used to automatically register new services to Consul when the containers are started.

[Consul-template](https://github.com/hashicorp/consul-template) is used to automatically rewrite the HAProxy configuration files when information stored in Consul changes.

### Naming services and handling versioning

When a container offering a service is started, it is named along with a version. For example "hello/v1". Also the container is tagged with the service type, for example "rest". The version can also be omitted and a simple name used instead as the literal name is used as part of the URL.

## Components

### Dockerfile for HAProxy image

This file is used to build the image. It's based on BusyBox, which results in a very small footprint. Configuration files are copied to place and consul-template command is executed.

{% highlight dockerfile linenos %}
FROM progrium/busybox
MAINTAINER Ilkka Anttonen version: 0.2

# Update wget to get support for SSL
RUN opkg-install haproxy wget

# Download consul-template
RUN ( wget  --no-check-certificate https://github.com/hashicorp/consul-template/releases/download/v0.10.0/consul-template_0.10.0_linux_amd64.tar.gz -O /tmp/consul_template.tar.gz && gunzip /tmp/consul_template.tar.gz && cd /tmp && tar xf /tmp/consul_template.tar && cd /tmp/consul-template* && mv consul-template /usr/bin && rm -rf /tmp/* )

# Copy configuration files to place
COPY files/haproxy.json /tmp/haproxy.json
COPY files/haproxy.ctmpl /tmp/haproxy.ctmpl

CMD ["consul-template", "-config=/tmp/haproxy.json"]
{% endhighlight %}

### haproxy.json

This file tells consul-template what to do when consul sends signals through the socket. As we're providing the DNS server as part of the startup command we can find the consul service by using the dns name consul.service.consul.

In the command section we tell consul-template to run haproxy command after every change. The flag -sf tells it to kill all the previous running instances with defined PIDs which are queried with pidof-command. This causes the HAProxy to reload the configuration but prevents multiple instances being running simultaneously.

{% highlight json linenos %}
consul = "consul.service.consul:8500"
template {
  source = "/tmp/haproxy.ctmpl"
  destination = "/etc/haproxy/haproxy.cfg"
  command = "haproxy -f /etc/haproxy/haproxy.cfg -sf $(pidof haproxy) &"
}
{% endhighlight %}

### haproxy.ctmpl

The template file that is filled by consul-template when Consul backend information changes. It uses [go templating language](http://golang.org/pkg/text/template/). Detailed information about HAProxy configuration can be found [here](http://www.haproxy.org/download/1.5/doc/configuration.txt).

{% highlight apache linenos %}
{% raw %}
global
    log 127.0.0.1   local0
    log 127.0.0.1   local1 notice
    debug
    stats timeout 30s
    maxconn {{with $maxconn:=key "service/haproxy/maxconn"}}{{$maxconn}}{{else}}4096{{end}}
{% endraw %}{% raw %}
defaults
    log global
    option httplog
    option dontlognull
    mode http{{range ls "service/haproxy/timeouts"}}
    timeout {{.Key}} {{.Value}}{{else}}
    timeout connect 5000
    timeout client  50000
    timeout server  50000{{end}}
{% endraw %}{% raw %}
frontend http-in
    bind \÷*:80{{range $i,$a:=services}}{{$path:=.Name}}{{range .Tags}}{{if eq . "rest"}}
    acl app{{$i}} path_beg -i /{{$path}}{{end}}{{end}}{{end}}
    {{range $i,$a:=services}}{{range .Tags}}{{if eq . "rest"}}
    use_backend srvs_app{{$i}} if app{{$i}}{{end}}{{end}}{{end}}
{% endraw %}{% raw %}
{{range $i,$a:=services}}{{$path:=.Name}}{{range .Tags}}{{if eq . "rest"}}
backend srvs_app{{$i}}
    mode http
    balance roundrobin
    option forwardfor
    option httpchk HEAD / HTTP/1.1\r\nHost:localhost
    reqrep ^([^\ ]*\ /){{$path}}[/]?(.*)     \1\2{{range $c,$d:=service $a.Name}}
    server host{{$c}} {{.Address}}:{{.Port}} check{{end}}{{end}}{{end}}{{end}}
{% endraw %}{% raw %}
listen stats *:1936
    stats enable
    stats uri /
    stats hide-version
    stats auth someuser:password
{% endraw %}
{% endhighlight %}

The default section has the debug mode set on (line 4). This causes HAProxy to echo all the logging to stdout which is not desired in production, but can be useful in development environment. For more serious use it can be commented out.

Line 6 demonstrates how a value can be accessed from Consul key/value-store. A default value can also be specified, which will be inserted to the result file if no value is found from Consul. Same principle is used on lines 12-16 with timeouts. The default values can be overridden using the key/value-store.

On lines 19-20 all the services having the tag "rest" defined are looped and an acl entry is generated.

On lines 21-22 the same loop is executed again and use_backend configuration is written.

Between lines 24 and 31 the backend configuration is formed. Http health checks are defined so that when an existing service is scaled, the requests shouldn't be routed there until it is ready to serve.

This template works with multiple nodes as well as with multiple instances of the same service.

The template can be expanded to support also other types than rest/http like database connections.

## Starting local test environment

### Create Docker node

Create the Docker Machine node and point local client to it

{% highlight bash %}
docker-machine create -d virtualbox dev
eval "$(docker-machine env dev)"
{% endhighlight %}

### Start Consul

{% highlight bash %}
docker run --name consul -d -h dev -p `docker-machine ip $DOCKER_MACHINE_NAME`:8300:8300 -p `docker-machine ip $DOCKER_MACHINE_NAME`:8301:8301 -p `docker-machine ip $DOCKER_MACHINE_NAME`:8301:8301/udp -p `docker-machine ip $DOCKER_MACHINE_NAME`:8302:8302 -p `docker-machine ip $DOCKER_MACHINE_NAME`:8302:8302/udp -p `docker-machine ip $DOCKER_MACHINE_NAME`:8400:8400 -p `docker-machine ip $DOCKER_MACHINE_NAME`:8500:8500 -p 172.17.42.1:53:53 -p 172.17.42.1:53:53/udp progrium/consul -server -advertise `docker-machine ip $DOCKER_MACHINE_NAME` -bootstrap-expect 1
{% endhighlight %}

Here the IP address is acquired from the Docker Machine. Consul offers a DNS service on Docker bridge that other containers can use.

### Start Registrator

{% highlight bash %}
docker run -d -v /var/run/docker.sock:/tmp/docker.sock -h registrator --name registrator gliderlabs/registrator consul://`docker-machine ip $DOCKER_MACHINE_NAME`:8500
{% endhighlight %}

Registrator is responsible for registering (and de-registering) containers to Consul. It receives signals through the Docker socket, which is exposed to it.

### Start the example service

The example application returns a timestamp, hostname and some information as JSON. The environment variable SERVICE_NAME gives the name of the service which is then picked up by registrator and updated to Consul. Environment variable SERVICE_TAGS is similarly updated to Consul. Service name is used in the URL to route to the correct Docker container.

{% highlight bash %}
docker run -d -e SERVICE_NAME=hello/v1 -e SERVICE_TAGS=rest -h hello1 --name hello1 -p :80 sirile/scala-boot-test
{% endhighlight %}

## Testing template generation

A dry run of a template can be generated with

{% highlight bash %}
docker run --dns 172.17.42.1 --rm sirile/haproxy consul-template -config=/tmp/haproxy.json -dry -once
{% endhighlight %}

Here we tell the HAProxy container to run consul-template command. As it is configured to find Consul from the address consul.service.consul, we're telling it to use Consul provided DNS for discovery.

On an environment where the "service/haproxy/maxconn" is set to 1024 on Consul and one hello/v1 service is running, the following result is generated:

{% highlight apache %}
global
    log 127.0.0.1   local0
    log 127.0.0.1   local1 notice
    debug
    stats timeout 30s
    maxconn 1024

defaults
    log global
    option  httplog
    option  dontlognull
    mode http
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend http-in
    bind *:80
    acl app7 path_beg -i /hello/v1

    use_backend srvs_app7 if app7


backend srvs_app7
    mode http
    balance roundrobin
    option forwardfor
    option httpchk HEAD / HTTP/1.1\r\nHost:localhost
    reqrep ^([^\ ]*\ /)hello/v1[/]?(.*)     \1\2
    server host0 192.168.99.100:32768 check

listen stats *:1936
    stats enable
    stats uri /
    stats hide-version
    stats auth someuser:password

{% endhighlight %}

## Running the HAProxy

Running the HAProxy can be be done with

{% highlight bash %}
docker run -d -e SERVICE_NAME=rest --name=rest --dns 172.17.42.1 -p 80:80 -p 1936:1936 sirile/haproxy
{% endhighlight %}

HAProxy stats can be checked from port 1936, for example http://192.168.99.100:1936.

## Testing that things work

Now you should be able to access the hello service from http://192.168.99.100/hello/v1. The correct IP can be checked with `docker-machine ip $DOCKER_MACHINE_NAME`.

Logs can be accessed through Docker with `docker logs rest`.

The HAProxy service can also be found from consul under the name "rest.service.consul". In a later blog I'll demostrate how the local environment can be joined to the DNS service provided by Consul.

## Next steps

In the next part I'll demonstrate how the horizontal scaling with adding more Docker nodes works.
