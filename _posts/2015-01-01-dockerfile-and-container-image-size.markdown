---
layout: post
title: "Dockerfile and container image size"
---

## General

As I was creating a [minicontainer (minibox)]({% post_url 2014-12-31-fat-containers-microcontainers-and-minicontainers %}) (container that is based on BusyBox, but still contains everything that is needed to run as standalone without the need to use volume containers) I noticed that the resulting containers were way larger than they should have been. I only started paying attention to this as I was trying to create as small a container as possible.

After inspecting the resulting image with `docker history` I noticed that the steps that were executed while installing packages, like having the file downloaded, unpacked using `gunzip` and after that unpacked using `tar` all left a large intermediate container in the history.

I had tried to minimize the image after creation by executing `rm -rf /tmp/*` but that seemed to have no effect on the resulting image size.

## Solution

I tried combining all the steps needed to install a given piece of software into a single liner and that did the trick. So instead of installing Java like:

{% highlight bash %}
RUN wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" -O /tmp/jre.tar.gz http://download.oracle.com/otn-pub/java/jdk/8u25-b17/jre-8u25-linux-x64.tar.gz
RUN gunzip /tmp/jre.tar.gz
RUN (cd /opt && tar xf /tmp/jre.tar)
RUN rm -rf /tmp/*
{% endhighlight %}

the command became:

{% highlight bash %}
RUN ( wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" -O /tmp/jre.tar.gz http://download.oracle.com/otn-pub/java/jdk/8u25-b17/jre-8u25-linux-x64.tar.gz && gunzip /tmp/jre.tar.gz && cd /opt && tar xf /tmp/jre.tar && rm /tmp/jre.tar)
{% endhighlight %}

This resulted in an image of an expected size. For an image containing Java, Elasticsearch and Logstash, the reduction in size was from ~850MB to 351.1MB as compared to installing all of the applications in multiple steps.

## Example

Java on BusyBox with the commands as separate steps

{% highlight docker %}
FROM progrium/busybox
MAINTAINER Ilkka Anttonen version: 0.1

# Udate wget to support SSL
RUN opkg-install wget

# Get and install Java
RUN wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" -O /tmp/jre.tar.gz http://download.oracle.com/otn-pub/java/jdk/8u25-b17/jre-8u25-linux-x64.tar.gz
RUN gunzip /tmp/jre.tar.gz
RUN ( cd /opt && tar xf /tmp/jre.tar )
RUN rm /tmp/jre.tar)

# Link Java into use, wget-ssl updates libpthread which causes Java to break
RUN ln -sf /lib/libpthread-2.18.so /lib/libpthread.so.0
RUN ln -s /opt/jre1.8.0_25/bin/java /usr/bin/java
{% endhighlight %}

leads to the following image:

{% highlight docker %}
REPOSITORY             TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
sirile/testjava        latest              0df3d2bce73c        24 hours ago       407.4 MB
{% endhighlight %}

When inspected with `docker history` it looks like:

{% highlight bash %}
IMAGE               CREATED              CREATED BY                                      SIZE
0df3d2bce73c        About a minute ago   /bin/sh -c ln -s /opt/jre1.8.0_25/bin/java /u   25 B
e143d449780d        About a minute ago   /bin/sh -c ln -sf /lib/libpthread-2.18.so /li   23 B
202555d24a9f        About a minute ago   /bin/sh -c rm /tmp/jre.tar                      0 B
65f30ff37126        4 minutes ago        /bin/sh -c ( cd /opt && tar xf /tmp/jre.tar )   168.5 MB
0860f824b63b        4 minutes ago        /bin/sh -c gunzip /tmp/jre.tar.gz               168.8 MB
19c2913337e8        4 minutes ago        /bin/sh -c wget --no-check-certificate --no-c   62.76 MB
c655df8a63f9        24 hours ago         /bin/sh -c opkg-install wget                    2.565 MB
6dbf326c3ce2        29 hours ago         /bin/sh -c #(nop) MAINTAINER Ilkka Anttonen v   0 B
6f114d3139e3        11 weeks ago         /bin/sh -c #(nop) CMD [/bin/sh]                 0 B
7acf13620725        11 weeks ago         /bin/sh -c opkg-cl install http://downloads.o   461.1 kB
7fff0c6f0b8d        11 weeks ago         /bin/sh -c opkg-cl install http://downloads.o   69.19 kB
8db8f013bfca        11 weeks ago         /bin/sh -c #(nop) ADD file:c63e5d6cb74e15cf63   9.15 kB
bbd692fe2ca1        11 weeks ago         /bin/sh -c #(nop) ADD file:3bae78d4720418c677   103 B
21082221cb6e        11 weeks ago         /bin/sh -c #(nop) ADD file:393caa32e4f0f6eef6   220 B
015fb409be0d        11 weeks ago         /bin/sh -c #(nop) ADD file:2052bcafea75adbdd2   4.268 MB
e2fb46397934        11 weeks ago         /bin/sh -c #(nop) MAINTAINER Jeff Lindsay <pr   0 B
511136ea3c5a        18 months ago                                                        0 B
{% endhighlight %}

After modifying the Dockerfile

{% highlight docker %}
FROM progrium/busybox
MAINTAINER Ilkka Anttonen version: 0.1

# Udate wget to support SSL
RUN opkg-install wget

# Get and install Java
RUN ( wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" -O /tmp/jre.tar.gz http://download.oracle.com/otn-pub/java/jdk/8u25-b17/jre-8u25-linux-x64.tar.gz &&   gunzip /tmp/jre.tar.gz && cd /opt && tar xf /tmp/jre.tar && rm /tmp/jre.tar)

# Link Java into use, wget-ssl updates libpthread which causes Java to break
RUN ln -sf /lib/libpthread-2.18.so /lib/libpthread.so.0
RUN ln -s /opt/jre1.8.0_25/bin/java /usr/bin/java
{% endhighlight %}

the resulting image is

{% highlight docker %}
REPOSITORY             TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
sirile/testjava        latest              5e0d93d4a5c0        24 hours ago        175.9 MB
{% endhighlight %}

and when inspected with `docker history` it looks like

{% highlight bash %}
IMAGE               CREATED             CREATED BY                                      SIZE
5e0d93d4a5c0        24 hours ago        /bin/sh -c ln -s /opt/jre1.8.0_25/bin/java /u   25 B
3d1c972185dd        24 hours ago        /bin/sh -c ln -sf /lib/libpthread-2.18.so /li   23 B
01f2e42fda1e        24 hours ago        /bin/sh -c ( wget --no-check-certificate --no   168.5 MB
c655df8a63f9        24 hours ago        /bin/sh -c opkg-install wget                    2.565 MB
6dbf326c3ce2        30 hours ago        /bin/sh -c #(nop) MAINTAINER Ilkka Anttonen v   0 B
6f114d3139e3        11 weeks ago        /bin/sh -c #(nop) CMD [/bin/sh]                 0 B
7acf13620725        11 weeks ago        /bin/sh -c opkg-cl install http://downloads.o   461.1 kB
7fff0c6f0b8d        11 weeks ago        /bin/sh -c opkg-cl install http://downloads.o   69.19 kB
8db8f013bfca        11 weeks ago        /bin/sh -c #(nop) ADD file:c63e5d6cb74e15cf63   9.15 kB
bbd692fe2ca1        11 weeks ago        /bin/sh -c #(nop) ADD file:3bae78d4720418c677   103 B
21082221cb6e        11 weeks ago        /bin/sh -c #(nop) ADD file:393caa32e4f0f6eef6   220 B
015fb409be0d        11 weeks ago        /bin/sh -c #(nop) ADD file:2052bcafea75adbdd2   4.268 MB
e2fb46397934        11 weeks ago        /bin/sh -c #(nop) MAINTAINER Jeff Lindsay <pr   0 B
511136ea3c5a        18 months ago                                                       0 B
{% endhighlight %}

## Conclusion

Combining the steps that would otherwise leave behing a temporary file or folder is recommended as otherwise the intermediary containers created during the build are still retained as part of the end image.
