---
layout: post
title:  "Notable containers"
date:   2014-11-29 00:31:37
categories: docker containers
---
While getting familiar with different ways of using Docker I have bumped
into a few pretty useful containers.

# Base for Microcontainers

A good BusyBox based container is [progrium/busybox]. It contains a package manager
(opkg) and includes most of the libraries to enable running applications on top of it.
It's decently sized at under 5 MB.

# Base for volume containers

As volume containers don't need any functionality, but you can't build them on
top of the scratch container, the next best alternative is [tianon/true].

# Getting Ubuntu Utopic to stay alive

When working in an environment where it would be useful to leave a container
running a fully fledged OS running in the background so that you can later
`docker exec` into it consider doing the following:

```bash
docker run -d --name ubuntu ubuntu:utopic sleep infinity
docker exec -ti ubuntu /bin/bash
```
