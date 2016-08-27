---
layout: post
title:  "Microservices in Microcontainers with Docker"
categories: docker
---

This article was originally published on [DZone](http://architects.dzone.com/articles/microservices-microcontainers).

## Going towards minimalism

After first using [fat Docker containers to build PAAS like
capabilities](http://www.slideshare.net/IlkkaAnttonen/open-slava-2014ilkkaanttonen)
I have now switched my thinking towards simpler containers. The size of the
core-container crept to over a gigabyte whereas microcontainers can be kept very
small, in the megabytes or tens of megabytes range instead of hundreds of
megabytes.

## Contained standalone services

If a container was small enough it would enable a Microservice per
Microcontainer kind of pattern. Combined with service registration and discovery
with a mechanism like [Consul](http://consul.io) it would provide a lot of
flexibility in distributing services. If a service contains the runtime as well
then getting them to run would equal to just firing up the container and having
it register itself.

If the container networking is provided by for example
[Weave](https://github.com/zettio/weave), the services could be distributed over
multiple hosts or cloud providers.

## Basebox for Microcontainers

BusyBox is a commonly used as a minimal Linux image in for example routers and
embedded systems. A very good Docker implementation can be found from
[progrium/busybox](https://github.com/progrium/busybox). It also has the project
that can be used to create a new base BusyBox image, but that's anything but
trivial with heaps of configuration options. For this exercise the container
image progrium provides in the Docker.io registry is sufficient.

The progrium/busybox Docker image includes opkg as the package manager. There
are quite a few packages, but the most active development seems to be aimed
towards ARM and not  x86_64 architecture.

## Getting node to run in a Microcontainer

For testing the Microservices pattern I decided to try to get node running as
creating a test application would be trivial. After checking the opkg registries
I couldn't find a prepackaged node for the x86_64 architecture anywhere.

As the virtualized Ubuntu image I use as the Docker host has the same x86_64
architecture as the BusyBox container, the same libraries and executables can be
used.

### Checking the linked libraries

The linked libraries of an executable can be checked with the `ldd` command. I
installed node on the host and running `ldd` there gives the following result:

~~~bash
vagrant@node1:~$ ldd /usr/bin/node
linux-vdso.so.1 =>  (0x00007fffdd5fe000)
libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f41e8f3f000)
librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f41e8d37000)
libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f41e8a32000)
libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f41e872c000)
libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f41e8516000)
libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f41e82f7000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f41e7f31000)
/lib64/ld-linux-x86-64.so.2 (0x00007f41e914b000)
~~~

Of those only libstdc++ isn't readily available in the BusyBox image /lib
directory.

### Trial and error

Sure way to see if an executable works is to run it. After Docker starts
supporting build containers, things will get a lot simpler, but for now it's
easiest to build the programs in the host and directly call them from its
filesystem by sharing the host filesystem to the container. The following
command gives a shell in the container and exposes the host filesystem as /host.
You can quit the container with CTRL+D and the filesystem is cleaned after the
exit.

~~~bash
vagrant@node1:~$ sudo docker run -v /:/host -ti --rm --name testbox progrium/busybox /bin/sh
~~~

You can try running the node executable (note that quite often the real
executable is behind a few symbolic links):

~~~bash
/# /host/usr/bin/nodejs
/host/usr/bin/nodejs: error while loading shared libraries: libstdc++.so.6: cannot open shared object file: No such file or directory
~~~

As expected the executable complains about the missing library.

### Adding the link and retrying

After adding the symbolic link the command can be retried:

~~~bash
/# ln -s /host/usr/lib/x86_64-linux-gnu/libstdc++.so.6 /lib/libstdc++.so.6
/# /host/usr/bin/nodejs
>
~~~

Node starts successfully. Now we know that by adding the libstdc++ library to
the BusyBox image we can get node running.

## Keeping the images small by sharing the runtime

