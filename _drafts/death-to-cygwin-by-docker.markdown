---
layout: post
title: "death-to-cygwin-by-docker"
date: "2015-05-01 20:57"
---

Most of the open source is developed on Linux or OS X platforms. Quite a lot of developers are using Windows. This has led to a lot of ingenious ways in getting things to run on Windows. One of the most commonly used is to install Cygwin and compile software to run natively on Windows.

As of Docker version 1.6 running things on Windows is extremely easy and ..

## Installing prerequisites

Installing VirtualBox

Installing Docker client version 1.6

Installing Docker Machine version 0.2.0

## Example of running ..

Create a local Docker execution environment with Docker Machine

{% highlight bash %}
docker-machine create -D virtualbox dev
{% endhighlight %}

Point the local docker client against Docker Machine created instance.

...

Build your own image or directly start an image from Docker.io registry.

docker run ..

#
