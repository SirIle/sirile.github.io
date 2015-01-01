---
layout: post
title: "Fat containers, microcontainers and minicontainers"
---

## General

The learning curve of Docker seems to go in the following order:

1. trying to get existing applications to run in containers in any way possible while minimizing the changes to the applications
2. trying to minimize the container size and to adhere to Docker best practices like one process per container
3. going overboard with minimizing container sizes and heavily using volume containers for sharing common files between the containers
4. finding the middle ground between extremes and using the smallest possible base image and minimizing the container size while still having the containers portable

## Fat containers

First step leads to fat containers. These have pretty much everything and the kitchen sink included. I talked about these in [OpenSlava](http://www.slideshare.net/IlkkaAnttonen/open-slava-2014ilkkaanttonen). I used supervisor for running multiple processes per container, rsyslog for log aggregation and consul for service registration and discovery. One container could easily be over a gigabyte in size.

## Microcontainers

Step two is an evolvement as skills with Docker increase. I guess most either stay in step two or at least skip over step three. I decided to [go a bit]({% post_url 2014-11-06-Microservices-in-Microcontainers-with-Docker %}) with [further]({% post_url 2014-12-04-java-and-nodejs-in-microcontainers-with-docker %}) and found out that although sharing runtimes with volume containers is feasible, in real life the orchestration support for using volume containers is either non-existent or minimal. That would lead to having to use shell scripts for orchestration which is clumsy and error prone.

I still used Consul for service registration and discovery but switched to runit for running multiple processes per container as it is statically linked and doesn't require python runtime.

## Minicontainers

After exploring microcontainers I decided to go for minicontainers (or miniboxes). I still try to minimize the container size by using BusyBox as the base ([progrium/busybox](https://github.com/progrium/busybox) is my favourite) but include the relevant runtimes in the containers so that they are easily portable and can be orchestrated. When building a [minicontainer for Logstash and Elasticsearch](https://github.com/SirIle/miniboxes/tree/master/minilogbox) I also found out that the way the Dockerfile is written [heavily affects the resulting image size]({% post_url 2015-01-01-dockerfile-and-container-image-size %}).

## Conclusion

Based on my experience with Docker so far I'll keep on using the minicontainer paradigm for now as it seems to bring out the best in Docker. As advanced networking or service registration and discovery still aren't covered by Docker itself, running Consul in every container seems feasible. For advanced networking I'll keep on using Weave if I need to distribute the containers across multiple hosts.

The age of the fat containers has already gone (if it ever was) and microcontainers with the required volume containers are too difficult with the current orchestrators, so minicontainers it is then.
