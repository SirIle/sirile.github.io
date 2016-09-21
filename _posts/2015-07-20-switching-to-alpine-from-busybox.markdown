---
layout: post
title: "Switching to Alpine from BusyBox"
date: "2015-07-20 10:37"
---

{::options parse_block_html="true" /}
<div class="toc">
Contents

<!--lint disable -->
* toc
{:toc}
</div>

## General

I have used Progrium's excellent [BusyBox](https://github.com/progrium/busybox)
image as the base of my minicontainers for quite some time. Recently it seems
that the upstream changes cause all the packages installed with the opkg package
manager it uses to [fail](https://github.com/progrium/busybox/issues/25) as
OpenWRT switched to musl.

Easiest way forward is to switch to using Alpine as it also has a minimal size.

## Package manager

Alpine uses uses
[apk](http://wiki.alpinelinux.org/wiki/Alpine_Linux_package_management) as the
package management tool instead of opkg that progrium's BusyBox image used. It
seems that apk package are maintained quite well and all the packages I have
needed so far are there.

## Running Java on Alpine

When I earlier tried out Alpine I ran into problems when trying to run Oracle
Java 8 on it. Things have progressed since then and people like [Vlad
Frolov](https://github.com/frol) have managed to get the Oracle JDK 8 to run.
The image is called
[frol/docker-alpine-oraclejdk8](https://github.com/frol/docker-alpine-oraclejdk8)
and I'm switching to using it as the base for the minicontainers as it seems to
be updated regularly and there is no reason for me to duplicate the effort done
by Vlad.

## Next steps

I'll immediately convert all my images to use Alpine as the base as at the
moment most of them won't build as they try to install wget with https support
and as that package doesn't run, the build fails.