I started playing with idea of just having the application code in the container
and storing larger executables and libraries so that they could be shared by the
Microcontainers. This way updating a runtime like a nodejs version would be
trivial.

### Volume containers

Docker offers a concept called Volume Containers which can be used to share
information between containers inside a single host.

The Volume Container needs to be built once in each host, but as the base image
can be built with a Dockerfile and is probably available from a registry,
building the container is a one-liner. A good base image for a Volume Container
is tianon/true which is basically a scratch image with only assembler based true
command.

### Example Dockerfile for a volume container

~~~bash
FROM tianon/true
MAINTAINER Ilkka Anttonen version: 0.1
VOLUME /opt/files
ADD consul /opt/files/consul
ADD libstdc++.so.6 /opt/files/libstdc++.so.6
ADD node /opt/files/node
~~~

This file adds Consul and node executables as well as the libstdc++ library to
the container. A version of this can be found from Docker.io with the name
sirile/filebox.

## Application container

Symbolic linking in Microcontainers can be used to provide linked libraries for
the runtime environment as well as providing the executables from the volume
container. Symbolic links enable the Microcontainer to be built even though the
actual files are not there as they will be provided by the volume container at
runtime. Trying to start the Microcontainer without providing the volume
container will fail at runtime as the libraries or the executables can't be
found.

### Example Dockerfile of Nodebox container

~~~bash
FROM progrium/busybox
MAINTAINER Ilkka Anttonen version: 0.1
RUN ln -s /opt/files/node /usr/bin/node
RUN ln -s /opt/files/libstdc++.so.6 /lib/libstdc++.so.6
~~~

This file adds the symbolic links in place. This can be found from Docker.io
with the name sirile/nodebox.

## Example application

The example application is a very simple node.js and express based application
which provides a "Hello world" response to a http request.

### Dockerfile for the example application

~~~bash
FROM sirile/nodebox
MAINTAINER Ilkka Anttonen version: 0.1
ADD express.js express.js
ADD node_modules node_modules
CMD "node" "express"
~~~

This file adds the node modules and the main Javascript file in place and
executes the application. This can be found from Docker.io with the name
sirile/nodeboxtest.

### Building the volume container

The Volume Container filebox is built with the command:

~~~bash
vagrant@node1:~$ sudo docker run --name filebox sirile/filebox
~~~

This downloads the image from the Docker.io registry if needed.

### Building and running the demo application container

Demo application can be run with the command:

~~~bash
vagrant@node1:~$ sudo docker run --volumes-from filebox -ti --name testbox -p 3000:3000 --rm sirile/nodeboxtest
Example app listening at http://0.0.0.0:3000
~~~

The application can be accessed from the host port 3000 with a browser. You can
quit it by pressing CTRL+C.

### Images

The size of the images is pretty small:

~~~bash
vagrant@node1:~$ sudo docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
sirile/nodeboxtest         latest              a52297dd66a1        5 hours ago         5.774 MB
sirile/filebox             latest              4fda74df99f9        5 hours ago         25.56 MB
progrium/busybox           latest              6f114d3139e3        3 weeks ago         4.808 MB
vagrant@node1:~$
~~~

The size of the container for the demo application is 5.774MB. This could be
made even smaller, but at this size the container offers decent functionality in
case it needs to be accessed for debugging.

### Accessing the container

If you need to join the running session and see what is going on inside the
container you can use:

~~~bash
sudo docker exec -ti nodeboxtest /bin/sh
~~~

This opens a shell inside the running container and you can then look around and
install debug tooling etc.The command is available from Docker 1.3 onwards. That
way running sshd isn't required in the container and nsenter isn't needed.

## Next steps

Next step is adding the service registration capability to the microcontainer by
using Consul. One microcontainer per host will act as the Consul server node and
all the microcontainers (microservices) in that node will register themselves on
it.

The Consul servers on different hosts can then be joined together and that way
Consul will provide a DNS based service discovery across hosts. This requires
advanced networking using for example Weave and is a topic for the next blog.
