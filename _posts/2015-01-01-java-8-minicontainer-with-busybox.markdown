---
layout: post
title: "Java 8 JRE minicontainer with BusyBox on Docker"
commentIssueId: 1
---

**Update on 31.3.2015**  _Updated the Java version to 1.8.0_40._

## Installing Java 8 on BusyBox

After spending [some time]({% post_url 2014-11-06-Microservices-in-Microcontainers-with-Docker %}) with [microcontainers]({% post_url 2014-12-04-java-and-nodejs-in-microcontainers-with-docker %}) I got interested in trying to make the smallest feasible self contained container that includes a working recent Java 8 JRE. I'm fond of using BusyBox (especially [progrium/busybox](https://github.com/progrium/busybox)) as the base image as it contains most of the libraries needed to be usable while still being under 5 MB in size.

I decided to call containers built this way [minicontainers as opposed to
microcontainers]({% post_url
2014-12-31-fat-containers-microcontainers-and-minicontainers %}) that share the
runtimes using volume containers. Volume container support is still pretty
non-existent in orchestrators so self-contained containers are more portable
between hosts.

### Downloading the installation packets

As the Oracle's Java download page requires the user to accept things by
selecting radio boxes, getting the file to download automatically isn't exactly
trivial. There was an excellent [answer on
stackoverflow](http://stackoverflow.com/questions/10268583/how-to-automate-download-and-installation-of-java-jdk-on-linux)
for this exact issue which described the command to use:

~~~bash
wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" -O /tmp/jre.tar.gz http://download.oracle.com/otn-pub/java/jdk/8u40-b26/jre-8u40-linux-x64.tar.gz
~~~

One problem with using wget in BusyBox is that the wget that comes with the
progrium/busybox image doesn't understand https or ssl in general. Luckily there
is a package for wget-ssl that can be installed directly with opkg manager that
does support https.

### Libpthread hack

As installing wget-ssl updates libpthreads to such a version that Java
executable breaks, the following is needed to get Java to run again:

~~~bash
RUN ln -sf /lib/libpthread-2.18.so /lib/libpthread.so.0
~~~

## Dockerfile

~~~bash
FROM progrium/busybox
MAINTAINER Ilkka Anttonen version: 0.2

# Udate wget to support SSL
RUN opkg-install wget

# Get and install Java
RUN ( wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" -O /tmp/jre.tar.gz http://download.oracle.com/otn-pub/java/jdk/8u40-b26/jre-8u40-linux-x64.tar.gz &&   gunzip /tmp/jre.tar.gz && cd /opt && tar xf /tmp/jre.tar && rm /tmp/jre.tar)

# Link Java into use, wget-ssl updates libpthread which causes Java to break
RUN ln -sf /lib/libpthread-2.18.so /lib/libpthread.so.0 && ln -s /opt/jre1.8.0_40/bin/java /usr/bin/java
~~~

## Building

The container can be built by running

~~~bash
docker build --rm -t sirile/minijavabox .
~~~

The resulting image is 178.7MB in size.

## Testing

After building, the image can be run with

~~~bash
docker run --rm -ti --name javabox sirile/minijavabox /bin/sh
/ # java -version
java version "1.8.0_40"
Java(TM) SE Runtime Environment (build 1.8.0_40-b26)
Java HotSpot(TM) 64-Bit Server VM (build 25.40-b25, mixed mode)
/ #
~~~
