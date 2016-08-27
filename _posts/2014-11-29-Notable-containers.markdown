---
layout: post
title:  "Notable containers"
date:   2014-11-29 00:31:37
categories: docker
---

While getting familiar with different ways of using Docker I have bumped
into a few pretty useful containers.

## Base for Microcontainers

A good BusyBox based container is [progrium/busybox][progrium/busybox]. It
contains a package manager (opkg) and includes most of the libraries to enable
running applications on top of it. It's decently sized at about 4.8 MB.

## Base for volume containers

As volume containers don't need any functionality, but you can't build them on
top of the scratch container, the next best alternative is
[tianon/true][tianon/true].

## Getting Ubuntu Utopic to stay alive

When working in an environment where it would be useful to leave a container
running a fully fledged OS running in the background so that you can later
`docker exec` into it consider doing the following:

~~~bash
docker run -d --name ubuntu ubuntu:utopic sleep infinity
docker exec -ti ubuntu /bin/bash
~~~

This should leave the container running but also cause no CPU strain as it will
be sleeping before another shell is opened to it using the `docker exec`
command.

[progrium/busybox]: https://github.com/progrium/busybox
[tianon/true]:      https://registry.hub.docker.com/u/tianon/true
