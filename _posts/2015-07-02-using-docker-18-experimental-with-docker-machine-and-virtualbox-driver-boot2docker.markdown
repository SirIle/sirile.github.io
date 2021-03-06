---
layout: post
title: Using Docker 1.10 experimental with docker-machine and VirtualBox driver (boot2docker)
commentIssueId: 5
---

{::options parse_block_html="true" /}
<div class="toc">
Contents

<!--lint disable -->
* toc
{:toc}
</div>

**Update on 20.11.2015** New experimental version is 1.10. The image can be
found from [here](http://sirile.github.io/files/boot2docker-1.10.iso). Major
experimental thing is the user namespaces.

**Update on 27.8.2015** I ran the build for newest experimental and the image
for boot2docker for experimental 1.9 can be found from
[here](http://sirile.github.io/files/boot2docker-1.9.iso). I updated the
examples to use 1.9. The post can be found from both URLs.

## TL;DR

To use boot2docker.iso which includes the 1.10 experimental Docker (at the
moment _Docker version 1.10.0-dev, build 0cdc96c, experimental_), create the
machine with

~~~bash
docker-machine create -d virtualbox --virtualbox-boot2docker-url=http://sirile.github.io/files/boot2docker-1.10.iso dev
~~~

I'll explain the creation process of the image if you want to do it yourself.

## General

As I was following an [article about Docker experimental
features](https://github.com/docker/docker/blob/master/experimental/compose_swarm_networking.md)
I started to follow the instructions. Getting the Docker client 1.8 experimental
installed on Mac was easy enough following the prerequisites chapter. Then I
tried to connect to an existing local Docker Machine instance and ran into
problems as it complained that the API versions on the client and server didn't
match. I tried upgrading the Docker Machine instance (boot2docker), but version
1.7 (and API version 1.19) was the newest it would do and Docker 1.8 client
requires API version 1.20.

It took me quite a while to figure out that the example uses DigitalOcean and
gives a parameter called
`--engine-install-url="https://experimental.docker.com"` as a parameter. That
script takes care of installing the experimental version on the server, but it
can't be used on boot2docker. Boot2docker uses a special flavor of Linux where
most of it (like /usr/local/bin) is immutable after creation. So even if you
manually ssh to the server and modify a file, after the next reboot the changes
are gone.

## Options

As a first option (after hacking the running boot2docker instance and figuring
out that isn't the way forward) I tried to use Docker Machine with the _none_
driver. I created a host using Vagrant and installed the experimental Docker 1.8
there. Getting the server to play along nicely with Docker Machine was quite
tedious as LTS based security is basically a requirement and configuring it was
more work than I was willing to do.

The next option was to explore the boot2docker ISO creation and see if I could
replace the docker-executable with the experimental version. Thanks to the smart
[boot2docker build
mechanism](https://github.com/boot2docker/boot2docker/blob/master/doc/BUILD.md)
this was really easy.

## Extending the boot2docker image

As I only needed to replace the existing docker executable with a new version,
the new Dockerfile is just

~~~bash
FROM boot2docker/boot2docker
RUN curl -L https://experimental.docker.com/builds/Linux/x86_64/docker-latest > $ROOTFS/usr/local/bin/docker && chmod +x $ROOTFS/usr/local/bin/docker
RUN /make_iso.sh
CMD ["cat", "boot2docker.iso"]
~~~

It can also be cloned from [GitHub](https://github.com/SirIle/boot2docker-experimental).

After building it on a suitable Docker environment (2 GB memory seems to be
required) with

~~~bash
docker build -t my-boot2docker-img .
~~~

the new image can be exported with

~~~bash
docker run --rm my-boot2docker-img > boot2docker-1.10.iso
~~~

The resulting file is the one [hosted here](http://sirile.github.io/files/boot2docker-1.10.iso).

## Next steps

The same script can be used to update the boot2docker.iso to the latest
experimental Docker executable release, it's not tied to 1.10.
