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

## Conclusion

Combining the steps that would otherwise leave behing a temporary file or folder is recommended as otherwise the intermediary containers created during the build are still retained as part of the end image.
