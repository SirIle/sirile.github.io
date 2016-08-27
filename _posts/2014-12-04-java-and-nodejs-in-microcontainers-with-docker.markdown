---
layout: post
title:  "Java and Node.js in Microcontainers with Docker"
categories: docker
---

This article was also published in [DZone](http://java.dzone.com/articles/java-and-node-microcontainers).

In the [first article]({% post_url
2014-11-06-Microservices-in-Microcontainers-with-Docker %}) I presented a way to
create Microcontainers which use volume containers to share runtimes and
executables so that the actual application containers can be kept small. The
containers are based on BusyBox
([progrium/busybox](https://github.com/progrium/busybox)) and necessary
libraries are symbolically linked so that the executables work in the very light
environment.

In this article I take the concept forward with a Java runtime environment and
service registration and discovery using [Consul](http://www.consul.io).

The concept is built on Docker's ability to use multiple volume containers per
application container. This allows the container to have a mix of different
executables available and provide a specialized, minimal runtime for a specific
application like Jetty.

I present an example which runs Consul for service registration and discovery in
all containers, a Node.js-based hello world application, a Java-based hello
world application on [Jetty](http://eclipse.org/jetty/) and
[cAdvisor](https://github.com/google/cadvisor) for container monitoring. The
set-up can be run with [Weave](https://github.com/zettio/weave) based networking
with static internal IP addresses or without it.

The concept is built in three parts:

-   Fileboxes containing the executables and runtime environments. The source
code can be found from github at
[SirIle/fileboxes](https://github.com/SirIle/fileboxes).

-   Microboxes setting up the application containers with everything linked into
place. The source code can be found from
[SirIle/microboxes](https://github.com/SirIle/microboxes).

-   Microtestboxes containing the actual applications for the demo and runit
configuration extending the microboxes. The source code is available at [SirIle/microtestboxes](https://github.com/SirIle/microtestboxes).

This image shows how the application containers are built by extending
containers. It also shows which fileboxes are needed for the application
containers to work and which ports are exposed.

![Fileboxes_and_Containers](/images/Fileboxes_and_containers.png)

Corresponding docker.io trusted builds have been made so trying out the examples
doesn't require local building. The filebox volume containers need to be created
locally.

## Fileboxes

Fileboxes are special containers that only define volumes. They are based on
[tianon/true](https://registry.hub.docker.com/u/tianon/true/dockerfile/) image
which only contains an assembler version of `true` command and causes minimum
overhead (125 bytes) on the file container.

### Basefilebox

Basefilebox contains the components that are provided to all containers. For
this demo that means Consul for service registration and discovery between
containers and runit to allow Consul and the actual application to run at the
same time in the container. The buildfile copies the executables from the image
to the correct location in the volume.

#### Consul

As Consul is statically linked it is extremely easy to get it to run. Main idea
is to have a Consul agent running in all containers so that service discovery is
trivial. As all the containers get an own IP either from the Docker itself or
from an external network configuration mechanism like Weave, services can be
left running in the default ports and use DNS A-records for service discovery.
This simplifies things as there is no need to query for ports or extra
information, which could be done using SRV records or a REST-based API.

A shell script is used to start the Consul which checks if an environment
variable called DC is defined. If it is, the script waits for the Weave
interface to become available before getting the IP of that interface and
announcing that with Consul. If it isn't, the IP of the container is received
from the /etc/hosts file.

#### Runit

Runit is used to control multiple processes running in the microcontainer. It is
used instead of supervisord as it is very small and doesn't require any extra
libraries to be available. It consists of two commands: `runsv` and `runsvdir`.

#### Dockerfile for the base container

~~~bash
FROM tianon/true
MAINTAINER Ilkka Anttonen version: 0.1
VOLUME /opt/files
COPY consul /opt/files/consul
COPY runsv /opt/files/runsv
COPY runsvdir /opt/files/runsvdir
~~~

### Javafilebox

#### Oracle JRE 8

The Java runtime is quite large. I looked around for stripped versions, but most
either weren't very compatible or were quite old. In the end I figured that
sharing the runtime-executables via a volume container could be a good trade-off
between compatibility and size. This way the volume container is built once and
then all the containers that need the Java runtime on that host can just include
it. Without any extra modifications, the size of the image is 168.5 MB. Building
the volume container is extremely quick, but downloading the image will take
some time.

#### Dockerfile for the Java 8 container

~~~bash
FROM tianon/true
MAINTAINER Ilkka Anttonen version: 0.1
VOLUME /opt/files/java
COPY jre1.8.0_25 /opt/files/java/jre
~~~

### Nodejsfilebox

Node executable that was built on the Ubuntu host only requires the libstdc++
library to run. This nodefilebox includes the executable and the library. I
thought about including npm as well, but decided against it as the container is
meant to be a runtime instead of a development environment.  This way the
application with all the dependencies can be built on host (or later on a
build-container) and the resulting files can just be copied to the application
container.

#### Dockerfile for the Nodejs volume container

~~~bash
FROM tianon/true
MAINTAINER Ilkka Anttonen version: 0.1
VOLUME /opt/files/node
COPY libstdc++.so.6 /opt/files/node/libstdc++.so.6
COPY node /opt/files/node/node
~~~

### Jettyfilebox

This filebox contains the unpacked Jetty 9 distributable. To be usable, the
container running this must also use the Javafilebox to provide the JRE.

#### Dockerfile for the Jettyfilebox

~~~bash
FROM tianon/true
MAINTAINER Ilkka Anttonen version: 0.1
VOLUME /opt/files/jetty
COPY jetty-distribution-9.2.4.v20141103 /opt/files/jetty/jetty
~~~

### Building the volume containers from the images

The volume containers can be built with the following four commands:

~~~bash
docker run --name=basefilebox sirile/basefilebox
docker run --name=nodejsfilebox sirile/nodejsfilebox
docker run --name=java8filebox sirile/java8filebox
docker run --name=jettyfilebox sirile/jettyfilebox
~~~

This also downloads the container images to the local registry. The JRE image is
rather large, others are small.

## Microboxes

These are containers that stack on top of each other to build the runtime
environments. They link the executables and libraries that the filebox volume
containers provide to the file system.  Some of them can be started by
themselves instead of extending with applications for development and testing
purposes. More information about this can be found from corresponding README.md
files in Github.

### Basebox

A base Microcontainer built on top of progrium/busybox. Not that useful by
itself, but acts as a base for running other applications. Includes symbolic
links for Consul and runit executables. It also includes basic configuration for
runit, basic consul configuration including DNS server port specification for
the normal port 53 so that it can be used as the local DNS server. The container
fires up runit which starts the execution of consul and also other runit
controlled processes that can be configured under /etc/service.

Everything built on top of Basebox requires basefilebox volume container for the
executables.

#### Dockerfile for Basebox

~~~bash
FROM progrium/busybox
MAINTAINER Ilkka Anttonen version: 0.1
RUN ln -s /opt/files/runsv /sbin/runsv
RUN ln -s /opt/files/runsvdir /sbin/runsvdir
RUN ln -s /opt/files/consul /usr/bin/consul
COPY consul /etc/service/consul/run
COPY config-consul.json /etc/consul.d/config-consul.json
CMD "/sbin/runsvdir" "/etc/service"
~~~

### Consulbox

A Microcontainer running on top of sirile/basebox which offers a consul server
and consul UI. It also contains a start script for the Consul server which can
adapt to Weave provided networking if the environment variable DC has been set.
Otherwise it uses the normal IP of the container.

#### Dockerfile for Consulbox

~~~bash
FROM progrium/busybox
MAINTAINER Ilkka Anttonen version: 0.1
COPY consul_ui /opt/consul_ui
COPY startConsul.sh /usr/bin/startConsul.sh
RUN ln -s /opt/files/consul /usr/bin/consul
CMD '/usr/bin/startConsul.sh'
~~~

### Javabox

A Microcontainer for a Java based application. The version of the JRE is decided
by the volume container that has the files.

#### Dockerfile for Javabox

~~~bash
FROM sirile/basebox
MAINTAINER Ilkka Anttonen version: 0.1
RUN ln -s /opt/files/java/jre/bin/java /usr/bin/java
~~~

### Jettybox

Jettybox build file does nothing at the moment as there is no need to link Jetty
executables to the file system because Jetty can be started directly with a
`java -jar` command. The container is still created and the actual application
container is built on top of it.

### Nodejsbox

A Microcontainer that can be used to run nodejs applications.

#### Dockerfile for Nodejsbox

~~~bash
FROM sirile/basebox
MAINTAINER Ilkka Anttonen version: 0.1
RUN ln -s /opt/files/node/node /usr/bin/node
RUN ln -s /opt/files/node/libstdc++.so.6 /lib/libstdc++.so.6
~~~

## Microtestboxes

### Jettytestbox

The Jetty example container contains just the example application and
configuration plus the runit configuration to start Jetty. The container
registers itself to Consul and tells that it provides a jettyhello-service. As
the container is based on basebox, it automatically launches the runit defined
in that Dockerfile.

Example war-file was downloaded from [Apache Tomcat
page](https://tomcat.apache.org/tomcat-6.0-doc/appdev/sample/). It contains a
servlet and a jsp that can be used to demonstrate that the Jetty environment is
fully functional.

#### Dockerfile for Jettytestbox

~~~bash
FROM sirile/jettybox
MAINTAINER Ilkka Anttonen version: 0.1
COPY jetty /etc/service/jetty/run
COPY start.ini start.ini
COPY sample.war webapps/sample.war
COPY hello.json /etc/consul.d/hello.json
~~~

### Nodejstestbox

A very simple nodejs and express based hello world application. The container
registers itself to Consul and tells that it provides a nodehello-service.

#### Dockerfile for Nodejstestbox

~~~bash
FROM sirile/nodejsbox
MAINTAINER Ilkka Anttonen version: 0.1
COPY express.js express.js
COPY node_modules node_modules
COPY node /etc/service/node/run
COPY hello.json /etc/consul.d/hello.json
~~~

## Running the example application

The [github project](https://github.com/SirIle/microtestboxes) contains shell
scripts which can be used to start the applications. As docker.io trusted builds
have been defined, local building is not necessary.

### Running with Weave provided networking

This can be started with the `startTestboxes.sh` script, which does the
following:

~~~bash
sudo weave launch
sudo weave run 10.0.1.1/22 -p 8500:8500 -h consul -e DC=west --name consul --volumes-from basefilebox sirile/consulbox
sudo weave run 10.0.1.2/22 --volumes-from basefilebox --volumes-from nodejsfilebox -e DC=west --link consul:consul -h nodebox --name nodebox --dns 127.0.0.1 -p 80:80 sirile/nodejstestbox
sudo weave run 10.0.1.3/22 --volumes-from basefilebox --volumes-from java8filebox --volumes-from jettyfilebox -e DC=west --link consul:consul -h jettybox --name jettybox --dns 127.0.0.1 -p 81:80 sirile/jettytestbox
sudo weave run 10.0.1.4/22 --volume=/:/rootfs:ro --volume=/var/run:/var/run:rw --volume=/sys:/sys:ro --volume=/var/lib/docker/:/var/lib/docker:ro --publish=8080:8080 --name=cadvisor google/cadvisor:latest
~~~

### Running without Weave

Running the example without Weave networking can be done with the
`startTestboxesNoWeave.sh` script which does:

~~~bash
docker run -p 8500:8500 -h consul -d --name consul --volumes-from basefilebox sirile/consulbox
docker run --volumes-from basefilebox -d --volumes-from nodejsfilebox --link consul:consul -h nodebox --name nodebox --dns 127.0.0.1 -p 80:80 sirile/nodejstestbox
docker run --volumes-from basefilebox -d --volumes-from java8filebox --volumes-from jettyfilebox --link consul:consul -h jettybox --name jettybox --dns 127.0.0.1 -p 81:80 sirile/jettytestbox
docker run --volume=/:/rootfs:ro --volume=/var/run:/var/run:rw --volume=/sys:/sys:ro --volume=/var/lib/docker/:/var/lib/docker:ro --publish=8080:8080 --detach=true --name=cadvisor google/cadvisor:latest
~~~

### Testing the application using a browser

There are several applications available:

-   Consul UI can be accessed at port 8500 (for example
[http://10.10.10.30:8500](http://10.10.10.30:8500))
-   Nodejs based test application runs at port 80
([http://10.10.10.30](http://10.10.10.3))
-   Jetty based test application runs at port 81 at context /sample
([http://10.10.10.30:81/sample](http://10.10.10.30:81/sample))
-   cAdvisor runs in port 8080 ([http://10.10.10.30:8080](http://10.10.10.30:8080))

### Testing Consul based connectivity

Consul provided DNS can be used to connect to for example a back-end database
that has registered itself as a service. For now there are three services
registered: consul, nodehello and jettyhello. Calling between containers can be
demonstrated with:

~~~bash
$ docker exec jettybox ping nodehello.service.consul
PING nodehello.service.consul (10.0.1.2): 56 data bytes
64 bytes from 10.0.1.2: seq=0 ttl=64 time=0.053 ms
64 bytes from 10.0.1.2: seq=1 ttl=64 time=0.207 ms
~~~

This shows that the Consul provided DNS resolves the address
nodehello.service.consul to the Weave given internal IP address of nodebox
container. Using Weave the container could well be running on a separate host,
even on another service provider.

## Conclusions

This is the list of the container images:

~~~bash
REPOSITORY             TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
sirile/jettytestbox    latest              b949a448fdac        5 hours ago         4.815 MB
sirile/nodejstestbox   latest              3e546fa4663d        5 hours ago         5.775 MB
sirile/consulbox       latest              38b40fd0b0a8        6 hours ago         5.782 MB
sirile/nodejsfilebox   latest              34b1ad35b57f        6 hours ago         9.819 MB
sirile/java8filebox    latest              d6b4cb7c3079        9 hours ago         168.5 MB
sirile/basefilebox     latest              f3a56a6cb60e        9 hours ago         15.78 MB
sirile/jettyfilebox    latest              f1a14fe1f665        9 hours ago         14.7 MB
google/cadvisor        0.6.1               5c12ab6c98aa        2 days ago          17.55 MB
google/cadvisor        latest              5c12ab6c98aa        2 days ago          17.55 MB
tianon/true            latest              d222574ccc4a        2 days ago          125 B
zettio/weave           git-3cb6f1b3cc4b    6e720cb2ccd9        3 days ago          10.83 MB
zettio/weave           latest              6e720cb2ccd9        3 days ago          10.83 MB
~~~

As can be observed, the application containers are very small, around 5 MB. The
memory usage varies from about 25MB of the nodejs container to around 100 MB for
the untuned Jetty container. Running services as Microservices with the runtime
bundled with the service is a feasible thing to do, especially if the runtimes
are shared with volume containers which helps to keep the application containers
small. This leads to extremely fast build and startup times for the application
containers.

## Next steps

The goal of this exercise was to prove that stand alone Microservices are doable
on Microcontainers with Docker. They are. They can provide a lot of flexibility
in creating the runtime distributed environment especially when service
registration and discovery is used along with advanced networking so that the
services can reside on multiple nodes.

Next step could be to implement a real multi-tiered application on this base
instead of just simple example applications. Suggestions are welcome.
