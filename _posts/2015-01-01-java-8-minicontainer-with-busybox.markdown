---
layout: post
title: "Java 8 JRE minicontainer with BusyBox on Docker"
---

## Installing Java 8 on BusyBox

After spending [some time]({% post_url 2014-11-06-Microservices-in-Microcontainers-with-Docker %}) with [microcontainers]({% post_url 2014-12-04-java-and-nodejs-in-microcontainers-with-docker %}) I got interested in trying to make the smallest feasible self contained container that includes a working recent Java 8 JRE. I'm fond of using BusyBox (especially [progrium/busybox](https://github.com/progrium/busybox)) as the base image as it contains most of the libraries needed to be usable while still being under 5 MB in size.

I decided to call containers built this way [minicontainers as opposed to microcontainers]({% post_url 2014-12-31-fat-containers-microcontainers-and-minicontainers %}) that share the runtimes using volume containers. Volume container support is still pretty non-existent in orchestrators so self-contained containers are more portable between hosts.

### Downloading the installation packets

As the Oracle's Java download page requires the user to accept things by selecting radio boxes, getting the file to download automatically isn't exactly trivial. There was an excellent [answer on stackoverflow](http://stackoverflow.com/questions/10268583/how-to-automate-download-and-installation-of-java-jdk-on-linux) for this exact issue which described the command to use:

{% highlight bash %}
wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u25-b17/jdk-8u25-linux-x64.tar.gz
{% endhighlight %}

One problem with using wget in BusyBox is that the wget that comes with the progrium/busybox image doesn't understand https or ssl in general. Luckily there is a package for wget-ssl that can be installed directly with opkg manager that does support https.

### Libpthread hack

As installing wget-ssl updates libpthreads to such a version that Java executable breaks, the following is needed to get Java to run again:

{% highlight bash %}
RUN ln -sf /lib/libpthread-2.18.so /lib/libpthread.so.0
{% endhighlight %}

## Dockerfile

{% highlight docker %}
FROM progrium/busybox
MAINTAINER Ilkka Anttonen version: 0.1

# Udate wget to support SSL
RUN opkg-install wget

# Get and install Java
RUN ( wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" -O /tmp/jre.tar.gz http://download.oracle.com/otn-pub/java/jdk/8u25-b17/jre-8u25-linux-x64.tar.gz && gunzip /tmp/jre.tar.gz && cd /opt && tar xf /tmp/jre.tar && rm /tmp/jre.tar)

# Link Java into use, wget-ssl updates libpthread which causes Java to break
RUN ln -sf /lib/libpthread-2.18.so /lib/libpthread.so.0
RUN ln -s /opt/jre1.8.0_25/bin/java /usr/bin/java
{% endhighlight %}

## Building

The container can be built by running

{% highlight bash %}
docker build --rm -t sirile/minijavabox .
{% endhighlight %}

The resulting image is 175.9MB in size.

## Testing

After building, the image can be run with

{% highlight bash %}
docker run --rm -ti --name javabox sirile/minijavabox /bin/sh
/ # java -version
java version "1.8.0_25"
Java(TM) SE Runtime Environment (build 1.8.0_25-b17)
Java HotSpot(TM) 64-Bit Server VM (build 25.25-b02, mixed mode)
/ #
{% endhighlight %}
